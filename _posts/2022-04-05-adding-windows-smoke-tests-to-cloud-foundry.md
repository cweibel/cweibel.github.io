---
layout: post
title: "Adding Windows Smoke Tests to Cloud Foundry"
date: 2022-04-05
---

## Opening A Window To Clear the Smoke (Tests)

![pic](https://raw.githubusercontent.com/cweibel/ghost_blog_pics/master/ahmed-zayan-kQebUnFAOos-unsplash-2.jpg)

Photo by [Ahmed Zayan](https://unsplash.com/@zayyerrn?utm_source=unsplash&utm_medium=referral&utm_content=creditCopyText) on [Unsplash](https://unsplash.com)

Cloud Foundry has supported running Windows Diego Cells for a few years now but until recently I had not had a reason to use them.  

The [instructions](https://github.com/cloudfoundry/cf-deployment/blob/main/operations/README.md) for modifying `cf-deployment` is fairly straight forward for adding ops files to enable Windows:

 - [windows2019-cell.yml](https://github.com/cloudfoundry/cf-deployment/blob/e20e11dd243f57555e3e0ad281e6797fbfde8152/operations/windows2019-cell.yml) - This is the starting point for adding a Windows Deigo Cell instance group.
 - [use-online-windows2019fs.yml](https://github.com/cloudfoundry/cf-deployment/blob/e20e11dd243f57555e3e0ad281e6797fbfde8152/operations/use-online-windows2019fs.yml) - This configures where to pull down the `windows2019fs` from the interwebs.  There is an `offline` variant as well, but I was not able to get this to work.
 - [use-latest-windows2019-stemcell.yml](use-latest-windows2019-stemcell.yml) - Exactly what it sounds like


What was missing?  I couldn't find a way to run smoke tests against the Windows Diego Cells.  The support for Windows exists in the [`cf-smoke-tests`](https://github.com/cloudfoundry/cf-smoke-tests-release/tree/main/jobs/smoke_tests_windows) bosh release, so a quick copy of the existing `smoke_tests` job from `cf-deployment.yml`, adding `enable_windows_tests: true` and `windows_stack: windows2016` and a whiff of smoke, here is the ops file that can be included with cf-deployment:

```
- path: /instance_groups/-
  type: replace
  value:
    azs:
    - z1
    instances: 1
    jobs:
    - name: smoke_tests_windows
      properties:
        bpm:
          enabled: true
        smoke_tests:
          enable_windows_tests: true
          windows_stack: windows2016
          api: "https://api.((system_domain))"
          apps_domain: "((system_domain))"
          client: cf_smoke_tests
          client_secret: "((uaa_clients_cf_smoke_tests_secret))"
          org: cf_smoke_tests_org
          space: cf_smoke_tests_space
          cf_dial_timeout_in_seconds: 300
          skip_ssl_validation: true
      release: cf-smoke-tests
    - name: cf-cli-7-linux
      release: cf-cli
    lifecycle: errand
    name: smoke-tests-windows
    networks:
    - name: default
    stemcell: windows2019
    vm_type: minimal
```

Once this is deployed, to run the errand:

```
$ bosh -d cf run-errand smoke_tests_windows

Using environment 'https://192.168.5.56:255555' as user 'admin'
Using deployment 'cf'
Task 134
...
shortened for the sake of scrolling...
...
#############################################################################
      Running smoke tests
      C:\var\vcap\packages\goland-1.13-windows\go\bin\go.exe
      c:\var\vcap\packages\smoke_tests_windows\bin\ginkgo.exe
      [1648831644] - CF-Isolation-Segment-Smoke-Tests - 4/4 specs SSSS SUCESS! 10.8756455s PASS
      [1648831644] - CF-Logging-Smoke-Tests - 2/2 specs ++ SUCESS! 1m5.5649573s PASS
      [1648831644] - CF-Runtime-Smoke-Tests - 2/2 specs ++ SUCESS! 1m9.5844699s PASS
      Ginkgo run 3 suites in 3m9.6523124s
      Tests Suite Passed
      Smoke Tests Complete, exit status 0
Stderr   - 
1 errand(s)
Succedded
```

## Something which doesn't work

If you add the `--keep-alive` to the `bosh run-errand` command, you'll need to rerun the `run-errand` command without the keep-alive option to get subsequent runs of the smoke tests to pass.  Part of the scripting moves (instead of copies) some of the files around, so you only get a single attempt to run the tests for a particular vm instance.

Enjoy!  
