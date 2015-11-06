# Know Your Sidekiq Testing Rights

Sidekiq is a background job processor for Ruby that is relied on by thousands of
Rails apps. It is regularly used to process slow or resource intensive work in
the background; sending email, for example. The results and operations of
background jobs are often critical to the business and its users. It is
essential that new signups get a confirmation email, accounts are charged for
usage, and incoming data is indexed for searching. As with any critical part of
your application infrastructure, *proper testing is paramount*. By no
coincidence, Sidekiq has fantastic support for various means of testing.

The performance and reliability of your application's test suite hinges on the
granularity with which you test each component. Features that exercise the
entire stack are tested at a higher level than the individual methods that
comprise a particular class. That same granular approach applies directly to
testing background jobs and the Sidekiq workers that process them.

You need to know which testing modes Sidekiq is packaged with and the best
practices for testing both jobs and workers. This post will walk you through
evaluating each testing paradigm and show examples of how to work effectively
with each.

Before proceeding it will serve us well to define a couple of commonly used
Sidekiq terms: "job" and "worker". A "job" is an operation to be processed in
the background. Sidekiq manages a queue of jobs within Redis as simple JSON data
structures. A "worker" is a Ruby class, with a little Sidekiq sugar mixed in,
that is responsible for executing a job. When Sidekiq is ready to process a job,
a corresponding worker is initialized and passed the job's arguments.

## Know Your Modes

There are three distinct testing "modes" that Sidekiq ships with. Here's the
breakdown, straight from the [GitHub wiki][stw]:

* Fake — No job processing is performed, jobs are only enqueued
* Inline — Jobs are enqueued and then processed synchronously
* Disabled — No special handling for testing, jobs are enqueued in Redis and
  processed normally

Before diving into the particulars for each paradigm, let's review how to
properly configure Sidekiq within your test suite. Changing the testing mode is
done on the `Sidekiq::Testing` module, which is a constant and therefore global.
This can cause unexpected behavior for randomized or parallelized tests, so it
is advisable to change the mode within the context of a block when possible.

```ruby
require 'minitest/autorun'
require 'sidekiq/testing'

class IndexWorkerTest < Minitest::Test
  def setup
    Sidekiq::Testing.fake!
  end

  def test_that_mode_can_change_within_a_block
    Sidekiq::Testing.inline do
      IndexWorker.perform_async(model.id)

      assert model.reload.indexed?
    end
  end
end
```

Testing your Sidekiq workers with `RSpec` is smooth and wonderful, and there's
even a [gem for it][rs]. However, this post is about Sidekiq and `Minitest` is
the default, so we'll stick with it for the examples to keep the cognitive
overhead to a minimum.

## Unit Testing Workers — Start at the Bottom

Unit testing workers is simple and fast. Always start at this most granular
level for testing branching logic, edge cases, or complex logic within workers.

A worker's initializer doesn't take any arguments, so the object isn't
instantiated with any state. This nudges your worker's methods toward a
functional style of passing objects and arguments around. Wonderfully, testing
these types of pure input/output methods is simple and straight forward.

Since jobs, along with their arguments, are persisted to Redis before they are
dequeued and executed, a [best practice][sbp] is to pass simple identifiers to
the `#perform_async` method call instead of heavier instances of
`ActiveModel::Record`. Instead, are often fetched from the database within a
worker's `#perform` method. This can make unit testing workers more expensive
and less unit-like, so it is best to have additional methods defined so
`#perform` can delegate down and you can unit-test with test doubles or
non-persisted models.

Here is an example of a worker responsible for indexing new events so they are
searchable by end users. The `#perform` method calls `.find` to load the model
and then hands it off to `#index_event` where the real work of indexing takes
place.

```ruby
class IndexWorker
  include Sidekiq::Worker

  def perform(event_id)
    event = Event.find(event_id)

    index_event(event)
  end

  def index_event(event)
    event.index!
  end
end
```

In our test case, we can avoid persisting any events to the database and instead
pass them directly to the `index_event` method.

```ruby
class IndexWorkerTest < Minitest::Test
  def test_indexing_event
    worker = IndexWorker.new
    event  = Event.new

    worker.index_event(event)

    assert event.indexed?
  end
end
```

With complex workers that coordinate multiple tasks or rely on helper methods,
this style of isolated unit testing really pays off. Tests can be faster and
more focused.

## Testing Worker Queuing — When Testing the Boundaries

Sidekiq's fake testing mode operates similarly to the `ActionMailer` testing
API, in which jobs are queued up in a `.deliveries` array rather than being
executed immediately. Jobs within the queue can be queried, inspected, and
optionally "drained" to process enqueued jobs. This mode is activated simply
with the `fake!` directive:

```ruby
Sidekiq::Testing.fake!
```

Testing with fake mode is the first level of integration testing with other
objects, one step beyond unit testing workers. Testing this way promotes
decoupled and faster tests, as the worker doesn't have to perform any actual
work.

