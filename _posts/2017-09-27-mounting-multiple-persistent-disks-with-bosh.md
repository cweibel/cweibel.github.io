---
layout: post
title: "Mounting Multiple Persistent Disks With BOSH"
date: 2017-09-29
---

![map](https://raw.githubusercontent.com/cweibel/ghost_blog_pics/master/steve-smith-_IovdZSUQBg-unsplash.jpg)



Photo by [Steve Smith](https://unsplash.com/@varrak?utm_source=unsplash&utm_medium=referral&utm_content=creditCopyText) on [Unsplash](https://unsplash.com/s/photos/uri?utm_source=unsplash&utm_medium=referral&utm_content=creditCopyText)


There are many use cases for needing multiple persistent disks to be mounted to a BOSH deployed VM. We are working on a [DC/OS BOSH Release](https://github.com/cloudfoundry-community/dcos-boshrelease) which requires multiple disks to be mounted to the `mesos-agent`servers, one for `/var/vcap/store` and one as a raw disk for Docker.

Before we dive into how this is done there are a couple of important bits:

 - The disks are mounted and a symlink is provided
 - The disks start as unformatted
 - None of the disks will be automatically be mounted to `/var/vcap/store`

Fear not, we will work around these last two issues.

Let's start!

## BOSH Manifest Changes

There are a couple of items required in a manifest for a job to leverage multiple disks:

 - A `consumes:` key with the list of mappings to persistent disks.
 - A `persistent_disks` array mapping of the consumed disks list to disk pool names
 - A `disk_pools` or `disk_type` array of disk sizes and names

Here is an example stub manifest defining two disks to be mounted, one called `portworx` (used as raw disk) and another called `store` (to be mounted to `/var/vcap/store`):

```
...
instance_groups:
- instances: 1
  jobs:
  - name: mesos-agent
    release: dcos
  - consumes:
      portworx:
        from: portworx      # <=== Maps to the `persistent_disks` name
      store:
        from: store         # <=== Maps to the `persistent_disks` name
    name: portworx
    release: dcos
  name: mesos-agent
  networks:
  - name: dcos_z1
  persistent_disks:
  - name: store
    type: 10GB               # <== Maps to `disk_pool` or `disk_types` name
  - name: portworx
    type: 100GB              # <== Maps to `disk_pool` or `disk_types` name
  resource_pool: default
...
```

The disk pools are defined as one of the two options below, notice that these map to the `persistent_disks` in the `mesos_agent` instance group above.

```
# Older BOSH v1 style disk pools:
disk_pools:
- disk_size: 10240
  name: 10GB
- disk_size: 100240
  name: 100GB

# Or for newer BOSH v2 disk descriptions:
disk_types:
- disk_size: 10240
  name: 10GB
- disk_size: 100240
  name: 100GB
```

At this point if the deployment manifest was deployed, two disks would be mounted on the `mesos-agent` vm in `/var/vcap/instance/disks` as:

```
lrwxrwxrwx 1 root root 9 Sep 29 14:12 portworx -> /dev/xvdg
lrwxrwxrwx 1 root root 9 Sep 29 14:12 store -> /dev/xvdf
```

But there is nothing mounted to `/var/vcap/store` and `/dev/xvdf` is not formatted yet. You can do this manually, however we modified the BOSH release with a pre-start script to do this work automatically for us as seen in the next section.

## BOSH Release Changes

Any BOSH Job which you would like to leverage multiple disks needs to format the disks. An [example pre-start script](https://github.com/cloudfoundry-community/dcos-boshrelease/blob/master/jobs/portworx/templates/bin/pre-start.erb) shows off the basics of checking if the filesystem exists and then symlinking a disk to `/var/vcap/store`:

```
#!/bin/bash
 
STORE_DIR=/var/vcap/store
STORE=/var/vcap/instance/disks/<%= link("store").p("name") %> 
if ! e2fsck -n $STORE    #Checks if the filesystem exists
then
  mkfs.ext4 -F $STORE   #...if not make the filesystem
fi

#  `/var/vcap/store` is not defined for you, you have to do it yourself:
if ! mountpoint -q $STORE_DIR  
then
  mkdir -p $STORE_DIR
  mount -t ext4 $STORE $STORE_DIR
fi
```

If we deploy the manifest with this modified release you will have a 10GB ext4 formatted drive symlinked to `/var/vcap/store`.

In the examples above the `portworx` disk was ignored because it is being presented in raw format to Docker to manage. If you wanted to format and use this disk as well we could have added the following to the pre-start script:

```
...
PORTWORX_DIR=/var/vcap/portworx
PORTWORX=/var/vcap/instance/disks/<%= link("portworx").p("name") %> 
if ! e2fsck -n $PORTWORX
then
  mkfs.ext4 -F $PORTWORX
fi
if ! mountpoint -q $PORTWORX_DIR  
then
  mkdir -p $PORTWORX_DIR
  mount -t ext4 $PORTWORX $PORTWORX_DIR
fi
```

This results in the second disk being with 100GB available at `/var/vcap/portworx` formatted as ext4.

## Retrieving the Device

In a BOSH release if you need the mapping of the device, you can `readlink`:

```
DISKPATH=$(readlink /var/vcap/instance/disks/<%= link("portworx").p("name") %>)
```

If you echoed `DISKPATH` you would get the value `/dev/xvdg`.

## Gotchas'n'stuff

The disks are not automatically deleted if the deployment is deleted, there is a 5 day grace period before the orphaned disks are cleaned up by BOSH. Verify your BOSH settings before relying on this feature.

Instead of embedding the prestart script in the DC/OS BOSH Release it may be more beneficial to create a separate `multidisk-boshrelease` and colocate it on the instance groups so it is easier to reuse. Look for an update on this!

Everyone deserves nice things, multiple disks via BOSH is a nice thing, be sure to share your tips on the greatness of BOSH!