---
layout:     post
title:      Web Application with ActiveRecord Solely
date:       2016-06-11 18:07:19
summary:    only using ActiveRecord to build a simple web application
categories: developing
---

Have you wonder if you could use ActiveRecord outside of Rails?

let's see how we can use ActiveRecod solely to build a simple web application step by step.

## Dependencies

* Ruby 2.2
* ActiveRecord 4.2.5

## Creating a Project

```bash
mkdir backend
cd backend
```

## Starting the Project from Test

We’ll start with a test. Create an empty test directory

```bash
mkdir -p test/models
```

We use MiniTest to create a test file: test/app/models/event_test.rb

{% highlight ruby lineanchors %}
# test/app/models/event_test.rb
class EventTest < MiniTest::Unit::TestCase
  def test_it_works
    Event.create(:name => 'test_event')
    assert_equal true, Event.find_by_name('test_event').present?
  end
end
{% endhighlight %}

Run it

```bash
ruby test/app/models/event_test.rb
```

You’ll get an error

```bash
test/app/models/event_test.rb:1:in <main>: 'uninitialized constant MiniTest (NameError)'
```

Then create a test_helper file: test/test_helper.rb

{% highlight ruby lineanchors %}
# test/test_helper.rb
require 'minitest/autorun'

# test/app/models/event_test.rb
require './test/test_helper'
class EventTest < MiniTest::Unit::TestCase
  # ...
end
{% endhighlight %}

Run it again and get a new error

```bash
EventTest#test_it_works:
NameError: 'uninitialized constant EventTest::Event'
    test/app/models/event_test.rb:5:in test_it_works
```

## Creating the Gemfile
Add a Gemfile to manage our dependencies and implement our first Model

{% highlight ruby lineanchors %}
# Gemfile
source 'https://rubygems.org'

gem 'activerecord', :require => 'active_record'
{% endhighlight %}

Run `bundle install` and it will generate the Gemfile.lock

Now create an environment file: config/environment.rb

{% highlight ruby lineanchors %}
# config/environment.rb
require 'bundler'
Bundler.require(:default)
{% endhighlight %}

And create first Model: app/models/event.rb

{% highlight ruby lineanchors %}
# app/models/event.rb
class Event < ActiveRecord::Base
end
{% endhighlight %}

Then require all ActiveModels in environment.rb

{% highlight ruby lineanchors %}
# config/environment.rb
require 'bundler'
Bundler.require(:default)

# require active_model
Dir.glob(File.join(File.dirname(__FILE__), "../app/**/*.rb")).each {|file| require file}
{% endhighlight %}

And require environment.rb in test_help.rb

{% highlight ruby lineanchors %}
# test/test_helper.rb
require 'minitest/autorun'
require './config/environment'
{% endhighlight %}

Run test again

```bash
EventTest#test_it_works:
ActiveRecord::ConnectionNotEstablished: No connection pool for Event
```

Now we have an event model, but it does not connect to a database

## Creating a Database Connection

Create a database config: config/database.yml

{% highlight ruby lineanchors %}
# config/database.yml
adapter: sqlite3
database: db/development.sqlite3
pool: 5
timeout: 5000
{% endhighlight %}

Add sqlite3 dependency to Gemfile

{% highlight ruby lineanchors %}
# Gemfile
source 'https://rubygems.org'

gem 'activerecord', :require => 'active_record'
gem 'sqlite3'
{% endhighlight %}

Require database.yml in the enviornment.rb

{% highlight ruby lineanchors %}
# config/environment.rb
require 'yaml'
require 'bundler'
Bundler.require(:default)

# ..

db_options = YAML.load(File.read('./config/database.yml'))
ActiveRecord::Base.establish_connection(db_options)
{% endhighlight %}

Because we altered the Gemfile, we need to run `bundle install` fisrt, and run test again

```bash
EventTest#test_it_works:
ActiveRecord::StatementInvalid: Could not find table 'events'
```

Now the Event model is connected to a database, but this database does not contain the table: events

## Creating migrations

Create first migration file: db/migrate/0001_create_events.rb

{% highlight ruby lineanchors %}
# db/migrate/0001_create_events.rb
class CreateEvents < ActiveRecord::Migration
  def change
    create_table :events do |t|
      t.string :name
      t.timestamps
    end
  end
