# Service Communication With Pub/Sub

The HTTP protocol was designed for synchronous communication between two
entities. For instance, a browser requesting a stylesheet, or a server charging
with a payment processor. Those are synchronous operations where nothing can
proceed without an immediate response.

Often communication can be asynchronous, like when queueing up work to be
performed in the background. It is possible to use HTTP for these lengthy
requests via [long polling][lp] or [server push][sp], but it belies the
strengths of HTTP. Long held direct HTTP requests don't scale beyond single
server-to-server communication, and they require greater resources as service
scales.

There doesn't always need to be direct communication between two services.
Message queues and distributed logging are two ways to accomplish fully
asynchronous, many-to-many communication, but there is a more fundamental pattern
available. The [Publish/Subscribe][ps] pattern, also known as Pub/Sub, is the
most basic flavor of event based messaging. It allows messages to be broadcast
out to other services which may or may not be listening. Those services are free
to pick up communication events they know how to respond to, and may respond with
asynchronous communications of their own.

### Getting To Know Pub/Sub

> [Services] should be able to register to receive only the events they need, and
> never be sent the events they don't need. We don't want to spam our [services]!
>
> The Pragmatic Programmer—Dave Thomas & Andy Hunt

A Pub/Sub provider brokers messages between publishers and subscribers.
Published messages are characterized into channels, which can be as broad or
narrow as necessary. Subscribers register to listen to one or more channels,
and then receive published messages only for those channels. Subscribers have no
knowledge of publishers, and publishers aren't aware of subscribers. This
decoupling allows a dynamic network topology that can scale beyond processes and
applications, well into the realm of multiple services.

Pub/Sub is most fitting when there are no limits on the number of subscribers.
In this situation, an unknown number of of subscribers may respond to any
messages.  This makes it unsuitable for queue-like operations, where an event
must be popped off and handled exactly once. However, it is perfectly suited to
synchronization operations, where a central publisher wishes to replicate data
in parallel to multiple subscribers.

Pub/Sub as a means of distributed communication is simple to understand and
simple to implement. It just so happens that Redis has an outstanding, albeit
simple, Pub/Sub implementation. Chances are, Redis is already in your stack, so
there isn't a barrier to using it. No additional services are required and the
overhead is low.

### Warming Up With the Redis CLI

Let's make all this abstract talk of Pub/Sub more concrete. We'll replicate
simple inter-process communication over Pub/Sub using the `redis-cli`. Assuming
you have Redis installed, open two separate terminal sessions. In the first
session use the [SUBSCRIBE][rdsu] command to bind to the `foo` channel:

```bash
$ redis-cli SUBSCRIBE foo
Reading messages... (press Ctrl-C to quit)
1) "subscribe"
2) "foo"
3) (integer) 1
```

The subscribe command is blocking, so the process will wait for any published
events. In the other session use the [PUBLISH][rdpu] to publish an event on the
`foo` channel:

```bash
$ redis-cli PUBLISH foo "bar"
(integer) 1
```

Back in the original process you'll see the message was passed through:

```
1) "message"
2) "foo"
3) "bar"
```

The message comes through with three arguments: command type, channel name, and
the payload. Any processes that subscribe to the same channel will receive
identical messages. This simple message passing scales beyond processes and into
disparate servers, enabling basic distributed messaging.

Now we'll move publishers and subscribers into Ruby to synchronize data between
a central source and multiple subscribers.

### Publishers and Subscribers in Ruby

Simulating publish and subscribe behavior from within the same Ruby process is
tricky business, requiring threads and passing behavior. Rather than go down
that rabbit hole we'll use two separate processes for demonstration. First, a
wrapper around channel subscription:

```ruby
# subscriber.rb
require 'redis'

class Subscriber
  def subscribe(channel)
    Redis.current.subscribe(channel) do |on|
      on.subscribe do |channel, _|
        puts "subscribed to #{channel}"
      end

      on.message do |channel, message|
        puts "#{channel} received #{message}"
      end
    end
  end
end
```

