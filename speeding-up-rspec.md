# Speeding up RSpec in `ascent-web`
## On the hunt for slow tests

Which tests are slow?  Profiling all 2278 RSpec examples (`rspec --profile
2278`), there are only a couple that are significantly slower than the rest.

![RSpec example times](rspec-example-times.png "RSpec example times")

In other words, fixing a few slow tests is not going to have much of an impact.
*All* of the tests are slow.

### It's got to be Rails...right?

RSpec listed `./spec/lib/batches/batch_filter_spec.rb:35` as fastest example at 0.017 seconds.

**So why does it take 11.6 seconds to run that test by itself?**

I had a couple of ideas that were dead ends:

1. Tests that don't `require 'spec_helper'` run faster.  If you do use `spec_helper`, it loads Rails and Rails autoloads
   all of our code (see `config/application.rb`).  Disabling for the test environment breaks a bunch of tests, and it
   turns out that [Rails autoloading can be
   complicated](http://urbanautomaton.com/blog/2013/08/27/rails-autoloading-hell/).
2. Our `$LOAD_PATH` has over 200 directories in it, which shouldn't be necessary.  I fixed that and fixed the
   `require` statments that were off, but that didn't improve the time any.

The penalty is a one-time, startup penalty anyway.  It's agonizing when running a single test, but it doesn't have a
great impact on the overall test suite.  [Spork](https://github.com/sporkrb/spork),
[Zeus](https://github.com/burke/zeus) and [Spin](https://github.com/jstorimer/spin) exist to ease this burden by
pre-loading Rails, but they add some configuration burden and can lead to incorrect test results if you're not careful.

### Application layers

Does one layer of the application take most of the 2+ minutes to run the suite?  Not really:

- 228 controller tests: 11.1s
- 329 gateway tests: 17.3s
- 280 interactor tests: 12.3s

### Memory usage?

No, it's not really this either.  The `rspec` process gets as high as 1074M VIRT size.  My machine has a lot more free
memory than that, so that's not likely to be the issue.

### Profiling helpers

The problem is revealed when you add profiling to each RSpec hook that's running (stop a test in the debugger and look
at the example class's internals to figure out which hooks are active).  Taking the total time for each file that adds
some sort of `Before` or `After` hook, I got these total times over the entire test suite spent in each helper:

- `app_config_spec_helper`: 0.603s
- `capybara`: 0.119s
- `fabrication_spec_helper`: 0.231s
- `mongoid_spec_helper`: **48.98s**
- `rspec-rails`: 0s
- `spec_helper`: 2.6s

`mongoid_spec_helper` runs a total of **1595 times**, taking 0.031 seconds each time it runs - all by itself!  Running
all tests that need `mongoid_spec_helper` yields 0.046s per example, vs. 0.013s per example for those that do not need
to access Mongo. 

So now we have a lead for what's taking so long.

## Testing with Mongo
### Keeping test databases in memory

Databases used in small tests are usually pretty small, and spinning up an
external database can be a pain.  Some databases offer a [memory-only
mode](http://hsqldb.org/doc/guide/ch01.html#N101CA) when you can't (or don't
want to) break the dependency on the database.

I did some searching for ways to get MongoDB to run this way, and I didn't find anything quite like that.  You can,
however, run `mongodb` in a RAM disk ([download more](http://www.downloadmoreram.com/) if you're running low) and get a
reasonable performance improvement.  My tests ran **28% faster** when I did the following:

```sh
$ mount -o size=1G -t tmpfs none /mnt/tmpfs
$ mongod --smallfiles --noprealloc --nojournal --dbpath /mnt/tmpfs
```

This however, became unnecessary with the next finding.

### Something that works even better: `Mongoid.purge!`

We've been using `DatabaseCleaner` before, which only supports the `:truncate`
method.  Mongoid offers a [faster
way](https://github.com/mongoid/mongoid/blob/master/lib/mongoid/config.rb#L166)
to clear out the database before each test.

In case anyone is running RSpec in development or production environments (i.e.
`RAILS_ENV=development bundle exec rspec`), purging is limited to just the test
environment.  Note that tests requiring `mongoid_spec_helper` default to the
test environment.  You don't have to do anything special to get fast
performance.

When you do this, you only spend **8.9s** in `mongoid_spec_helper` over the life of the test suite.

```ruby 
# spec/mongoid_spec_helper.rb
ENV['RAILS_ENV'] ||= 'test'
...
RSpec.configure do |config|
  #Retain pre-test data in development/production environments; purge in test to maintain cleanliness and to run faster
  if ENV['RAILS_ENV'] == 'test'
    config.before(:each) { Mongoid.purge! }
  else
    require 'database_cleaner'
    config.before(:suite) do
      DatabaseCleaner.strategy = :truncation
      DatabaseCleaner.orm = "mongoid"
    end
    config.before(:each) { DatabaseCleaner.clean }
  end
end
```

Going for the win and using `Mongoid.purge!` with a RAMdisk'd `mongodb` didn't
do any better than just using `Mongoid.purge!` by itself, so it wasn't worth
the trouble of changing how the service starts up.

## Redirecting output

I stumbled upon this by accident, when I noticed RSpec running faster when I
redirected its output.  The `progress` formatter seems to flush `$stdout` for
each dot.

Running `bundle exec rspec --out <file> ...` does the trick, but it also makes
you go through another stop of looking at the file when you want to see
results.  I created `script/rspec` to automate the process of running the specs
and looking at the file.

If all specs pass, you'll see the summary:

```sh
ascent-web$ ./script/rspec spec/lib/assay_configuration/lots/lot_gateway_spec.rb 
RSpec output being saved to tmp/rspec.txt
Finished in 0.48446 seconds
38 examples, 0 failures
```

If any specs fail, you'll see the whole output:

```sh
ascent-web$ ./script/rspec spec/lib/assay_configuration/lots/lot_gateway_spec.rb
RSpec output being saved to tmp/rspec.txt
rspec ./spec/lib/assay_configuration/lots/lot_gateway_spec.rb:439 # LotGateway#update given an ID for an existing assay given a Lot with #compounds overwrites Lot#compounds with UpdateLotRequestEntity#compounds
rspec ./spec/lib/assay_configuration/lots/lot_gateway_spec.rb:442 # LotGateway#update given an ID for an existing assay given a Lot with #compounds returns LotEntity#compounds as [LotCompoundEntity ...]
```

We could also try [Fuubar](https://github.com/thekompanee/fuubar#installation)
to get a progress bar, while maintaining the increased speed.

## Results

At the end of the day, here are the improvements that I applied.

| Changes | Process time | RSpec time |
| :----   | ----:        | ----:      |
| None | 2m21s | 2m11s |
| Redirect output | 1m59s | 1m41s |
| Redirection + `Mongoid.purge!` | 1m3s | 45s |

Overall, that's a **55% improvement**.

### Thoughts ahead

At the time of this writing, we had **1330 examples that load `mongoid_spec_helper`**.  That's more tests relying on
Mongo than not, and we only have a handful (23) of gateway classes (where you often do want to test I/O with Mongo).
This suggests that there are some tests out there that - perhaps indirectly - gain access to Mongo when they don't need
to.

Each example that can access Mongo takes extra time, so be sure you're breaking dependencies with good engineering
practices when it's practical to do so.
