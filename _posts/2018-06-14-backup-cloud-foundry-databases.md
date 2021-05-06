---
layout: post
title: "Manual Backup of ccdb/uaadb Databases on Cloud Foundry"
date: 2018-06-14
---

![map](https://raw.githubusercontent.com/cweibel/ghost_blog_pics/master/steve-smith-FqU7kOoJum8-unsplash.jpg)




Photo by [Steve Smith](https://unsplash.com/@varrak?utm_source=unsplash&utm_medium=referral&utm_content=creditCopyText) on [Unsplash](https://unsplash.com/s/photos/uri?utm_source=unsplash&utm_medium=referral&utm_content=creditCopyText)

For whatever reason you may want to make a database backup for `uaadb`,  `ccdb` or any of the other 6 databases in `cf-deployment` and copy the backup to a jumpbox or other server. The instructions below will walk through two options for performing the backups and getting the backup to a secondary server. Option 1 assumes the jumpbox cannot access the databases directly, Option 2 assumes the jumpbox can access the database directly.

The examples use `mysql` but it is a similar process for `postgres`, just swap `mysqldump` for `pg_dump`.

## Requirements

You must have the following to perform a backup of the `ccdb` or `uaadb` database instance:

 - Access to BOSH
 - BOSH SSH access to a `cloud_controller` or `api `for `ccdb`; `uaa` vm for `uaadb` in your CF deployment
 - Access to a jumpbox or similar with a few GB of persistent storage

## Login

Target the CF deployment and bosh ssh to the `cloud_controller/0` instance, specify your own alias and deployment, the example below is for `myenv`:

```
bosh -e myenv -d my-cf-deployment ssh cloud_controller/0
```

Once connected via bosh ssh, connect as the root user and retrieve the creds to the database:

```
sudo -i
vim /var/vcap/jobs/cloud_controller_ng/config/cloud_controller_ng.yml
```

In this file search for a line similar to the following which contains the cleartext credentials to the ccdb database:

```
db: &db
  database: "mysql2://databaseusername:databasepassword@mysql.service.cf.internal:3306/ccdb"
```

For `uaadb` connect via `bosh ssh uaa/0` and search for the file `/var/vcap/jobs/uaa/config/uaa.yml` and look for lines similar to:

```
database:
  url: jdbc:mysql://mysql.service.cf.internal:3306/uaa
  username: databaseusername
  password: databasepassword
```

## Option 1 - Create backup on `cloud_controller/0` or `uaa/0`

While in the ssh session to `cloud_controller/0` for ccdb or `uaa/0` for uaadb, if the `mysqldump` and `mysql` client tools were not previously installed on the vm, run the following:

```
sudo apt-get update
sudo apt-get install mysql-client
```

Now you can perform the backup of the database, substitute values for `host`, `user` and `password` from the `cloud_controller_ng.yml` or `uaa.yml` file:

```
#ccdb

mysqldump --host=mysql.service.cf.internal --user=databaseusername --password=databasepassword --single-transaction ccdb > /tmp/ccdb.sql
uaadb:


#uaadb

mysqldump --host=mysql.service.cf.internal --user=databaseusername --password=databasepassword --single-transaction uaadb > /tmp/uaadb.sql
```

Set the file permissions so `bosh scp` can copy the files later, `bosh scp` doesn't work against files owned by root:

```
chown vcap:vcap /tmp/ccdb.sql   #ccdb
chown vcap:vcap /tmp/uaadb.sql  #uaadb
```

Now exit the ssh connection to the `cloud_controller/0` or `uaa/0` vm, from the jumpbox run a `bosh scp` command to download the backup to the jumpbox:

```
bosh -e myenv -d my-cf-deployment scp cloud_controller/0:/tmp/ccdb.sql .
bosh -e myenv -d my-cf-deployment scp uaa/0:/tmp/uaadb.sql .
```


## Option 2 - Perform Backup From Jumpbox

While on the `cloud_controller/0` or `uaa/0` vm get the DNS lookup for the mysql nodes:

```
# dig mysql.service.cf.internal

; <<>> DiG 9.9.5-3ubuntu0.17-Ubuntu <<>> mysql.service.cf.internal
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 21422
;; flags: qr aa rd ra; QUERY: 1, ANSWER: 2, AUTHORITY: 0, ADDITIONAL: 0

;; QUESTION SECTION:
;mysql.service.cf.internal.    IN    A

;; ANSWER SECTION:
mysql.service.cf.internal. 0    IN    A    10.20.2.20
mysql.service.cf.internal. 0    IN    A    10.20.2.21
```

Now you can perform the backup of the database, substitute values for `host`, `user` and `password` from the `cloud_controller_ng.yml` or `uaa.yml` file, use one of the two ip addresses from the DNS lookup in the previous step for the host parameter:

ccdb:

```
mysqldump --host=10.20.2.21 --user=databaseusername --password=databasepassword --single-transaction ccdb > ccdb.sql
```

uaadb:

```
mysqldump --host=10.20.2.21 --user=databaseusername --password=databasepassword --single-transaction uaadb > uaadb.sql
```

## Verify Backup

Verify you have the full file by running a `tail`. If the backup was complete it should end with:

```
...
-- Dump completed on 2018-06-14 10:22:53
```

## Hey, What About the Other CF Databases?

Rinse, repeat the instructions about, reference [https://cweibel.github.io/blog/2020/07/07/connect-to-cf-dbs](https://cweibel.github.io/blog/2020/07/07/connect-to-cf-dbs) for a list of the databases and where their configuration files are that hide the clear text values.