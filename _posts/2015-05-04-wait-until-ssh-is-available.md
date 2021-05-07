---
layout: post
title: "Wait Until SSH Is Available"
date: 2015-05-04
---

![map](https://raw.githubusercontent.com/cweibel/ghost_blog_pics/master/rene-porter-P7cKOCQmilY-unsplash.jpg)


Photo by [René Porter](https://unsplash.com/@reneporter?utm_source=unsplash&utm_medium=referral&utm_content=creditCopyText) on [Unsplash](https://unsplash.com/s/photos/uri?utm_source=unsplash&utm_medium=referral&utm_content=creditCopyText)


[Author's Note in 2021: This is old and should not be used for modern deployments as-is but is kept for historical reasons]

Ever run into a situation where you need to ssh into a newly created server but you aren't sure that the server is listening on the ssh port yet? For the Terraform [OpenStack](https://github.com/cloudfoundry-community/terraform-openstack-cf-install) install of Cloud Foundry the Bastion server isn't immediately available for the provision script to run.

Below is a short bash script that will wait until ssh is available on the target server. If the server isn't ready yet, wait 10 seconds and try again for a maximum of 10 attempts.

```
#Wait until SSH on Bastion server is working
keyPath=$(terraform output key_path | sed -e "s#^~#$HOME#")
scriptPath="provision/provision.sh"
targetPath="/home/ubuntu/provision.sh"
bastionIP=$(terraform output bastion_ip)
maxConnectionAttempts=10
sleepSeconds=10

#Wait until SSH on Bastion server is working
echo "Attempting to SSH to Bastion server..."
index=1

while (( $index <= $maxConnectionAttempts ))
do
  ssh -i ${keyPath} ubuntu@$bastionIP
  case $? in
    (0) echo "${index}> Success"; break ;;
    (*) echo "${index} of ${maxConnectionAttempts}> Bastion SSH server not ready yet, waiting ${sleepSeconds} seconds..." ;;
  esac
  sleep $sleepSeconds
  ((index+=1))
done
```

Enjoy!