---
layout: post
title: "BOSH + UAA With Signed Certificates - Part II"
date: 2016-08-09
---

![map](https://raw.githubusercontent.com/cweibel/ghost_blog_pics/master/steve-smith-3bus5RKqlOs-unsplash.jpg)



Photo by [Steve Smith](https://unsplash.com/@varrak?utm_source=unsplash&utm_medium=referral&utm_content=creditCopyText) on [Unsplash](https://unsplash.com/s/photos/uri?utm_source=unsplash&utm_medium=referral&utm_content=creditCopyText)



In the second part of configuring UAA with BOSH we'll cover changes which are needed for Health Monitor which may not be obvious from the tutorial found at [http://bosh.io/docs/director-users-uaa.html](http://bosh.io/docs/director-users-uaa.html).

Part I of this tutorial is here:



## Change Health Manager Authentication

In your deployment manifest you should have the `user` and `password` defined similar to:

```
hm:
  director_account:
    user:  hm_user
    password: hm_password
```

You've removed all the local accounts from BOSH so you can no longer use a `user` & `password` and instead need to use `client_id` and `client_secret` much like we did in the Shield example in Part I. We do this in two steps, the first defines a new UAA client and then we use these client credentials for the `hm:director_account` properties. You can reuse the same user and password of the local account:

```
uaa:
  clients:
    hm_user:
      authorities: bosh.admin
      authorized-grant-types: client_credentials
      override: true
      scope: bosh.admin
      secret: hm_password

hm:
  director_account:
    client_id:   hm_user
    client_user: hm_password
```

## Verify via Logs

SSH onto the microbosh director and tail `/var/vcap/sys/log/health_monitor/health_monitor.log`, if you get a `401` error you likely copy/pasted the creds incorrectly, are still using `user` & `password` instead of `client_id` & `client_secret` or need another cup of coffee:


> [2016-08-08T14:06:55.175865 #25522] INFO : [ALERT] Alert @ 2016-08-08 14:06:55 UTC, severity 3: Cannot get deployments from director at https://10.8.6.4:25555/deployments: 401 Not authorized: '/deployments'

Run the logs for at least a minute watching for these requests. No `401`s and you should be all set, Health Monitor will once again watch over your deployments once it logs into Bosh via UAA. Enjoy!