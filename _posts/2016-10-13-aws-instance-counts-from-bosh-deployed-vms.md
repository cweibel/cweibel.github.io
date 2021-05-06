---
layout: post
title: "AWS Instance Counts From BOSH Deployed VMs"
date: 2016-10-13
---

![map](https://raw.githubusercontent.com/cweibel/ghost_blog_pics/master/steve-smith-Emj_SMW_D9g-unsplash.jpg)



Photo by [Steve Smith](https://unsplash.com/@varrak?utm_source=unsplash&utm_medium=referral&utm_content=creditCopyText) on [Unsplash](https://unsplash.com/s/photos/uri?utm_source=unsplash&utm_medium=referral&utm_content=creditCopyText)


Ever wonder how many of the VMS in AWS Console were deployed via BOSH? How about the number of instance types so you can start using reserved instances?

The script below will show you the number of BOSH deployed VMS you have for each defined director on AWS.

A few assumptions:

 - You can log into each BOSH Director from a single jump box
 - You have a BOSH Client credentials created
 - You have AWS credentials (hint: find your `bosh-init` manifest)

In the example below there are two directors called `useast2-a` and `useast2-b`, each has a set of unique credentials for BOSH and AWS.

```
function set_env_useast2_a {
  export REPORT_ENV="useast2_a"
  export BOSH_CLIENT=reporting
  export BOSH_CLIENT_SECRET=boshSecret
  export AWS_ACCESS_KEY_ID=AKIANICETRYNOTSODUMB
  export AWS_SECRET_ACCESS_KEY="viooijijobniAoaieuAjblisdfiueBoiuwp8eufg"
  export AWS_DEFAULT_REGION="us-east-2"
  bosh target 192.168.0.2
}

function set_env_useast2_b {
  export REPORT_ENV="useast2_b"
  export BOSH_CLIENT=reporting
  export BOSH_CLIENT_SECRET=boshSecret2
  export AWS_ACCESS_KEY_ID=AKIANICETRYNOTSOTUFF
  export AWS_SECRET_ACCESS_KEY="viooijijobniAoaieuAjblasdfffdadgggdeehgb"
  export AWS_DEFAULT_REGION="us-east-2"
  bosh target 192.168.0.3
}

function generate_files {
  list_of_instances=$(bosh vms --details | grep " i-" | cut -d '|' -f 7)
  aws ec2 describe-instances --instance-ids ${list_of_instances} --query 'Reservations[*].Instances[*].[Placement.AvailabilityZone, InstanceType]' --output text > ${REPORT_ENV}.out
  sleep 1
  aws ec2 describe-instances --instance-ids ${list_of_instances} --query 'Reservations[*].Instances[*].[Placement.AvailabilityZone, InstanceType]' --output text >> ${AWS_DEFAULT_REGION}.out
  cat ${REPORT_ENV}.out | sort -nr | uniq -c > ${REPORT_ENV}.count.out
  cat ${AWS_DEFAULT_REGION}.out | sort -nr | uniq -c > ${AWS_DEFAULT_REGION}.count.out
}

for set_env in useast2_a useast2_b; do
  eval set_env_${set_env}
  generate_files
done

for aws_region in "us-east-2"; do   # <=== Add more regions here if you have more than 1
  echo "Region: ${aws_region}"
  cat ${aws_region}.count.out
done
```

Note that you can extend the script to handle multiple regions by simply adding any uniquely defined `AWS_DEFAULT_REGIONs` in the final `for` loop. You will also want to wipe out the *.out files between runs.