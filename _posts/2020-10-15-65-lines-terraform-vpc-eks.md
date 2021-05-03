---
layout: post
title: "65 Lines of Terraform For a New VPC + EKS + Node Group + Fargate Profile"
date: 2020-10-15
---

![beaver](https://raw.githubusercontent.com/cweibel/ghost_blog_pics/master/ignacio-amenabar-2dkgXTfPfTg-unsplash-3.jpg)

Photo by [Ignacio Amenábar](https://unsplash.com/@amenabarladrondeguevara?utm_source=unsplash&utm_medium=referral&utm_content=creditCopyText) on [Unsplash](https://unsplash.com/s/photos/quokka?utm_source=unsplash&utm_medium=referral&utm_content=creditCopyText)


As the title alludes, spinning Kubernetes on Amazon EKS is now a trivial exercise with Terraform. So simple even I can do it. So can you!

## Requirements

Not much:

 - An AWS account with [access keys](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_credentials_access-keys.html#Using_CreateAccessKey)
 - [Terraform](https://learn.hashicorp.com/tutorials/terraform/install-cli) and [AWS CLI](https://docs.aws.amazon.com/cli/latest/userguide/cli-chap-install.html)s installed
 - Knowledge on what EKS, Node Groups, and Fargate are. If not, take a quick look at [this post](https://cweibel.github.io/blog/2020/10/14/20Q-fargate)
 - An internet connection
 - Glass of Dr. Pepper to enjoy while the scripts run

## What Are We Going To Make?

The section below will walk you through deploying an EKS cluster with:

 - A new VPC with all the necessary subnets, security groups, and IAM roles required
 - A master node running Kubernetes `1.18` in the new VPC
 - A Fargate Profile, any pods created in the `default` namespace will be created as Fargate pods
 - A Node Group with 3 nodes across 3 AZs, any pods created to a namespace other than `default` will deploy to these nodes.

## CREATE ME A CLUSTER!!

Start by setting your environment variables:

```
export AWS_ACCESS_KEY_ID=AKIAGETYOUROWNCREDS2
export AWS_SECRET_ACCESS_KEY=Nqo8XDD0cz8kffU234eCP0tKy9xHWBwg1JghXvM4
export AWS_DEFAULT_REGION=us-east-2
```

Create a new folder and create a file called `cluster.tf` with the contents:

```
locals {
  cluster_name = "my-eks-cluster"
  cluster_version = "1.18"
}

module "vpc" {
  source = "git::ssh://git@github.com/reactiveops/terraform-vpc.git?ref=v5.0.1"

  aws_region = "us-east-2"
  az_count   = 3
  aws_azs    = "us-east-2a, us-east-2b, us-east-2c"

  global_tags = {
    "kubernetes.io/cluster/${local.cluster_name}" = "shared"
  }
}

module "eks" {
  source       = "git::https://github.com/terraform-aws-modules/terraform-aws-eks.git?ref=v12.2.0"
  cluster_name = local.cluster_name
  cluster_version = local.cluster_version
  vpc_id       = module.vpc.aws_vpc_id
  subnets      = module.vpc.aws_subnet_private_prod_ids

  node_groups = {
    eks_nodes = {
      desired_capacity = 3
      max_capacity     = 3
      min_capaicty     = 3

      instance_type = "t2.small"
    }
  }
  manage_aws_auth = false
}

resource "aws_iam_role" "iam_role_fargate" {
  name = "eks-fargate-profile-example"
  assume_role_policy = jsonencode({
    Statement = [{
      Action = "sts:AssumeRole"
      Effect = "Allow"
      Principal = {
        Service = "eks-fargate-pods.amazonaws.com"
      }
    }]
    Version = "2012-10-17"
  })
}
resource "aws_iam_role_policy_attachment" "example-AmazonEKSFargatePodExecutionRolePolicy" {
  policy_arn = "arn:aws:iam::aws:policy/AmazonEKSFargatePodExecutionRolePolicy"
  role       = aws_iam_role.iam_role_fargate.name
}
resource "aws_eks_fargate_profile" "aws_eks_fargate_profile1" {

  depends_on = [module.eks]

  cluster_name           = local.cluster_name
  fargate_profile_name   = "fg_profile_default_namespace"
  pod_execution_role_arn = aws_iam_role.iam_role_fargate.arn
  subnet_ids             = module.vpc.aws_subnet_private_prod_ids
  selector {
    namespace = "default"
  }
}
```

Now you can deploy the cluster:

```
terraform init
terraform apply
```

Eventually if all goes well you will get output similar to:

```
...
aws_eks_fargate_profile.aws_eks_fargate_profile1: Still creating... [1m40s elapsed]
aws_eks_fargate_profile.aws_eks_fargate_profile1: Still creating... [1m50s elapsed]
aws_eks_fargate_profile.aws_eks_fargate_profile1: Still creating... [2m0s elapsed]
aws_eks_fargate_profile.aws_eks_fargate_profile1: Still creating... [2m10s elapsed]
aws_eks_fargate_profile.aws_eks_fargate_profile1: Still creating... [2m20s elapsed]
aws_eks_fargate_profile.aws_eks_fargate_profile1: Creation complete after 2m23s [id=my-eks-cluster:fg_profile_default_namespace]

Apply complete! Resources: 63 added, 0 changed, 0 destroyed.
```

Now, switch to your Dr. Pepper, wait 20 or so minutes and when complete you can create a kubeconfig file to login by:

```
aws eks --region us-east-2 update-kubeconfig --name my-eks-cluster
```

## You Made That Look Easy

Sure I did. But, that's because several other people have done a ton of work. Much of this (minus the Fargate bits) are originally sourced from [this blog post](https://www.fairwinds.com/blog/terraform-and-eks-a-step-by-step-guide-to-deploying-your-first-cluster).

Within that blog post the author references a couple of excellent repos that I wish I had known about before creating my own (ugly) VPC Terraform (which I'm not sharing due to the previous comment). Specifically:

 - [https://github.com/FairwindsOps/terraform-vpc](https://github.com/FairwindsOps/terraform-vpc) will create you a new VPC with public and private subnets and other bits needed for EKS. Note, this could also be easily used for Cloud Foundry or other distributed multi-vm bits of software.
 - [https://github.com/terraform-aws-modules/terraform-aws-eks](https://github.com/terraform-aws-modules/terraform-aws-eks) which has a very active community to support/patch/fix using Terraform to spin EKS. A minor side note: this repo is great for spinning EKS and Node Groups, just not Fargate Profiles. There are plenty of changes you can make to the configurations, explore them to customize the install for your needs. Using spot instances might be a fun trick to throw into the mix.


Explore a few changes you can make easily with just the Terraform about, such as:

 - `instance_type = "t2.small"` are the size of the Node Group EC2 VMs. If you increase the size (t3.large) you'll be able to push more pods to the cluster.
 - `max_capacity     = 3` is the number of of EC2 VMs that make up the Node Group. Increase this to a larger number to push more pods.
- `cluster_version = "1.18"` Is the version the master will be created with. That also controls the version of AMI used for the Node Group and Fargate EC2 instances. `1.16` and `1.17` work as well, as of this writing 1.18 was just released.
 - `namespace = "default"` means all pods deployed to the `default` namespace will go to Fargate. You might rather have a special namespace called `fargate-only` or similar so the `default` namespace pods go to the Node Group instead.

## All Done, Tear it Down

Once you are done giving AWS money, run:

```
terraform destroy
```

When prompted on Do you really want to destroy all resources (the world)?, type `Yes`.  It is 2020 after all.

Take another sip of Dr. Pepper, you deserve it!