The `Subscriber` class in this example is only a thin wrapper around
`Redis#subscribe`, which binds to the specified channel and then writes out
published messages. The wrapper around `Redis#publish` is even simpler:

```ruby
# publisher.rb
require 'redis'

class Publisher
  def publish(channel, message)
    Redis.current.publish(channel, message)
  end
end
```

Just as we used one terminal sessions with `redis-cli` before, now we'll start a
subscribe process in one session and publish to it from another.

```bash
$ ruby -rSubscriber -e "Subscriber.new.subscribe('custom/channel')"
> subscribed to custom/channel
```

The process will block while it waits for messages to arrive. Send a message to
the "custom/channel":

```bash
$ ruby -rPublisher -e "Publisher.new.publish('custom/channel', 'hello there')"
```

It is written out in the first session:

```
> custom/channel received hello there
```

And that's all there is to Pub/Sub between processes in Ruby. To make use of
Pub/Sub for data synchronization we must layer predictable channels and
structured messages on top of simple messaging.

### Structuring Channels and Messages

The previous toy examples leave out all of the details necessary to actually
fold distributed messaging into an application. What you need, at a minimum, are
conventions for setting predictable channel names. Channel names that compose to
become more specific are ideal. For example, in an application where readers can
leave comments on a post you may have a channel named `posts/1` and another
channel for comments named `posts/1/comments`. Updates to the post itself are
broadcast to the `posts/1` channel, while new comments are sent to the
`posts/1/comments` channel.

Message events also need to be structured in a predictable format. Any transport
that would be used for HTTP is fitting for messages, but JSON is the obvious
choice. Continuing the comment synchronization example above, here is an example
of a "comment updated" payload:

```json
{
  action: "updated",
  channel: "posts/1/comments",
  meta: {},
  model: {
    "id": 100,
    "body": "This is my new comment"
  }
)
```

Just like any API, or the canonical [JSON-API][js-api], payload consistency is
key. When designing communication you must be aware that, unlike HTTP, there
aren't any headers or versions, the only means of filtering messages is by
channel name.

### Subscribing To Events Between Languages

Pub/Sub is a bare bones protocol that is entirely language agnostic. Every major
language has a usable Redis library, so the possibilities are wide open.
As with any multi-service architecture, there are opportunities to
lean on other languages and frameworks where your primary platform may fall
short.

For example, Ruby is notoriously bad at maintaining multiple concurrent
connections. That's a perfect opportunity to write a small service in a platform
like Node.js, which can hold long standing connections with browsers. The
primary application then broadcasts events out to each service, which can then
relay them to the browser.

Even a framework like [Phoenix][phx], built in the highly asynchronous
[Elixir][ex] language, relies on Redis' Pub/Sub to expand websocket support
beyond a single web host.

### The Final Countdown

Pub/Sub makes one-to-many and many-to-many communication between services
possible where HTTP falls flat. Allowing applications to broadcast events to
arbitrary services is a path to highly scalable, fault tolerant, and loosely
coupled services.

It must be noted that Redis' implementation has notable caveats. Unlike some
other implementations, such as [ZeroMQ][zmq], Redis has no guarantees on message
delivery, no acknowledgements, and no persistence in the event of service or
network failure. It's triumph is simplicity and ubiquity, but you'll want to
look further into the world of Pub/Sub if persistence and reliability are
paramount.

[ps]: https://en.wikipedia.org/wiki/Publish%E2%80%93subscribe_pattern
[lp]: https://en.wikipedia.org/wiki/Push_technology#Long_polling
[sp]: https://en.wikipedia.org/wiki/Push_technology#HTTP_server_push
[rdsu]: http://redis.io/commands/subscribe
[rdpu]: http://redis.io/commands/publish
[js-api]: js-api
[zmq]: zmq
[phx]: phx
[ex]: ex
