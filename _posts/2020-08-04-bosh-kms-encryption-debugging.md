---
layout: post
title: "BOSH + KMS Disk Encryption Debugging"
date: 2020-08-14
---

![beaver](https://raw.githubusercontent.com/cweibel/ghost_blog_pics/master/markus-winkler-cS2eQHB7wE4-unsplash-2.jpg)

Photo by [Markus Winkler](https://unsplash.com/@markuswinkler?utm_source=unsplash&utm_medium=referral&utm_content=creditCopyText) on [Unsplash](https://unsplash.com/s/photos/quokka?utm_source=unsplash&utm_medium=referral&utm_content=creditCopyText)

Today, we made an attempt to encrypt a persistent volume in a BOSH deployment using a KMS key and experienced an odd error which was not obvious on how to debug.

Hopefully, after reading this you'll be able to identify and quickly fix your problem if you run into this scenario!

## Scenario

We've been tasked with making sure that a Postgres BOSH deployment on AWS has a KMS encrypted disk. In it's current state the deployment does not utilize an encrypted disk so we added a configuration to the manifest which looks like:

```
...
jobs:
- instances: 1
  name: postgres
  persistent_disk_pool: database_z1

disk_pools:
- cloud_properties:
    kms_key_arn: arn:aws:kms:us-west-2:86753090000:key/7f134ad7-1da5-40d7-b7a1-445f099b3203
    type: standard
  disk_size: 1024
  name: database_z1
...
```

The `kms_key_arn` property was added to the existing manifest with a KMS key which was manually created.

## Expected Behavior

Like any changes to a persistent disk, BOSH will make changes in this order:

 - Create the new volume
 - `monit stop all` jobs on the VM
 - Attach the new volume along with the old volume
 - Copy all the files from the old volume to the new volume
 - Detach the old volume
 - `monit start all` the jobs on the VM
 - Delete the old volume

In our case the new disk was expected to be encrypted and the old disk unencrypted.

## Deployment Attempt

Having added the parameters to add disk encryption the BOSH deploy is initiated but results in a non obvious CPI error:

```
bosh -e my-director -d postgres deploy postgres.yml

...
00:15:52 | Deprecation: Ignoring cloud config. Manifest contains 'networks' section.
00:15:52 | Preparing deployment: Preparing deployment (00:00:24)
00:17:22 | Preparing package compilation: Finding packages to compile (00:00:00)
00:17:33 | Updating instance postgres: postgres/47ee7f36-7bbe-4f31-b2e7-cfa7aabae804 (0) (canary) (00:00:22)
L Error: Unknown CPI error 'Unknown' with message 'The volume 'vol-b2c6bd9ea3bc3ef6' does not exist.' in 'create_disk' CPI method
00:17:55 | Error: Unknown CPI error 'Unknown' with message 'The volume 'vol-b2c6bd9ea3bc3ef6' does not exist.' in 'create_disk' CPI method
Started Thu Aug 13 00:15:52 UTC 2020
Finished Thu Aug 13 00:17:55 UTC 2020
Duration 00:02:03
Task 2700 error
Updating deployment:
Expected task '2700' to succeed but state is 'error'
Exit code 1
```

That's not good. The Postgres VM still has its old unencrypted disk attached so no loss of data or unavailability has occurred. However, we aren't using an encrypted disk still.

## Symptoms in the BOSH Logs

The problem reveals itself a bit in the BOSH task logs with the `--debug` option added:

```
bosh task 2700 --debug

...
few thousand lines of logs, then:
...
[Aws::EC2::Client 200 0.242032 0 retries] create_volume(size:1,availability_zone:"us-west-2a",volume_type:"standard",encrypted:true,kms_key_id:"arn:aws:kms:us-west-2:86753090000:key/7f134ad7-1da5-40d7-b7a1-445f099b3203")
Creating volume 'vol-b2c6bd9ea3bc3ef6'
[Aws::EC2::Client 200 0.088644 0 retries] describe_volumes(volume_ids:["vol-b2c6bd9ea3bc3ef6"])
Waiting for vol-b2c6bd9ea3bc3ef6 to be available, retrying in 2 seconds (1/54)
[Aws::EC2::Client 400 0.128502 0 retries] describe_volumes(volume_ids:["vol-b2c6bd9ea3bc3ef6"]) Aws::EC2::Errors::InvalidVolumeNotFound The volume 'vol-b2c6bd9ea3bc3ef6' does not exist.
Finished create_disk in 9.59 seconds
...
```

So, from that output you can see the BOSH CPI attempts to create the volume with various parameters (200 response), the first attempt to describe the volume (200 response), but isn't ready and the second attempt results in a 400 response.

Odd.

## Recreating the Issue with the AWS CLI

Trying to create a volume with the AWS CLI is equally head scratching. The `create-volume` command will run with no issues, but attempting to get the status or retrieve information about the volume fails because it is already terminated:

```
$ aws ec2 create-volume --availability-zone us-west-2a \
--volume-type standard \
--size 1 \
--encrypted \
--kms-key-id arn:aws:kms:us-west-2:86753090000:key/7f134ad7-1da5-40d7-b7a1-445f099b3203

{
    "AvailabilityZone": "us-west-2a",
    "CreateTime": "2020-08-13T19:57:09+00:00",
    "Encrypted": true,
    "KmsKeyId": "arn:aws:kms:us-west-2:86753090000:key/7f134ad7-1da5-40d7-b7a1-445f099b3203",
    "Size": 1,
    "SnapshotId": "",
    "State": "creating",
    "VolumeId": "vol-0f521ea98951011cf",
    "Tags": [],
    "VolumeType": "standard",
    "MultiAttachEnabled": false
}

$ aws ec2 describe-volume-status --volume-ids vol-0f521ea98951011cf

An error occurred (InvalidVolume.NotFound) when calling the DescribeVolumeStatus operation: 
The volume 'vol-0f521ea98951011cf' does not exist.
```

What's going on here?

Let's see if that KMS key exists:

```
aws kms list-keys | grep 445f099b3203
```

Nope, a clue!

## Solution

Make sure the KMS key exists before attempting to create encrypted persistent disks with BOSH, my dear Watson!

For each new `kms_key_arn` in the manifest below:

```
disk_pools:
- cloud_properties:
    kms_key_arn: arn:aws:kms:us-west-2:86753090000:key/7f134ad7-1da5-40d7-b7a1-445f099b3203
```

Make sure that it exists via the AWS CLI:

```
aws kms list-keys | grep "arn:aws:kms:us-west-2:86753090000:key/7f134ad7-1da5-40d7-b7a1-445f099b3203"
```

One final hint: make sure the the AWS access key and secret (or instance profile) used to configure the BOSH CPI are also the same credentials used when configuring the AWS CLI so that you know you are using the same IAM Policy.

Happy BOSHing!

