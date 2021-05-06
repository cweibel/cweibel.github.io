---
layout: post
title: "Verifying Cloud Foundry Route Counts"
date: 2017-04-28
---

![map](https://raw.githubusercontent.com/cweibel/ghost_blog_pics/master/steve-smith-KAWmWGx2X_4-unsplash.jpg)



Photo by [Steve Smith](https://unsplash.com/@varrak?utm_source=unsplash&utm_medium=referral&utm_content=creditCopyText) on [Unsplash](https://unsplash.com/s/photos/uri?utm_source=unsplash&utm_medium=referral&utm_content=creditCopyText)

Occasionally you may run into an issue where the routes between the routers and nats are not in a consistent state. To detect this issue log onto a server with connectivity to your BOSH director and copy the following `bash` script onto it

Let's get our ducks in a row!

```
#!/usr/bin/env bash

DEPLOYMENT=$1
RUSER=$2
RPASSWORD=$3

if [[ -z ${DEPLOYMENT} ]]; then
  echo "No deployment specified."
  exit 1
fi

if [[ -z ${RUSER} ]]; then
  echo "No router user specified"
  exit 1
fi

if [[ -z ${RPASSWORD} ]]; then
  echo "No router user password specified"
  exit 1
fi

if ! [ -x "$(command -v jq)" ]; then
  echo 'jq is not installed and is requried - please install' >&2
  exit 2
fi

for ip in $(bosh vms $DEPLOYMENT| grep router_z | grep -oE "[0-9]+\..*[0-9]+"); do
  echo Router IP Address: $ip
  temp_length=$(curl -s http://$RUSER:$RPASSWORD@$ip:8080/routes | jq length)
  echo Router Routes length: $temp_length
  temp_length=
done
```

To use this script you will need to retrieve the router username and password from your deployment manifest and call the script with the following parameters:

```
script <deployment name> <router.status.user> <router.status.password>
```

This will generate output similar to:

```
Router IP Address: 10.8.8.100
Router Routes length: 12121
Router IP Address: 10.8.8.101
Router Routes length: 12121
Router IP Address: 10.8.8.102
Router Routes length: 4336
```

If there are inconsistencies you likely just need to `monit restart` on all of the `nats` servers.