---
layout: post
title: "Adding Users To EKS Kubernetes Clusters"
date: 2021-09-08
---

![map](https://raw.githubusercontent.com/cweibel/ghost_blog_pics/master/pablo-garcia-saldana-lPQIndZz8Mo-unsplash-2.jpg)



Photo by [Pablo García Saldaña](https://unsplash.com/@garciasaldana_?utm_source=unsplash&utm_medium=referral&utm_content=creditCopyText) on [Unsplash](https://unsplash.com/s/photos/uri?utm_source=unsplash&utm_medium=referral&utm_content=creditCopyText)


# Adding Users To EKS Kubernetes Clusters


Creating an Amazon EKS Cluster is a fun experience with any number of tools (Terraform, eksctl, AWS Console) to create you first or 100th cluster.

What isn't so much fun?  Giving someone other than yourself access to the cluster.  



## Option 1: Adding IAM Users and IAM Roles

I won't repeat what AWS has so expertly documented at [https://docs.aws.amazon.com/eks/latest/userguide/add-user-role.html](https://docs.aws.amazon.com/eks/latest/userguide/add-user-role.html).  It really is a good article.  Go read it.  I'll wait.


A couple highlights from the documentation:

 - The user or assumed role which originally created the EKS Cluster always has full access to the cluster. Note that this user/role DOES NOT APPEAR in the configmap.
 - AWS IAM Authenticator does not permit a path in the role ARN used in the configuration map. Therefore, before you specify rolearn, remove the path. For example, change 

   ```
   arn:aws:iam::<123456789012>:role/<team>/<developers>/<eks-admin>
   ``` 

   to 

   ```
   arn:aws:iam::<123456789012>:role/<eks-admin>
   ``` 

  according to [this](https://docs.aws.amazon.com/eks/latest/userguide/troubleshooting_iam.html#security-iam-troubleshoot-ConfigMap) link.
 - The group mapping to `system:masters` is also purposeful as it indicates that members of this ARN will have full cluster admin access as documented [here](https://docs.aws.amazon.com/eks/latest/userguide/add-user-role.html).  This group is associated to the `ClusterRoleBinding` named `cluster-admin` which is associated with a `ClusterRole` also named helpfully `cluster-admin`.  These are wired up automatically for you in every EKS cluster. 
 - Any combination of IAM roles and IAM users can be added to the configmap




## Option 2: Old Fashioned Service Account


Sometimes you just need a `kubeconfig` not tied to any IAM users or roles which can connect to the cluster for CI/CD or, well, laziness :)


The script below will create a Kubernetes Service Account and generate a `kubeconfig` file you can target, save it as `create-sa.sh`:

    
```    
name=$1
namespace=$2
role=$3
clusterName=$namespace
server=$(kubectl config view --minify | grep server | awk {​'print $2'}​)
serviceAccount=$namekubectl create -n $namespace sa $serviceAccount
kubectl create -n $namespace clusterrolebinding ${​namespace}​-${​role}​ --clusterrole ${​role}​ --serviceaccount=$namespace:$serviceAccount --namespace $namespace
set -o errexit
secretName=$(kubectl --namespace $namespace get serviceAccount $serviceAccount -o jsonpath='{​.secrets[0].name}​')
ca=$(kubectl --namespace $namespace get secret/$secretName -o jsonpath='{​.data.ca\.crt}​')
token=$(kubectl --namespace $namespace get secret/$secretName -o jsonpath='{​.data.token}​' | base64 --decode)

echo "
---
apiVersion: v1
kind: Config
clusters:
  - name: ${​clusterName}
  ​  cluster:      
  ​    certificate-authority-data: ${​ca}​      
  ​    server: ${​server}​
contexts:  
  - name: ${​serviceAccount}​@${​clusterName}​    
  - context:      
      cluster: ${​clusterName}​      
      namespace: ${​serviceAccount}​      
      user: ${​serviceAccount}
​users:  
​  - name: ${​serviceAccount}​    
​    user:      
​      token: ${​token}​
​current-context: ${​serviceAccount}​@${​clusterName}​" > sa.kubeconfig
```    
    
Pass in the 3 parameters (service account name, namespace, cluster role) to the script, the `kubeconfig` file `sa.kubeconfig` will be generated at can then be targeted:

```
# Run the script
create-sa.sh my-sa kube-system cluster-admin

# Reference the kubeconfig file the script created
export KUBECONFIG=./sa.kubeconfig

# Test the new kubeconfig file
kubectl get nodes  
``` 
  
Credit goes to Alexander Lukyanov who shared this during a fun pairing session!