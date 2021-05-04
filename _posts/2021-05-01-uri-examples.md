---
layout: post
title: "URI Examples for Postgres on AWS and Azure"
date: 2021-05-01
---

![map](https://raw.githubusercontent.com/cweibel/ghost_blog_pics/master/xavier-von-erlach-KjyuH25GS48-unsplash.jpg)

Photo by [Xavier von Erlach](https://unsplash.com/@xavier_von_erlach?utm_source=unsplash&utm_medium=referral&utm_content=creditCopyText) on [Unsplash](https://unsplash.com/s/photos/uri?utm_source=unsplash&utm_medium=referral&utm_content=creditCopyText)
  

Sometimes you need a bit of URI formatting in your life but can't remember the syntax and googling this seems to be hard to find useful example.  If you are using Cloud Foundry, here are some examples to crack into your `ccdb`

## Postgres

This example will add the `citext` extension to the `ccdb` database.  Never know when you'll need that.

```
psql 'postgres://${master_username}:${master_password}@${rds_instance_hostname}/ccdb' -c "CREATE EXTENSION IF NOT EXISTS citext;"
```

Weird edge case: Azure - the user name is not a regular name like `admin` or `bob`, they include a funky funky `@` made up of the `user+@+server`, the example below assumes you'll be running it through terraform where ascii 40 = `@`

```
psql 'postgres://${master_username}%40${server_name}:${master_password}@${instance_hostname}/ccdb' -c "CREATE EXTENSION IF NOT EXISTS citext;"
```

## MySQL

For mysql you can do this by not putting a space between the -p and the password itself:

```
mysql -h ${rds_instance_hostname} -u ${master_username} --database master -p${master_password} -e "CREATE DATABASE IF NOT EXISTS ccdb;"
```


I know its an old blog post but I reference it every 6 months: [https://starkandwayne.com/blog/using-a-postgres-uri-with-psql/](https://starkandwayne.com/blog/using-a-postgres-uri-with-psql/)