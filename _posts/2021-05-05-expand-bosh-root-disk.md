---
layout: post
title: "How to Expand the Root Disk on a BOSH Director in AWS"
date: 2018-08-22
---

![map](https://raw.githubusercontent.com/cweibel/ghost_blog_pics/master/blake-cheek-WP51CpHSxYU-unsplash-2.jpg)



Photo by [Blake Cheek](https://unsplash.com/@blakecheekk?utm_source=unsplash&utm_medium=referral&utm_content=creditCopyText) on [Unsplash](https://unsplash.com/s/photos/uri?utm_source=unsplash&utm_medium=referral&utm_content=creditCopyText)


BOSH Stemcells utilize a 3GB ephemeral disk for the root partition.  This has traditionally been a forcing function for operators and BOSH release maintainers to make sure to properly place there files to a set of standards which do not include writing to the root partition.

Not everyone follows these recommendations.  These are their stories.

## Deploying the BOSH Director

Each of BOSH Director is maintained via the `bosh` v2 CLI.  In many organizations this is done with a command similar to:

```
bosh create-env manifest.yml --state=manifest-state.json
```

Depending on what is in the configuration of your `manifest.yml` this will either end with success or tears.


## Debugging - Out of Disk During `create-env`

This happens when `os-conf` release is (mis)used with `apt-get install` or BOSH releases are used which don't put their files into `/var/vcap/` are added.  Either during the compilation phase or during pre-start the root partition fills up since it is only 3GB.

To avoid this problem, during the `create-env` once the new VM is created and packages start to compile, use AWS Console to select the new EC2 instance and navigate to the volumes.  Select the 3GB disk, click `Modify` and increase the disk to 20GB.

Now SSH onto the new BOSH Director and run:

```
sudo -i   # c1oudc0w at prompt
growpart /dev/xvda 1
resize2fs /dev/xvda1
```

After running a quick `df -h` you can verify that the root partition has been expanded to be 20GB of usable disk.

## Does This Persist Across Deploys?

Nope.  You'll need to do this every time you recreate the vm associated with the BOSH Director.  Think of it as motivation to fix your BOSH releases!