It isn't appropriate for full integration testing or situations where you want
the process jobs during a test. For example, indexing data in a background job
when you want to make assertions on finding the data during the test. The inline
testing mode, discussed next, is best for these cases.

Worker queues are global and will therefore persist between tests. Unchecked,
your tests will bleed state and your test suite will become order dependent. To
combat this, be sure to clear jobs between tests:

```ruby
def teardown
  Sidekiq::Worker.clear_all
end
```

Our event indexing application allows users to post new events, which need to be
indexed immediately. The collaboration of those two objects is wrapped up in a
form object that can be tested separately.

```ruby
class EventForm
  attr_reader :worker

  def initialize(worker: IndexWorker)
    @worker = worker
  end

  def create(params)
    Event.new(params).tap do |event|
      worker.perform_async(event.id) if event.valid?
    end
  end
end
```

The form object triggers the event indexing worker, but the test isn't concerned
with what work is being done in the worker—it only needs to verify that the job
was enqueued.

```ruby
class EventFormTest < Minitest::Test
  def test_enqueuing_index
    form  = EventForm.new(worker: IndexWorker)
    event = form.create(id: 123, name: 'New Event')

    assert_equal 1, IndexWorker.jobs.length
    assert_equal [123], IndexWorker.jobs.first['args']
  end
end
```

The job queue is simply an array filled with very simple hash structures
representing the job:

```ruby
{ "class"      => "IndexWorker",
  "args"       => [1],
  "retry"      => true,
  "queue"      => "default",
  "jid"        => "c7b41a718032908180a1de9c",
  "created_at" => 1440171298.594218 }
```

Making assertions against a job is extremely straight forward and doesn't
require a domain specific testing API. This style of testing can also be
accomplished with stubs, but that's more invasive and less flexible.

## Testing Inline Processing — When You Need Results

Inline testing mode performs enqueued jobs synchronously within the same
process, rather than asynchronously in a separate, dedicated process. This
closely mirrors production behavior, but without the difficulty of multiple
processes and race conditions.

```ruby
Sidekiq::Testing.inline!
```

Inline mode bypasses Redis integration entirely. Jobs are pushed into queues and
then immediately popped off and executed.

Inline processing is ideal for feature tests involving separate communicating
processes, for example full stack tests that use [capybara][cap]. By avoiding
asynchronous job processing, you gain more predictable test runs, ones where you
aren't plagued by periodic race conditions.

Now we want to add a new integration test to our event management app. The new
test simulates the happy path of a user posting a new event and then trying to
find that event through search:

```ruby
class SubmitAndSearchTest < ActionDispatch::IntegrationTest

  # Note that inline mode must be configured during test setup, running inside
  # of a testing block won't propagate between client (browser) and server
  # (rails) processes.

  def setup
    Sidekiq::Testing.inline!
  end

  def test_search_after_event_submission
    sign_in!

    visit '/events'

    post_an_event(name: 'Meetup')
    search_for_event('Meetup')

    assert_equal '/search', page.current_path
    assert_contains 'Meetup', page.body
  end
end
```

The test not only requires an event to be saved to the database, but the event
must also be indexed to show up in search results. Using the `inline` mode for
testing means that the indexing job is processed immediately and the results
will include the new event. If the job were processed asynchronously, there is a
good chance that the controller would respond before anything were indexed, a
textbook race condition causing flickering tests.

## Testing Disabled — When You Must Know Everything Works

Disabling testing altogether reverts Sidekiq back to asynchronous processing.
This is the default mode that Sidekiq runs in. It relies on separate clients and
servers, and a real Redis instance for queuing.

Disabled mode isn't meant for running tests; it's called "disabled" for a
reason! It is, however, an excellent way to verify that Sidekiq has the correct
configuration for Redis.

```ruby
class SidekiqConfigurationTest < Minitest::Test
  def setup
    configure = -> (config) do
      config.redis = { url: ENV.fetch('REDIS_URL') }
    end

    Sidekiq.configure_client(&configure)
    Sidekiq.configure_server(&configure)
  end

  def test_server_enqueing_works
    Sidekiq::Testing.disable! do
      event = Event.create(name: 'Reception')

      IndexWorker.perform_async(event.id)
    end
  end
end
```

## Exercise Your Rights

During everyday application testing, you always test a model at the unit level.
You then test how a model coordinates with other models at the boundary level.
Finally, you test how it integrates with the database and other services at the
functional level. Treat Sidekiq the exact same way! Follow these simple
guidelines and you'll be off to a great start:

* Use fast, finely grained, unit tests to work through edge cases and refine
  worker behavior.
* Test the boundaries of your jobs when you're focusing on collaboration with
  other objects.
* Test workers inline when you need to test the output of the jobs in the
  context of the rest of the system.

[stw]: https://github.com/mperham/sidekiq/wiki/Testing
[sbp]: https://github.com/mperham/sidekiq/wiki/Best-Practices
[rs]:  https://github.com/philostler/rspec-sidekiq
[cap]: https://github.com/jnicklas/capybara
