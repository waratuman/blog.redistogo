---
title: Configurable Instances & Notifications
author: James R. Bracy
layout: post
---

By default all paid plans use [AOF](http://code.google.com/p/redis/wiki/AppendOnlyFileHowto)
for persistence. This is the safest option but is the best for everyone. You
should change this and several other default options to suit your specific
needs. Today we released the ability to change several of the config options.
All of the changes happen instantly with no down time for the Redis instance.

One of the requested features from our users has been notifications for when
an instance is nearing its memory limit. Starting today all paid plans will
receive an email when the instance is using more than 80% of its maximum
memory. This is important because the memory limits are hard. If you try to go
over this limit Redis will just reject any write request.
