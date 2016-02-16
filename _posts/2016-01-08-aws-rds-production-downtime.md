---
layout: post
title: "Testing AWS RDS failover downtime"
image:
  feature: diavolo.jpg
  credit: Richelmo Giupponi
excerpt: "Getting comfortable and experienced with the inevitable RDS downtime
when modifying instance types/sizes"
tags: [rds, aws, databases, ops]
comments: true
---

> *TL;DR* The MultiAZ RDS automatic failover mechanism is measured.
RDS downtime, scheduled or not, is confirmed to be around 2 minutes, as 
per AWS documentation, while singleAZ downtime results to be much longer (~ 10
mins).
Uncommon stuck-read-replica problem is reported and debugged. Finally, AWS
management console reflects changes in DB state with a noticeable latency.



So the time has come when the core of your infrastructure, your database
instances, have to be resized. Unlike other AWS products, it is not possible to
perform such operation without downtime, as the database is by definition the single,
stateful source of truth and therefore zero-downtime procedures such as Elastic
Beanstalk [rolling
deployments](http://docs.aws.amazon.com/elasticbeanstalk/latest/dg/using-features.rolling-version-deploy.html)
and [rolling configuration
updates](http://docs.aws.amazon.com/elasticbeanstalk/latest/dg/using-features.rollingupdates.html)
cannot be implemented.

As a consequence, in order to be able to supervise without panic the failover
procedure and accurately predict production downtime, it is important to
simulate and time the whole process.

## What AWS says

[From AWS documentation:](http://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/Concepts.MultiAZ.html)

_In the event of a planned or unplanned outage of your DB instance, Amazon RDS
automatically switches to a standby replica in another Availability Zone if you
have enabled Multi-AZ. The time it takes for the failover to complete depends on
the database activity and other conditions at the time the primary DB instance
became unavailable. Failover times are typically 60-120 seconds. However, large
transactions or a lengthy recovery process can increase failover time. When the
failover is complete, it can take additional time for the RDS console UI to
reflect the new Availability Zone.
The failover mechanism automatically changes the DNS record of the DB instance
to point to the standby DB instance._

Sweet. We then want to induce a failover in order to be prepared to what it will
happen in production. We start by restoring a MultiAZ deployment from a
snapshot.  Little or no traffic at all would stream in and out of the testing
DB, but no important difference was detected between test and production
downtime during failover (data below refer to the actual production resizing procedure).

## Cool in action 

We setup a naive TCP endpoint check to monitor the health of the RDS endpoint

{% highlight bash %}
date
  while [ 1 ]
  do
    nc -vvvz xxx.yyy.us-east-1.rds.amazonaws.com 3306
    sleep 5
    date
  done
date
{% endhighlight %}

We then modify the instance using the AWS console (selecting Apply
Now) and start the process. The database instance state in the AWS console
changes in a little time to Modifying...

We seat back and watch

{% highlight bash %}
Thu Aug 27 13:04:12 CEST 2015
Connection to xxx.yyy.us-east-1.rds.amazonaws.com 3306 port
[tcp/mysql] succeeded!
Thu Aug 27 13:04:17 CEST 2015
Connection to xxx.yyy.us-east-1.rds.amazonaws.com 3306 port
[tcp/mysql] succeeded!
{% endhighlight %}

We can see that the RDS endpoint still replies.
Another simple script, not included here, also monitors DB health at the application
level  (SELECT NOW()), resulting in the same behaviour described here.
{% highlight bash %}
Thu Aug 27 13:04:22 CEST 2015
Connection to xxx.yyy.us-east-1.rds.amazonaws.com 3306 port
[tcp/mysql] succeeded!
Thu Aug 27 13:04:28 CEST 2015
Connection to xxx.yyy.us-east-1.rds.amazonaws.com 3306 port
[tcp/mysql] succeeded!
{% endhighlight %}
...
After more than 6 minutes the TCP check fails...
...
{% highlight bash %}
Thu Aug 27 13:11:43 CEST 2015
nc: connect to xxx.yyy.us-east-1.rds.amazonaws.com port
3306 (tcp) failed: Connection refused
Thu Aug 27 13:11:48 CEST 2015
nc: connect to xxx.yyy.us-east-1.rds.amazonaws.com port
3306 (tcp) failed: Connection timed out
{% endhighlight %}
Finally,  2:13 minutes after the first downtime poll, we regain connectivity (at 
application level too).
{% highlight bash %}
Thu Aug 27 13:14:01 CEST 2015
Connection to xxx.yyy.us-east-1.rds.amazonaws.com 3306 port
[tcp/mysql] succeeded!
Thu Aug 27 13:14:06 CEST 2015
Connection to xxx.yyy.us-east-1.rds.amazonaws.com 3306 port
[tcp/mysql] succeeded!
{% endhighlight %}

The six minutes before endpoint failure suggest that, in a multi-AZ deployment,
instance modifications are not applied to the primary instance, but rather to
the standby instance, most probably shutting down the RDS virtual machine and
restarting it allocating the modified resources.

Once the standby instance is up and running, and more importantly replicating
from the primary, the primary instance can be marked as unhealthy, triggering 
the failover process. As the documentation says, failover duration can depend on
the database state, but in any case it would leave the clients without access to
the database. At this point, the system is ready to promote the newly
typed/sized standby instance as the new primary.  Assuming that replication of
the final state of failing/ex-primary instance to the ex-standby instance has
been successful, RDS then changes the DNS endpoint to the promoted instance,
which starts to speak to the reconnecting clients.

{% highlight bash %}
[giovanni@giovanni-kerad ~/KeradCode/goldenmanager.com$for I in $(seq 1 360); do date; dig +short
xxx.yyy.us-east-1.rds.amazonaws.com; echo; sleep 10; done
Wed Sep  9 13:03:33 CEST 2015
ec2-11-22-33-44.compute-1.amazonaws.com.
11.22.33.44
{% endhighlight %}
...
{% highlight bash %}
Wed Sep  9 13:14:49 CEST 2015
ec2-11-22-33-44.compute-1.amazonaws.com.
11.22.33.44

Wed Sep  9 13:14:59 CEST 2015
ec2-11-22-33-44.compute-1.amazonaws.com.
44.33.22.11
{% endhighlight %}

We repeated the whole process a few times, resizing instances of different
type/size, and we can confirm that the downtime lasts on average a little bit
more than 2 minutes.


## The not so cool things

- During the tests, it was manifest that the AWS management console lagged behind
  the truth state of the instance. Most notably, after up to ~ 10 minutes after
  the failover completed successfully, i.e. after the clients could query the
  endpoint of new instance, the console would still indicate the instance as
  'Modifying'

- At times, replica instances attached to the master node, if any, can loose the
  master after failover. It only happened once in around ten test/live
  resizing, actually during the production resize.  Contacting AWS support solved
  the issue. However, [this is a known issue](https://aws.amazon.com/rds/faqs/)
  and a workaround is suggested to minimize the likelihood of a lost replica, at
  the cost of probable performance penalty.

- Quite obviously, modifying a SingleAZ instance would take much longer, summing
  up the time to modify the secundary master replica and then reboot the system.
  A rough estimate is of about 10 minutes.


## Conclusions

AWS RDS offers a great and seamless service when it comes to high availability:
the possibility of a one-click, automatic and relatively short downtime to recover from
failure or scaling up/down of database instances is extremely appealing to those that have
had to put up with rescaling database servers, replication failures, etc. No
more long cold hours at night in data centers.
