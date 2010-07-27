---
title: Resque with Redis To Go
author: James R. Bracy
layout: post
---

[Resque](http://github.com/defunkt/resque) is a queueing system that is backed
by Redis. Common use cases include sending emails and processing data. For more
information about Resque itself, visit [http://github.com/defunkt/resque](http://github.com/defunkt/resque).
This tutorial will cover setting up Resque with Rails and Redis To Go.

Being a Rails programmer, the easiest solution for background processing was
[Delayed Job](http://github.com/tobi/delayed_job). Delayed Job did well when
the queue was not very large or when the processing of data could wait for a
few hours. Since it uses the database as a queue, performance of the site
would take a hit. Delayed Job also lacked the ability to scale. Some of the
projects in which I was involved required data to be processed in near-real
time. After fighting with Delayed Job for too long I tried Resque, a queuing
system for Ruby based on Redis.

Delayed Job checks the queue every second. When you have 1 worker, that's 1
request per second. This is fine for a few workers, but as you start to add
more you're talking about a bigger and bigger load against your database
just to check state. This is happening exactly when you don't want it - when
you're trying to scale!  At FlightCaster we could have 400+ jobs running
simultaneously to process data, The load on the database alone from 400 req/s
for Delayed Job to get the next task off the queue would crush the database,
making it impossible to run. With Resque all 400 jobs can run without any
issues.

Every single website and project that I have been involved has at one point
required a queuing system.  No longer will I even touch Delayed Job, Resque
has had such a significant impact and has such great performance as well as
a built in interface for introspection into the queue.

This tutorial is very similar to the systems that [FlightCaster](http://flightcaster.com/),
[Juice In the City](http://www.juiceinthecity.com/), and even [Redis To Go](http://redistogo.com/)
uses.

Enough with the context, lets get started.

## Set Up Rails

This is going to be a Rails 3 app, so get the latest gem.

    $ sudo gem install rails --pre
   
Create the application:

    $ rails new cookie_monster
    $ cd cookie_monster
    
Modify the Gemfile to include Resque.

    source 'http://rubygems.org'

    gem 'rails', '3.0.0.rc'
    gem 'sqlite3-ruby', :require => 'sqlite3'
    gem 'resque'
    gem 'SystemTimer'
    
Install all of the gems and dependencies using [Bundler](http://gembundler.com/).

    $ bundle install

## Set Up Redis To Go

Go to [Redis To Go](http://redistogo.com/) and sign up for the free plan. Once
you have an instance, grab the URL given to you and modify the
`config/initializers/resque.rb` as follows:

    ENV["REDISTOGO_URL"] ||= "redis://username:password@host:1234/"
    
    uri = URI.parse(ENV["REDISTOGO_URL"])
    Resque.redis = Redis.new(:host => uri.host, :port => uri.port, :password => uri.password)

## Create a Job

Jobs are Ruby classes or modules that respond to the `perform` method. A good
place to put jobs that perform background work would be in `app/jobs`. Create
the job named `Consume` in `app/jobs/eat.rb`

    module Eat
      @queue = :food
      
      def perform(food)
        puts "Ate #{food}!"
      end
    end

Inside `config/initializers/resque.rb` place the following code so that
`app/jobs/eat.rb` is loaded.

    Dir["#{Rails.root}/app/jobs/*.rb"].each { |file| require file }
    
## Enqueue the Job

The main use case for using a queuing system is to take prevent a task that
could take more than a second from blocking a request. This will often happen
in either the model or controller. Create a controller named `eat`.

    $ rails g controller eat
    
Create a route for the controller in `config/routes.rb`  

    CookieMonster::Application.routes.draw do
      match 'eat/:food' => 'eat#food'
    end

Create the action in the controller. This action will put a job on the queue
and return the request, leaving any work to be done outside of the request.

    class EatController < ApplicationController

      def food
        Resque.enqueue(Eat, params[:food])
        render :text => "Put #{params[:food]} in fridge to eat later."
      end

    end

## Start a Worker

Open `lib/tasks/resque.rb` and place the following inside.
    
    require 'resque/tasks'
    
    task "resque:setup" => :environment

This will load the Resque tasks and load the environment which is required for
doing any work.
    
To start a worker that will pull work off of all queues run the command:

    $ rake resque:work QUEUE=*

## Start the Server

In a separate terminal start the rails server.

    $ rails s
    
## Test the Application  

Pull up a browser and go to `http://localhost:3000/eat/cookie`. You should get
the following response.

    Put cookie in fridge to eat later.
    
In the terminal where you started the worker it should have outputted:

    Ate cookie!
  
## Introspection  

One of the most useful aspects of Resque is the ability to perform
introspection. Resque has a great interface that can be give you and
understanding of what is going on. The best part is that you can load it in a
subpath with Rack's `URLMap`.

Open `config.ru` and replace what is in the file with the following code. This
will boot your app as the root and give you `/resque` as the web front end to
Resque.

    require ::File.expand_path('../config/environment',  __FILE__)

    require 'resque/server'
    run Rack::URLMap.new \
      "/"       => CookieMonster::Application,
      "/resque" => Resque::Server.new

From `/resque` you can see what is in your queues, the workers and what they
are currently doing, and the ability to view any failed jobs.
  
## Deploy to Heroku

This part can be done two different ways, using the [Heroku Redis To Go](http://addons.heroku.com/redistogo)
add-on or signing up for  directly. If you are using Heroku you need to be
part of the beta program for access.

The only adjustment that we need to make is to map the rake task `jobs:work`
to `resque:work` and set the queues to watch. After making these changes the
Heroku workers will work wonderfully. Open `lib/tasks/resque.rb` and replace
what is in there with the following:

    require 'resque/tasks'

    task "resque:setup" => :environment do
      ENV['QUEUE'] = '*'
    end

    desc "Alias for resque:work (To run workers on Heroku)"
    task "jobs:work" => "resque:work"

Now create the Heroku app and deploy.
    
    $ git init
    $ git ci -am "Initial Commit."
    $ heroku create
    $ heroku addons:add redistogo
    $ git push heroku master

Looks like we hit a snag. Rails RC1 was just release and requires Bundler 1.0.0.rc.1.
So open up the `Gemfile` and change the Rails version to beta 4. 

    gem 'rails', '3.0.0.beta4'
    
Also, remove the `Gemfile.lock` if it is under version control.

    $ git rm Gemfile.lock
    
Try to push again:

    $ git push heroku master
    
Now if you got to [http://warm-beach-19.heroku.com/eat/cookie](http://warm-beach-19.heroku.com/eat/cookie) (the example app)
a job will be placed on the Resque queue. You can view the queue from
[http://warm-beach-19.heroku.com/resque](http://warm-beach-19.heroku.com/resque)

The example app source code is located at [http://github.com/waratuman/cookie-monster](http://github.com/waratuman/cookie-monster).

## Conclusion

This should get you up and running with Resque and on your way to a scalable
solution. If you have any question, either leave a comment in the Hacker News
post or email james at redistogo.