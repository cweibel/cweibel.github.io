---
layout: post
title: "How to Reload Buildpacks in Cloud Foundry"
date: 2018-08-22
---

![map](https://raw.githubusercontent.com/cweibel/ghost_blog_pics/master/ricardo-viana--tYsPFKMm7g-unsplash.jpg)



Photo by [Ricardo Viana](https://unsplash.com/@ricardoviana?utm_source=unsplash&utm_medium=referral&utm_content=creditCopyText) on [Unsplash](https://unsplash.com/s/photos/uri?utm_source=unsplash&utm_medium=referral&utm_content=creditCopyText)

## Made A Mess of Your Buildpacks?

If you have made a mess of your default system buildpacks and want to delete and reload them you can in just a few simple steps.

### Step 1 - Remove existing Buildpacks

Start by logging into the CF CLI with cloud_controller.admin permissions, once there list the existing buildpacks:

```
cf buildpacks
```

Now you can loop through and remove the buildpacks, adjust the names as needed to your output paying attention to `-` versus `_`:

```
cf delete-buildpack staticfile_buildpack -f
cf delete-buildpack java_buildpack -f
cf delete-buildpack ruby_buildpack -f
cf delete-buildpack nodejs_buildpack -f
cf delete-buildpack go_buildpack -f
cf delete-buildpack python_buildpack -f
cf delete-buildpack php_buildpack -f
cf delete-buildpack binary_buildpack -f
cf delete-buildpack dotnet_core_buildpack -f
```

Verify that the buildpacks have been removed by running another `cf buildpacks`

### Step 2 - Modify API VM

Start by BOSH ssh'ing into any of the API nodes and navigate to `/var/vcap/jobs/cloud_controller_ng/bin`

Make a copy of the `post-start` script, we'll be editing the file and will need to put it back when done:

```
cp post-start post-start.orig
```

Now modify the `post-start` file to contain:

```
#!/usr/bin/env bash

set -ex
export LANG="en_US.UTF-8"

CC_JOB_DIR="/var/vcap/jobs/cloud_controller_ng"
CONFIG_DIR="${CC_JOB_DIR}/config"
CC_PACKAGE_DIR="/var/vcap/packages/cloud_controller_ng"

export CLOUD_CONTROLLER_NG_CONFIG="${CONFIG_DIR}/cloud_controller_ng.yml"
export BUNDLE_GEMFILE="${CC_PACKAGE_DIR}/cloud_controller_ng/Gemfile"

source "${CC_JOB_DIR}/bin/ruby_version.sh"
source /var/vcap/packages/capi_utils/syslog_utils.sh
tee_output_to_sys_log "cloud_controller_ng.$(basename "$0")"

function install_buildpacks {

  pushd "${CC_PACKAGE_DIR}/cloud_controller_ng" > /dev/null
    chpst -u vcap:vcap bundle exec rake buildpacks:install

    if [[ $? -ne 0 ]]; then
      echo "Buildpacks installation failed"
      exit 1
    fi
  popd > /dev/null

}

function main {
  install_buildpacks
}

main

exit 0
```

### Step 3 - Verify and Cleanup

Now run the `post-start` script.  Once it is complete you can validate everything went well by running a `cf buildpacks` command.

Perform a `bosh recreate api` on the deployment or remove your temporary `post-start` script as to not cause issues with next deployment:

```
cd /var/vcap/jobs/cloud_controller_ng/bin
rm post-start
mv post-start.orig post-start
```

At this point you have your refreshed buildpacks and fixed your API node. Enjoy!