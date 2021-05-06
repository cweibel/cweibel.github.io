---
layout: post
title: "Connecting to an App Container in Cloud Foundry"
date: 2017-01-18
---

![map](https://raw.githubusercontent.com/cweibel/ghost_blog_pics/master/steve-smith-qQDOyPWfdQ8-unsplash.jpg)



Photo by [Steve Smith](https://unsplash.com/@varrak?utm_source=unsplash&utm_medium=referral&utm_content=creditCopyText) on [Unsplash](https://unsplash.com/s/photos/uri?utm_source=unsplash&utm_medium=referral&utm_content=creditCopyText)

You may find yourself in need to connect to a container running on either a DEA runner or Diego Cell. There are two different methods depending on which backend you use which are listed below.

## DEA Runner

Connect via ssh to one of the runners.

```
bosh ssh runner_z1/0
```

Switch to the root user and obtain the list of all Warden information on the server, including app and service credentials:

```
sudo -i
cat /var/vcap/data/dea_next/db/instances.json 
```

The output will appear similar to this:

```
...
"application_id": "b39f03c7-27c5-4239-86fd-bfb8dd194489",
"application_version": "d2cf9c78-c7f3-4280-8877-5e44c267e0e2",
"application_name": "my-awesome-cf-app-3",
"application_uris": [
  "my-awesome-cf-app-3-release-candidate.run.aws.domain.io"
],
"droplet_sha1": "c8bd3a6bc98bf8fd6e6e9b03c5eb690db07d263d4",
"state": "RUNNING",
"warden_job_id": 51,
"warden_container_path": "/var/vcap/data/warden/depot/1a5lipbhhrt",
"warden_host_ip": "10.54.0.69",
...
```

Observe the `warden_container_path` for the application you wish to connect to, navigate to this folder and connect to the container:


```
cd /var/vcap/data/warden/depot/1a5lipbhhrt
./bin/wsh

root@1a5lipbhhrt:~#   
```

You are now connected as the `root` user on the application and debug as you wish. Simply exit when done.

## Diego Cell

When running against Diego the list of applications running on a particular cell is not as easily discovered. One way of doing this is to add `toolbelt-veritas` onto the `database` job in the Diego deployment. If you aren't familiar with the `toolbelt` release it is located here: [https://github.com/cloudfoundry-community/toolbelt-boshrelease](https://github.com/cloudfoundry-community/toolbelt-boshrelease)

Start by connecting to one of the Diego `database` instances:

```
bosh ssh database/0
```

Now you can connect to veritas and dump the list of applications and cells they are running on. If you are using a newer version of Cloud Foundry you likely have certificates and keys to communicate with BBS which need to be exported. Below should leverage default values for a straight copy-paste:

```
export BBS_CERT_FILE=/var/vcap/jobs/bbs/config/certs/server.crt
export BBS_KEY_FILE=/var/vcap/jobs/bbs/config/certs/server.key
export BBS_ENDPOINT=https://bbs.service.cf.internal:8889
veritas dump-store
```


This will output similar to:

```
Tasks
LRPs
  521b47f5-48fd-4054-8c10-a50a1b2c34b4-8ea82812-a6ff-4f27-bfd8-c764ea0627ad
    6 preloaded:cflinuxfs2 (256 MB, 1024 MB, 3 CPU)
      8080 => cw-diego.run.aws.domain.io
       0: d9a451d6-9d27-4699-4866-d7ccf828b635 cell-9 [RUNNING for 15h25m22.374563333s]
       1: 83afe77a-9b8a-48fd-7570-b7dae67e72eb cell-0 [RUNNING for 26m44.413924206s]
       2: d11191ef-65d4-4705-5c1b-47532180efd0 cell-6 [RUNNING for 26m44.289806726s]
       3: 5f0a000f-ec80-4c1a-56dd-e3a67154fafd cell-3 [RUNNING for 26m45.111574374s]
       4: f1ee061a-84d8-47ad-77c4-6feab22c4a70 cell-5 [RUNNING for 26m44.371298781s]
       5: 455cbf70-566f-4e3a-4444-2645f9a2c7ff cell-1 [RUNNING for 26m45.056561196s]
...
```

Let's assume you want to attach to this application on `cell-3`, number 3 on the list. Connect to the server:

```
bosh ssh cell/3
```

Use gaol to connect to the guid for the instance of the application on this cell:

```
gaol shell 5f0a000f-ec80-4c1a-56dd-e3a67154fafd
#
```

You are now connected as the `root` user.


## More Honey!!


There are several ways to get this information using the `ccdb`, various API calls, etc. This is simply one way of finding which applications are running on a runner or cell and connecting to them. 

Enjoy!
























