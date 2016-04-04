# Surrogate WebSockets Through Elixir

ActionCable is coming to Rails 5 and brings with it the promise of using WebSockets directly in Rails.
Ruby has a notoriously bad concurrency story, and that certainly extends into the realm of WebSockets and pubsub.
ActionCable may be suitable for a small number of authenticated sessions, but scaling persistent connections to thousands or tens-of-thousands won't be easy.
That's alright, the beauty of the internet is that you can mix and match technologies as you see fit!

There are plenty of other languages that your application can lean on to achieve real-time over WebSockets.
In the recent past you may have reached to Node.js for this kind of work, but there is a better option.
Pubsub is by its nature a distributed system, and it's no secret that Erlang is a superstar in the distributed systems realm.
Rather than reach for Erlang directly we'll leverage Elixir's productive tooling to build out a WebSocket micro-service.

By using a surrogate micro-service, your Rails application remains the authority on business logic.
It still handles all database interactions, sending email, and plugs away at the other work Rails excels at—it just doesn't hold thousands of connections.

## Gluing PubSub Together

You may recall from the previous article on [service communication with pubsub][pbr] that it is very easy to publish messages from Ruby.
As a refresher, here is the simple `Publisher` class:

```ruby
require 'redis'

class Publisher
  def publish(channel, message)
    Redis.current.publish(channel, message)
  end
end
```

The Rails application will publish score updates whenever data changes:

```ruby
# from a controller or a background worker

publisher.publish('teams-123-scores', score.to_json)
```

Publishing is the easy part though!
So long as you have a consistent naming scheme for channels and serialize your messages it is very straightforward.
What about the Elixir side?

## Getting Started With Compatibility

