---
layout: post
title: "BOSH Director Time Drift"
date: 2017-01-18
---

![map](https://raw.githubusercontent.com/cweibel/ghost_blog_pics/master/steve-smith-06RtG58gEog-unsplash.jpg)



Photo by [Steve Smith](https://unsplash.com/@varrak?utm_source=unsplash&utm_medium=referral&utm_content=creditCopyText) on [Unsplash](https://unsplash.com/s/photos/uri?utm_source=unsplash&utm_medium=referral&utm_content=creditCopyText)

Earlier today I was greeted with this message while performing a BOSH deployment for SHIELD:


> Error 100: Unknown CPI error 'Unknown' with message 'AWS was not able to validate the provided access credentials'


I double checked the AWS Access and Secret permissions. I even deployed from my laptop a test deployment with the same keys with no issue. So the problem was not with the keys.

After digging around I found [this article](https://forums.aws.amazon.com/thread.jspa?messageID=722197) which pointed me to check the time on the BOSH Director. Why? The BOSH AWS CPI is using the AWS CLI to perform actions which leverage a token which is time sensitive.

To verify compare the time at Google to the local server:

```
root:~# date
Wed Jan 18 15:30:50 UTC 2017

root@:~# sudo date -s "$(wget -qSO- --max-redirect=0 google.com 2>&1 | grep Date: | cut -d' ' -f5-8)Z"
Wed Jan 18 15:27:50 UTC 2017
```

The BOSH Director was living 3 minutes in the future. A quick resetting of the clock:

```
sudo date --set="2017-01-18 15:30:00.000" 
```

Running `bosh deploy` again ran successfully after the change.

The long term solution is to figure out why NTP didn't keep the date in sync. By default there is a cron job for root which is supposed to update ntp every 15 minutes:

```
root:~ crontab -l
0,15,30,45 * * * * /var/vcap/bosh/bin/ntpdate
0 0 * * * /var/vcap/jobs/director/bin/task_logrotate >>/var/vcap/sys/log/director/task_logrotate.log 2>&1
```

I'll update this post once I figure out what the issue is with the two defined NTP servers.