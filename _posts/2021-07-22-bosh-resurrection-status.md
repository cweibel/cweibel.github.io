---
layout: post
title: "Checking the Current Status of BOSH Resurrection"
date: 2021-07-22
---

![confusion](https://raw.githubusercontent.com/cweibel/ghost_blog_pics/master/daniel-tuttle-vBOHVZG6L50-unsplash.jpg)

Photo by [Daniel Tuttle](https://unsplash.com/@danieltuttle?utm_source=unsplash&utm_medium=referral&utm_content=creditCopyText) on [Unsplash](https://unsplash.com)



I recently had an issue where AWS sent notifications that they would be stopping EC2 instances on a certain date and time.  Turns out, they meant it.  

Normally this is no big deal for BOSH deployed VMs.  BOSH would notice the VM is missing and recreate it through the CPI with it's awesome self-healing resurrection feature.

I waited.

Noting happened.

Huh...

Turns out someone had turned off the resurrection feature with the BOSH CLI command:

```
bosh update-resurrection off
```

Ah.  Ok.  To turn resurrection back on I can simply run:

```
bosh update-resurrection on
```

But how do I tell what the **current** state of resurrection is?  Is it on?  Is it off?  BOSH CLI to the rescue? Nope.  There is no BOSH CLI command to retrieve the current value.

## Where is this kept?

Inside the `bosh` database of course!  There is a table called `director_attributes` which contains the UUID of the BOSH Director and the current status of resurrection.


This is what it looks like with resurrection disabled:

```
bosh=# select * from director_attributes;
                value                 |        name         | id
--------------------------------------+---------------------+----
 b0062f10-blah-1224-blah-fd0dcc751234 | uuid                |  1
 true                                 | resurrection_paused |  2
(2 rows)
```

A quick `bosh update-resurrection on` to enable resurrection results in the table looking like:

```
bosh=# select * from director_attributes;
                value                 |        name         | id
--------------------------------------+---------------------+----
 b0062f10-blah-1224-blah-fd0dcc751234 | uuid                |  1
 false                                | resurrection_paused |  2
(2 rows)
```

Side note:  turning resurrection `on` means the table value is really `false`.  Resurrection `off` means the table value is really `true`.  Having the field being `resurrection_paused` caused me to take a double take to make sure I had those values correct.

Second note: if there is no row for resurrection that means that this value has not been set yet.  As soon as you run the `bosh update-resurrection` command the row will be created.