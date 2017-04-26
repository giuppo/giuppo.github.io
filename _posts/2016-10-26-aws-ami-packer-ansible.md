---
layout: post
title: "Building AWS AMIs with Packer-Ansible: tips and tricks"
image:
  feature: cresta.jpg
  credit: Richelmo Giupponi
excerpt: "Some tips and tricks on how to use Packer and Ansible with the base AWS Linux AMI."
tags: [ec2, aws, ansible, packer]
comments: true
---

Continuous deploy with immutable infrastructure requires an automatic and robust workflow for building virtual machines. This article lists some tips and tricks on how to use [Packer](https://www.packer.io/) and [Ansible](https://www.ansible.com) with the base AWS Linux AMI.
Using [Packer](https://www.packer.io/) and [Ansible](https://www.ansible.io), a simple bash script “wrapper” can be responsible to build, provision and deploy AMIs into your AWS infrastructure.
<BR>
<BR>

Immutable infrastructure practices require to treat virtual machines as disposable tools, meant to be thrown away and recreated from scratch when we want/must change something (new application code, software upgrades, OS tweaking, …).
<BR>
<BR>

Packer is a tool to portably create machine images. It can build VM artifacts for different platforms such as AWS EC2, Azure, DigitalOcean, Docker, GCE, …
Code attached here refers to the cooking of AWS EC2 instances.
<BR>
<BR>

Creating virtual machines portably and à-la-carte sounds cool, but what is even cooler is that Packer can also be instructed to wrap your infrastructure provisioner of choice ( Ansible, Chef, Puppet, Salt and some others) while baking a machine.
Packer building logic and parameters are defined in a JSON template file: please refer to [intro](https://www.packer.io/intro/) and [docs](https://www.packer.io/docs/) pages for details.
<BR>
<BR>

In the following, we list tips and tricks adopted to automatically build and provision Sequra AWS machines based on [encrypted AMIs](https://engineering.sequra.es/2016/09/building-an-aws-ami-with-encrypted-root-device/).
<BR>
<BR>

## Wrap packer templates into bash scripts

The first trick that we suggest is to create a script that wraps both the JSON Packer template and the commands/variables necessary to execute the command, i.e.


{% highlight bash %}
###
# ... params management
###

AWS_ACCESS_KEY_ID=${AWS_ACCESS_KEY_ID:-'your_aws_access_key_id_here'}
AWS_SECRET_ACCESS_KEY=${AWS_SECRET_ACCESS_KEY:-'your_aws_access_secret_key_here'}

cat > $PACKER_TEMPLATE << PACKER
 ...
   ... Your Packer template, with bash and ENV variables to make it more flexible
 ...
PACKER

time packer build -var "aws_access_key=$AWS_ACCESS_KEY_ID"     \
                  -var "aws_secret_key=$AWS_SECRET_ACCESS_KEY" \
                  $PACKER_TEMPLATE

rm $PACKER_TEMPLATE
{% endhighlight %}

Putting AWS credentials in environment variables might not be the best solution, but it is not the focus here. Feel free to implement different ways to pass them to packer.

## Non-standard location of sftp-server command on AWS Linux AMIs

This tip can save you from setting debug options in Ansible and Packer to skim through logs in order to understand why you cannot provision an AWS Linux machine with the Packer test template found in the documentation.


{% highlight bash %}
...
  "provisioners": [
    {
      ...
      "sftp_command": "/usr/libexec/openssh/sftp-server -e"
      ...
    }
  ]
...
{% endhighlight %}

## Packer does not support Ansible remote_user in playbooks

Packer connects to instances to push and run Ansible code only with the user you specify in the builder of the packer template.
Full stop. No can do. Just try some other solution.


{% highlight bash %}
...
  "builders": [{
    ...
    "ssh_username": "ec2-user",
    ...
  }],
...
{% endhighlight %}

The same happens for ansible_user in configuration files, i.e. it is useless.
<BR>
<BR>

This is a [poorly documented](https://github.com/hashicorp/packer/issues/3828) packer “feature” that can cause headaches. It is actually [under discussion](https://github.com/hashicorp/packer/issues/3889) and the issue we opened seems to be taken as case study.

## ec2-user to non-root users

Another trick we implemented is due to the fact that Ansible 2.1 and above versions by default do not allow to become an unprivileged user from non-root users. There are [sound security reasons](http://docs.ansible.com/ansible/become.html#becoming-an-unprivileged-user) for this, but this is a pity as on AWS AMIs sudo-ed root users are called ec2-user (or ubuntu), and this security measure then impedes become_user tasks, a very common pattern when provisioning with Ansible.
In our case, we can use the workaround that Ansible people suggest to modify this default behaviour, i.e. we can put


{% highlight bash %}
allow_world_readable_tmpfiles = True
{% endhighlight %}

in your ansible configuration file.

## Conclusions

We have shown here that, with a bit of effort, it is possible to couple the great power of Ansible and Packer to build AWS Linux machines: all testing was done with version 2016.09.0, but the tips will probably apply to more versions. This can be used in a continuous delivery/deploy solution in order to allow immutable infrastructure pipelines, where phrases as ‘cap stage deploy’, ‘apt-get update’, … are not very welcome.



