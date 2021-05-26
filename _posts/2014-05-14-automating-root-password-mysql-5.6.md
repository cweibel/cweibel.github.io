---
layout: post
title: "Automating Root Password on MySQL 5.6"
date: 2014-05-18
---

![map](https://raw.githubusercontent.com/cweibel/ghost_blog_pics/master/steve-smith-3bus5RKqlOs-unsplash.jpg)



Photo by [Steve Smith](https://unsplash.com/@varrak?utm_source=unsplash&utm_medium=referral&utm_content=creditCopyText) on [Unsplash](https://unsplash.com/s/photos/uri?utm_source=unsplash&utm_medium=referral&utm_content=creditCopyText)


So, I spent an unreasonable amount of time attempting to automate the install of MySQL 5.6 for a client. Apparently one of the newer features in MySQL is the creation of a temporary password file when MySQL is first intalled. I want to use this password file to reset the root password as the root user is locked out from all functionality until the password is reset. I ran into the same problem as this user: [http://forums.mysql.com/read.php?11,584314,584314#REPLY](http://forums.mysql.com/read.php?11,584314,584314#REPLY)

I found a work around but couldn't reply to that forum (closed), thus I'm documenting it here!

## So What Do I Do With The .mysql_secret File?

Use it to change the root login password for the db. That's it. You can't do anything else with the root login until the password is reset.

If you view the contents of the file you will see what the temporary password is for the root user.

```
[root@myserver html]# cat /root/.mysql_secret
# The random password set for the root user at Wed May 14 14:06:38 2014 (local time): th09GAcGAnfVP_t4
```

You won't have the same password as seen above.

To cleanly extract your password for bash scripting use:

```
#!/usr/bin/env bash
mysql_secret=$(awk '/password/{print $NF}' /root/.mysql_secret)
```

## Automating Changing The Password

Here is the whole solution:

```
#!/usr/bin/env bash
mysql_secret=$(awk '/password/{print $NF}' /root/.mysql_secret)
mysqladmin -u root --password=${mysql_secret} password myNewCoolPassword
```

Now you can use the root login to create databases/tables/views/and other nifty database objects


## Pitfalls

You cannot use mysql to change the password on this account for the first time when passing in a SET PASSWORD query.

```
#!/usr/bin/env bash
mysql_secret=$(awk '/password/{print $NF}' /root/.mysql_secret)
mysql -u root --password=$mysql_secret -e "SET PASSWORD = PASSWORD('mynewpassword');"
```

Results in an initially unhelpful error message:

```
#!/usr/bin/env bash
ERROR 1862 (HY000): Your password has expired. To log in you must change it using a client that supports expired passwords.
```

Thanks Wayne and Tyler for coming up with the solution.