---
layout: post
title: "Measuring AWS RDS production downtime"
excerpt: "Getting comfortable and experienced with the inevitable RDS downtime
when modifying instance types"
tags: [rds, aws, databases, ops]
comments: true
---

> *TL;DR* RDS downtime, scheduled or not, is reduced to around 2 minutes using an 
automatic failover mechanism available for multi-AZ deployments. 
At times, replica instances attached to the master node, if any, can loose the
master after failover: contacting AWS support would solve the issue (they
ackwnoledge it).
It is important to test all these mechanisms before RDS instance modification, 
in order to be able supervise the fail over procedure and accurately predict production downtime.
Using a single-AZ deployment, outage due to resizing can last much longer (~ XX minutes).
TODO: test restoring from snapshot or modifying up and down once.


So the time has come when the core of your infrastructure, your database
instances, have to be resized. 

Unlike other AWS products, it is not possible to
perform such operation seamlessly, i.e. avoiding downtime.
The reason for this is the fact that 
oElastic beanstalk has (put links) rolling deployment, rolling configuration update and
CNAME swap -> they all give no downtime
TODO Explain why it is impossible to do it
without downtime, such as Elastic Beanstalk rolling updates or batches
deployment (where the ELB end point acts as permanent reference, and instances 
are stateless, i.e. application is without state, so it does not matter how many
are running concurrently)


## What AWS says

[From AWS documentation:](http://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/Concepts.MultiAZ.html)

In the event of a planned or unplanned outage of your DB instance, Amazon RDS
automatically switches to a standby replica in another Availability Zone if you
have enabled Multi-AZ. The time it takes for the failover to complete depends on
the database activity and other conditions at the time the primary DB instance
became unavailable. Failover times are typically 60-120 seconds. However, large
transactions or a lengthy recovery process can increase failover time. When the
failover is complete, it can take additional time for the RDS console UI to
reflect the new Availability Zone.
The failover mechanism automatically changes the DNS record of the DB instance
to point to the standby DB instance. 


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

We can then correlate our console action and the connectivity pattern to draw
some conclusions.  

We can evince that, in a multi-AZ deployment, instance modifications are not
applied to the primary instance, but rather to the standby instance, most
probably shutting down the RDS virtual machine and restarting it allocating the
modified resources.

Once the standby instance is up and running, and more importantly replicating
from the primary, the primary instance can be marked as unhealthy, triggering 
the fail over process. This, as doc says, can depend on the database state, but will end up leaving
the clients without access to the database. At this point, the system is ready 
to promote the newly sized standby instance as the new primary. 
Assuming that replication of the final state of failing/ex-primary instance to the ex-standby instance has
been successful, RDS then changes the DNS endpoint to the promoted instance, which 
starts to speak to the clients. 

We repeated the whole process for a few times, resizing instances of different
size, and we can confirm that the downtime implied lasts on average a little bit
more than 2 minutes.

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

## The not so cool things

TODO


- console takes a lot
Console takes a long time, much longer than 2 minutes we measured.
- restoring from single-AZ
- Replica problem 


## Single-AZ

## Conclusions

TODO
Handle with care! DB is core and as we said comparing to Beanstalk, difficult to 
treat and scale (it is the STATE of your business, everything else is stateless,
a cache or software)  as we compared it with Elastic Beanstalk
Test Test Test. It is cheap and can make things less stressfullmake things less
stressful


