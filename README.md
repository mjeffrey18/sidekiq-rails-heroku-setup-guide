# Setup Sidekiq and Redis on Rails and use in Heroku Production

Hey Everyone, 
I thought I would share this information with everyone to help anyone looking to setup and background job process on their rails development machine (which also works in Heroku production).

##### References:
- [Ruby on Rails](http://rubyonrails.org/ "Ruby on Rails")
- [Redis](https://redis.io/ "Redis")
- [Sidekiq](http://sidekiq.org/ "Sidekiq")

### Summary
Basically sidekiq is a background queuing and job processing worker, which allows you to pass off certain jobs to a background worker and not let the front end users notice any performance lag which the large processes are working in the background.
Sidekiq using Redis, which in-memory datastore, this has a dedicated server to hold the information sidekiq uses.

### Installation

To setup sidekiq you must first have a running rails application, a database server (I personally use PostgreSQL) and a Redis server. Technically the PG DB is not actually required to run Sidekiq, however, you cannot operate rails without one and most production environments will run all 4 mechanisms on separate instances i.e. Rails running on puma server as a process, PG as a database service, Redis as a data store service and Sidekiq as a worker process.
Basically in Heroku your should use at least 2 dyno's, 1 for rails and 1 for sidekiq.
Redis and postgresql run as addon services.
During this tutorial I’m going to skip setting up Rails and PostgreSQL, as I will assume that is already setup. 

#### Redis Setup

Using a Mac you can follow these commands to install Redis:

```bash
brew install redis
```

This will run through the installation process and install 

##### Production Redis
In Heroku production I suggest you use heroku-redis, its free, good for 30MB and 30 connections, I’ve not had any issues in over for over a year since using it.
Be sure to use the a separate redis production Add-on, if your using redis already as a cache store. Caching and background jobs don’t work well on the same instance.

#### Process Manager Setup
Install Foreman (or) Hivemind
```bash
gem install foreman
```
or
```bash
brew install hivemind
```
> Note - I used to use foreman but now I use hivemind, up to you, not a big deal at this point.

### Setup Rails

In Rails Gemfile
```ruby
gem 'sidekiq'
gem 'puma' # if not already there
```

Run bundler
```bash
bundle install
```

Create 2 new files in your rails ./ home directory 

	.procfile
	.procfile.dev

Add this code to `Procfile`
```yaml
web:    bundle exec puma -C config/puma.rb
worker: bundle exec sidekiq -e production -C config/sidekiq.yml
```

Add this code to `Procfile.dev`
```yaml
redis:  redis-ser
worker: bundle exec sidekiq
rails:  PORT=3000 bundle exec rails server
```

Create a file `config/sidekiq.yml`
```yaml
development:  
  :concurrency: 5
production:  
  :concurrency: 5
:queues:
  - critical
  - high
  - default
  - mailers
 ```

Create a file `config/initializers/sidekiq.rb`
```ruby
if Rails.env.production?
  Sidekiq.configure_client do |config|
    config.redis = { url: ENV['REDIS_URL'], :size => 1, network_timeout: 5 }
  end

  Sidekiq.configure_server do |config|
    config.redis = { url: ENV['REDIS_URL'], :size => 7, network_timeout: 5 }
  end
end
```

Puma file to be updated `config/puma.rb`
```ruby
workers Integer(ENV['WEB_CONCURRENCY'] || 2)
threads_count = Integer(ENV['RAILS_MAX_THREADS'] || 5)
threads threads_count, threads_count

preload_app!

rackup      DefaultRackup
port        ENV['PORT']     || 3000
environment ENV['RACK_ENV'] || 'development'

on_worker_boot do
  ActiveRecord::Base.establish_connection
end

# Allow puma to be restarted by rails restart command.
plugin :tmp_restart
```

> Note - (if) your use puma worker killer add this to `config/application.rb' also
```ruby
before_fork do
  require 'puma_worker_killer'

  PumaWorkerKiller.config do |config|
    config.ram                        = (ENV["PUMA_WORKER_KILLER_RAM"] || 512).to_i # mb, change as needed according to your dyno available mem
    config.frequency                  = (ENV["PUMA_WORKER_KILLER_FREQUENCY"] || 30).to_i # seconds, change as needed
    config.percent_usage              = (ENV["PUMA_WORKER_KILLER_PERCENT_USAGE"] || 0.96).to_f # change percentage as needed
    config.rolling_restart_frequency  = (ENV["PUMA_WORKER_KILLER_ROLLING_RESTART_FREQUENCY"] || 12).to_i * 3600 # 12 hours in seconds
  end
  PumaWorkerKiller.start
end
```

Be sure to have `puma worker killer` in your gemfile...
```ruby
gem "puma_worker_killer"
```

Edit this file `config/application.rb` with a single line
```ruby
require_relative 'boot'
require 'rails/all'
Bundler.require(*Rails.groups)

module Blog
  class Application < Rails::Application
    # ONLY the follow line is to be added
    config.active_job.queue_adapter = :sidekiq
  end
end
```

Now create a new route in `config/routes.rb`
```ruby
require 'sidekiq/web'
authenticate :user, lambda { |u| u.super_admin? } do
  mount Sidekiq::Web => '/sidekiq'
end
get '/sidekiq' => redirect('/')
```

If you do not have a super_admin or admin you can use this route instead:_
```ruby
require 'sidekiq/web'
authenticate :user do
  mount Sidekiq::Web => '/sidekiq'
end
get '/sidekiq' => redirect('/')
```

> note - this means any user signed in can access the sidekiq dashboard - if they know about the url
> I suggest at least having a admin or super_admin access

Create new directory in application folder `app/workers`
Otherwise you can also use ActiveJob since we changed the default queue adapter to sidekiq
I'll show you both options later to get up and running

From here you can start creating workers to run in the background

If your using this is production you need to set some environment variables as follows:

	RACK_ENV: production
	REDIS_PROVIDER: # Production redis server URL to be pasted here
	REDIS_URL: # Production redis server URL to be pasted here

Stop any rails servers you have currently running and install of running `rails s` to start rails, install run this command
```bash
foreman start -f Procfile.dev
```
or if your using hivemind
```bash
hivemind Procfile.dev
```

This will start redis, sidekiq and rails from 1 terminal window. You can pressing `CMD+C` anytime to stop the 3 processes

Create a worker

```bash
rails g sidekiq:worker RunDailyReports
```

This will create a new file called `app/workers/run_daily_report_worker.rb`
```ruby
class RunDailyReportWorker
  include Sidekiq::Worker

  def perform(name, count)
    # do something
  end
end
```

Ok so that’s how to create a sidekiq job, how about ActiveJob, lets create a new job also, so you have any option you want

```bash
rails generate job RunDailyReport
```

This will create a new file called `app/jobs/run_daily_report_job.rb`
```ruby
class RunDailyReportJob < ApplicationJob
  queue_as :default

  def perform(*args)
    # Do something later
  end
end
```

Both look nearly the same, however to have sidekiq work in activejob, you need to add the following line
```ruby
class RunDailyReportJob < ApplicationJob
include Sidekiq::Worker # New
  queue_as :default

  def perform(*args)
    # Do something later
  end
end
```

Now we can run any of the following commands from anywhere in rails or the terminal

##### Sidekiq Worker

	RunDailyReportWorker.perform_async
	RunDailyReportWorker.perform_async(arg)
	RunDailyReportWorker.perform_async(1,2)
	RunDailyReportWorker.perform_in(5.minutes)
	RunDailyReportWorker.perform_async

##### ActiveJob

	RunDailyReportJob.perform_now
	RunDailyReportJob.perform_later
	RunDailyReportJob.perform_now(arg)
	RunDailyReportJob.perform_later(1,2)
	RunDailyReportJob.perform_now


The above setup is just a basic setup to get you up and running for small to moderate app, running on Heroku.
> Note - Make sure to turn on the Heroku worker dyno and add the ENV variables 

Please check out the referenced documentation for more advanced usage of the services mentioned.
