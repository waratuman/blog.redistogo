---
title: Resque Scheduler
author: James R. Bracy
layout: post
---

[Rescue Scheduler](http://github.com/bvandenbos/resque-scheduler) is an
extension to [Resque](http://github.com/defunkt/resque) that provides both
scheduled and delayed jobs. This service can replace [cron](http://en.wikipedia.org/wiki/Cron) and bring the
ability to schedule task to execute at a certain time.

This tutorial is based off the previous tutorial on [using Resque with Redis To
Go](http://blog.redistogo.com/2010/07/26/resque-with-redis-to-go/). We will be
using the same `cookie_monster` application.

Before we begin you should be aware of the difference between a scheduled job
and a delayed job. A scheduled job is a task that is specified to be run at a
certain time. For example you may want to update any indexes at midnight
everyday; this is a scheduled job. A delayed job is a job that gets put on the
queue with a specified time to run at. For example you may want to schedule a
followup email to go out a few days after a user has signed up.

Now lets get started.

# Setup

First check out the `cookie_monster` app from [github](http://github.com/waratuman/cookie-monster).

    $ git clone git://github.com/waratuman/cookie-monster.git
    $ cd cookie-monster

# Install Resque Scheduler

Modify the `Gemfile` to include `resque-scheduler`.

    gem 'resque-scheduler'
    
Install dependencies.

    $ bundle install

# Configure Resque Scheduler

Resque Scheduler run as a process. To run it we will end up calling `rake
resque:scheduler`. Before that can happen the `lib/tasks/resque.rake` file will
need to be updated to load these task. Added the following line to the file:

    require 'resque_scheduler/tasks'

Now you should be able to run `rake resque:scheduler`. Since the crontab for
Resque Scheduler has not been set the process will schedule any jobs. It will
however look for delayed jobs that need to be run.

Lets go ahead and create the cron tab for Resque Scheduler. Create the file
`config/resque_schedule.yml` and insert the following text:

    breakfast:
      cron: 0 8 * * *
      class: Eat
      args: scrambled egg
      description: Breakfast
    lunch:
      cron: 0 12 * * *
      class: Eat
      args: penut-butter and jelly sandwich
      description: Lunch
    dinner:
      cron: 0 6 * * *
      class: Eat
      args: Steak
      description: Dinner
    midnight_snack:
      cron: 0 0 * * *
      class: Eat
      args: Milk & Cookies
      description: Midnight Snack

We still need to load the schedule when we initialize Resque. To do this open
`config/initializers/resque.rb` and add the following lines.
  
    require 'resque_scheduler'
    Resque.schedule = YAML.load_file("#{Rails.root}/config/resque_schedule.yml")

# Start the Scheduler Process

All that is left is to run the scheduler process. 

    $ rake resque:scheduler

Note that if the scheduler process is not running when a scheduled job needs
to be enqueued, it will not be put on the queue once the scheduler process is
running again. In some cases this may not be an issue, it may be fine for
`cookie-monster` to miss breakfast on occasion. FlightCaster needed jobs to be
run every minute. The system was build so that if one job was missed the next
one would pick up the work. An alternative solution would have been to have a
schedule job put delayed jobs on the queue to run at the specified times. Then
if the scheduler process died the jobs would be performed once the process was
up again.

## Delayed Jobs

To schedule a job to take place after a certain amount of time is almost as
easy as enqueuing a job. Simple specify the time you want to have the Job run
at.

    Resque.enqueue_at(2.days.from_now, Eat, 'leftovers')

## Web Additions

Resque scheduler also extends the Resque web UI, so you get a great looking
interface from which you can see delayed jobs and scheduled jobs.

<img src="/resources/images/scheduled_jobs.png" alt="Resque Scheduler Web Extentsion" style="width: 100%;" />

<img src="/resources/images/delayed_jobs.png" alt="Resque Scheduler Web Extentsion" style="width: 100%;" />

## Conclusion

[Rescue Scheduler](http://github.com/bvandenbos/resque-scheduler) offers a simple
and easy way to added scheduling abilities to any queueing system and is well
worth the move from cron.