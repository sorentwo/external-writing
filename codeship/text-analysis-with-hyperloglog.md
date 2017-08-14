# Text Analysis With HyperLogLog

Counting distinct values is trivial with small datasets, but becomes dramatically harder for streams with millions of distinct points.
Absolute counting accuracy can be attained through a set, but with the undesirable trade off of linear memory growth.
What you really want when dealing with enormous datasets is something with predictable storage and performance characteristics.
That is precisely what HyperLogLog (HLL) is—an algorithm optimized for counting the distinct values within a stream of data.
Enormous streams of data.
It is an algorithm that has become [invaluable for real time analytics][res].
The canonical use case is tracking unique visitors to a website, but the use cases extend far beyond that.

Before HyperLogLog there was the LogLog algorithm.
Like HLL, it traded absolute accuracy for performance, speed, and a minor error rate.
The author of LogLog, Philippe Flajolet, knew the error rate was too high and endeavoured to create a successor that had fewer flaws.
The result was HyperLogLog, which boasts a lower error rate (0.81%) and even better performance.
Luckily, to make use of the HLL goodness you don't need to implement the data structure yourself.
Redis has a [highly tuned implementation][anti] of HLL available as a native data structure.

## Using HLL Within Redis

> The HyperLogLog data structure can be used in order to count unique elements
> in a set using just a small constant amount of memory, specifically 12k bytes
> for every structure (plus a few bytes for the key itself).

Redis provides a couple of simple but powerful commands for working with HyperLogLog structures.
The commands are all prefixed with `PF`, in honor of Philippe Flajolet, and are simple to experiment with.
The Redis documentation is excellent and thoroughly describes the use of each command in more detail than we need.
For now, we'll skim the basic usage of the `PFADD`, `PFCOUNT`, and `PFMERGE` commands.

Start a new `redis-cli` session to experiment with and follow along:

```
127.0.0.1:6379> PFADD words this is a sentence about stuff
(integer) 1
127.0.0.1:6379> PFADD words this is another sentence about stuff
(integer) 1
127.0.0.1:6379> PFCOUNT words
(integer) 7
127.0.0.1:6379> PFADD alt-words stuff is a great word
(integer) 1
127.0.0.1:6379> PFCOUNT words alt-words
(integer) 9
```

What we've done here is use `PFADD` to create two structures, one with the key `words` and another with the key `alt-words`.
After creating the keys we used `PFCOUNT` to check the cardinality at each key.
After the creation of `alt-words` we call `PFCOUNT` with multiple keys, which merges the structures in memory and gives us the combined cardinality.
It is that simple.
With such a tiny data set we can easily see that the counts are correct.
It gets far more interesting when estimating larger uncountable datasets.

## Setting Up an Experiment

To make things interesting let's set up an experiment that leverages HLL outside the realm where it is normally applied.
Cardinality as a measurement only has a single dimension.
It becomes more informative when given a second dimension, such as change over time.
We can use HLL to measure the changes in a stream of data, for example, the vocabulary of a writer over their career.
This approach could be used on a twitter stream, a blog feed, or a series of books.
Given that Redis uses linear counting for structures with less than 40k distinct values, we'll want to use the most expansive sample set available.
The complete works of Charles Dickens are [freely available][gut] in plain text format.
That should make for an expansive sample!
Let's see if Dickens was sporting a 40k word vocabulary.

My hypothesis is that there will be a significant jump in vocabulary between each book—in addition to the small changes in subject and characters.

## Adding and Counting the Data

The interface for adding and counting values is extremely simple.
Here we'll define a simple Ruby script that will download a sample of books as raw text.
Each book is then processed line by line, lightly normalized, and then pushed into Redis:

```ruby
require 'open-uri'
require 'redis'

URLS = %w[
  http://www.gutenberg.org/ebooks/98.txt.utf-8
  http://www.gutenberg.org/ebooks/1400.txt.utf-8
  http://www.gutenberg.org/ebooks/730.txt.utf-8
  http://www.gutenberg.org/ebooks/766.txt.utf-8
  http://www.gutenberg.org/ebooks/19337.txt.utf-8
  http://www.gutenberg.org/ebooks/700.txt.utf-8
]

BOOKS = URLS.map(&File.method(:basename))
REDIS = Redis.current

URLS.each do |url|
  text = open(url)
  name = File.basename(url)

  text.each_line do |line|
    REDIS.pfadd(name, line.split(/\s+/).map(&:downcase))
  end
end
```

With all of the words stored we can do some light exploration using `PFCOUNT`:

```ruby
BOOKS.each do |name|
  puts "#{name}: #{REDIS.pfcount(name)}"
end

puts "All: #{REDIS.pfcount(*BOOKS)}"
```

That outputs the word count for each book, followed by the merged value for all of the works:

```
98.txt.utf-8:    18,583
1400.txt.utf-8:  21,384
730.txt.utf-8:   20,532
766.txt.utf-8:   31,601
19337.txt.utf-8: 7,405
700.txt.utf-8:   24,045
All:             65,454
```

Astoundingly the cumulative distinct word count is over *65k* words!
Also astounding, but less apparent from the printed results, is that the values are reported instantaneously.

## Powerful Analysis, Even Without Big Data

This is an intentionaly simple example of how to use HLL for a trivial problem.
However, once you grasp the fundamentals of the data structure it opens up new avenues of analysis.

Instinctually, when given a problem dealing with unique values most developers would immediately reach for a set.
Set's are perfectly suited to storing unique values, but that storage comes at a cost.
In sutations where the cardinality is important such as unique visits, distinct search terms, or vocabulary; precise values are immaterial.
Keep HyperLogLog in mind, a datastructure uniquely suited to the task of counting distinct values.


[anti]: http://antirez.com/news/75
[res]: http://research.neustar.biz/2012/10/25/sketch-of-the-day-hyperloglog-cornerstone-of-a-big-data-infrastructure/
[gut]: http://www.gutenberg.org/ebooks/author/37
