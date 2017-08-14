# Folding Window Functions Into Rails

You've heard of window functions in PostgreSQL, but you aren't quite sure what they are or how to use them. On the surface they seem esoteric and their use-cases are ambiguous. Something concrete would really help cement when window functions are the right tool for the job. We have our work cut out for us!  Through this post we'll see:

1. How to recognize where a window function is helpful.
2. How to implement a scope to introduce window functions.
3. How to use tests to drive a switch from pure ruby to a window function.

## An Anecdotal Example

You've recently finished shipping a suite of features for an application that helps travelers book golf trips. Things are looking good, and a request comes in from your client:

> Our application started by being the go-to place to find golf trips, and our users love it. Some of the resorts that list trips with us also offer some non-golf events, such as tennis, badminton, and pickleball. When we begin listing other trips it would be great to highlight our user's favorite trips for each category. Can you do that for us?

Why, of course you can do that! The application lets potential traveler's flag trips they are interested in as favorites, providing a reliable metric that we can use to rank trips. With the simple addition of a `category` for each trip we can also filter or group trips together. This seems straight forward enough, what are we waiting for?

## Survey the Scene

After a refresher look through the schema you are ready to get to work. A look at the `trips` table reveals that it currently has these relevant fields: `name`, `category`, and `favorites`.

Where the application currently lists the top ranked trips we will instead list the top two for each category. Some tests will help verify that we're getting the expected results.

```ruby
class TripTest < ActiveSupport::TestCase
  setup do
    Trip::CATEGORIES.each do |category|
      5.times do |favorites|
        Trip.create!(
          name: "#{category}-#{favorites}",
          category: category,
          favorites: favorites
        )
      end
    end
  end

  test 'find top trips by category' do
    trips = Trip.popular_by_category(per: 2)

    assert_equal 8, trips.length
    assert_equal [2,2,2,2], trips.group_by(&:category).map(&:last).map(&:length)
    assert trips.all? { |trip| trip.favorites > 2 }
  end
end
```

The test seeds the test database with twenty trips across four categories. The `popular_by_category` method should only return eight trips in total, the most popular two from each category.

Approaching this in pure ruby is clear, readable and concise. All of the trips are loaded into memory, grouped by category, ranked according to the number of favorites, and then the requested amount is sliced off of the top. Do note that sorting is comprised of both favorites and name, which is necessary to force deterministic sorting in the event of trips that are equally popular.

```ruby
class Trip < ActiveRecord::Base
  CATEGORIES = %w[golf tennis badminton pickleball]

  def self.popular_by_category_orig(per: 2)
    all.group_by(&:category).flat_map do |category, subset|
      subset.sort_by do |trip|
        [-trip.favorites, trip.name]
      end.take(per)
    end
  end
end
```

As a wizened developer, you immediately recognize that loading all of the trips into memory simply to end up with only eight results is horribly inefficient. The pure Ruby code is concise and readable, but it isn't suitable for production usage.

## Move the Logic to PostgreSQL

Between various sub-selects, `GROUP BY` with aggregates and multiple queries, [there is more than one way to do it][tim] in SQL. You remember reading about one advanced feature of PostgreSQL that may be particularly adept at solving this category problem—[window functions][tw]. Succinctly put, directly from the documentation:

> A window function performs a calculation across a set of table rows that are somehow related to the current row.
> <cite>[Postgres Documentation][tw]</cite>

The important part of that phrase is the power of calculating across related rows. In our case the rows are related by category, and the calculation being performed is ordering them within those categories. In the realm of window functions this is handled with an [`OVER` clause][swf]. There are additional expressions for fine tuning the window , but we can achieve all we need with `PARTITION BY` and `ORDER BY` expressions. Dropping into `psql`, partition the data set by category:

```postgressql
SELECT category, favorites, row_number() OVER (PARTITION BY category) FROM trips;
```

