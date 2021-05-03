---
layout: post
title: "Using BOSH Tags for Cost Center Charge Backs on AWS"
date: 2020-08-12
---

![beaver](https://raw.githubusercontent.com/cweibel/ghost_blog_pics/master/devin-avery-B3u-SJbCy0U-unsplash-2.jpg)

Photo by [Devin Avery](https://unsplash.com/@devintavery?utm_source=unsplash&utm_medium=referral&utm_content=creditCopyText) on [Unsplash](https://unsplash.com/s/photos/filter?utm_source=unsplash&utm_medium=referral&utm_content=creditCopyText)


Scenario: You log into the AWS Console, look at the EC2 tab and wonder whether the hundreds of thousands of instances are being used, are still needed, and who owns them.

Your recovery from this situation is a slow process, but BOSH can help!

## BOSH Tags

BOSH deployed VM instances can be deployed with [tags](https://bosh.io/docs/manifest-v2/#tags). These are an arbitrary list of key/value combinations that can be attached to any VM which BOSH creates. These tags can be leveraged to help quickly and easily identify ownership of the resources created.

For example, a simple deployment can have the following tags added to it:

 - Team - The group or Business Unit who deployed the instance, using the value `Platform` for the example.
 - Environment - In the case where production, development and test are all mixed into the same account this value can help differentiate the relative importance of the VM, using the value `Test` for the example.
ChargeBackBU - The id of the cost center to charge back the costs associated with the VM, using the value `ABC123` for the example.
 - The full manifest looks like the following, more details on this particular deployment can be found at [https://starkandwayne.com/blog/hey-bosh-gimme-a-vm/](https://starkandwayne.com/blog/hey-bosh-gimme-a-vm/):

```
---
name: emptyvm

tags:
  Team:         Platform
  Environment:  Test
  ChargeBackBU: ABC123

stemcells:
- alias: default
  os: ubuntu-xenial
  version: latest

releases: []

update:
  canaries: 2
  max_in_flight: 1
  canary_watch_time: 5000-60000
  update_watch_time: 5000-60000

instance_groups:
- name: emptyvm
  azs: [z1]
  instances: 1
  jobs: []
  vm_type: default
  stemcell: default
  persistent_disk: 10240
  networks:
  - name: default
```


## Use the AWS CLI to List EC2 Instances by Tag

Now you can easily identify the EC2 instances associated with a tag.

In the example below, all of the instances which should be charged back to the cost center `ABC123` can be listed by:

```
aws ec2 describe-instances --filters "Name=tag:ChargeBackBU,Values=ABC123"  \
   --query Reservations[*].Instances[*].[InstanceId] --output text

i-026c888b8b239c8dd
i-0896ddc5d2c06eff8
i-04445fa23de114605
i-061b8549cc2f74a3c
i-08266e8e04f40164e
```

Use AWS Console to List EC2 Instances by Tag

The same data can be obtained in the AWS Console. In the console navigate to `EC2` > `Instances`, in the "Add Filter" dropdown select `Tag Keys` > `ChargeBackBU` and hit enter. The VMs associated with this tag and value will be displayed similar to:

![chargeback](https://github.com/cweibel/ghost_blog_pics/blob/master/chargeback.png?raw=true?chargeback.png)

## What's Next?

The example used shows how to add tags to a single deployment. To add the tags to ALL deployments add the tags to the BOSH Runtime Configuration. Add the following snippet to the runtime config:

```
tags:
  Team:         Platform
  Environment:  Test
  ChargeBackBU: ABC123
```

As you rollout future deployments (or redeploy existing ones) the tags will get added to the VMs automatically!