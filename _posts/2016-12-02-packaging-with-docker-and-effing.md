---
layout: post
title: "Packaging software with Docker and Effing"
image:
  feature: cresta.jpg
  credit: Richelmo Giupponi
excerpt: "How we can use docker and the Effing Package Management to greatly simplify, speed and account the packaging process"
tags: [ec2, aws, docker, effing]
comments: true
---

Using pre-packaged software is the quickest and most robust way to provision instances.
However, building software packages can be tedious as 1) it must be done on a clone of the instances to be provisioned and 2) it requires specific knowledge on the type of package/platform that has to be created. Also, 3) a repeatable/automatic workflow might be difficult to create.
<BR>
<BR>

In this post, we show how we can use [docker](https://www.docker.com/) and the [Effing Package Manager](https://github.com/jordansissel/fpm) to greatly simplify, speed and account the packaging process.
<BR>
<BR>


The idea is to create a base Docker image for each OS we need packages for. Additionally, the Effing Package Management (fpm) tool, used to build packages, is included in the base image.
<BR>
<BR>

In effing’s own words…


{% highlight bash %}
...
The goal of fpm is to make it easy and quick to build packages such as rpms, debs, OSX packages, etc.
...
It should be easy to say "here's my install dir and here's some dependencies; please make a package"
...
{% endhighlight %}

With some practice, it certainly achieves its goal.

## Base images

You start from a public image of the chosen target OS/architecture. In our case, we use the public image for the AWS Linux AMI and we season it with all the dependencies we require to build software.
As more software packages are built with this method, the number of dependencies in the docker base image will of course increase, but it is likely you will not have to rebuild the base image very often.


{% highlight bash %}
FROM amazonlinux
MAINTAINER Giovanni Giupponi "mymail@gmail.com"

RUN yum -y update && \
    yum -y install ruby23.x86_64 ruby-devel.noarch \
                   git wget tar bzip2              \
                   perl-App-cpanminus.noarch       \
                   openssl-devel libpcap-devel     \
                   pcre-devel zlib-devel

RUN yum -y group install "Development Tools"

RUN gem  install fpm
{% endhighlight %}

You can rearrange the docker layers creation in order to optimize the docker build process. In any case, after having empowered the base instance
to compile the target software, the effing package management tool is installed (last line).

## The packages directory 

We create a directory where to store the base dockerfiles and one subdirectory for each software package, containing accessory files/scripts.


{% highlight bash %}
$ ls packages
README.md
Dockerfile.packetizer_amazonlinux_base
Dockerfile.packetizer_ubuntu_14.04_base
collectd
ossec
rubies
snort
{% endhighlight %}

This directory can be versioned in git, for example in your infrastructure repository.



## Example: collectd

For each package, we typically define a Dockerfile, an entrypoint and a build script.


{% highlight bash %}
$ ls packages/collectd/amazonlinux
Dockerfile.packetizer_amazonlinux_collectd build.sh entrypoint.sh
{% endhighlight %}

The Dockerfile is responsible for the tailored compilation and installation of target software


{% highlight bash %}
cat Dockerfile.packetizer_amazonlinux_collectd
FROM packetizer_amazonlinux_base
MAINTAINER Giovanni Giupponi "mymail@gmail.com"

ENV COLLECTD collectd-5.6.1
ENV PACKAGE_DIR /opt/package

RUN wget https://storage.googleapis.com/collectd-tarballs/$COLLECTD.tar.bz2 -O-
| tar -xj
WORKDIR $COLLECTD
RUN ./configure --sysconfdir=/etc --exec-prefix=/usr

RUN make DESTDIR=$PACKAGE_DIR all install
WORKDIR $PACKAGE_DIR

COPY ./entrypoint.sh ./entrypoint.sh
RUN chmod 755 ./entrypoint.sh
ENTRYPOINT ["./entrypoint.sh"]
{% endhighlight %}

The entrypoint, given a docker image built from the Dockerfile, is responsible to package the software using fpm. Creating the package while the container is running permits to mount local filesystem to ….


{% highlight bash %}
#!/bin/bash
PACKAGE_NAME=seq_collectd_5.6.1_aws_linux.rpm
fpm -s dir -t rpm -C .                    \
  -p /tmp/$PACKAGE_NAME                   \
  --name sequra_collectd_5.6.1_aws_linux  \
  etc/ opt/ usr/
{% endhighlight %}

Each package is then created launching the build.sh script, which creates the image and then runs it.


{% highlight bash %}
#!/bin/bash
docker build -t packetizer_amazonlinux_collectd -f Dockerfile.packetizer_amazonlinux_collectd  .
docker run --rm -v $PWD:/tmp packetizer_amazonlinux_collectd
{% endhighlight %}

When build.sh finishes, you can find the package in the current directory.


{% highlight bash %}
$ ls
Dockerfile.packetizer_amazonlinux_collectd build.sh entrypoint.sh seq_collectd_5.6.1_aws_linux.rpm
{% endhighlight %}


## Conclusions

In this post, we show how we can avoid some headaches and save time building packages with Docker the Effing Package Management.
Packages assembly is simplified by fpm, and docker allows to build them locally, without the need to spin up a remote clone instance, saving time. Also, versioning workflow files in git increases accountability and flexibility.
<BR>
<BR>

Generated packages can then be put into a private mirror repo to be consumed by configuration management systems. As an alternative, they can also be uploaded to S3 to be wget-ed and localinstall-ed during provisioning.
