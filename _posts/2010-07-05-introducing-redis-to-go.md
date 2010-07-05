---
title: Introducing Redis To Go
author: James R. Bracy
editor: Laura Price
layout: post
---

Redis is a in-memory database that has recently had major uptake. In many ways
it can be thought of as a database for data-structures. The ability to store
different data-structures is what gives Redis its strength and flexibility.
Since first using Redis, I now find it working its way into just about every
one of my projects.

Being a Rails programmer, the easiest solution for background processing was
Delayed Job. Delayed Job did well when the queue was not very large or when
the processing of data could wait for a few hours. Since it uses the database
as a queue, performance of the site would take a hit. Delayed Job also lacked
the ability to scale. Some of the projects in which I was involved required
data to be processed in near-real time. After fighting with Delayed Job for
too long I tried Resque, a queuing system for Ruby based on Redis.

One of the most significant gains was the ability to recover from a full data
loss. When using Delayed Job, the recovery time was measuring in hours.
Typically it would take 6 to 8 hours for the system to fully recover. After
switching to Resque, the system was able to recover in under 30 minutes. That
is a significant increase and the key was switching from Delayed Job to
Resque. (To be fair there were other optimizations that were made but Resque
had the most significant impact.)

Every single website and project that I have been involved has at one point
required a queuing system.  No longer will I even touch Delayed Job, Resque
has had such a significant impact and has such great performance as well as a
built in interface for introspection into the queue.

Setting up a server with Redis to power Resque can sometimes be daunting as it
is another server in the system to manage and one more moving part. 

And so, we introduce Redis To Go: a new Redis server ready for you in under a
minute.  Now using Redis in your application is dead simple. 

It doesn't matter if you are using it for a queuing system or not. Do you want
to do AB Testing with Vanity:  Get a Redis server from Redis To Go and deploy
without the overhead of setting up a server and managing it.

Redis To Go was born out of a need that I have experience on several projects.
If all of these projects are using Redis why not build a provisioning system
to meet this need and lower the overall cost of each indivual project? Redis
To Go meets one of my needs and I hope that it will meet your needs as well.

Redis To Go will initially be offered as a Heroku add-on.   If you are
interested in the current alpha program or the beta program that will be
launching in the next week or two, shoot me an email at james at redistogo.