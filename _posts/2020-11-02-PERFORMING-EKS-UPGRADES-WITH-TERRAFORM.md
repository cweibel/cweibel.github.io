# PERFORMING EKS UPGRADES WITH TERRAFORM

Nov 03, 2020
by Chris Weibel

![quackquack](https://images.unsplash.com/photo-1558911350-c1e2588e6c7d?ixid=MnwxMjA3fDB8MHxwaG90by1wYWdlfHx8fGVufDB8fHx8&ixlib=rb-1.2.1&auto=format&fit=crop&w=2840&q=80)

Photo by [Joel Thorner](https://unsplash.com/@joelthorner?utm_source=unsplash&utm_medium=referral&utm_content=creditCopyText) on [Unsplash](https://unsplash.com/t/animals?utm_source=unsplash&utm_medium=referral&utm_content=creditCopyText)

In a [previous blog post](https://www.starkandwayne.com/blog/65-lines-of-terraform-for-a-new-vpc-eks-node-group-fargate-profile/) we've shown you how to deploy EKS quickly and easily with Terraform.  AWS recently release version v1.18 of Kubernetes on EKS so now is the perfect opportunity to see how to upgrade an EKS cluster using Terraform.

For the rest of this blog it is assumed that you've used [https://github.com/cweibel/example-terraform-eks/tree/main/eks_and_fargate](https://github.com/cweibel/example-terraform-eks/tree/main/eks_and_fargat) to spin up your EKS cluster and a single managed Node Group.

## Performing Upgrades

Upgrades can be done through either the AWS Console UI or via Terraform.  We'll assume that you want to continue to use Terraform to manage EKS after you've bootstrapped the environment.

Let's follow the mama duck!

Upgrades are done in two or more phases.  The first phase involves updating the version of Kubernetes on the master/control node.  The subsequent phases involve updating the one or more Node Groups you have defined.

### Step 1 - Upgrade the master

This is straight forward.  In this repo set cluster.tf local variables to the desired version:

```
locals {
  cluster_version = "1.18"   # Assuming you initially deployed 1.17
}
```

Perform a `terraform apply` and perform an update in-place:

```
Resource actions are indicated with the following symbols:
  ~ update in-place

Terraform will perform the following actions:

  # module.eks.aws_eks_cluster.this[0] will be updated in-place
  ~ resource "aws_eks_cluster" "this" {
        arn                       = "arn:aws:eks:us-west-2:123456789012:cluster/eks-cweibel2"
        certificate_authority     = [
            {
                data = "LS0tLREDACTEDtLS0tLQo="
            },
        ]
        created_at                = "2020-09-16 16:07:38.259 +0000 UTC"
        enabled_cluster_log_types = []
        endpoint                  = "https://1234AB1AEB234567FA5EBBAA67ED8BC9.gr7.us-west-2.eks.amazonaws.com"
        id                        = "eks-cweibel2"
        identity                  = [
            {
                oidc = [
                    {
                        issuer = "https://oidc.eks.us-west-2.amazonaws.com/id/1234AB1AEB234567FA5EBBAA67ED8BC9"
                    },
                ]
            },
        ]
        name                      = "eks-cweibel2"
        platform_version          = "eks.3"
        role_arn                  = "arn:aws:iam::123456789012:role/eks-cweibel220200916160718353000000001"
        status                    = "ACTIVE"
        tags                      = {}
      ~ version                   = "1.17" -> "1.18"

        timeouts {
            create = "30m"
            delete = "15m"
        }

        vpc_config {
            cluster_security_group_id = "sg-081a41a4f850bc69b"
            endpoint_private_access   = false
            endpoint_public_access    = true
            public_access_cidrs       = [
                "0.0.0.0/0",
            ]
            security_group_ids        = [
                "sg-084cb02c3a6d6442c",
            ]
            subnet_ids                = [
                "subnet-05fa7d7ec68ecfae8",
                "subnet-07c8750e73172d61d",
                "subnet-0c9b19e2856e8b867",
            ]
            vpc_id                    = "vpc-05f0ba84696234a43"
        }
    }

Plan: 0 to add, 1 to change, 0 to destroy.
```

This will take a few minutes to run, don't jump ahead until this step finishes.

### Step 2a - Upgrading Node Groups

Once the master node has been upgraded to the newer version, each of the Node Groups can be upgraded, following the mama duck.  This is done by tainting the NodeGroup resources:

```
terraform taint "module.eks.module.node_groups.random_pet.node_groups[\"eks_nodes\"]"
terraform taint "module.eks.module.node_groups.aws_eks_node_group.workers[\"eks_nodes\"]"
```

This will not do an in-place upgrade.  What it will do is:

 1. Spin an entirely new NodeGroup set of EC2 instances using the newer AMI.
 1. Once all the new worker nodes are healthy, the older nodes will be drained and status will be `Ready,SchedulingDisabled`.
 1. After the pods are moved off of the old workers, the underlying EC2 instances are terminated and the upgrade is complete

While this is happening, to get a "BOSH like experience to know the state of the upgrade" you can run a watch `kubectl get nodes` command.  In the example below the new workers have been successfully added and the old workers are in the process of draining:

```
$ kubectl get nodes
NAME                                         STATUS                     ROLES    AGE   VERSION
ip-10-20-68-201.us-west-2.compute.internal   Ready                      <none>   17m   v1.18.1-eks-4c6976
ip-10-20-71-150.us-west-2.compute.internal   Ready,SchedulingDisabled   <none>   13m   v1.17.3-eks-2ba888
ip-10-20-76-255.us-west-2.compute.internal   Ready                      <none>   17m   v1.18.1-eks-4c6976
ip-10-20-83-114.us-west-2.compute.internal   Ready                      <none>   17m   v1.18.1-eks-4c6976
ip-10-20-73-121.us-west-2.compute.internal   Ready,SchedulingDisabled   <none>   13m   v1.17.3-eks-2ba888
ip-10-20-75-133.us-west-2.compute.internal   Ready,SchedulingDisabled   <none>   13m   v1.17.3-eks-2ba888
```

If you have additional Node Groups (managed or unmanaged) you can now loop through and taint the worker groups.  To help prevent bumping in AWS EC2 resource quotas you'll likely only want to perform upgrades to one worker group at a time.

### Step 2b - Upgrading Fargate

Once the `master` node has been upgraded to the newer version, any newly created fargate pods will deploy EC2 instances with AMIs based on the version the master has.

To upgrade existing Fargate pods, there is no `terraform` to do this.  The pods needs to be destroyed and recreated.  To do this without downtime for the app itself, Kubernetes Deployments should be leveraged which can do rolling upgrades.

For example, if there is a Deployment called `nginx` which has 3 replicas:

```
$ kubectl get pods
NAME                               READY   STATUS    RESTARTS   AGE
nginx-deployment-f8d6d5c66-62s4f   1/1     Running   0          51m
nginx-deployment-f8d6d5c66-7tmc9   1/1     Running   0          52m
nginx-deployment-f8d6d5c66-qlkf5   1/1     Running   0          50m
```

You can see the 3 Fargate nodes that are associated to these 3 pods:

```
$ kubectl get nodes
NAME                                                 STATUS   ROLES    AGE   VERSION
fargate-ip-10-20-79-20.us-west-2.compute.internal    Ready    <none>   49m   v1.17.8-eks-e16311
fargate-ip-10-20-80-167.us-west-2.compute.internal   Ready    <none>   50m   v1.17.8-eks-e16311
fargate-ip-10-20-83-190.us-west-2.compute.internal   Ready    <none>   51m   v1.17.8-eks-e16311
ip-10-20-67-247.us-west-2.compute.internal           Ready    <none>   84m   v1.17.3-eks-2ba888
ip-10-20-77-210.us-west-2.compute.internal           Ready    <none>   85m   v1.17.3-eks-2ba888
ip-10-20-86-121.us-west-2.compute.internal           Ready    <none>   85m   v1.17.3-eks-2ba888
```

Perform upgrade of `master` from 1.17 to 1.18, any new fargate pods created will use the 1.18 AMI.  To rotate the existing Fargate Pods, perform a rolling update:

```
$ kubectl -n fgnamespace rollout restart deployment nginx-deployment
deployment.apps/nginx-deployment restarted
```

To see the status of the rollout:

```
kubectl rollout status deployments/nginx-deployment
Waiting for deployment "nginx-deployment" rollout to finish: 1 out of 3 new replicas have been updated...
Waiting for deployment "nginx-deployment" rollout to finish: 1 out of 3 new replicas have been updated...
Waiting for deployment "nginx-deployment" rollout to finish: 1 out of 3 new replicas have been updated...
Waiting for deployment "nginx-deployment" rollout to finish: 2 out of 3 new replicas have been updated...
Waiting for deployment "nginx-deployment" rollout to finish: 2 out of 3 new replicas have been updated...
Waiting for deployment "nginx-deployment" rollout to finish: 2 out of 3 new replicas have been updated...
Waiting for deployment "nginx-deployment" rollout to finish: 1 old replicas are pending termination...
Waiting for deployment "nginx-deployment" rollout to finish: 1 old replicas are pending termination...
deployment "nginx-deployment" successfully rolled out
```

If you look at the list of nodes at this point, the Fargate Pods are now all leveraging the v1.18.1 AMI

```
$ kubectl get nodes
NAME                                                 STATUS   ROLES    AGE     VERSION
fargate-ip-10-20-69-132.us-west-2.compute.internal   Ready    <none>   2m56s   v1.18.1-eks-a84824
fargate-ip-10-20-69-55.us-west-2.compute.internal    Ready    <none>   4m14s   v1.18.1-eks-a84824
fargate-ip-10-20-86-53.us-west-2.compute.internal    Ready    <none>   97s     v1.18.1-eks-a84824
ip-10-20-67-247.us-west-2.compute.internal           Ready    <none>   90m     v1.17.3-eks-2ba888
ip-10-20-77-210.us-west-2.compute.internal           Ready    <none>   91m     v1.17.3-eks-2ba888
ip-10-20-86-121.us-west-2.compute.internal           Ready    <none>   91m     v1.17.3-eks-2ba888
```

### Opinion: Import Notes for Fargate

Fargate is an odd duck when it comes to an Ops team being responsible for performing upgrades.  Unless the Fargate Pods are recreated via rolling upgrades there is no way for Ops personnel to do this work without looping through the namespaces in the Fargate Profile and performing rolling upgrades of the deployment.  If pods aren't associated to a Kubernetes Deployment, such as an app that only has a pod spec, tooling needs to be created to orchestrate the restart.

Otherwise the responsibility of performing upgrades to Fargate Pods falls to the app owner.  This could be dangerous from a compliance/security perspective.

## What's Next?

Need a dash of Cloud Foundry deployed on top of your Terraform managed EKS cluster?  Checkout [Deploying KubeCF to EKS](https://www.starkandwayne.com/blog/deploying-kubecf-to-eks-revisited/), much of that blog is automated into [https://github.com/cweibel/example-terraform-eks/tree/main/eks_for_kubecf_v2](https://github.com/cweibel/example-terraform-eks/tree/main/eks_for_kubecf_v2).  If you have any questions, please feel free to ask in the comments section below.

Enjoy!

