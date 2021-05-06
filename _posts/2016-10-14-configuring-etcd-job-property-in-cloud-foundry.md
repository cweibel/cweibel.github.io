---
layout: post
title: "Configuring etcd.cluster Job Property in Cloud Foundry"
date: 2016-10-14
---

![map](https://raw.githubusercontent.com/cweibel/ghost_blog_pics/master/steve-smith-KAWmWGx2X_4-unsplash.jpg)



Photo by [Steve Smith](https://unsplash.com/@varrak?utm_source=unsplash&utm_medium=referral&utm_content=creditCopyText) on [Unsplash](https://unsplash.com/s/photos/uri?utm_source=unsplash&utm_medium=referral&utm_content=creditCopyText)


A few new job properties were added to the etcd bosh release. One of these is etcd.cluster and reading the [spec file](https://github.com/cloudfoundry-incubator/etcd-release/blob/327fb92f6c1ee0aefca7a3ebcf017e3b98257995/jobs/etcd/spec#L36) you get the helpful hint that it is to represent `Information about etcd cluster`.

So what is this job property supposed to be set to? After a bit of trial and error and then reading the template the answer is simple, you need to know the names of your etcd jobs and the number of them. With this information in hand you can configure the property correctly.

Let's assume we have the deployment manifest for our etcd jobs as:

```
- instances: 2       # (A)
  name: etcd_z1      # (B)
  properties:
    metron_agent:
      zone: z1
    networks:
      apps: cf1
   ...
- instances: 1       # (C)
  name: etcd_z2      # (D)
  properties:
    metron_agent:
      zone: z2
    networks:
      apps: cf2
   ...
```

Now that you know the names and quantities of etcd jobs you can fill out the `etcd.cluster` property. You will have 1 instance of cluster for each `etcd` job:

```
properties:
  etcd:
    cluster:
    - instances: 2     # Value from (A)
      name: etcd_z1    # Value from (B)
    - instances: 1     # Value from (C)
      name: etcd_z2    # Value from (D)
```

Make sure you use `instances` and not `instance`, that error is annoying to track down.

If you forget to populate this field you will get an error similar to:

```
1 error(s) detected:
 - $.properties.etcd.cluster: must define etcd cluster at env level
```

Even worse, you could have guessed at what the values were supposed to be and wind up with an error similar to:

```
  Error 100: Unable to render instance groups for deployment. Errors are:
   - Unable to render jobs for instance group 'etcd_z1'. Errors are:
     - Unable to render templates for job 'etcd'. Errors are:
       - Error filling in template 'etcd_bosh_utils.sh.erb' (line 33: undefined method `-' for nil:NilClass)
```

























