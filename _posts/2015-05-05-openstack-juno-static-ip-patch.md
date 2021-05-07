---
layout: post
title: "Openstack Juno Static IP Patch"
date: 2015-05-05
---

![map](https://raw.githubusercontent.com/cweibel/ghost_blog_pics/master/jakob-owens-ZBadHaTUkP0-unsplash.jpg)


Photo by [Jakob Owens](https://unsplash.com/@jakobowens1?utm_source=unsplash&utm_medium=referral&utm_content=creditCopyText) on [Unsplash](https://unsplash.com/s/photos/uri?utm_source=unsplash&utm_medium=referral&utm_content=creditCopyText)


[Author's Note in 2021: This is old and should not be used for modern deployments as-is but is kept for historical reasons]

There is a [bug](https://review.openstack.org/#/c/149905/2/nova/network/neutronv2/api.py) in OpenStack Juno which prevents BOSH from assigning static IPs. This can be quite an issue when trying to install Cloud Foundry on [OpenStack](https://github.com/cloudfoundry-community/terraform-openstack-cf-install).

There is a very simple fix:

## Log into Controller Node

From the Mirantis Fuel Dashboard navigate to Nodes > Controller and click on the gear icon. The FQDN will list the server we will need to log into. For this example we will use `node-100.prod.myorg.com`

ssh into this server:

> ssh node-100

Then assign all the environment variables by running:

> source openrc

Now to find the list of compute nodes run:

> nova service-list

An example of this output is (some columns removed for easier reading):

```
+----+------------------+-------------------------+---------+
| Id | Binary           | Host                    | Status  |
+----+------------------+-------------------------+---------+
| 1  | nova-consoleauth | node-100.prod.myorg.com | enabled |
| 2  | nova-scheduler   | node-100.prod.myorg.com | enabled |
| 3  | nova-conductor   | node-100.prod.myorg.com | enabled |
| 4  | nova-cert        | node-100.prod.myorg.com | enabled |
| 5  | nova-compute     | node-105.prod.myorg.com | enabled |
| 6  | nova-compute     | node-106.prod.myorg.com | enabled |
| 7  | nova-compute     | node-107.prod.myorg.com | enabled |
| 8  | nova-compute     | node-108.prod.myorg.com | enabled |
| 9  | nova-compute     | node-109.prod.myorg.com | enabled |
+----+------------------+-------------------------+---------+
```

## Fix The Compute Node



In the output above the first compute node is `node-105`, ssh to this server

> ssh node-105

Once on this server, open api.py file

> vi /usr/lib/python2.6/site-packages/nova/network/neutronv2/api.py

Replace the line:

> port_req_body['port']['fixed_ips'] = [{'ip_address': fixed_ip}]

with

> port_req_body['port']['fixed_ips'] = [{'ip_address': str(fixed_ip)}]

Save the file and restart nova-compute:

> service openstack-nova-compute restart

## Fix the remaining compute nodes

Now you have a choice, you can repeat the previous section for the remaining compute nodes (node-106, node-107, node-108 and node-109) or leverage the first server we fixed, copy the `api.py` file and restart nova-compute.

To leverage the first compute server's patch get the list of remaining compute servers we need to patch.

When we ran `nova service-list` on the controller node there were 5 compute nodes listed, starting with `node-105` and ending with `node-109`. We've already done `node-105` so we still need to patch `node-106` through `node-109`.

```
for i in {106..109}
do
   scp node-106:/usr/lib/python2.6/site-packages/nova/network/neutronv2/api.py node-$i:/usr/lib/python2.6/site-packages/nova/network/neutronv2/api.py
   ssh node-$i service openstack-nova-compute restart
done
```

That's it! Now you can deploy Cloud Foundry and use static IPs.

