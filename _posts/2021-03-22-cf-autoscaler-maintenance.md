
---
layout: post
title: "Maintenance For Cloud Foundry Autoscaler"
date: 2021-03-22
---

![map](https://raw.githubusercontent.com/cweibel/ghost_blog_pics/master/bofu-shaw-HIcAsbl2dBA-unsplash.jpg)

Photo by [Bofu Shaw](https://unsplash.com/@hikeshaw?utm_source=unsplash&utm_medium=referral&utm_content=creditCopyText) on [Unsplash](https://unsplash.com/s/photos/uri?utm_source=unsplash&utm_medium=referral&utm_content=creditCopyText)
  


This is a pretty niche area of Cloud Foundry, move along if you aren't using [CF Autoscaler](https://www.cloudfoundry.org/blog/cloud-foundry-app-autoscaler-project-has-graduated/) for, oh, 4 years with several thousand apps being monitored for scaling.  My copy of autoscaler is a wee bit older as well, modern versions I believe store much of this in memory.

## Patient Symptoms

I had an issue where the SHIELD backup of the autoscaler databases was taking longer and longer, finally got a ticket so I could take a look.
On first inspection the db was pretty chubby, don’t run into 50GB+ postgres boxes often anymore.  Hopped on the instance and dumped the top 10 relations in size:

```
    schema_name     |       relname        |  size   | table_size
--------------------+----------------------+---------+-------------
 public             | idx_instance_metrics | 23 GB   | 24695881728
 public             | appinstancemetrics   | 15 GB   | 15856132096
 public             | index_app_metrics    | 4722 MB |  4951744512
 public             | app_metric           | 2753 MB |  2886754304
 public             | scalinghistory       | 686 MB  |   719765504
 public             | scalingcooldown      | 784 kB  |      802816
 public             | policy_json          | 96 kB   |       98304
 public             | binding              | 72 kB   |       73728
 public             | pk_binding           | 64 kB   |       65536
 information_schema | sql_features         | 56 kB   |       57344
(10 rows)
```

After a bit of poking, grabbed the schemas for the table `appinstancemetrics`, `app_metric`  and `scalinghistory` and eventually figured out each contains 30 days of history.

At this point, cool, I don’t have to do anything fancy to rotate out the data but the `INDEX`es are bigger than the tables themselves.  Figured out the code does `DELETE FROM` queries which leads to index bloat.  After doing `REINDEX TABLE` commands was able to shrink down the monster indexes:

```
    schema_name     |       relname        |  size   | table_size
--------------------+----------------------+---------+-------------
 public             | appinstancemetrics   | 15 GB   | 15856132096
 public             | idx_instance_metrics | 8746 MB |  9170403328
 public             | app_metric           | 2753 MB |  2886754304
 public             | index_app_metrics    | 1783 MB |  1869365248
 public             | scalinghistory       | 686 MB  |   719765504
 public             | scalingcooldown      | 784 kB  |      802816
 public             | policy_json          | 96 kB   |       98304
 public             | binding              | 72 kB   |       73728
 public             | pk_binding           | 64 kB   |       65536
 information_schema | sql_features         | 56 kB   |       57344
(10 rows)
```

## Surgery To Fix The Problem

Cool, select statements are peppier now and its using less disk, but the time to do a backup hasn’t changed because INDEXes are included in pg_dump
To make this faster I need to either throw more hardware (nope) or configure for less data to be held.  Eventually found that the number of days of history in each of these tables is controlled in the bosh release at:

```
  autoscaler.operator.instance_metrics_db.cutoff_days:
    description: "the cutoff days when pruning instancemetrics database"
    default: 30
  autoscaler.operator.app_metrics_db.cutoff_days:
    description: "the cutoff days when pruning appmetrics database"
    default: 30
  autoscaler.operator.scaling_engine_db.cutoff_days:
    description: "the cutoff days when pruning scalingengine database"
    default: 30
```

I got to play with Postgres.  T’was a good day.