In the Elixir world [Phoenix][phx] is an obvious choice for hosting surrogate websocket connections.
There are two pubsub implementations broadly available for Phoenix, `Redis` and `PG2` (using Erlang's distributed named process groups).
As it turns out, both modules rely on Erlang encoded terms, so neither are conducive to communication outside the Erlang ecosystem.
Developing an alternate pubsub module for Phoenix is beyond the scope of this article.
Instead we'll build a simple system that allows Ruby and Elixir to communicate over pubsub with a minimal API.

## Setting Up Elixir and Redis PubSub

From here on I'll assume you are familiar with Elixir, the `mix` build tool, and testing with [ExUnit][exu].

The first step is creating a surrogate project:

```bash
mix new surrogate && cd surrogate
```

Now we'll install the Redis client.
There are a few different Redis clients in the Erlang world, but we'll be using the native Elixir client [Redix][red].
It is easy to configure, easy to test, and wonderfully written.
Open the generated `mix.exs` and modify the `deps`:

```elixir
defp deps do
  [{:redix, "~> 0.3"}]
end
```

Now retrieve the dependencies:

```bash
mix deps.get
```

That's enough to get the project set up and explore pubsub with a small test.
The only functions our module needs are `subscribe/1`, `unsubscribe/1`, and `broadcast/2`, that is consistent with the `Phoenix.PubSub` API.
Our test gets started by testing `subscribe/1`:

```elixir
defmodule SurrogateTest do
  use ExUnit.Case

  alias Surrogate.PubSub

  test "subscribe/1 links to a topic" do
    {:ok, _pid} = PubSub.start_link

    :ok = PubSub.subscribe("topic.1")
    :ok = PubSub.subscribe("topic.2")

    {:ok, other_conn} = Redix.start_link
    {:ok, _} = Redix.command(other_conn, ~w(PUBLISH topic.1 hello))
    {:ok, _} = Redix.command(other_conn, ~w(PUBLISH topic.2 world))

    assert_receive "hello"
    assert_receive "world"
  end
```

The test first establishes a link to a new surrogate server and then uses the `subscribe/1` api to register the test process for published messages.
Next it starts another link directly through Redix and publishes messages on both of the subscribed topics.
Finally, it verifies that both of the simple `hello` and `world` messages were received by the test process.

Actually defining the `Surrogate.PubSub` module is a bit more involved.
Retaining a list of subscribed processes is essential, as every connection from the web is a separate process.
In order to retain state we must use an `Agent` or a `GenServer`.
As our module combines state and functions to manipulate that state we will use the `GenServer` behaviour.

```elixir
defmodule Surrogate.PubSub do
  use GenServer

  def start_link(opts \\ []) do
    GenServer.start_link(__MODULE__, opts, [name: __MODULE__])
  end

  ## Callbacks

  def init(opts) do
    {:ok, conn} = Redix.PubSub.start_link(opts)

    {:ok, %{conn: conn, subs: %{}}}
  end
end
```

The `GenServer` behavior is mixed in via `use`, and we then define the `start_link/1` convenience function.
As soon as the process is started it links itself to `Redix.PubSub` and retains a connection to Redis.
Now we're ready to add the `subscribe/1` api method.

```elixir
def subscribe(topic) do
  GenServer.call(__MODULE__, {:subscribe, topic})
end

## Callbacks

def handle_call({:subscribe, topic}, {pid, _}, %{conn: conn, subs: subs} = state) do
  :ok = Redix.PubSub.subscribe(conn, topic, self())

  tops = Map.get(subs, pid, MapSet.new)
  subs = Map.put(subs, pid, MapSet.put(tops, topic))

  {:reply, :ok, %{state | subs: subs}}
end
```

Subscribe issues a `handle_call/3`, part of the `GenServer` behaviour, to synchronously subscribe the caller process to a topic.
Simultaneously it adds the calling process (seen here as `{pid, _}`) to the server's state.
Every process is a key in the `subs` map, and its value is a set of all the topics the process is subscribed to.
This mapping is critical for receiving published messages and managing unsubscribes later.

Running the tests now doesn't quite work, because the `Redix.PubSub` module sends out-of-band messages back to the pubsub process and they aren't being handled.
To handle these out-of-band messages we have to use the `handle_info/2` callback:

```elixir
def handle_info({:redix_pubsub, :message, message, topic}, %{subs: subs} = state) do
  for {pid, topics} <- subs do
    if MapSet.member?(topics, topic), do: send pid, message
  end

  {:noreply, state}
end
def handle_info(_msg, state) do
  {:noreply, state}
end
```

There! Now the module can properly handle subscriptions and forwarding messages to the subscribed processes.

```bash
mix test test/surrogate_test.exs
Compiled lib/surrogate.ex
.

Finished in 0.1 seconds (0.06s on load, 0.04s on tests)
1 test, 0 failures

Randomized with seed 117850
```

## What Else Does PubSub Need?

The `subscribe/1` function is *almost* enough for one way communication between the Rails app and the Surrogate.
There isn't any need to support `broadcast/2` (which would send messages back upstream), leaving `unsubscribe/1`.

When the current process is unsubscribed it should no longer receive any forwarded messages.
The implementation of `unsubscribe/1` is simple, effectively `subscribe/1` in reverse.
First, add a new test case:

```elixir
test "unsubscribe/1 unlinks a topic" do
  {:ok, _pid} = PubSub.start_link

  :ok = PubSub.subscribe("topic.1")
  :ok = PubSub.unsubscribe("topic.1")

  {:ok, other_conn} = Redix.start_link
  {:ok, _} = Redix.command(other_conn, ~w(PUBLISH topic.1 hello))

  refute_receive "hello"
end
```

The test refutes that any message is broadcast after unsubscribing to "topic.1".
Of course this initially fails, now we make it pass:

```elixir
def unsubscribe(topic) do
  GenServer.call(__MODULE__, {:unsubscribe, topic})
end

## Callbacks

def handle_call({:unsubscribe, topic}, {pid, _}, %{subs: subs} = state) do
  tops = Map.get(subs, pid, MapSet.new)
  subs = Map.put(subs, pid, MapSet.delete(tops, topic))

  {:reply, :ok, %{state | subs: subs}}
end
```

Again we rely on the power of `call`, which includes the calling processes' `pid` in the second argument.
The handler finds the entry for the calling `pid` and removes the specified topic from its list.
When future messages are published that topic won't be found and the message won't be forwarded.
This approach is simple for illustration purposes, but it is quite naive and leaves behind dangling references to processes.
Production grade implementations, such as `Phoenix.PubSub`, run periodic GC to clean up after unsubscribes.

## Stitching Into WebSockets

Our connections are built on top of [Plug][plug], which comes with an adapter for the [Cowboy][cowb] server.
Every http connection that comes to cowboy is wrapped in separate process.
That includes websocket connections, which are initiated as http connections and then upgraded to websockets.
That's perfect for our pubsub setup!
Once a connection is upgraded the server simply waits for `topic` subscriptions and forwards them on to the `Surrogate` module.

Add both `plug` and `cowboy` to `deps` in `mix.exs`, ensure they are started as `applications`, and define `Surrogate` as the application to start.

```elixir
def application do
  [mod: {Surrogate, []},
   applications: [:cowboy, :plug, :logger]]
end

defp deps do
  [{:redix, "~> 0.3"},
   {:plug, "~> 1.0"},
   {:cowboy, "~> 1.0"}]
end
```

In order to treat `Surrogate` as an application it needs a base module that uses the Application behaviour.
Forgive me if what follows is a little like the "how to draw an owl" process.
There are a lot of details in application setup that are outside the scope of this article, and it is necessary to get a real server.

```elixir
defmodule Surrogate do
  use Application

  alias Plug.Adapters.Cowboy

  def start(_, _) do
    import Supervisor.Spec

    children = [
      Cowboy.child_spec(:http, Surrogate.Server, [], [dispatch: dispatch]),
      worker(Surrogate.PubSub, [])
    ]

    opts = [strategy: :one_for_one, name: Surrogate.Supervisor]

    Supervisor.start_link(children, opts)
  end

  defp dispatch do
    [{:_, [{"/ws", Surrogate.Socket, []},
           {:_, Cowboy.Handler, {Surrogate.Server, []}}]}]
  end
end
```

The Surrogate module will is defined as a proper Erlang app, which supervises the pubsub and cowboy workers.
The supervision ensures that any time a worker crashes a new one is started up in its place.
Notice that the number of modules has expanded, now there are also `Surrogate.Server` and `Surrogate.Socket` modules.
These will handle our incoming `HTTP` and `WebSocket` connections, respectively.

First, the `Surrogate.Server` module is defined as a simple plug that always returns an "OK" text response.

```elixir
defmodule Surrogate.Server do
  import Plug.Conn

  def init(_opts) do
  end

  def call(conn, _opts) do
    conn
    |> put_resp_content_type("application/text")
    |> send_resp(200, "OK")
  end
end
```

We can verify that the server is working now by starting the application up with `iex -S mix` and visiting `localhost:4000`.
If everything went according to plan you'll see a friendly "OK" in your browser.
Now the final module needed to relay messages, `Surrogate.Socket`:

```elixir
defmodule Surrogate.Socket do
  @behaviour :cowboy_websocket_handler

  def init(_, _req, _opts) do
    {:upgrade, :protocol, :cowboy_websocket}
  end

  ## Callbacks

  def websocket_init(_type, req, _opts) do
    {:ok, req, %{}, 60_000}
  end
end
```

The module uses the `cowboy_websocket_handler` behaviour, which requires callback handlers just like a `GenServer`.
Upgrading to a websocket is handled by `websocket_init/3`, which is called whenever a new connection comes in to `localhost:4000/ws`.
Once connected we can begin sending and receiving messages through `websocket_handle/3`.

```elixir
alias Surrogate.PubSub

def websocket_handle({:text, "ping"}, req, state) do
  {:reply, {:text, "pong"}, req, state}
end
def websocket_handle({:text, "subscribe|" <> topic}, req, state) do
  PubSub.subscribe(topic)

  {:ok, req, state}
end
def websocket_handle({:text, "unsubscribe|" <> topic}, req, state) do
  PubSub.unsubscribe(topic)

  {:ok, req, state}
end
def websocket_handle({:text, _}, req, state) do
  {:ok, req, state}
end
```

Each handler will pattern match on the incoming message, allowing the server to selectively handle `subscribe` and `unsubscribe` messages accordingly.
At this point the server can handle incoming messages, but it is unable to relay subscribed messages being returned from pubsub.
As with all out-of-band messages to a server those are handled with an `_info/3` callback:

```elixir
def websocket_info(message, req, state) do
  {:reply, {:text, message}, req, state}
end
```

Here the `message` is the exact value being published to the channel, and it is passed along to the websocket connection directly.
Finally, we can test the round trip between Ruby, Elixir, and a web browser.
Restart the surrogate server interactively with `iex -S mix`, reload the page, and open up the JavaScript console.

```javascript
let ws = new WebSocket("ws://localhost:4000/ws")
ws.onmessage = (message) => console.log(message.data)
ws.send("subscribe|topic.1")
```

All messages published to `topic.` are fed to the console, whether they come from Ruby, a Redis CLI instance, or the Elixir shell.
For example, when a 'score' object is serialized and published from Ruby, this is exposed as `data` in the console:

```javascript
'{"id":12345,"value":"new score"}'
```

## Monolith This Isn't

The system we've built is more complex than a monolith, but it will scale simply and perform reliably.
It will easily handle thousands of topics and thousands of connections, even on the lightest weight hardware.
Line for line it may outweigh a simple naive Node.js implementation, but it is entirely fault tolerant right out of the box.
That's the benefit we get from building on top of Erlang.

This system is real-time *only*, there aren't any mechanisms to rewind or catch up to missed messages after temporary disconnections.
That is simply the nature of pub-sub.
Paired with a lack of channel authorization it is perfectly suited to pushing updates over broadly available public channels.
Think real time sports updates, leader boards, stock tickers, etc.

This is not the "beautiful monolith" approach, because that simply doesn't exist when you enter the realm of distributed systems.
It relies on an external database to broker communication, which is a service in itself.
When external dependencies are already at play, it is wise to leverage languages and frameworks that are perfectly suited to the task.

[phx]: http://phoenixframework.org/
[pbr]: codeship.com/pubsub-ruby
[red]: https://github.com/whatyouhide/redix
[exu]: https://github.com/elixir-lang/elixir/blob/v1.2.3/lib/ex_unit/lib/ex_unit.ex#L1
[plug]: https://github.com/elixir-lang/plug
[cowb]: https://github.com/ninenines/cowboy
