---
layout:     post
title:      Sinatra with RABL
date:       2016-06-27 12:55:19
summary:    contiune to build a simple web application with Sinatra and RABL
categories: developing
---

Assume you have built a simple API app with ActiveRecord solely. If you didn’t, you could check this [link](http://sullivanlee.info/developing/2016/06/12/activerecord-only/).

Now, let’s continually make this application complete.

## Dependencies

* Ruby 2.2
* ActiveRecord 4.2.5
* Sinatra 1.4.7
* RABL 0.11.6


## Set up the Sinatra

As before, we’re going to write a very simple test to make sure that everything is correct.

{% highlight ruby lineanchors %}
# test/app/apis/event_test.rb
require './test/test_helper'

class APITest < MiniTest::Unit::TestCase
  def app
    EventAPI
  end

  def test_hello_world
    get '/'
    assert_equal "Hello, World!", last_response.body
  end
end
{% endhighlight %}

To this, we need to add Sinatra and RABL to the Gemfile and run `bundle install`.

{% highlight ruby lineanchors %}
# Gemfile
source 'https://rubygems.org'

gem 'sinatra'
gem 'activerecord', :require => 'active_record'
gem 'sqlite3'
gem 'rabl'
{% endhighlight %}

Run the test.

```bash
ruby test/app/apis/event_test.rb
```

Got an error.

```bash
APITest#test_hello_world:
NoMethodError: undefined method `get' for #<APITest:0x007fb9efd591b0>
    test/apis/event_test.rb:9:in `test_hello_world'
```

We need the Rack-test gem for the API tests, so add it to the Gemfile.

{% highlight ruby lineanchors %}
# Gemfile
source 'https://rubygems.org'

gem 'sinatra'
gem 'activerecord', :require => 'active_record'
gem 'sqlite3'
gem 'rabl'

group :test do
  gem 'rack-test'
end
{% endhighlight %}

Then we configure it at test_help.rb

{% highlight ruby lineanchors %}
# test/test_helper.rb
ENV['RACK_ENV'] = 'test'

require 'minitest/autorun'
require './config/environment'

# ..

class Minitest::Unit::TestCase
  include Rack::Test::Methods
end
{% endhighlight %}

Run the test again.

```bash
APITest#test_hello_world:
NameError: uninitialized constant APITest::EventAPI
    test/apis/event_test.rb:5:in `app'
```

Let’s create the EventAPI class.

{% highlight ruby lineanchors %}
# app/apis/event_api.rb
require './config/environment'

class EventAPI < Sinatra::Base
  get '/' do
    "Hello, World!"
  end
end
{% endhighlight %}

Run it again and the test should pass.

```bash
Finished in 0.028669s, 34.8804 runs/s, 34.8804 assertions/s.

1 runs, 1 assertions, 0 failures, 0 errors, 0 skips
```

## Combine Sinatra with the RABL

Now we write a little complex test.

{% highlight ruby lineanchors %}
# test/app/apis/event_test.rb
# ..

def test_event
  event = create(:event)
  get "/events/#{event.id}"

  data = JSON.parse last_response.body
  assert_equal 200, last_response.status
  assert_equal event.id, data['event']['id']
  assert_qeual event.name, data['event']['name']
end
{% endhighlight %}

Assume we have a route "GET /events/:id".

First, we use faker for fake data and factory_girl for setting test data.

{% highlight ruby lineanchors %}
# Gemfile
source 'https://rubygems.org'

gem 'sinatra'
gem 'activerecord', :require => 'active_record'
gem 'sqlite3'
gem 'rabl'

group :test do
  gem 'rack-test'
  gem 'factory_girl', "~> 4.0"
  gem 'faker'
end
{% endhighlight %}

Second, configure the test_help.rb

{% highlight ruby lineanchors %}
# test/test_helper.rb
ENV['RACK_ENV'] = 'test'

require 'minitest/autorun'
require './config/environment'

# ..

class Minitest::Unit::TestCase
  include Rack::Test::Methods
  include FactoryGirl::Syntax::Methods
end

FactoryGirl.find_definitions
{% endhighlight %}

Factories can be defined anywhere, but will be automatically loaded after calling `FactoryGirl.find_definitions` if factories are defined in files at the following locations:

```bash
test/factories.rb
spec/factories.rb
test/factories/*.rb
spec/factories/*.rb
```

Now create event_factory.rb.

{% highlight ruby lineanchors %}
# test/factories/event_factory.rb
FactoryGirl.define do
  factory :event do
    name Faker::Beer.name
  end
end
{% endhighlight %}

Try to run the test.

```bash
APITest#test_event:
JSON::ParserError: 757: unexpected token at '<h1>Not Found</h1>'
```

Add a route to match "GET /events/:id"

{% highlight ruby lineanchors %}
# app/apis/event_api.rb
require './config/environment'

class EventAPI < Sinatra::Base
  # ..
  
  get '/events/:id' do
    @event = Event.find(params['id'])
    Rabl::Renderer.json(@event, 'event')
  end
end
{% endhighlight %}

Run the test again.

```bash
APITest#test_event:
RuntimeError: Cannot find rabl template 'event' within registered ([]) view paths!
```

At the end of the route, we called Rabl::Renderer and tried to render a template "event".

First, we create an "event" template.

{% highlight ruby lineanchors %}
# app/views/event.rabl
object @event
attributes :id, :name
{% endhighlight %}

Second, configure RABL veiw_path in environment.rb.

{% highlight ruby lineanchors %}
# config/environment.rb
# ..

Rabl.configure do |config|
  config.view_paths = [File.join(File.dirname(__FILE__), "../app/views/")]
end
{% endhighlight %}

Run the test and That’s working!

```bash
Finished in 0.063928s, 31.2852 runs/s, 62.5703 assertions/s.

2 runs, 4 assertions, 0 failures, 0 errors, 0 skips
```

The final project tree looks like this:

```bash
├── Gemfile
├── Gemfile.lock
├── Rakefile
├── app
│   ├── apis
│   │   └── event.rb
│   ├── models
│   │   └── event.rb
│   └── views
│        └── event.rabl
├── config
│   ├── database.yml
│   └── environment.rb
├── db
│   ├── development.sqlite3
│   └── migrate
│       └── 0001_create_events.rb
├── test
│   ├── app
│   │   ├── apis
│   │   │   └── event_test.rb
│   │   └── models
│   │       └── event_test.rb 
│   └── test_helper.rb
```
