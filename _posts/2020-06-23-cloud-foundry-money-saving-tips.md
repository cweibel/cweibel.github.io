---
layout: post
title: "Cloud Foundry Money Saving Tips"
date: 2020-06-23
---

![pic](https://raw.githubusercontent.com/cweibel/ghost_blog_pics/master/michael-longmire-lhltMGdohc8-unsplash-2.jpg)

Photo by [Michael Longmire](https://unsplash.com/@f7photo?utm_source=unsplash&utm_medium=referral&utm_content=creditCopyText) on [Unsplash](https://unsplash.com/s/photos/quokka?utm_source=unsplash&utm_medium=referral&utm_content=creditCopyText)

Companies are looking for ways to cut costs and keep staff while offering the same level of service to their developers and customers. Hopefully, the suggestions below will help you identify a few opportunities to save money!

There are a few ways operators can save money on Cloud Foundry deployments. There are pros and cons to each approach, so please understand these before deciding to act on any suggestions.

## Right-Size Instance Types

Using Prometheus dashboards, the amount of CPU, memory, and disk can be monitored for each virtual machine. Over time the usage patterns of each of these resources can be graphed to show which is over and under provisioned.


If you deployed to GCP, "Custom Machine Types" could be leveraged in which the vCPU and RAM can be configured separately. They also offer up suggestions on scaling instance types to different sizes after they've pulled instance statistics for a period of time.

Pros:

 - You can keep the control plane VMs "lean and mean" using the VM instance type which closely matches the actual usage needs.
 - You'll need Prometheus or a similar tool to scrape resource metrics which should be deployed anyway to monitor the deployments.

Cons:

 - This is not a "Day 1" configuration as the usage patterns of each instance group VM type will take a bit of time to establish.
 - Changes will need to be done in the Cloud Config for the BOSH director to pick up on and the change rolled out during the following deployment.

## Non HA Sandbox

Sandbox does not need to be highly available since it is only needed when testing upgrades. It should however attempt to match the instance types declared in the Cloud Config to match higher environments.

There is an ops file named `scale-to-one-az.yml` which scales the environment to a single AZ and one VM per instance group. This ops file is included and maintained in cf-deployment.

"Small Form Factor" or other types of deployments for which components are co-located on a reduced number VMs is not supported natively with ops files in cf-deployment and is not recommended since it would be an out-of-band modification which would need constant testing with each new release of cf-deployment. The feature is supported and maintained with versions of PCF and Genesis.

Pros:

 - Keeps the number of VMs down to an officially supported minimal configuration and thus the cost to a minimum.

Cons:

 - Not many. You obviously can't test HA scenarios.
 - May need to provide an override to the cell counts if there are more than a few trivial apps which need to be tested during upgrades.

## Use of Spot Instances

Spot Instances on AWS are regular but excess capacity EC2 instances which are auctioned off at a discounted rate. At any time, with just a few minutes notice, AWS can take the instance(s) away from you to sell to someone who has bid a higher price if the region is at capacity.

This solution leverages the power of BOSH to recreate VMs that have gone unresponsive after a period of time. The BOSH CPI for AWS can be configured with a bid price to provision VMs. The BOSH CPI can also be configured to fall back to On Demand EC2 Instances if the spot instances cannot be provisioned.

Pros:

 - Spot Instances can be as much as 90% off "On-Demand" EC2 pricing
 - The BOSH Director AWS CPI already supports Spot Instance usage
 - Potentially an excellent way for Sandbox or other POC environments to be provisioned cheaply
 - Potentially for Dev: configured per VM type so you can have the control plane work on regular On Demand EC2 instances while cell vm types could be on Spot Instances.

Cons:

 - Some regions are busier than other resulting in Spot Instance movement being more volatile
 - Don't use this for the BOSH Director itself. Chicken, meet egg.
 - Requires coordination with the billing team between ops and infrastructure groups to make sure Spot Instances are an available option.

## Pipeline Starting and Stopping Sandbox

It is potentially possible to spin down environments like Sandbox over nights/weekends. To do so, you would want to do the spin down and spin up in the following order:

 - Stop monit or shutdown the BOSH Director
 - Stop the instances for the desired deployments the BOSH Director was  - managing, like Cloud Foundry as the IaaS layer
 - Save Money
 - Start the instances at the IaaS layer
 - Start the BOSH Director

Pros:

 - You won't be paying for VMs no one is using at night or on weekends. AWS does not charge for stopped instances. They do however continue to charge for EBS volumes attached to the stopped instances.

Cons:

 - Many public clouds, like AWS, will recycle ephemeral volumes after a period of time meaning your VMs when turned back on will not have the necessary packages and jobs on the VMs and will require a `bosh recreate` to bring back up.
 - If you leverage Spot Instances you may not be saving enough money to make the effort worth it.
 - You'll need to write and maintain the IaaS script to spin the resources up and down
 - If the team in other time zones winds up leveraging the Sandbox environments there may not be enough offset between the teams to warrant a scheduled spin down/up.
