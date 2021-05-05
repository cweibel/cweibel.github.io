---
layout: post
title: "Where is PostgreSQL Getting That $%&^ing Configuration Parameter From?"
date: 2019-09-20
---

![map](https://raw.githubusercontent.com/cweibel/ghost_blog_pics/master/steve-smith-Cn4UdXazVmM-unsplash.jpg)


Photo by [Steve Smith](https://unsplash.com/@varrak?utm_source=unsplash&utm_medium=referral&utm_content=creditCopyText) on [Unsplash](https://unsplash.com/s/photos/uri?utm_source=unsplash&utm_medium=referral&utm_content=creditCopyText)

Like a squirrel looking for a lost nut, finding where a configuration parameter is being set in PostgreSQL can be a pain in the (fluffy) tail.

For example, the `log_min_duration_statement` which is used to configure the threshold duration of a query before it is logged can be set in multiple ways.  Here are 7 ways I know of to set the values:

 1. Add `log_min_duration_statement = 0` to `postgresql.conf`
 1. Add `log_min_duration_statement = 0` to a conf file declared in `postgresql.conf` with `include =`
 1. Add `log_min_duration_statement = 0` to a conf file in a folder declared in `postgresql.conf` with `include_dir`
 1. Add the value to the command line when starting Postgres
 1. Use an `ALTER SYSTEM` command, this will set the value in `postgresql.auto.conf`
 1. Use an `ALTER DATABASE` command which will set the value for the current database
 1. Use `set_config` to set the value for the current transaction or session

I *believe* I have these in the correct override order, so if the value was set in all 7 locations, the value in `set_config` would be used.  Let me know in the comments if there are other ways of setting the value.

## Look up the current value

So how do you know what the current values is of a particular configuration parameter?

There is a wonderful system view which stores where ALL the configuration parameters are coming from called `pg_settings`.  Let's take a look at a particular configuration parameter `log_min_duration_statement` to decipher how and where the value is being set:

### Example 1

We'll start with a non-typical way of assigning our example configuration parameter:

```
mydb=# select * from pg_settings where name = 'log_min_duration_statement';
-[ RECORD 1 ]---+-----------------------------------------------------------------------
name            | log_min_duration_statement
setting         | 60000
unit            | ms
category        | Reporting and Logging / When to Log
short_desc      | Sets the minimum execution time above which statements will be logged.
extra_desc      | Zero prints all queries. -1 turns this feature off.
context         | superuser
vartype         | integer
source          | database
min_val         | -1
max_val         | 2147483647
enumvals        |
boot_val        | -1
reset_val       | 60000
sourcefile      |
sourceline      |
pending_restart | f
```

From the above we can see the current value is `60000ms` (setting & unit), the value is an integer (vartype) and the source is `database`.   `database`, in this context means the value was set with an `ALTER DATABASE` command (#6 in our list):

```
ALTER DATABASE mydb SET log_min_duration_statement TO 60000;
```

So any value set in `postgresql.conf`, config files, config folders, startup parameters and any `ALTER SYSTEM` settings were all ignored.

Hint: if you want to undo this customization run a RESET command while connected to the database:

```
ALTER DATABASE mydb RESET log_min_duration_statement;
```

You'll need to reset your client connection to see the change take effect.

This configuration is per database, which can be a bit confusing, if you run the exact same query against a different database, such as the `postgres` database (see the next example), you may see that the value comes from a completely different source.

### Example 2

In the next example the value is set in a configuration file:

```
postgres=# select * from pg_settings where name = 'log_min_duration_statement';
-[ RECORD 1 ]---+------------------------------------------------------------------------------------
name            | log_min_duration_statement
setting         | 120000
unit            | ms
category        | Reporting and Logging / When to Log
short_desc      | Sets the minimum execution time above which statements will be logged.
extra_desc      | Zero prints all queries. -1 turns this feature off.
context         | superuser
vartype         | integer
source          | configuration file
min_val         | -1
max_val         | 2147483647
enumvals        |
boot_val        | -1
reset_val       | 120000
sourcefile      | /var/postgres/bin/postgresql.conf
sourceline      | 623
pending_restart | f
```

Specifically the configuration value `120000ms` (setting & unit) is coming from the file `/var/postgres/bin/postgresql.conf` (sourcefile) from line `623` (sourceline).

This type of configuration where the value is set in a configuration file such as `postgresql.conf` is the most common.

### Example 3

This one is set in a configuration file, but with a twist:

```
postgres=# select * from pg_settings where name = 'log_min_duration_statement';
-[ RECORD 1 ]---+------------------------------------------------------------------------------------
name            | log_min_duration_statement
setting         | 30000
unit            | ms
category        | Reporting and Logging / When to Log
short_desc      | Sets the minimum execution time above which statements will be logged.
extra_desc      | Zero prints all queries. -1 turns this feature off.
context         | superuser
vartype         | integer
source          | configuration file
min_val         | -1
max_val         | 2147483647
enumvals        |
boot_val        | -1
reset_val       | 30000
sourcefile      | /var/postgres/bin/postgresql.auto.conf
sourceline      | 3
pending_restart | f
```

Our configuration parameter is listed as a configuration file (source) but the sourcefile points to a `postgresql.auto.conf` which means the value was set with an ALTER SYSTEM statement similar to:

```
ALTER SYSTEM SET log_min_duration_statement = 30000;
```

When changing this value you need to reset the client connection to see the change go into effect.

There is a slightly different RESET command to undo this customization:

```
ALTER SYSTEM RESET log_min_duration_statement;
```

### Example 4

In the final example the configuration parameter is assigned with a `set_config()` function from `pg_settings` looks like:

```
postgres=# select * from pg_settings where name = 'log_min_duration_statement';                                                                                                                                                     -[ RECORD 1 ]---+-----------------------------------------------------------------------
name            | log_min_duration_statement
setting         | 180000
unit            | ms
category        | Reporting and Logging / When to Log
short_desc      | Sets the minimum execution time above which statements will be logged.
extra_desc      | Zero prints all queries. -1 turns this feature off.
context         | superuser
vartype         | integer
source          | session
min_val         | -1
max_val         | 2147483647
enumvals        |
boot_val        | -1
reset_val       | 30000
sourcefile      |
sourceline      |
pending_restart | f
```

The configuration value is listed as `session` and none of the other `source*` fields are populated which means it was set with a SQL query similar to:

```
SELECT set_config('log_min_duration_statement', '180000', false);
```
To reset the value simply terminate the current connection and reconnect the client.

## Final Thoughts

The `pg_settings` view will show you the current configuration parameter values be used, hopefully this article helped to decipher how these values are being set! 

If not, yell at your monitor like I did for 15 minutes :) 