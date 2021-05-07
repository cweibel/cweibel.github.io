---
layout: post
title: "How To Install Splunk and Boundary Plugin on CentOS 6.5"
date: 2014-06-23
---

![map](https://raw.githubusercontent.com/cweibel/ghost_blog_pics/master/rene-porter-fnZs1yDS8JY-unsplash.jpg)


Photo by [René Porter](https://unsplash.com/@reneporter?utm_source=unsplash&utm_medium=referral&utm_content=creditCopyText) on [Unsplash](https://unsplash.com/s/photos/uri?utm_source=unsplash&utm_medium=referral&utm_content=creditCopyText)


[Author's Note in 2021: This is old and should not be used for modern deployments as-is but is kept for historical reasons]

## Basic Setup

As root execute the following to download and install some basic linux apps:

```
	sudo su -
	yum install git wget vim htop lsof iftop -y
```

## Download the Installer

Now we will download the appropriate linux installer for Splunk

> wget -O splunk-6.1.1-207789-linux-2.6-x86_64.rpm 'http://www.splunk.com/page/download_track?file=6.1.1/splunk/linux/splunk-6.1.1-207789-linux-2.6-x86_64.rpm&ac=&wget=true&name=wget&platform=Linux&architecture=x86_64&version=6.1.1&product=splunk&typed=release';

New or additional downloads are available at [http://www.splunk.com/download?r=header](http://www.splunk.com/download?r=header)

## Run the installer

> rpm -i splunk-6.1.1-207789-linux-2.6-x86_64.rpm 

## Start the Splunk daemon and application

Using `--accept-license` allows for a silent startup with no user intervention required

> ./opt/splunk/bin/splunk start --accept-license

## Splunk Scheduler

Schedule Splunk to start whenever the server is rebooted.

> ./opt/splunk/bin/splunk enable boot-start -user root

## Access the UI

You will now be able to access the UI by navigating to from a browser:

> `http://ip_address_of_server:8000`

## Configure Splunk Listenter

To configure Splunk to listen on TCP 514 so that logs can be sent to the server

While still logged into the UI, navigate to Settings > Data Inputs > TCP > New

 - Under "TCP port" enter "514"
 - Under "Set Sourcetype" select "Manual"
 - In "Source type" enter "syslog"

Then click "Save", Splunk should now be listening on port 514


### Firewall Notes:

If this server is remote to you you may need to open the firewall for TCP 8000, for simple proof of life you can disable the firewall by executing

/etc/init.d/iptables stop

### Install Boundary Plugin

Now, to install the Boundary plugin for Splunk, clone the repo to a specific folder

```
cd /opt/splunk/etc/apps
git clone https://github.com/boundary/boundary_splunk_app.git
mv boundary_splunk_app/ boundary/
```


### Restart Splunk

Now restart Splunk so the Boundary plugin is loaded.

```
./opt/splunk/bin/splunk stop
./opt/splunk/bin/splunk start
```

### Install Meters

Now for any servers which you want to install the Boundary Meter, log onto that server as root, download the script and add your credentials

```
curl -3 -s https://app.boundary.com/assets/downloads/setup_meter.sh > setup_meter.sh
chmod +x setup_meter.sh
./setup_meter.sh -d -i <organization>:<organization id>
```

### Boundry Credentials

To get you Boundary credentials for the previous step, log into Boundary:

 - In the User drop down in the upper right corner select "User Settings" which should be at [https://app.boundary.com/account](https://app.boundary.com/account)
 - Your organization keys should be listed on this screen.


**Special Note: Jeremy - this is where you use Bill's credentials and each stack is associated with a particular organization**