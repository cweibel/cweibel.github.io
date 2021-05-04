---
layout: post
title: "LDAP In Concourse, Why Hast Thou Errored on Me?"
date: 2019-11-18
---

![map](https://raw.githubusercontent.com/cweibel/ghost_blog_pics/master/marko-horvat-uulf3173LPU-unsplash.jpg)

Photo by [Marko Horvat](https://unsplash.com/@lemondyt) on [Unsplash](https://unsplash.com/s/photos/uri?utm_source=unsplash&utm_medium=referral&utm_content=creditCopyText)

## What we were doing

Recently, we were helping a client to integrate logging into Concourse. Deploying Concourse with the [`concourse-bosh-deployment`](https://github.com/concourse/concourse-bosh-deployment/tree/master/cluster) is fairly easy with a base `concourse.yml` and features added with various ops files. One of the available ops files adds LDAP authentication which the client wanted.  We wound up with a deployment similar to:

```
bosh deploy -d control_plane_concourse concourse.yml \
  -o operations/ldap.yml \
  -o operations/add-main-team-ldap-users.yml \
  -o operations/tls.yml \
  -o operations/tls-vars.yml \
  -o operations/credhub.yml \
  -o operations/credhub-path-prefix.yml
```

## Symptoms

After deploying Concourse to use LDAP authentication, we tried logging in.  No dice.  So we bosh ssh’d onto the Web VM and looked at the logs in `/var/vcap/sys/log/web`.   Scrolling through the logs found this error:

```
"level":"error","source":"atc","message":"atc.dex.event","data":{"fields":{},
  "message":"Failed to login user: ldap: entry missing following required attribute(s):
  [\"\"]","session":"7"}
```

## Solution

There were no errors during the BOSH deploy but obviously we were missing something.  After a bit of trial and we error discovered that the following needed to be populated:

 - [ldap_user_search_id_attr](https://github.com/concourse/concourse-bosh-deployment/blob/master/cluster/operations/ldap.yml#L26), should not be empty
 - [ldap_user_search_email_attr](https://github.com/concourse/concourse-bosh-deployment/blob/master/cluster/operations/ldap.yml#L30), should be set to mail
 - [ldap_user_search_username](https://github.com/concourse/concourse-bosh-deployment/blob/master/cluster/operations/ldap.yml#L24), should not be empty
 - 
These values can be set in CredHub or provided as [`vars-file'](https://bosh.io/docs/cli-int/) when performing the BOSH deployment. After a redeployment Concourse authentication to LDAP worked as expected!