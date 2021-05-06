---
layout: post
title: "Using AWS CLI with IAM Instance Profile"
date: 2017-10-24
---

![yoda_koala](https://raw.githubusercontent.com/cweibel/ghost_blog_pics/master/henrique-felix-yADA15HFGxM-unsplash.jpg)


Photo by [Henrique Félix](https://unsplash.com/@henriquefelix?utm_source=unsplash&utm_medium=referral&utm_content=creditCopyText) on [Unsplash](https://unsplash.com/?utm_source=unsplash&utm_medium=referral&utm_content=creditCopyText)

Every once in a while you will find an organization which will not give you AWS Console access so you have to become handy using the AWS CLI for managing the infrastructure underneath BOSH. Fear not, the CLI can be used to retrieve even more information than the Console.

You will need to retrieve a set of credentials which come in two flavors:

 - AWS Access and Secret keys
 - IAM Profiles

## Retrieving AWS Access and Secret

If you leverage AWS Access and Secret keys they are defined in your deployment manifest for BOSH:

```
cloud_provider:
  properties:
    aws:
      access_key_id: AKIATONYANDBRUCE
      region: us-east-1
      secret_access_key: c9someReallyLongPassword4meR
```

Now you can use this information to configure the AWS CLI which can be installed with [these instructions](https://docs.aws.amazon.com/cli/latest/userguide/installing.html):

```
aws config
AWS Access Key ID []: enter cloud_provider.properties.aws.access_key_id here
AWS Secret Access Key []: enter cloud_provider.properties.aws.secret_access_key here
Default region name []: enter cloud_provider.properties.aws.region here
Default output format [json]: just hit enter
```

Now you can discover ELB (which you would have in front of your CF Routers) information by:

```
aws elb describe-load-balancers
```

If you also want information about your EC2 instances:

```
aws ec2 describe-instances
``` 


## Retrieving AWS IAM Profile

If you are using IAM profiles your deployment manifest for BOSH will be configured similar to:

```
cloud_provider:
  properties:
    aws:
      credentials_source: env_or_profile
      iam_instance_profile: bosh-profile
      region: us-east-1
```

No `access_key_id` or `secret_access_key` here. The AWS CLI can be configured to use a Role but requires a few additional bits of information:

 - a profile name
 - role_arn

To get the `role_arn` ssh onto the jumpbox or BOSH Director with the IAM Profile and run:

```
curl http://169.254.169.254/latest/meta-data/iam/info
```

This will output something similar to the following, with `InstanceProfileArn` containing the value we need:

```
{
  "Code" : "Success",
  "LastUpdated" : "2017-10-24T17:16:15Z",
  "InstanceProfileArn" : "arn:aws:iam::12345678912:instance-profile/bosh-profile",
  "InstanceProfileId" : "AIPANOTBRUCEORTONEY"
}
```

For this example we'll call our profile `prodaccess`. Using the `arn_role` and profile name info you can craft an AWS config file in `~/.aws/config` similar to:

```
[default]
output = json
[profile prodaccess]
profile_arn = arn:aws:iam::12345678912:instance-profile/bosh-profile
source_profile = default
region = us-east-1
```

Now the aws elb command can be executed with the addition of a `--profile` parameter:

aws elb describe-load-balancers --profile prodaccess
Similarly you can retrieve ec2 instance information:

```
aws ec2 describe-instances --profile prodaccess
```

## Bring it Home!

We all deserve to have nice things. Access to the AWS CLI is a great tool when managing BOSH deployed resources whether controlled by IAM Profiles or traditional AWS Access and Secret Keys.

This tool can also be used to force a reboot of a VM which BOSH has lost control of and `bosh cck` isn't fixing. 