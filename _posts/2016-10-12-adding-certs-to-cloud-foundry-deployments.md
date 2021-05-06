---
layout: post
title: "Adding Certificates to Cloud Foundry Deployments (cf-release era)"
date: 2016-10-13
---

![map](https://raw.githubusercontent.com/cweibel/ghost_blog_pics/master/steve-smith-4hVY30sVXso-unsplash.jpg)



Photo by [Steve Smith](https://unsplash.com/@varrak?utm_source=unsplash&utm_medium=referral&utm_content=creditCopyText) on [Unsplash](https://unsplash.com/s/photos/uri?utm_source=unsplash&utm_medium=referral&utm_content=creditCopyText)



**Update** These instructions are for `cf-release`.  Anyone using newer versions of `cf-deployment` will have a `variables:` section in their manifest to create any needed certs via `credhub`.

Now, back to our story...


We recently added etcd TLS to several environments and leveraged the certificate creation scripts in `cf-release/scripts`. These are wonderful little scripts but leave it as an exercise to copy and paste in the contents of the flat files into your deployment manifest. After my second copy-pasta a colleague (thanks Tom) created a helpful script to copy the certs into my clipboard.

In this example we will create certificates needed for etcd for CF v243. Start by getting the CF release, checking out the correct release and switching to the scripts folder:

```
git clone git@github.com:cloudfoundry/cf-release.git
cd cf-release
git checkout v243
cd scripts
```

Now that we are in the scripts folder there are several helpful scripts to generate certs for several CF components. In this case we'll create the certs for etcd:

```
./generate-etcd-certs
```

This will create a folder called `etcd-certs` and inside this folder you will see all the files created:

```
-r--r--r--  1.5K client.crt
-r--r--r--  891B client.csr
-r--r-----  1.6K client.key
-r--r--r--  1.8K etcd-ca.crt
-r--r-----  3.2K etcd-ca.key
-r--r--r--  918B etcdCA.crl
-r--r--r--  1.8K peer-ca.crt
-r--r-----  3.2K peer-ca.key
-r--r--r--  1.6K peer.crt
-r--r--r--  1.0K peer.csr
-r--r-----  1.6K peer.key
-r--r--r--  918B peerCA.crl
-r--r--r--  1.6K server.crt
-r--r--r--  1.0K server.csr
-r--r-----  1.6K server.key
```


So now begins the awkward cat-copy-paste into your deployment manifest, unless...

Create a file named `certs_please.yml` in the `etcd-certs` folder and copy in the following contents:

```
client_cert: (( file "./client.crt"  ))
client_key: (( file "./client.key" ))
peer_cert: (( file "./peer.crt"  ))
peer_key: (( file "./peer.key" ))
server_cert: (( file "./server.crt" ))
server_key: (( file "./server.key" ))
ca_cert: (( file "./etcd-ca.crt" ))
peer_ca_cert: (( file "./peer-ca.crt" ))
```

Now run the [`spruce`](https://github.com/geofffranks/spruce) command to copy the contents to your clipboard buffer:

```
spruce merge certs_please.yml | pbcopy
```

Now paste into your deployment manifest. Less mess, less debugging copy-pasta errors.