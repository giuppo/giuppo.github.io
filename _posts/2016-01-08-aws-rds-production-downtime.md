---
layout: post
title: "Measuring AWS RDS production downtime"
excerpt: "Getting comfortable and experienced with the inevitable RDS downtime when modifying instances"
tags: [rds, databases, rds, aws]
comments: true
---

RDS is arguably the least seamless service of the AWS platform. The act of

While . This is
also probably the reason why 

> TL;DR When resizing AWS RDS instances, it is better to practice beforehand
 what will happpen, to measure in order to plan production downtime
 communication etc. AWS promise of a few minutes quantifies better 2-3 minutes.
 When a replica is attached, things can go pearshaped and support has to be
poked (it is a known issue)


## What AWS says


## How it actually happens

{% highlight bash %}
date
  while [ 1 ]
  do
    nc -vvvz strike-virginia.c3ryd7sklhx3.us-east-1.rds.amazonaws.com 3306
    sleep 5
    date
  done
date
{% endhighlight %}

{% highlight bash %}
[giovanni@giovanni-kerad ~/KeradCode/goldenmanager.com$for I in $(seq 1 360); do date; dig +short
strike-virginia.c3ryd7sklhx3.us-east-1.rds.amazonaws.com; echo; sleep 10; done
Wed Sep  9 13:03:33 CEST 2015
ec2-52-20-165-239.compute-1.amazonaws.com.
52.20.165.239
{% endhighlight %}

DNS switch

{% highlight bash %}
Wed Sep  9 13:14:49 CEST 2015
ec2-52-20-165-239.compute-1.amazonaws.com.
52.20.165.239

Wed Sep  9 13:14:59 CEST 2015
ec2-52-22-6-238.compute-1.amazonaws.com.
52.22.6.238

Wed Sep  9 13:15:09 CEST 2015
ec2-52-22-6-238.compute-1.amazonaws.com.
52.22.6.238
{% endhighlight %}
Actually salvia consectetur, hoodie duis lomo YOLO sunt sriracha. Aute pop-up brunch farm-to-table odio, salvia irure occaecat. Sriracha small batch literally skateboard. Echo Park nihil hoodie, aliquip forage artisan laboris. Trust fund reprehenderit nulla locavore. Stumptown raw denim kitsch, keffiyeh nulla twee dreamcatcher fanny pack ullamco 90's pop-up est culpa farm-to-table. Selfies 8-bit do pug odio.

<figure class="half">
    <a href="/images/RDS_downsize_tiempo_master.png"><img
src="/images/RDS_downsize_tiempo_master.png"></a>
    <a href="/images/RDS_downsize_tiempo_slave.png"><img
src="/images/RDS_downsize_tiempo_slave.png"></a>
    <figcaption>Caption describing these two images.</figcaption>
</figure>


{% highlight bash %}
Thu Aug 27 13:04:12 CEST 2015
Connection to strike-virginia.c3ryd7sklhx3.us-east-1.rds.amazonaws.com 3306 port
[tcp/mysql] succeeded!
Thu Aug 27 13:04:17 CEST 2015
Connection to strike-virginia.c3ryd7sklhx3.us-east-1.rds.amazonaws.com 3306 port
[tcp/mysql] succeeded!
{% endhighlight %}
RUN THE DOWNSCALE COMMAND...DB Instance state in AWS console changes in a little
time to Modifying
{% highlight bash %}
Thu Aug 27 13:04:22 CEST 2015
Connection to strike-virginia.c3ryd7sklhx3.us-east-1.rds.amazonaws.com 3306 port
[tcp/mysql] succeeded!
Thu Aug 27 13:04:28 CEST 2015
Connection to strike-virginia.c3ryd7sklhx3.us-east-1.rds.amazonaws.com 3306 port
[tcp/mysql] succeeded!
{% endhighlight %}
…. ~ 6 minutes afterwards….refused
{% highlight bash %}
Thu Aug 27 13:11:43 CEST 2015
nc: connect to strike-virginia.c3ryd7sklhx3.us-east-1.rds.amazonaws.com port
3306 (tcp) failed: Connection refused
Thu Aug 27 13:11:48 CEST 2015
nc: connect to strike-virginia.c3ryd7sklhx3.us-east-1.rds.amazonaws.com port
3306 (tcp) failed: Connection timed out
{% endhighlight %}
…. 2:13 minutes afterwards….connected again.
{% highlight bash %}
Thu Aug 27 13:14:01 CEST 2015
Connection to strike-virginia.c3ryd7sklhx3.us-east-1.rds.amazonaws.com 3306 port
[tcp/mysql] succeeded!
Thu Aug 27 13:14:06 CEST 2015
Connection to strike-virginia.c3ryd7sklhx3.us-east-1.rds.amazonaws.com 3306 port
[tcp/mysql] succeeded!
{% endhighlight %}

### 

Maybe mention that when downscaling replica, which is obviously not MultiAZ, the
endpoint of course stays the same, connection is lost for some minutes, AND
IP of the replica instance would change (i.e. stop and start change IP, as in
EC2 with no Elastic IP).
