---
title: Resque with Sinatra
author: James R. Bracy
layout: post
---

[Rails](http://rubyonrails.org/) isn't the only framework that [Heroku](http://heroku.com/)
supports and a few people have been trying to get [Resque](http://github.com/defunkt/resque)
up and running with [Sinatra](http://www.sinatrarb.com/). Here is a guide to
get you started using Resque with Sinatra.

## Setup

Add the [Redis To Go](http://redistogo.com) add-on to your Heroku application.
Adding the add-on provides an environmental variable named `REDISTOGO_URL`.

Create a file called `app.rb` and add the following:

    require 'rubygems'
    require 'sinatra'
    require 'resque'
    require 'system_timer'
    
    uri = URI.parse(ENV["REDISTOGO_URL"])
    Resque.redis = Redis.new(:host => uri.host, :port => uri.port, :password => uri.password)
    
    get '/' do
      "<a href='/eat/mango'>eat something</a>
       <a href='/resque'>resque</a>"
    end

## Create a Job

Jobs are Ruby classes or modules that respond to the perform method. Create
the job named Consume in `app.rb`.
    
    class Eat
      @queue = :food
    
      def self.perform(food)
        puts "Ate #{food}!"
      end
    end

## Enqueue the Job

In this case I'm just going to create an application that will put a job on
the queue any time the url is hit. This will go in `app.rb` as well.

    get '/eat/:food' do
      Resque.enqueue(Eat, params['food'])
      "Put #{params['food']} in fridge to eat later."
    end
    
The `app.rb` file should now contain the following:

    require 'rubygems'
    require 'sinatra'
    require 'resque'
    require 'system_timer'

    uri = URI.parse(ENV["REDISTOGO_URL"])
    Resque.redis = Redis.new(:host => uri.host, :port => uri.port, :password => uri.password)

    module Eat
      @queue = :food

      def self.perform(food)
        puts "Ate #{food}!"
      end
    end

    get '/' do
      "<a href='/eat/mango'>eat something</a>
       <a href='/resque'>resque</a>"
    end

    get '/eat/:food' do
      Resque.enqueue(Eat, params['food'])
      "Put #{params['food']} in fridge to eat later."
    end

## Start a Worker

Create the file `Rakefile` and add the following to it:

    require 'app'
    require 'resque/tasks'

    task "resque:setup" do
      ENV['QUEUE'] = '*'
    end

    desc "Alias for resque:work (To run workers on Heroku)"
    task "jobs:work" => "resque:work"

## Rackup

A Rackup file is needed in order to deploy to Heroku. Create the file
`config.ru`. This will tell it to run the Sinatra application and will also
server up the Resque Web Interface.

    require 'app'
    require 'resque/server'

    run Rack::URLMap.new \
      "/"       => Sinatra::Application,
      "/resque" => Resque::Server.new
    
## Run

Now lets run the application. Here I'll be using [Unicorn](http://unicorn.bogomips.org/),
but you can use [Thin](http://code.macournoyer.com/thin/) or any other server
that supports [Rack](http://rack.rubyforge.org/).

    $ unicorn config.ru

Try going to the `/eat/food` path and the `/resque` path. You should start to
see jobs pop onto the queue. To run the jobs simply run `rake jobs:work`. Don't
forgot to set the `REDISTOGO_URL` in your local development.

## Deploy

To deploy to Heroku, simply follow the [docs for deploying a Rack app](http://docs.heroku.com/rack).

If you have any questions feel free to email james at redistogo.