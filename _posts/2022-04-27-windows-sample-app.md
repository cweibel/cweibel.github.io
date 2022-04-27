---
layout: post
title: "Sample Windows Cloud Foundry App"
date: 2022-04-27
---

## Has it really been under your nose all along?

![pic](https://raw.githubusercontent.com/cweibel/ghost_blog_pics/master/sydney-rae--jRJJMr5vBE-unsplash.jpg)

Photo by [sydney Rae](https://unsplash.com/@srz?utm_source=unsplash&utm_medium=referral&utm_content=creditCopyText) on [Unsplash](https://unsplash.com)



Ever try to find a really simple Windows app to test against Cloud Foundry Windows Cells?  

Sometimes the most obvious answer is right under your nose.  Inside of the [`cf-smoke-tests`](https://github.com/cloudfoundry/cf-smoke-tests) are the tests used by Cloud Foundry which are safe to run against production.  

In general the tests work by creating a test org, space, quota, pushes an app, scales it, retrieve logs and tear it all back down.  There are tests for both the `cflinuxfs3` and `windows2019` which require cells of each type to exist in the Cloud Foundry deployment. 

What all this means is there is a simple Windows Cloud Foundry app inside of the smoke tests, here is how you use it:

```
git clone https://github.com/cloudfoundry/cf-smoke-tests.git
cd cf-smoke-tests/assets/dotnet_simple/Published

cf push imarealwindowsapp -s windows -b hwc_buildpack
```

In the example above we clone the repo and push an app called `imarealwindowsapp`, feel free to use whatever name you'd like.  To get the url of the app once it is deployed, run the following command and note the `routes:`:

```
$ cf app imarealwindowsapp
Showing health and status for app imarealwindowsapp in org system / space ops as admin...

name:              imarealwindowsapp
requested state:   started
routes:            imarealwindowsapp.apps.codex.starkandwayne.com
last uploaded:     Wed 27 Apr 17:05:55 UTC 2022
stack:             windows
buildpacks:        hwc

type:           web
instances:      1/1
memory usage:   1024M
     state     since                  cpu    memory         disk          details
#0   running   2022-04-27T17:07:00Z   0.1%   100.5M of 1G   44.8M of 1G
```

To test whether or not it was successful, you can curl the endpoint adding `https://` to the `routes:` value from the last command output:

```
$ curl https://imarealwindowsapp.apps.codex.starkandwayne.com -k

Healthy
It just needed to be restarted!
My application metadata: {"application_id":"b55b34e2-c434-4782-b44e-3f9f469dd70c","application_name":"imarealwindowsapp","application_uris":["imarealwindowsapp.apps.codex.starkandwayne.com"],"application_version":"1bd0703a-4f13-45c8-86cb-0632db5cd6bd","cf_api":"https://api.system.codex.starkandwayne.com","host":"0.0.0.0","instance_id":"f56eaa45-cad2-4ab8-6e75-1ea9","instance_index":0,"limits":{"disk":1024,"fds":16384,"mem":1024},"name":"imarealwindowsapp","organization_id":"d396b0c6-872f-46a2-a752-bdea51819c06","organization_name":"system","port":8080,"process_id":"b55b34e2-c434-4782-b44e-3f9f469dd70c","process_type":"web","space_id":"4e081328-2ac1-4509-8f51-ffcbfc012165","space_name":"ops","uris":["imarealwindowsapp.apps.codex.starkandwayne.com"],"version":"1bd0703a-4f13-45c8-86cb-0632db5cd6bd"}
My port: 8080
My instance index: 0
My custom env variable:

```

Finally, if you look at the logs you'll see that the app emits a timestamp tick every second, which is what the smoke tests look for to validate logging is working:

```
$ cf logs imarealwindowsapp
Retrieving logs for app imarealwindowsapp in org system / space ops as admin...

   2022-04-27T17:11:15.44+0000 [APP/PROC/WEB/0] OUT Tick: 1651079475
   2022-04-27T17:11:16.45+0000 [APP/PROC/WEB/0] OUT Tick: 1651079476
   2022-04-27T17:11:17.46+0000 [APP/PROC/WEB/0] OUT Tick: 1651079477
   2022-04-27T17:11:18.47+0000 [APP/PROC/WEB/0] OUT Tick: 1651079478
   2022-04-27T17:11:19.47+0000 [APP/PROC/WEB/0] OUT Tick: 1651079479
```

If you are curious on how to use this in a bosh errand to run the complete Cloud Foundry Windows Smoke Tests, be sure to visit [https://cweibel.github.io/blog/2022/04/05/adding-windows-smoke-tests-to-cloud-foundry](https://cweibel.github.io/blog/2022/04/05/adding-windows-smoke-tests-to-cloud-foundry)


Enjoy!  
