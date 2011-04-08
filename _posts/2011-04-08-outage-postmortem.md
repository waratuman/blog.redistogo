---
title: Outage Postmortem
author: James R. Bracy
layout: post
---

On April 6th `catfish.redistogo.com` became unresponsive. Typically our alert
system would have notified within a minute. But the primary system was
self-hosted. Our secondary notification systems do not check as often, but did
alert us to the issue.

Once we had been alerted that the server was unresponsive we attempted to reboot
the server, but were unable to get the server to reboot. As a result we
removed EBS volumes, brought up another instance to restore service to the
affected Redis instances.

As a result we are adding more external monitoring services that will alert
us when an instance becomes unresponsive.