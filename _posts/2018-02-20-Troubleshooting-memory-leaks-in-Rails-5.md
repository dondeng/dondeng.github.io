---
title: "Troubleshooting memory leaks in Rails 5"
image: postgres_loves_ruby.png
date: 2018-02-20 12:35:35
categories:
  - rails
tags:
  - rails
  - memory
---

So we recently did an upgrade of our application at Kopo Kopo, Inc to Rails 5.1 and Ruby 2.4.1. For an upgrade
coming from Rails 4.0 things went well I must say. The only two issues we experienced were

- An issue with our master-slave RDS replication where by controllers that are heavily 'read-centric' and whose
  db requests are sent to a read replica, all over sudden would have caching problems and present stale data on
  subsequent access to the same page/view. (This will be for another blog post)
- We seemed to be having memory issues after a while. We are on an Nginx-Passenger web server stack and during peak hours
  there always seemed to be situations where our server would get maxed out of its memory by the passenger processes. We summized
  that something was leaking memory and the culprit needed to be found.

This post covers problem number 2.

We figured that the first order of business would be to get relief from having to periodically scramble and restart our
app manually. To do this, we figured that we needed to take advantage of the [PassengerMaxRequest][1] configuration in
Phusion Passenger. This parameter (which has a default value of 0) determines the maximum number of requests an application process
will process. After reaching this threshold, the process is restarted. Any memory that may have been tied up,
is then released back to the OS. If you have a run-away process with a memory leak, this restart at the very least
ensures that memory is released back to the OS and the process is restarted without users being impacted.

It is important to note, as the documentation notes, the better parameter to use for this optimization would be the [PassengerMemoryLimit][2].
Also set to 0 by default, when specified this requires passenger to restart the application process when a certain memory threshold is reached.
Note that this parameter is only available in Passenger Enterprise and not the open source version.

Immediately we did this we noticed a big difference. The total private dirty RSS taken up by passenger processes was now peaking at around 3.5GB.
This is after a period where it would spike to consume all of 16GB a server.

[1]: https://www.phusionpassenger.com/library/config/apache/reference/#passengermaxrequests
[2]: https://www.phusionpassenger.com/library/config/apache/reference/#passengermemorylimit
