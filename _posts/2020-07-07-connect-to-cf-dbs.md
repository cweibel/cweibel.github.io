---
layout: post
title: "How to Connect to Cloud Foundry's Databases"
date: 2020-07-07
---

![beaver](https://raw.githubusercontent.com/cweibel/ghost_blog_pics/master/n-RFId0_7kep4-unsplash-4.jpg)

Photo by [N.](https://unsplash.com/@ellladee?utm_source=unsplash&utm_medium=referral&utm_content=creditCopyText) on [Unsplash](https://unsplash.com/s/photos/filter?utm_source=unsplash&utm_medium=referral&utm_content=creditCopyText)

Whether you are using PCF, cf-deployment, or KubeCF, there is an easy pattern to connect to the underlying MySQL or PostgreSQL databases that works regardless of the secrets provider (CredHub, Vault, or Kubernetes secrets). Makes you feel a bit like a pirate, right?

So, break out your sand shovels and let's go hunting for some connection strings!

## Finding the First Treasure

All flavors of Cloud Foundry have a Cloud Controller API service. This service is either a pod or VM. The configuration file used to start the API service is always in clear text. This includes configurations for the API to talk to the Cloud Controller database.

What does this mean?

Find the configuration file and you'll find the database connection string in clear text.

Indeed, `X` marks the spot!

## What You'll Need

While this sounds like a huge security hole, it is not. In order to gain access to the configuration files, you'll need `bosh ssh` access which typically is limited to a small set of operations folks.

Begin the Hunt
Let's start by going after the cloud_controller database. Depending on the flavor of the Cloud Foundry installer you'll either be hunting for a VM or a pod.

### PCF/cf-deployment

 - `bosh ssh api/0`  (or if older bosh ssh cloud_controller/0)
 - Navigate to `/var/vcap/jobs/cloud_controller_ng/config/cloud_controller_ng.yml`
   
   
### KubeCF

 - `k exec -it api-0 -- cat /var/vcap/jobs/cloud_controller_ng/config/cloud_controller_ng.yml`


### Parsing the Config File

Inside this file will be a reference to db:, this will contain your database connection string like:

```
   db: &db
   	database: "mysql2://databaseusername:databasepassword@mysql.service.cf.internal:3306/cloud_controller"
```

Congratulations! You can now use this connection string to attach manually to the cloud_controller database!

## UAA and the Other CF Databases

You can repeat the same pattern for UAA and other databases. When you find the component with the configuration file, scan the configuration file for the credentials and then manually make the connection to the database:


| Database |	vm/pod	Config | File Location |
| --- | --- | --- |
|uaa | uaa | `/var/vcap/jobs/uaa/config/uaa.yml`
|credhub| credhub| `/var/vcap/jobs/credhub/config/application/spring.yml`
|network_policy| api | `/var/vcap/jobs/policy-server/config/policy-server.json`
|diego | diego-api | `/var/vcap/jobs/bbs/config/bbs.json`
|locket| diego-api | `/var/vcap/jobs/locket/config/locket.json`
|routing-api | routing-api | `/var/vcap/jobs/routing-api/config/routing-api.yml`
|cloud_controller| api | `/var/vcap/jobs/cloud_controller_ng/config/cloud_controller_ng.yml`
|network_connectivity (silk)	| diego-api | `/var/vcap/jobs/silk-controller/config/silk-controller.json`

## Beware of the Treasure's Curse

Just because you have access to the databases of Cloud Foundry now, doesn't mean you should start dropping tables in production to exercise your ticketing system's ability to handle tickets-per-minute.

If you are running anything other than a SELECT statement be sure to back up the database first. For instructions on how to do this refer to: [https://www.starkandwayne.com/blog/manual-backup-of-ccdb-uaadb-on-cloud-foundry/](https://www.starkandwayne.com/blog/manual-backup-of-ccdb-uaadb-on-cloud-foundry/)

Enjoy!

(PS: put your sandbox toys away when you are done with them, otherwise dad will run over them with the lawnmower. Again. )