```
  category  | favorites |  row_number
------------+-----------+-------------
 badminton  |         0 |          1
 badminton  |         1 |          2
 badminton  |         2 |          3
 badminton  |         3 |          4
 golf       |         0 |          1
 golf       |         1 |          2
```

The `row_number` is a window function that calculates number of the current row within its partition. The row number becomes crucial when the partitioned data is then ordered:

```postgressql
SELECT category, favorites, row_number() OVER (
  PARTITION BY category ORDER BY favorites DESC
) FROM trips;
```

```
  category  | favorites | row_number
------------+-----------+------------
 badminton  |         3 |          1
 badminton  |         2 |          2
 badminton  |         1 |          3
 badminton  |         0 |          4
 golf       |         3 |          1
 golf       |         2 |          2
```

All that remains is limiting the results to the top ranked rows, and that can be done in Ruby.

## Move it to Rails

There aren't any constructs for `OVER` built into ActiveRecord. You must drop down and compose raw SQL to compose your queries. So long as you avoid `find_by_sql`, and use relation refining methods like `from` or `select` there isn't any loss in composability.

Only the `Trip` class itself needs to be modified, The tests can stay exactly as it is. A primary benefit of performing calculations in the database is that a scope can be used to return a relation for composability.

```ruby
class Trip < ActiveRecord::Base
  CATEGORIES = %w[golf tennis badminton pickleball]

  scope :popular_by_category, -> (per: 2) do
    from_ranked.where('trips.row_number <= ?', per)
  end

  private

  def self.from_ranked
    from <<-SQL.strip_heredoc
      (SELECT *, row_number() OVER (
        PARTITION BY category
        ORDER BY favorites DESC, name ASC
      ) FROM trips) AS trips
    SQL
  end
end
```

The relation is defined as a `scope` using a custom `from` clause to build up partitioned trips. The `where` clauses filters out any row numbers below the desired threshold, and only the favorites for each category are returned. The test still passes, and inspecting the results yields:

```
  category  | favorites | row_number
------------+-----------+------------
 badminton  |         3 |          1
 badminton  |         2 |          2
 golf       |         3 |          1
 golf       |         2 |          2
 pickleball |         3 |          1
 pickleball |         2 |          2
 tennis     |         3 |          1
 tennis     |         2 |          2
```

Those are precisely the results we're looking for!

## How Much Better Is It?

Composability, offloading work to the database and minimizing memory usage in Rails are all wonderful notions. But looking at numbers, how much faster is execution after the move to window functions? Clearly a test with twenty rows isn't going to show much, let's look at a more realistic production-like database size of 5,000 trips.

```ruby
Benchmark.ms { Trip.popular_by_category_original } #=> 149.14
Benchmark.ms { Trip.popular_by_category }          #=> 0.277
```

Not only is the window function version *539.3x* faster, it doesn't trigger a garbage collection cycle after each run. Your client gets exactly what they want, quickly, and with ample room for their site to grow without bottlenecks.

## Understanding is Most Important

> There was SQL before window functions and SQL after window functions: that's how powerful this tool is. Being that [big of a game changer] unfortunately means that it can be quite hard to grasp the feature.
>
> <cite>[Dimitri Fontaine][wf]</cite>

Window functions are an advanced feature of PostgreSQL, heck, it's even listed under "advanced features" in the documentation. Having a relatively simple use-case for them helps a lot when you are trying to wrap your head around the concept. For some tasks window functions can radically simplify queries, for other tasks they are completely irreplaceable.

[tw]: http://www.postgresql.org/docs/9.4/static/tutorial-window.html
[swf]: http://www.postgresql.org/docs/9.4/interactive/sql-expressions.html#SYNTAX-WINDOW-FUNCTIONS
[tim]: https://en.wikipedia.org/wiki/There's_more_than_one_way_to_do_it
[wf]: http://tapoueh.org/blog/2013/08/20-Window-Functions
