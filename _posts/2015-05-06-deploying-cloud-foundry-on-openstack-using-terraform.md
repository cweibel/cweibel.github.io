---
layout: post
title: "Deploying Cloud Foundry on Openstack Using Terraform"
date: 2016-08-09
---

![map](https://raw.githubusercontent.com/cweibel/ghost_blog_pics/master/jeshoots-com-fzOITuS1DIQ-unsplash.jpg)



Photo by [JESHOOTS.COM](https://unsplash.com/@jeshoots) on [Unsplash](https://unsplash.com/s/photos/uri?utm_source=unsplash&utm_medium=referral&utm_content=creditCopyText)


[Author's Note in 2021: This is old and should not be used for modern deployments as-is but is kept for historical reasons]

This manual will guide you through the steps necessary to deploy Cloud Foundry using Terraform on OpenStack. A tremendous amount of automation has been put in place to allow you to quickly deploy Cloud Foundry in an easy and repeatable way. Installing to AWS can be found [here](https://blog.starkandwayne.com/2015/04/06/deploy-cloud-foundry-on-aws-using-terraform/).

## Overview

There are four primary steps to deploying Cloud Foundry (and one to destroy), we will explore each step in the subsequent sections:

 - Prerequisites
 - Configure Install
 - Connect to Cloud Foundry
 - Deploy Services
 - Destroy the Deployment

## Step 1 - Prerequisites

### Local Configuration

We’ll assume that you have an OSX or Linux computer you are working from locally. There are a couple pieces of software which you will need installed:

 - git
 - Terraform 0.4.0+

### Install git
You likely already have git installed if you do any development locally. Follow the instructions here to install git: [http://git-scm.com/book/en/v2/Getting-Started-Installing-Git](http://git-scm.com/book/en/v2/Getting-Started-Installing-Git) a short summary is:

 - OSX - `brew install git`
 - RHEL/CentOS/Fedora - `sudo yum install -y git`
 - Ubuntu/Debian = `sudo apt-get install -y git`
 
### Install Terraform

Visit the following website to download a version of terraform for you local computer: [https://www.terraform.io/downloads.html](https://www.terraform.io/downloads.html)

After you download the appropriate zip, copy the files to a folder in your `PATH`, in the example below we use `~/bin`

```
mv ~/Downloads/terraform_0/* ~/bin
```

Open a new terminal window and run:

```
terraform -v
```

You should get the following output:

```
Terraform v0.4.2
```

## Step 2 - Configure Install

### Clone the Repo

On your local computer run the following:

```
git clone https://github.com/cloudfoundry-community/terraform-openstack-cf-install
cd terraform-openstack-cf-install
cp terraform.tfvars.example terraform.tfvars
```

### Edit Variable File

This will copy a terraform configuration to your local computer. Using your favorite text editor (in the examples we will use vi) edit the `terraform.tfvars` file and supply the the terraform paramters:

```
vi terraform.tfvars
```

The determine how to populate each of the parameters view [this](https://blog.starkandwayne.com/2015/05/06/configuring-openstack-for-cloud-foundry/) document.

After editing your file should look like:

```
network = "192.168"
auth_url="http://10.2.95.20:5000/v2.0"
tenant_name="terraform_cf"
tenant_id="cb4dd90cd995452ea989b7f8601132b"
username="cf_user"
password="cf_user"
public_key_path="/Users/chris/.ssh/cfkey.pub"
key_path="/Users/chris/.ssh/cfkey"
floating_ip_pool="net04_ext"
region="RegionOne"
network_external_id="aa801a43-688b-4949-b82e-74ead5e358cd"
```

Now you are ready to deploy, run:

```
make plan
make apply
make provision
```

Go get something to drink. It will take about an hour to deploy everything to OpenStack. Don’t panic if an error occurs during `make apply` or `make provision`, run the command again as OpenStack resources aren’t always available when requested.

When the installation has completed, your screen should output a series of values you will need to connect to the Cloud Foundry deployment, in our example we see:

```
Apply complete! Resources: 13 added, 0 changed, 0 destroyed.

The state of your infrastructure has been saved to the path
below. This state is required to modify and destroy your
infrastructure, so keep it safe. To inspect the complete state
use the `terraform show` command.

State path: terraform.tfstate

Outputs:

  auth_url                 = http://10.2.95.20:5000/v2.0
  bastion_ip               = 10.2.95.207
  cf_api                   = api.run.10.2.95.206.xip.io
  cf_boshworkspace_version = v1.1.4
  cf_domain                = XIP
  cf_fp_address            = 10.2.95.206
  cf_sg_id                 = baba2b1f-e924-4f37-90e4-b1d081389767
  cf_size                  = tiny
  internal_network         = 9ef1f4c7-4b7f-459f-9dfe-ceadd119c02a
  internal_network_id      = 9c806195-77f9-4d2a-bc47-a9b8672d6547
  key_path                 = /Users/chris/.ssh/id_rsa
  network                  = 192.168
  password                 = chris
  region                   = RegionOne
  router_id                = 0811cbe1-7457-497a-830d-8c192417b760
  tenant_name              = terraform_cf
  username                 = user_cf
```


## Step 3 - Connect To Cloud Foundry

In order to connect to Cloud Foundry, you will need to do the following locally (note that you can just use the Bastion server which will have these tools already installed):

 - Install the CF CLI Tool
 - Use the CF CLI Tool to connect to Cloud Foundry

### Install CF CLI - Local

There are two ways to push CF apps: from your laptop or the bastion server. If will only be using the bastion server you can skip this step as it is already installed on the bastion server. If you would like to push CF apps from your laptop:

Open a terminal window on your laptop and run:

> curl -s https://raw.githubusercontent.com/cloudfoundry-community/traveling-cf-admin/master/scripts/installer | bash

### Install CF CLI - Bastion Server

The CF CLI and other tools are already installed on the bastion server.

### Connect With the CF CLI

Now that the CF CLI tool has been installed, connect to Cloud Foundry using the (remember to substitute the IP address in `cf_api` with the value outputted from Step 2):

```
cf login --skip-ssl-validation -a api.run.10.2.95.206.xip.io -u admin -p c1oudc0wc1oudc0w
```

Thats it! You can now write your application locally and then push it to Cloud Foundry.

## Step 4 - Deploy Services

Now that we have Cloud Foundry deployed to OpenStack let’s deploy some services which the Cloud Foundry applications can consume, such as PostgreSQL and Redis.

### Modify terraform.tfvars

Add the following line to the terraform.tfvars file:

```
install_docker_services = "true"
```

Now you are ready to deploy, run:

```
make plan
make apply
make provision
```

After 20 or so minutes the Docker Services VM will be deployed. Even after the VM and jobs are running it will take about 10 minutes for all of the docker images to be imported and available for the Service Broker.
Register Services with Service Broker

In Step 3 you installed the CF CLI tools which we will use now to create a service broker. It may take up to 20 minutes for all of the unicorn web server to fetch all of the docker images. Execute the following on the bastion server (be sure to substitute the ip address for with the output if Step 2):

```
cf create-service-broker docker containers containers http://cf-containers-broker.run.10.2.95.206.xip.io
```

### Note 1
If you get an error indicating “cf” is not installed, run the following:

> curl -s https://raw.githubusercontent.com/cloudfoundry-community/traveling-cf-admin/master/scripts/installer | bash

### Note 2

If you forgot to login in Step 3 you will get the message ”No API endpoint set. Use `cf login` or `cf api` to target an endpoint.” If so, execute the following:

```
cf login --skip-ssl-validation -a api.run.52.0.125.51.xip.io -u admin -p c1oudc0wc1oudc0w
```

Once the service broker is created you can list the services available:

```
cf service-access
cf enable-service-access mysql56  # Enables a mysql 5.6 instance
```

## Step 5 - Tear Down

Terraform does not yet quite cleanup after itself. You can run make destroy to get quite a few of the resources you created, but you will probably have to manually track down some of the bits and manually remove them. Once you have done so, run make clean to remove the local cache and status files, allowing you to run everything again without errors. Locally run:

```
make destroy ; sleep 10 ; make destroy
```

Note that it may take a few runs to have all the objects destroyed. Once make destroy is successful, run the following to reset Terraform files for any subsequent runs:

```
make clean
```

## Resources

 - The primary repository is located [here](https://github.com/cloudfoundry-community/terraform-openstack-cf-install)
 - [Installing RVM](https://rvm.io/rvm/install)
[Installing Git](https://git-scm.com/book/en/v2/Getting-Started-Installing-Git)
 - [CF CLI Releases](https://github.com/cloudfoundry/cli/releases)
 - [Traveling CF CLI Install](https://github.com/cloudfoundry-community/traveling-cf-admin)
