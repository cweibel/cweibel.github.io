---
layout: post
title: "Running Your Own DNS Server For Cloud Foundry"
date: 2017-10-24
---

![map](https://raw.githubusercontent.com/cweibel/ghost_blog_pics/master/steve-smith-vHDMOmONWEc-unsplash.jpg)



Photo by [Steve Smith](https://unsplash.com/@varrak?utm_source=unsplash&utm_medium=referral&utm_content=creditCopyText) on [Unsplash](https://unsplash.com/s/photos/uri?utm_source=unsplash&utm_medium=referral&utm_content=creditCopyText)


## Overview

I ran across a nifty trick today to deploy a local DNS for Cloud Foundry. We all deserve nice things so I figured I would share.

We have a Cloud Foundry environment which is being migrated across infrastructures and need to test different scenarios keeping system and run urls the same as the old environment. We are not ready to expose the new environment yet. For now we need local DNS to resolve in the new environment.

## Implementation

***Warning: Here be Dragons***

This is a quick/dirty solution for temporary use. For a longer term solution consider using [`local-dns.yml`](https://github.com/cloudfoundry/bosh-deployment/blob/master/local-dns.yml) opsfile for bosh2 deployments. BOSH + apt-get anything makes operators sad in the long run.

We start by adding one more job to the CF deployment manifest:

```
jobs:
- name: dns
  instances: 1
  networks:
  - name: default
    static_ips:
    - 10.120.2.12
  resource_pool: medium_z1
  templates: []
```

This will add basically an empty vm which we can use to configure DNS. `bosh ssh` onto the server and install `dnsmasq`:

```
apt-get update
apt-get install dnsmasq -y
```

Edit the config file at `/etc/dnsmasq.conf` and add the dns mappsings you would like:

```
# Points to our CF HAProxy
address=/system.pr.starkandwayne.com/10.130.16.15
address=/run.pr.starkandwayne.com/10.130.16.15
```

To enable this DNS entry restart the `dnsmasq` service:

```
service dnsmasq restart
```

Now to get all of the vms in the Cloud Foundry deployment to use this DNS value, modify the `networks:` block and add the ip address of your new DNS server and redeploy:

```
networks:
- name: default
  subnets:
  - cloud_properties:
    dns:
    - 10.120.2.12
```

Now you can dig `system.pr.starkandwayne.com` from any server with the new DNS entry and get the ip address of the HAProxy server in CF:

```
; <<>> DiG 9.10.3-P4-Ubuntu <<>> system.pr.starkandwayne.com
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 2181
;; flags: qr aa rd ra; QUERY: 1, ANSWER: 1, AUTHORITY: 0, ADDITIONAL: 0

;; QUESTION SECTION:
;system.pr.starkandwayne.com. IN	A

;; ANSWER SECTION:
system.pr.starkandwayne.com. 0 IN	A	10.130.16.15

;; Query time: 0 msec
;; SERVER: 10.120.2.12#53(10.130.16.15)
;; WHEN: Wed Oct 25 00:37:16 UTC 2017
;; MSG SIZE  rcvd: 65
```

If you run `dig` from a server which does not have this dns server entry you'll get the old environment (ie: the customer only sees the old environment):

```
; <<>> DiG 9.10.3-P4-Ubuntu <<>> system.pr.starkandwayne.com
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 14649
;; flags: qr rd ra; QUERY: 1, ANSWER: 3, AUTHORITY: 0, ADDITIONAL: 1

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 4096
;; QUESTION SECTION:
;system.pr.starkandwayne.com. IN A

;; ANSWER SECTION:
system.pr.starkandwayne.com. 60 IN CNAME CF-System-7168675309.us-east-1.elb.amazonaws.com.
CF-System-7168675309.us-east-1.elb.amazonaws.com. 60 IN A 52.165.68.112
CF-System-7168675309.us-east-1.elb.amazonaws.com. 60 IN A 52.162.32.221

;; Query time: 28 msec
;; SERVER: 10.13.6.28#53(10.30.22.82)
;; WHEN: Wed Oct 25 00:42:38 UTC 2017
;; MSG SIZE  rcvd: 171
```

Once we are ready to expose this new environment to the world we simply remove the dns entry `10.120.2.12` from the `default` network, redeploy and register public DNS to our CF HAProxy node.

## Conclusions

I'm sure there are a dozen ways this could have been done, this one just happened to work for our use case. 