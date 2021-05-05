---
layout: post
title: "Registering (Someone Else's) Service Broker in Cloud Foundry"
date: 2019-03-15
---

![map](https://raw.githubusercontent.com/cweibel/ghost_blog_pics/master/emmy-smith-LEjEst7lLfU-unsplash.jpg)



Photo by [Emmy Smith](https://unsplash.com/@emsmith?utm_source=unsplash&utm_medium=referral&utm_content=creditCopyText) on [Unsplash](https://unsplash.com/s/photos/uri?utm_source=unsplash&utm_medium=referral&utm_content=creditCopyText)

So once in a while I need to debug whether BOSH is wired up correctly to spin vms and I need a quick deployment manifest that isn't 4k+ lines long.  This is what I use assuming a cloud-config with `defaults` defined:

```
---
name: emptyvm

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

A quick `bosh -d emptyvm deploy emptyvm.yml` and I have a vm I can `bosh ssh` into.

Enjoy!