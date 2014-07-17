# Speeding up RSpec in `ascent-web`

## Testing with Mongo
### Keeping test databases in memory

Databases used in small tests are usually pretty small, and spinning up an
external database can be a pain.  Some databases offer a [memory-only
mode](http://hsqldb.org/doc/guide/ch01.html#N101CA) when you can't (or don't
want to) break the dependency on the database.

I did some searching for ways to get MongoDB to run this way, and I didn't find
anything quite like that.  You can, however, run `mongodb` in a RAM disk
([download more](http://www.downloadmoreram.com/) if you're running low) and
get a reasonable performance improvement.  My tests ran 28% faster when I did
the following:

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
you go through another stop of loooking at the file when you want to see
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

| Changes | Process time | RSpec time |
| :----   | ----:        | ----:      |
| None | 2m21s | 2m11s |
| Redirect output | 1m59s | 1m41s |
| Redirection + `Mongoid.purge!` | 1m3s | 45s |

Overall, that's a **55% improvement**.
