---
layout: post
title: "Detect etcd Split Brain in Cloud Foundry)"
date: 2016-10-06
---

![map](https://raw.githubusercontent.com/cweibel/ghost_blog_pics/master/steve-smith-R0GuHOq4AlQ-unsplash.jpg)



Photo by [Steve Smith](https://unsplash.com/@varrak?utm_source=unsplash&utm_medium=referral&utm_content=creditCopyText) on [Unsplash](https://unsplash.com/s/photos/uri?utm_source=unsplash&utm_medium=referral&utm_content=creditCopyText)

While upgrading one of the development environments we had a bad configuration of the etcd properties. This resulted in three etcd servers spinning up which each elected themselves as leader. To detect this condition look at the leader key on each etcd server.

In the scenario below there are three etcd servers named:

 - etcd_z1/0
 - etcd_z1/1
 - etcd_z2/0

We have an added layer of SSL certs for the etcd cluster, in properties `requires_ssl: true` is set so when we curl consul we have to provide the ca cert, client and key files.

## What to look for

Start by logging into one of the `hm9000` servers:

```
bosh ssh hm9000_z1/0
```

Switch to the `root` user and query the `/leader` and `/self` endpoints for each of the 3 etcd servers:

```
for etcd in etcd-{z1-0,z1-1,z2-1}; do
  for role in self leader; do
    echo -n "${etcd} ${role}: "
    curl -k \
      --cacert /var/vcap/jobs/hm9000/config/certs/etcd_ca.crt \
      --cert /var/vcap/jobs/hm9000/config/certs/etcd_client.crt \
      --key /var/vcap/jobs/hm9000/config/certs/etcd_client.key \
      https://${etcd}.cf-etcd.service.cf.internal:4001/v2/stats/${role}
  done
  echo
done
```

**If each `/leader` endpoint outputs a different results for each etcd vm you have etcd in split brain.**

## How to fix

Start by looking at your configuration.

Add cluster definition for all etcd jobs. There should be 1 declaration for each cluster

```
properties:  
  etcd:
    cluster:
    - instances: 2    # <== Must match # instances defined for etcd_z1 job
      name: etcd_z1   # <== Must match job name etcd_z1 etcd job
    - instances: 1    # <== Must match # instances defined for etcd_z1 job
      name: etcd_z2   # <== Must match job name etcd_z2 etcd job
```

Verify your peer ca cert, peer cert and peer key are correctly populated:

```
properties:
  etcd:
    ca_cert: (( grab meta.etcd_ca_cert ))   
    client_cert: (( grab meta.client_cert" ))
    client_key: (( grab meta.client_key" ))
    server_cert: (( grab meta.server_cert" ))
    server_key: (( grab meta.server_key" ))
    peer_ca_cert: (( grab meta.peer_ca_cert" ))  # <== Check these!!  
    peer_cert: (( grab meta.peer_cert" ))        # <== Check these!!  
    peer_key: (( grab meta.peer_key" ))          # <== Check these!!  
```

You may also have a stale etcd cache. To clear it, `monit stop all` on all `etcd_z*` vms, remove the `/var/vcap/store/etcd/member` folder on each etcd vm and finally `monit start all` on each of the remaining etcd vms. This was pulled from the Recover from HM9000 Failure section at [https://docs.cloudfoundry.org/running/troubleshooting.html](https://docs.cloudfoundry.org/running/troubleshooting.html)