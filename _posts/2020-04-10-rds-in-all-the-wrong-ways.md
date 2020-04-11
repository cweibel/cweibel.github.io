---
layout: post
title: "How Not to Do RDS"
date: 2020-04-10
---

# A Few Ways to Not Do RDS

I took on the task today of using the `aws` CLI to spin up an RDS Postgres instance.  Simple, right?

There are a couple of really darn important details you need to pay attention to.  I'll point out some of the more egregious errors committed by yours truly.

Order of operations:

 - Create a VPC - Or have another team responsible for doing this, so this exercise is left to the reader.
 - Create a bunch of subnets in the VPC.  This was also something already done for me.
 - Create a `RDS Subnet Group`.  It is not good enough to simply creating plain ol' subnets and try and get an RDS deploy to magically use them.  It just doesn't work that way, you need an RDS Subnet Group
 - Deploy the RDS Database
 - Create a security group (in a subsequent blog)
 - Create additional databases, users and extensions (in a subsequent blog)

## Deploy the RDS Subnet Group

Options as listed on [https://docs.aws.amazon.com/cli/latest/reference/rds/create-db-subnet-group.html](https://docs.aws.amazon.com/cli/latest/reference/rds/create-db-subnet-group.html)
All options:
```
  create-db-subnet-group
--db-subnet-group-name <value>
--db-subnet-group-description <value>
--subnet-ids <value>
[--tags <value>]
[--cli-input-json <value>]
[--generate-cli-skeleton <value>]
```
Newbie mistake: *don't put all the vpc's subnets in here, only the subnets you want the rds instance deployed to.*  When the AWS backend picks were  to place the master it seems to use the first subnet sorted alphabetically. 


Working example:

```
aws rds create-db-subnet-group \
--db-subnet-group-name rds-wyldstyle \
--db-subnet-group-description "Subnet group for the new RDS WyldStyle instances" \
--subnet-ids "subnet-089firstsubnet" "subnet-0021secondsubnet"
```


## Deploying the RDS Instance

Now for the fun part, actually deploying the darn thing.

### Documentation of the CLI

The number of options in the aws cli almost hurts the head.  To see the complete available aws rds cli commands visit [https://docs.aws.amazon.com/cli/latest/reference/rds/create-db-instance.html](https://docs.aws.amazon.com/cli/latest/reference/rds/create-db-instance.html) while below is a shortened list for mere mortals to likely use:

```
  create-db-instance
[--db-name <value>]
--db-instance-identifier <value>
[--allocated-storage <value>]
--db-instance-class <value>
--engine <value>
[--master-username <value>]
[--master-user-password <value>]
[--db-security-groups <value>]
[--db-subnet-group-name <value>]
[--port <value>]
[--multi-az | --no-multi-az]
[--engine-version <value>]
[--publicly-accessible | --no-publicly-accessible]
[--storage-type <value>]
[--storage-encrypted | --no-storage-encrypted]
[--enable-iam-database-authentication | --no-enable-iam-database-authentication]
[--deletion-protection | --no-deletion-protection]
```

### Deploy the DB

```
aws rds create-db-instance \
    --allocated-storage 100 \
    --db-instance-class db.m5.large \
    --db-instance-identifier wyldstyle-cf \
    --engine postgres \
    --master-username master \
    --master-user-password builder \
    --db-name pieceofresistence \
    --port 5432 \
    --db-subnet-group-name rds-wyldstyle \
    --no-auto-minor-version-upgrade 
```

Note: 

 - `--db-subnet-group-name` IS NOT a subnet name, instead this is a separate RDS Subnet Group which must be created ahead of time
 - The database instance does not instantly provision, it can take a few minutes.  See the step below to see the current status of the db.

### List DB

```
aws rds describe-db-instances \
    --db-instance-identifier wyldstyle-cf
```
Leave off the `--db-instance-identifier` to see all the rds dbs deployed


### Connect to the Database

To connect to the database from a jumpbox within the VPC

> psql -h wyldstyle-cf.lordbusiness.us-west-2.rds.amazonaws.com -U master pieceofresistence

When prompted for the password use the `--master-user-password` you used in the `create-db-instance` command.

### Delete DB

```
aws rds delete-db-instance
--db-instance-identifier wyldstyle-cf
```
