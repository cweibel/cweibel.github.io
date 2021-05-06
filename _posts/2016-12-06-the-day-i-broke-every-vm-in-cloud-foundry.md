---
layout: post
title: "The Day I Broke Every VM in Cloud Foundry"
date: 2016-12-06
---

![map](https://raw.githubusercontent.com/cweibel/ghost_blog_pics/master/steve-smith-xayb-m1RJ08-unsplash.jpg)



Photo by [Steve Smith](https://unsplash.com/@varrak?utm_source=unsplash&utm_medium=referral&utm_content=creditCopyText) on [Unsplash](https://unsplash.com/s/photos/uri?utm_source=unsplash&utm_medium=referral&utm_content=creditCopyText)

On a recent upgrade I added Diego components to an existing DEA based CF deployment. All was well until I ran the `smoke_tests` errand and noticed logs were not being returned through Doppler to Loggregator so I ran through the usual suspects. At some point I ran `bosh vms` and was greeted with every single Cloud Foundry deployment in a failing state.

The culprit? `metron_agent` had fallen over on every VM. Good times. Picking a vm at random:

```
==[]=[ 18:35:50 ]=[ sb-cloudfoundry doppler_z1/0 ]=[ ~ ]=[]==
# monit summary
The Monit daemon 5.2.5 uptime: 1h 28m

Process 'consul_agent'              running
Process 'doppler'                   running
Process 'syslog_drain_binder'       running
Process 'metron_agent'              not monitored
Process 'sensu-client'              running
Process 'toolbelt'                  running
System 'system_localhost'           running
```

I found an interesting row among the misery listed in the `metron_agent.stderr.log`:

```
panic: No dopplers listening on [udp]
```

Which was interesting because I was ssh'd onto a Doppler vm.

Knowing a bit about how `metron_agent` connects to Doppler via key lookups in `etcd` I checked out the keys. This is most easily done by connecting to one of the `hm9000` instances which happens to have all the `etcd` certs and keys.

The following is a bit long but the basic point is traverse the tree of keys related to doppler to find one or more misconfigured keys.

```
one_of_my_etcd_servers="etcd_z1_0"
curl -k   --cacert /var/vcap/jobs/hm9000/config/certs/etcd_ca.crt \
   --cert /var/vcap/jobs/hm9000/config/certs/etcd_client.crt \
   --key /var/vcap/jobs/hm9000/config/certs/etcd_client.key \
   https://"${one_of_my_etcd_servers}".cf-etcd.service.cf.internal:4001/v2/keys | jq .
```

From this you can see there is a `/doppler` endpoint:

```
...
  {
  "key": "/doppler",
  "dir": true,
  "modifiedIndex": 4,
  "createdIndex": 4
},
...
```

One layer deeper at '/keys/doppler' you find the meta endpoint:

```
{
  "action": "get",
  "node": {
    "key": "/doppler",
    "dir": true,
    "nodes": [
      {
        "key": "/doppler/meta",
        "dir": true,
        "modifiedIndex": 4,
        "createdIndex": 4
      }
    ],
    "modifiedIndex": 4,
    "createdIndex": 4
  }
}
```
At `/keys/doppler/meta` now you can see that there are 3 known Doppler vms. Note that this piece will look different depending on the number of Doppler jobs you have defined in your deployment:

```
{  
  "action": "get",
  "node": {
    "key": "/doppler/meta",
    "dir": true,
    "nodes": [
      {
        "key": "/doppler/meta/z1",
        "dir": true,
        "modifiedIndex": 418,
        "createdIndex": 418
      },
      {
        "key": "/doppler/meta/z2",
        "dir": true,
        "modifiedIndex": 4,
        "createdIndex": 4
      },
      {
        "key": "/doppler/meta/z3",
        "dir": true,
        "modifiedIndex": 307,
        "createdIndex": 307
      },
    ],
    "modifiedIndex": 4,
    "createdIndex": 4
  }
}
```

Now you can loop through each of the `nodes` listed above and look at `/keys/doppler/meta/{each zone}` to get a list of job names:

```
{
  "action": "get",
  "node": {
    "key": "/doppler/meta/z1",
    "dir": true,
    "nodes": [
      {
        "key": "/doppler/meta/z1/doppler_z1",
        "dir": true,
        "modifiedIndex": 418,
        "createdIndex": 418
      }
    ],
    "modifiedIndex": 418,
    "createdIndex": 418
  }
}
```

At this layer you can see all the possible ports which Doppler should be listening on for `doppler_z1/0` including the UDP port the error log was referring to. This particular key looks ok:

```
{
  "action": "get",
  "node": {
    "key": "/doppler/meta/z1/doppler_z1",
    "dir": true,
    "nodes": [
      {
        "key": "/doppler/meta/z1/doppler_z1/4c738367-8eb1-4a57-b44e-771f4870db0e",
        "value": "{\"version\":1,\"endpoints\":[\"udp://10.160.55.25:3457\",\"tcp://10.160.55.25:3458\",\"ws://10.160.55.25:8081\"]}",
        "expiration": "2016-12-06T19:42:36.064990188Z",
        "ttl": 10,
        "modifiedIndex": 56766863,
        "createdIndex": 30947393
      }
    ],
    "modifiedIndex": 418,
    "createdIndex": 418
  }
}
```

However the next doppler does not have `nodes` like the previous example defined:

```
{
  "action": "get",
  "node": {
    "key": "/doppler/meta/z2/doppler_z2",
    "dir": true,
    "nodes": [],
    "modifiedIndex": 418,
    "createdIndex": 418
  }
}
```

This is our culprit! Performing a `monit restart all` on the `doppler_z2/0` vm registers that instance of Doppler with etcd. After 30 seconds the `metron_agent` on all the servers report running.

Yay!  Now hand me a wrench so we can work on the nifty car in the picture!