end
{% endhighlight %}

Create Rakefile to execute `db:migrate`

{% highlight ruby lineanchors %}
# Rakefile
namespace :db do
  desc "migrate your database"
  task :migrate do
    require 'bundler'
    Bundler.require
    require './config/environment'
    ActiveRecord::Migrator.migrate('db/migrate')
  end
end
{% endhighlight %}

Run `rake db:migrate` first, and this will generate a sqlite3 db: db/development.sqlite3

Then run test again. Finally, we pass the test

```bash
Finished in 0.068492s, 14.6003 runs/s, 14.6003 assertions/s.

1 runs, 1 assertions, 0 failures, 0 errors, 0 skips
```

The project now looks like this:

```bash
├── Gemfile
├── Gemfile.lock
├── Rakefile
├── app
│   └── models
│       └── event.rb 
├── config
│   ├── database.yml
│   └── environment.rb
├── db
│   ├── development.sqlite3
│   └── migrate
│       └── 0001_create_events.rb
├── test
│   ├── app
│   │   └── models
│   │       └── event_test.rb 
│   └── test_helper.rb
```

## Cleaning the Database

We don't want to actually write to the database when we run the test

Let's change the event_test.rb

{% highlight ruby lineanchors %}
# test/app/models/event_test.rb
class EventTest < MiniTest::Unit::TestCase
  def test_it_works
    Event.create(:name => 'test_event')
    assert_equal 1, Event.count
  end
end
{% endhighlight %}

Run the test, and it should pass. Run it again, and it will fail

We could use a gem like `database_cleaner`, but that seems like a big solution to a small problem

So we can write a method that will do something like `database_cleaner`

In the test_helper.rb, add this module

{% highlight ruby lineanchors %}
# test/test_helper.rb
require 'minitest/autorun'
require './config/environment'

module WithRollback
  def temporarily(&block)
    ActiveRecord::Base.connection.transaction do
      block.call
      raise ActiveRecord::Rollback
    end
  end
end
{% endhighlight %}

Let's change event_test.rb again

{% highlight ruby lineanchors %}
# test/app/models/event_test.rb
class EventTest < MiniTest::Unit::TestCase
  include WithRollback

  def test_it_works
    assert_equal 0, Event.count
    temporarily do
      Event.create(:name => 'test_event')
      assert_equal 1, Event.count
    end
    assert_equal 0, Event.count
  end
end
{% endhighlight %}

Now we can clean our changes after the test finishing

## Using Different Environments

Normally, we use different gems and databases in different environments: development, test or production

First, we change the database.yml

{% highlight ruby lineanchors %}
# config/database.yml
development:
  adapter: sqlite3
  database: db/development.sqlite3
  pool: 5
  timeout: 5000
test:
  adapter: sqlite3
  database: db/test.sqlite3
  pool: 5
  timeout: 5000
production:
  adapter: sqlite3
  database: db/production.sqlite3
  pool: 5
  timeout: 5000
{% endhighlight %}

Second, we add some gems to different groups

{% highlight ruby lineanchors %}
# Gemfile
source 'https://rubygems.org'

gem 'activerecord', :require => 'active_record'
gem 'sqlite3'

group :test do
  gem 'rack-test'
  gem 'faker'
  gem 'pry'
end
{% endhighlight %}

Then we change the environment.rb

{% highlight ruby lineanchors %}
# config/environment.rb
require 'yaml'
require 'bundler'
Bundler.require(:default, (ENV['RACK_ENV'] || 'development').to_sym)

# ..

db_options = YAML.load(File.read('./config/database.yml'))[ENV['RACK_ENV'] || 'development']
ActiveRecord::Base.establish_connection(db_options)
{% endhighlight %}

Now we can run a command like this

```bash
rake db:migrate RACK_ENV=test
```

And we add a default environment to test_help.rb

{% highlight ruby lineanchors %}
# test/test_helper.rb
ENV['RACK_ENV'] = 'test'

require 'minitest/autorun'
require './config/environment'

# ...
{% endhighlight %}

Run the test again ,and we can use a different database and additional gems :)
