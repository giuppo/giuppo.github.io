---
layout: post
title: "Building an AWS AMI with encrypted root device"
image:
  feature: tracce.jpg
  credit: Richelmo Giupponi
excerpt: "PCI-DSS recommends to use machines with all the volumes encrypted in order to prevent data leakage due to disk loss/theft. An AWS solution."
tags: [ec2, aws, pci-dss, ops]
comments: true
---

There is not much to dispute about the importance of encrypting data.
<BR>
<BR>
Elastic Block Store (EBS) encryption offers a solution to encrypt EBS volumes, i.e. machine “disks”. Data residing on encrypted EBS volumes are assured to be encrypted at rest, while moving between the volume and the instance and also when dumped into a volume snapshot.
However, unlike other services such as S3 Server-Side Encryptions (SSE-S3, SSE-KMS, SSE-C) or Redshift clusters (immutably encrypted on-demand during nodes launch), **AWS does not provide any guideline on how to create a base AMI with all the EBS volumes encrypted**. This article describes the solution we have implemented @ Sequra.
<BR>
<BR>
When hardening machines, it is very recommended to use a base AMI with all the volumes encrypted in order to prevent data leakage due to disk loss/theft.
It is actually required in order to comply with Payment Card Industry Data Security Standard (PCI-DSS v3.2, requirement 3.4.1)
As the guidance puts it:

> > …Full disk encryption helps to protect data in the event of physical loss of a disk…


It is not about questioning [AWS data centers security processes](https://d0.awsstatic.com/whitepapers/aws-security-whitepaper.pdf): it is about doing our own part within the [AWS shared responsibility](http://cloudacademy.com/blog/aws-shared-responsibility-model-security/) model.
<BR>
<BR>
Unfortunately, the AWS Linux AMI comes with an unencrypted root volume which is derived from an unencrypted EBS snapshot. I could not find any
community/marketplace AMI with an encrypted root volume: in any case, I much prefer using the native AWS Linux AMI as a base for a security critical instance.

### HOW

In order to create an AMI that will boot with an encrypted root volume, the trick is to make an encrypted copy of a snapshot of the unencrypted volume which is used as a root volume by the AMI of choice.
The id of the unencrypted snapshot can be obtained inspecting the AMI description, or you can just spawn an AMI instance and then use the aws-cli to
create a snapshot of the root volume. The latter procedure takes more time, but it allows you to update packages before snapshotting: a little more of scripting, but definitely worth it.

I strip down a lot of ancillary code used to start, tag, wait, check, destroy, … transient artifacts: it is recommended to write your own custom and reusable library to interact with AWS, as in [kubernetes utils](https://github.com/kubernetes/kubernetes/blob/master/cluster/aws/util.sh).
It all boils down to

{% highlight bash %}
encrypted_snapshot_id=$(create_encrypted_snapshot $snapshot_id $region)
{% endhighlight %}

where

{% highlight bash %}
function create_encrypted_snapshot {
  local snapshot_id=$1 
  local region=$2
  aws ec2 copy-snapshot --source-snapshot-id $snapshot_id --source-region $region
    --destination-region $region --encrypted --query SnapshotId --output text
}
{% endhighlight %}

You then register an AMI using such snapshot as root volume.

{% highlight bash %}
image_id=$(create_ami_from_snapshot $encrypted_snapshot_id)
{% endhighlight %}

where

{% highlight bash %}
function create_ami_from_snapshot {
  local encrypted_snapshot_id=$1
  local region=$2
  local when=$(date +%s)
  aws ec2 register-image --name "AWS Linux Sequra PCI ami $when" \
    --description "AWS Linux with root device encrypted" \
    --region $region \
    --output text \
    --architecture x86_64 \
    --virtualization-type hvm \
    --root-device-name "/dev/xvda" \
    --block-device-mappings
    "[{\"DeviceName\":\"/dev/xvda\",\"Ebs\":{\"SnapshotId\":\"$encrypted_snapshot_id\"}}]"
}
{% endhighlight %}

Voilà!



