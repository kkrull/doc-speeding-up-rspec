# Speeding up RSpec in `ascent-web`

## Redirecting output

I stumbled upon this by accident, but I noticed that RSpec was running noticably faster when I redirected its output.  The `progress` formatter seems to flush `$stdout` for each dot.

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

## `Mongoid.purge!`

We were using `DatabaseCleaner` before, which only supports the `:truncate`
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

## Results

| Changes | Process time | RSpec time |
| :----   | ----:        | ----:      |
| None | 2m21s | 2m11s |
| Redirect output | 1m59s | 1m41s |
| Redirection + `Mongoid.purge!` | 1m3s | 45s |

Overall, that's a **55% improvement**.
