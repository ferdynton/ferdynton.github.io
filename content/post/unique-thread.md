---
title: "Unique Thread"
date: 2018-05-22T18:37:46-05:00
---
I recently wrote a new library for Ruby called [Unique
Thread](https://github.com/ferdynton/unique_thread). It allows multiple Ruby
processes to communicate through Redis so only one of them is running a given
thread concurrently. The idea for this came to me when wanting to run a crontab
runner on a Rails app but only wanted one of the processes to be running at any
given time. If the process running the thread shuts down, I also wanted another
process to wake up and start running in its place.

Since then, I've found many other use cases where this strategy might be
useful. This post explores some use cases for Unique Thread and the technical
challenges.

## Usage

The most simple usage of Unique Thread goes like:

```ruby
UniqueThread.new('my-thread-name').run do
  loop do
    # Some code that needs to run every minute
    sleep(60)
  end
end
```

The block in the Unique Thread loops infinitely running some code and sleeping
for 60 seconds. It doesn't block the main thread, so it can be used on a Rails
initializer and the app will boot normally. If some event like a deployment
stops the process, another one will start the infinite loop in its place.

## Use cases

Beginning with the use case described in the header, running a process to
trigger cron tasks is the most common use case I can think for Unique Thread.
While there are many different tools in Ruby to run cron jobs, not all provide
a solution for how to run it reliably on production (without having an
exclusive process for it or being duplicated on every process). Unique Thread
solves the uniqueness problem but doesn't solve other ones like:

* Having small periods of time without a process running the thread
* Switching processes often and hence having more frequent repetitions of the
  block (in the context of cron, this would mean running more than one "tick"
  per minute)

In our case, this meant adding extra layers to our cron scheduler to store on
a database the times when our cron commands were run. The scheduler then checks
if the command should have run since then and now. This avoids running each
command twice and also resists the possibility of the app being down for over
one minute (although the commands will run late).

Another use case for Unique Thread is for pushing statsd metrics about the app
to a monitoring service. This is specially useful when the metrics come from
information in the database or Redis, like the amount of users that have been
online in the last hour or the latency of the Sidekiq queues. While submitting
more information than needed might not be harmful in these cases, the overhead
from performance and extra data might not be desired.

One final use case for Unique Thread is to pull data from external sources into
the app's database. Imagine that you wanted to keep a CSV file from an S3
bucket in sync with a local table. With Unique Thread you can run this
operation as often as every second. If this operation was ran by every process
it would run multiple times per second, which in some APIs could mean being
throttled.

## Technical challenges

Finally, I want to talk about why this problem is hard to solve. First of all,
Redis does not provide many complex operations to be ran atomically. This is
key for Unique Thread because it can't allow two processes to believe they're
the ones that should be running right now. It needs a way of doing two things
atomically:

* Check if the current process holds (or can hold) a distributed lock
* Acquire the lock when possible

Without atomic operations, two processes could think they can hold the lock and
then both acquire it, one of them being wrong. Thankfully, Redis allows running
small LUA scrips in its server and guarantees that it runs atomically. Unique
Thread uses this feature efficiently to minimize the performance hit on Redis
and guarantee uniqueness.

However, Unique Thread is not prepared for environments with distributed Redis
with master and slave. This is explained in more detail in [the Redis
documentation](https://redis.io/topics/distlock). The scope of this project for
me was to learn about lower level tools for concurrency and get the job done
simply for the majority of use cases. Concurrency is extremely hard!

One of the reasons concurrency is hard is because it's very hard to know it's
right. Running concurrent code and getting the right results doesn't guarantee
anything because it might have been a "lucky run" where everything went right.
There might be a way for your code to run and cause race condition, but finding
that possible way is very hard. This makes testing very important because
changing concurrent code is more difficult than usual. With Unique Thread I
spent a long time ensuring the code is well encapsulated so it can be easier to
understand. This already makes it easier to find flaws in the logic but it also
comes with a thorough test suite for all the individual components. The tests
also use small 'sleep' commands to suggest the Ruby VM to switch threads, which
is a very clumsy practice but makes the tests easier to understand.

If you've read until here, thank you for your interest! I hope you've learned
something or that Unique Thread helps your app. If you have any suggestions for
the code, feel free to [open an issue on the
repo](https://github.com/ferdynton/unique_thread/issues/new) or find me on
Twitter!

