---
layout: post
title: "Debugging Slow PostgreSQL on BOSH with pgBadger"
date: 2019-10-23
---

![map](https://raw.githubusercontent.com/cweibel/ghost_blog_pics/master/vincent-van-zalinge-u4TvWW1ldW8-unsplash.jpg)


Photo by [Vincent van Zalinge](https://unsplash.com/@vincentvanzalinge?utm_source=unsplash&utm_medium=referral&utm_content=creditCopyText) on [Unsplash](https://unsplash.com/s/photos/uri?utm_source=unsplash&utm_medium=referral&utm_content=creditCopyText)

Something is just not quite right, the CLI is sluggish and your senses are tingling that something is wrong.  Is it the database?  Maybe.  How do you know?

[pgBadger](https://github.com/darold/pgbadger/) is a nifty tool that reads through postgres logs and generates a report showing slow queries, table locks, DML distribution and several other metrics which are displayed in an easy to read format.  To use pgBadger we'll need to configure a few settings in the deployed instance of postgres.

What you will need:

 - A BOSH deployed VM with PostgreSQL on it
 - SSH access to the vm with PostgreSQL (`bosh ssh`)
 - pgBadger installed locally (`brew install pgbadger`)

## Scenario

In the example below we have a BOSH Director (`maybe_broken_bosh`) deployed by another BOSH Director (`master_bosh`).  This inner BOSH Director has UAA, Credhub and BOSHDB Postgres databases on it where we suspect the `boshdb` has something going wrong.  We'll enable the logging for this database and feed the logs into pgBadger to see what is going on.

## SSH and Connect to the Database

Start by making a SSH connection to the suspect BOSH Director vm:

```
bosh -e master_bosh -d maybe_broken_bosh ssh
```

Now connect to the postgres instances:

```
/var/vcap/packages/postgres-9.4/bin/psql -U vcap -h 127.0.0.1 postgres
```
A quick `\l` command will list all the databases:

```
postgres=# \l
                              List of databases
   Name    | Owner | Encoding |   Collate   |    Ctype    | Access privileges
-----------+-------+----------+-------------+-------------+-------------------
 bosh      | vcap  | UTF8     | en_US.UTF-8 | en_US.UTF-8 |
 credhub   | vcap  | UTF8     | en_US.UTF-8 | en_US.UTF-8 |
 postgres  | vcap  | UTF8     | en_US.UTF-8 | en_US.UTF-8 |
 template0 | vcap  | UTF8     | en_US.UTF-8 | en_US.UTF-8 | =c/vcap          +
           |       |          |             |             | vcap=CTc/vcap
 template1 | vcap  | UTF8     | en_US.UTF-8 | en_US.UTF-8 | =c/vcap          +
           |       |          |             |             | vcap=CTc/vcap
 uaa       | vcap  | UTF8     | en_US.UTF-8 | en_US.UTF-8 |
(6 rows)
```

Our potential troublemaker database is named `bosh` so let's connect to that db with a `\c` bosh command:

```
postgres=# \c bosh
You are now connected to database "bosh" as user "vcap".
```

## Set Configuration Parameters

There are a couple configuration parameters we'll need to check the values for and potentially change with this query:

```
SELECT name, setting, short_desc 
FROM pg_settings 
WHERE name IN 
('logging_collector', 'log_min_duration_statement', 'log_destination', 'log_line_prefix');
```

Right now these are configured as:

```
            name            | setting |                                 short_desc
----------------------------+---------+----------------------------------------------------------------------------
 log_destination            | stderr  | Sets the destination for server log output.
 log_line_prefix            | %t      | Controls information prefixed to each log line.
 log_min_duration_statement | -1      | Sets the minimum execution time above which statements will be logged.
 logging_collector          | off     | Start a subprocess to capture stderr output and/or csvlogs into log files.
```

Picking through those results:

 - The `log_destination` is set to `stderr` which is what we want.
 - `log_line_prefix` is set to `%t` which is ok, but a better value is `'%m [%p] '` but can only be changed by modifying the `postgresql.conf` file, more on that below.
 - `log_min_duration_statement` is set to `-1` which means no queries are logged.  For now, a better value is 0 to log all queries.  Later we'll change this to a higher value `10000` to log any queries which take longer than 10 seconds to run.
 - `logging_collector` is `off` and needs to be turned on so the logs are written out to the destination set by `log_destination`.


Ideally these are all the values which are suggested to be set when installing `pgbadger` with `homebrew`:

```
  log_destination = 'stderr'
  log_line_prefix = '%t [%p]: [%l-1] user=%u,db=%d '
  log_statement = 'none'
  log_duration = off
  log_min_duration_statement = 0
  log_checkpoints = on
  log_connections = on
  log_disconnections = on
  log_lock_waits = on
  log_temp_files = 0
  lc_messages = 'C'
```

To make the changes we need run the following queries:

```
ALTER DATABASE bosh SET log_min_duration_statement TO 0;
ALTER DATABASE bosh SET logging_collector TO true;
```

The second query will issue an "Error" which is really just letting you know you need to restart PostgreSQL which we'll be doing after the next step:

```
ERROR:  parameter "logging_collector" cannot be changed without restarting the server
```

Sadly the only way to make the necessary change to the `log_line_prefix` is to manually edit the configuration file on the BOSH Director as the value is [hardcoded in the release](https://github.com/cloudfoundry/bosh/blob/83b1650f88323f5f35ed2bca3059d2bc27642f36/jobs/postgres-9.4/templates/postgresql.conf.erb#L11).  The file `/var/vcap/jobs/postgres-9.4/config/postgresql.conf` should be modified to look like:

```
data_directory = '/var/vcap/store/postgres-9.4'

listen_addresses = '127.0.0.1'
port = 5432
max_connections = 200

ssl = false

shared_buffers = 32MB

log_line_prefix = '%m [%p] '     #<<<< Updated this line

datestyle = 'iso, mdy'

lc_messages = 'en_US.UTF-8'
lc_monetary = 'en_US.UTF-8'
lc_numeric = 'en_US.UTF-8'
lc_time = 'en_US.UTF-8'

default_text_search_config = 'pg_catalog.english'
```

The trailing whitespace after the right bracket in `'%m [%p] '` is purposely there.

To restart Postgres cleanly leverage `monit`:

```
bosh=# \q
bosh/cd832504-0cb1-4f24-bd15-25d531c1307f:~$ sudo -i
bosh/cd832504-0cb1-4f24-bd15-25d531c1307f:~# monit restart all
```

Before moving on, why did we use `ALTER DATABASE` commands?  Those configuration changes could have been done in the `postgresql.conf` file but that would have meant a redeploy to BOSH supplying those spec values to make them persistent.  With the `ALTER DATABASE` commands those configuration parameter values are stored inside the database and persist stemcell upgrades, BOSH "recreates" and other BOSH maintenance until you reset the values inside of the database.  See the `RESET` parameter in `Example 1` here [https://www.starkandwayne.com/blog/where-is-postgresql-getting-that-ing-configuration-parameter-from/](https://www.starkandwayne.com/blog/where-is-postgresql-getting-that-ing-configuration-parameter-from/) to undo setting the values.

## Find the logs

Normally PostgreSQL emits logs to a sub folder defined by the configuration parameter log_directory in the primary data folder defined by data_directory:

```
bosh=# SELECT name, setting, short_desc
bosh-# FROM pg_settings
bosh-# WHERE name IN ('log_directory', 'data_directory');
      name      |           setting            |                  short_desc
----------------+------------------------------+-----------------------------------------------
 data_directory | /var/vcap/store/postgres-9.4 | Sets the server's data directory.
 log_directory  | pg_log                       | Sets the destination directory for log files.
```

On the filesystem you will notice that the folder is empty:

```
# ls -alh /var/vcap/store/postgres-9.4/pg_log
total 8.0K
drwxr-xr-x  2 vcap vcap 4.0K Oct  7 17:39 .
drwx------ 19 vcap vcap 4.0K Oct 22 16:31 ..
```

So, where are the log files?  The BOSH release is routing the stderr logs to `/var/vcap/sys/log/postgres-9.4/`:

```
# ls -alh /var/vcap/sys/log/postgres-9.4
total 58M
drwxrwx---  2 vcap vcap 4.0K Oct 22 12:15 .
drwxr-x--- 12 root vcap 4.0K Oct 10 18:18 ..
-rw-------  1 vcap vcap  11K Oct 22 16:31 bpm.log
-rw-------  1 vcap vcap  40M Oct 22 16:42 postgres-9.4.stderr.log
-rw-------  1 vcap vcap 2.7M Oct 22 12:15 postgres-9.4.stderr.log.1.gz
-rw-------  1 vcap vcap 2.7M Oct 22 06:15 postgres-9.4.stderr.log.2.gz
-rw-------  1 vcap vcap 2.7M Oct 22 00:15 postgres-9.4.stderr.log.3.gz
-rw-------  1 vcap vcap 2.7M Oct 21 18:15 postgres-9.4.stderr.log.4.gz
-rw-------  1 vcap vcap 2.7M Oct 21 12:15 postgres-9.4.stderr.log.5.gz
-rw-------  1 vcap vcap 2.7M Oct 21 06:15 postgres-9.4.stderr.log.6.gz
-rw-------  1 vcap vcap 2.7M Oct 21 00:15 postgres-9.4.stderr.log.7.gz
-rw-------  1 vcap vcap 1.6K Oct 22 16:31 postgres-9.4.stdout.log
-rw-r-----  1 root root    0 Oct 10 18:18 pre-start.stderr.log
-rw-r-----  1 root root   25 Oct 10 18:18 pre-start.stdout.log
```

The `postgres-9.4.stderr.log*` files are the ones we want to run through pgBadger.

## Copy Logs Locally

First we need to change the file permissions a bit so bosh scp will work:

```
chmod 644 /var/vcap/sys/log/postgres-9.4/postgres-9.4.stderr*
```

Now from your laptop:

```
mkdir pgbadger-bosh-logs && cd pgbadger-bosh-logs

bosh -e master_bosh -d maybe_broken_bosh \
  scp bosh:"/var/vcap/sys/log/postgres-9.4/postgres-9.4.stderr.log*" . \
  --recursive
  
gunzip postgres-9.4.stderr.log.*.gz
```

What you should be left with is a folder with your unzipped log files:

```
➜  pgbadger-bosh-logs git:(master) ✗ ls -alh
total 635632
drwxr-xr-x   9 chris  staff   288B Oct 22 14:12 .
drwxr-xrwx  59 chris  staff   1.8K Oct 22 14:09 ..
-rw-r--r--   1 chris  staff   1.5M Oct 22 14:10 postgres-9.4.stderr.log
-rw-r--r--   1 chris  staff    52M Oct 22 14:10 postgres-9.4.stderr.log.1
-rw-r--r--   1 chris  staff    52M Oct 22 14:10 postgres-9.4.stderr.log.2
-rw-r--r--   1 chris  staff    51M Oct 22 14:10 postgres-9.4.stderr.log.3
-rw-r--r--   1 chris  staff    51M Oct 22 14:10 postgres-9.4.stderr.log.4
-rw-r--r--   1 chris  staff    52M Oct 22 14:10 postgres-9.4.stderr.log.5
-rw-r--r--   1 chris  staff    52M Oct 22 14:10 postgres-9.4.stderr.log.6
-rw-r--r--   1 chris  staff    52M Oct 22 14:10 postgres-9.4.stderr.log.7
```

## Run the Logs Through pgBadger

Now that you have the logs copied down locally you can run them through pgBadger.  In the folder with  the log files run the following:

```
pgbadger postgres-9.4.stderr.log* -f stderr \
            --prefix='%m [%p] ' --jobs 4 -o \
            pgbadger$(date +"%Y_%m_%d_at_%H_%M").html
```

This will will parse your logs into a nicely formatted report in the current folder.  A description of the various categories within the report are documented at [https://severalnines.com/database-blog/postgresql-log-analysis-pgbadger]( https://severalnines.com/database-blog/postgresql-log-analysis-pgbadger).  That website will also point out we could have turned on logging for locks and vacuums in the `PostgreSQL Logging Setup` section.  You can turn these on with `ALTER DATABASE...` commands like we did for `log_min_duration_statement`

I've included a pgBadger report for a BOSH Director PostgreSQL database which you can look at to see an example of the report at [https://bitbucket.org/cweibel/ghost_blog_pics/downloads/pgbadger2019_10_23_at_09_48.html](https://bitbucket.org/cweibel/ghost_blog_pics/downloads/pgbadger2019_10_23_at_09_48.html)

The data in the report should be able to identify if there are any areas of the database which are experiencing difficulties.  Pay particular attention to the `Top `> `Slowest Individual Queries` for problem queries pointing to bad indexes or explain plans.  Also look at the `Top` > `Most frequent queries` to see which queries are run most frequently because if these queries start taking a tad longer than normal the effect is compounded quickly.


## Other Interesting Databases

This same scenario can be replayed for the databases in Cloud Foundry.  Connect to the database, make the necessary configuration parameter changes and run the resulting logs through pgBadger.  Want to know why the CF CLI is running slow for some commands?  Take a peak at the `ccdb` database and see if there are slow queries which may have changed explain plans.

## Final Thoughts

Once you have figured out the source of your problem I would recommend generating a report on a regular schedule but modify `log_min_duration_statement` to catch slow moving queries by performing:

```
ALTER DATABASE bosh SET log_min_duration_statement TO 10000;  # 10 Seconds
```

This will cut down on the amount of logging done but still help to point to the fire in the database when you smell smoke.

Oh, and be good to badgers!