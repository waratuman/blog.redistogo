---
title: Introducing Redis To Go
author: James R. Bracy
editor: Laura Price
layout: post
---

[Redis](http://code.google.com/p/redis/), an advanced key-value store, has
recently seen a major uptake. In many ways it can be thought of as a database
for data-structures. The ability to store different data-structures is what
gives Redis its strength and flexibility. Since first using Redis, I now find
it working its way into just about every one of my projects.

With every project I have done making use of Redis I found the most
aggravating part being the addition of a new server to manage. Setting up a
server can be daunting and adds one more part to a system which must be
managed. My time was spent not on the projects but on managing the system. What
happens if the server goes down? How do I make sure Redis is up and running?
Where do I backup my data and how do I do it?

This madness ends today! [Redis To Go](http://redistogo.com/) is an answer to
all of my problems listed above. In under a minute a new Redis server is ready
to be used. Now using Redis in any project is dead simple.

Redis To Go is currently offered as an add-on to [Heroku](http://heroku.com/).
Deploying web applications has never been easier. Heroku has a vision of
keeping deployment simple and pain free. The Heroku platform is robust and
powerful. It is a deadly combination of simplicity and power. With once click
Redis can be added to your application. The add-on program that Heroku offers
is a great way to expand the vision of simple and powerful deployments and
Redis To Go is proud to be a part of it.

If you happen to not use Heroku, don't worry. In the coming weeks we will be
adding the ability to provision Redis directly from the [Redis To Go](http://redistogo.com/)
website. If you are interested either the Heroku beta program or the Redis To
Go website alpha please fill out this [form](http://spreadsheets.google.com/viewform?formkey=dFZfTmZ1YWJpRzdSb3V1Wl9QaWVqcWc6MQ).

Any feedback or questions? Email james at redistogo.