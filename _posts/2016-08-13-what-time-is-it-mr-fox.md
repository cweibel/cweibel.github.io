---
layout: post
title: "BOSH Init on AWS, What Time is it Mr Fox?"
date: 2016-08-03
---

![map](https://raw.githubusercontent.com/cweibel/ghost_blog_pics/master/sunyu-tIfrzHxhPYQ-unsplash.jpg)



Photo by [Sunyu](https://unsplash.com/@sunyu) on [Unsplash](https://unsplash.com/s/photos/uri?utm_source=unsplash&utm_medium=referral&utm_content=creditCopyText)

We ran into an interesting problem today while running `bosh-init` (2021 update: same error, but with `bosh create-env`) against AWS:

> CPI 'has_vm' method responded with error: CmdError{"type":"Unknown","message":"AWS was not able to validate the provided access credentials","ok_to_retry":false}`

This is a CPI error so after a bit of investigation confirmed our AWS keys were correct and valid with the awscli. After some more digging we found that the server we were running the `bosh-init` command from had drifted 7 minutes making the authentication token expired.

Update the ntp server configuration or manually move the minute hand on the server clock and get back to `bosh-init` glory.

You can use the following to sync of your time but best to fix your ntp settings:

```
sudo date -s "$(wget -qSO- --max-redirect=0 google.com 2>&1 | grep Date: | cut -d' ' -f5-8)Z"
date
```