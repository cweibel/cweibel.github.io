---
layout: post
title: "EKS Overview - What I've figured out"
date: 2021-05-04
---

![quoka](https://raw.githubusercontent.com/cweibel/ghost_blog_pics/master/natalie-su-6ZDY7Xht8_unsplash_smaller.png)

Photo by [Natalie Su](https://unsplash.com/@capillasn?utm_source=unsplash&utm_medium=referral&utm_content=creditCopyText) on [Unsplash](https://unsplash.com/s/photos/quokka?utm_source=unsplash&utm_medium=referral&utm_content=creditCopyText)



# EKS Overview
​
EKS is a (mostly) Managed Kubernetes service which has configuration options to offer 0 downtime upgrades, patching, a self healing control plane and access to other AWS services like IAM, KMS and RDS.  With both Serverless (Fargate) and traditional Worker Nodes (NodeGroups) available there is a mix of configurations which can meet user requirements.
​
## What does EKS Get You
​
 - A managed (master) node
 - Monitoring with CloudWatch
 - You do not get ssh access to the master node
 - You do not get to schedule pods on the master node
​
What it doesn't get you without additional configuration:
​
 - Worker nodes
 - Auto Scaling Groups
​
### Creating Workers
​
An EKS Cluster does not come with any worker nodes.  There are a few options here and note that you are not limited to a single choice, you can have as many of each type of workers as you would like, in any combination, on the same cluster:
​
 - NodeGroups, of which there are 3 main types:

   1. Self Managed (aka: unmanaged)
   2. Managed
   3. Managed with Launch Template
 
 - Fargate Profile, which has two modes:

   1. Namespace
   2. Namespace and Label


It bears repeating, the same EKS cluster can combine both Fargate and NodeGroups on the same cluster.
​
​
### What is Fargate on EKS
​
Fargate removes the requirement to manage worker nodes to run a pod.  When a pod is deployed to Fargate, AWS behind the scenes provisions an EC2 instance and attempts to roughly match the CPU and Memory requirements of the pod to an instance type.  The EC2 instance has kubelet installed on it and is added as a worker node to the cluster.  If the pod is removed, the EC2 instance is removed.  If 0 pods are deployed, there is 0 cost since there are no EC2 instances.  Scaling happens automatically since more EC2 instances are created automatically every time a pod is created.  There are pros and cons to this which are covered later. 

To use Fargate on EKS, an admin must configure one or more Fargate Profiles.  

It is possible to deploy an EKS Cluster to only support Fargate at bootstrap.  In this case the the namespaces `default` and `kube-system` will be added to the default Fargate Profile.  If this bootstrap option is not defined, no default Fargate Profile is automatically created but can be added at any time.
​
### Fargate Pros/Cons
​
Portions of this are sourced from [https://docs.aws.amazon.com/eks/latest/userguide/fargate.html](https://docs.aws.amazon.com/eks/latest/userguide/fargate.html)

Pros:
​
 - Compliance is baked in for SOC, PCI, FedRamp.  See [https://aws.amazon.com/compliance/services-in-scope/](https://aws.amazon.com/compliance/services-in-scope/) for more details
 - No admin required for scaling
 - No patching, maintenance is done automatically
 - Only pay for the pods you are running, not for any additional headroom/buffer you have for NodeGroups.
 
Cons:
​
 - No persistent storage is supported
 - Only works from within private subnets
 - Privileged containers are not allowed
 - Daemonsets not supported
 - HostPort/HostNetwork are not supported
 - No control on the EC2 instance type which is used, potentially inconsistent performance between pods.
 - ELB and NLB not supported, ALB Ingress Controller should be used
 - Not available in GovCloud
 - Fargate runs each pod in a VM-isolated environment without sharing resources with other pods. However, because Kubernetes is a single-tenant orchestrator, Fargate cannot guarantee pod-level security isolation. You should run sensitive workloads or untrusted workloads that need complete security isolation using separate Amazon EKS clusters.
 - You have no control (ssh or otherwise) of the underlying EC2 instance
 - Only certain combinations of CPU and Memory are supported:
 
  | CPU | Memory Values |
  | --- | --- |
  |0.25 vCPU | 0.5GB, 1GB, and 2GB|
  |0.5 vCPU	| Min. 1GB and Max. 4GB, in 1GB increments
  |1 vCPU	| Min. 2GB and Max. 8GB, in 1GB increments
  |2 vCPU	| Min. 4GB and Max. 16GB, in 1GB increments
  |4 vCPU	| Min. 8GB and Max. 30GB, in 1GB increments
   *(source: [https://aws.amazon.com/fargate/pricing/](https://aws.amazon.com/fargate/pricing/) )*
​
### Using Fargate Profiles on EKS
​
To enable Fargate on an EKS cluster you define a `Fargate Profile`.  A Fargate Profile comes in two flavors:
​
1. **Namespace** - You define a list of namespaces that you want pods in only these namespaces to deploy to Fargate.   If a pod is deployed in one of these namespaces it will become a Fargate Pod.  If the pod is deployed to a namespace not defined in the Fargate Profile, it will wind up in a NodeGroup or fail if no NodeGroups exist.
​
   This is all transparent to the person deploying the pod.  They cannot tell whether the pod landed on Fargate or a NodeGroup.  Other than picking the namespace the developer has no say in where the pod is deployed.
​
​
2. **Namespace and Label** - You define a list of namespaces and labels you want pods to deploy to Fargate.  If a pod is deployed to one of these namespaces AND has the appropriate label, it will deploy to Fargate.  If the pod is deployed and does not match a namespace AND label combination it will wind up in a NodeGroup or fail if no NodeGroups exist.
​
   This is semi-transparent to the person deploying the pod.  This combination allows the developer to chose whether or not their pod is scheduled on Fargate for a particular namespace by adding or removing a label from their pod spec.
​
### Using NodeGroups
​
NodeGroups can be thought of as traditional worker nodes.  They are EC2 instances with a kubelet built into the AMI image.  Just like a Cloud Foundry Cell, as many pods/containers as possible will get scheduled on these nodes until the resources run out.
​
As mentioned in the `Creating Workers` section there are a few main types of NodeGroups:
​
 - **Self Managed** - You have the most freedom to do as you want but are responsible for the lifecycle of the EC2 instance, patching the AMI and other maintenance.  Graceful draining and updating of nodes need an outside process. These nodes are also not managed from the EKS Console.  Auto scaling is an exercise left to the admin.
 - **Managed** - These have significantly less options but are easier to maintain.  Nodes are gracefully drained for upgrades and terminations. 
​
    You select from a list of 3 AMIs, a list of instance types and assign the disk size.  For the Scaling Group configuration you set the desired, minimum and maximum number of nodes.  You can also select the subnets and security groups as well as an SSH key to connect to the created EC2 instances.  
    
    These instances are managed from the EKS Console.  Upgrades to workers are not automatically applied but are easily launched from the EKS Console.
 - **Managed with Launch Templates** - These are like the `Managed` but allow you to bring your own AMI, use Spot Instances, additional Network Interfaces for those who enjoy creating their own pain and other optimizations around tenancy/GPU/termination and the like.
​
​
### Summary of Worker Types
​
A quick summary of the pros and cons to each type of EKS worker:
​
![compare worker types](https://miro.medium.com/max/700/1*2lhFtVTeM9n8LV6XS1vDEQ.png)
​
*(source: [https://blog.gruntwork.io/comprehensive-guide-to-eks-worker-nodes-94e241092cbe](https://blog.gruntwork.io/comprehensive-guide-to-eks-worker-nodes-94e241092cbe) )*
​
​
## EKS with KMS
​
Kubernetes Secrets can be encrypted in the etcd store by enabling `Envelope Encryption` when the EKS cluster is created.  A walk through example and additional info/diagrams can be found here [https://aws.amazon.com/blogs/containers/using-eks-encryption-provider-support-for-defense-in-depth/](https://aws.amazon.com/blogs/containers/using-eks-encryption-provider-support-for-defense-in-depth/).  Secrets using this feature are transparent to the developer managing the lifecycle of these secrets.
​
There are costs associated with KMS keys for creation and decryption which should not be ignored.  It is also unclear if KMS key rotation is possible.
​
It should also be noted that `etcd` volumes attached to the master are encrypted at disk-level using AWS managed keys.  It is unclear whether customers can supply their own KMS keys for this.
​
​
## Tools to Spin EKS (and Fargate)
​
### Using the `eksctl` CLI
​
Perhaps the easiest way of creating an EKS instance is using this tool.  It is really just a wrapper around an AWS managed CloudFormation script and it will create the vpc, subnets, roles and everything else required to support deploying an EKS cluster.
​
Examples:
​
Create an EKS cluster with a Self Managed NodeGroup:
​
```
eksctl create cluster --name my-eks-self-managed  
```
​
Create an EKS cluster with a Managed NodeGroup:
​
```
eksctl create cluster --name my-eks-managed --managed 
```
​
Create an EKS cluster with a default Fargate Profile.  This will put all nodes in namespaces `default` and `kube-system` into Fargate:
​
```
eksctl create cluster --name my-eks-fargate --fargate 
​
```
​
`eksctl` can also be configured with a yaml file to use an existing VPC, Subnet or other resources to the point that a duplicate cluster except for the name can be created.  The complete schema of all the options is documented at [https://eksctl.io/usage/schema/](https://eksctl.io/usage/schema/) To get an example configuration which you can base other examples on add the verbose level 4 flag to the create cluster command:
​
```
eksctl create cluster --name tom-managed --managed -v 4
```
​
This will create output similar to:
​
```
...
few hundred lines of output
...
2021-04-09T15:59:40-04:00 [✔]  EKS cluster "tom-managed" in "us-west-2" region is ready
2021-04-09T15:59:40-04:00 [▶]  cfg.json = \
{
    "kind": "ClusterConfig",
    "apiVersion": "eksctl.io/v1alpha5",
    "metadata": {
        "name": "tom-managed",
        "region": "us-west-2",
        "version": "1.17"
    },
    "iam": {
        "serviceRoleARN": "arn:aws:iam::868593321044:role/eksctl-tom-managed-cluster-ServiceRole-1BZI8D2TK63ZL",
        "withOIDC": false,
        "vpcResourceControllerPolicy": true
    },
    "vpc": {
        "id": "vpc-0375e8c98efb3efac",
...
```
​
Save the output to a file named `tom2.yaml`, change the cluster name references from `tom` to `tom2` and deploy the second EKS cluster:
​
```
eksctl create cluster -f tom2.yml
```
​
​
### Terraform
​
There are a few community repos which can spin a VPC and deploy an EKS cluster, an example of the one we've been using is documented at [https://github.devtools.predix.io/industrial-cloud-pcs/terraform-eks-poc](https://github.devtools.predix.io/industrial-cloud-pcs/terraform-eks-poc).

This repo assumes that you have admin access to an AWS Account and that a VPC, subnet, roles, and other AWS resources required will all be created.