---
layout: post
title: "More Limes: Running Multiple KubeCF Deployments on One Kubernetes Cluster"
date: 2020-03-18
---

![pic](https://raw.githubusercontent.com/cweibel/ghost_blog_pics/master/herry-sutanto-H18UoPLPvcE-unsplash-2.jpg)

Photo by [Herry Sutanto](https://unsplash.com/@sutanto?utm_source=unsplash&utm_medium=referral&utm_content=creditCopyText) on [Unsplash](https://unsplash.com/s/photos/limes?utm_source=unsplash&utm_medium=referral&utm_content=creditCopyText)


In a [previous blog post](https://cweibel.github.io/blog/2020/01/15/running-cloud-foundry-on-kubernetes-using-kubecf) we discovered how to deploy a single KubeCF with a single cf-operator. Exciting stuff! What if you wanted to deploy a second KubeCF? A third?

With a couple minor changes to subsequent installs you can deploy as many instancess of KubeCF as you like, each in their own namespaces.

## Concepts

Creating the first KubeCF is done by Helm installing the cf-operator, configuring a `values.yaml` file for KubeCF and finally Helm installing KubeCF. Each of the two operators will exist in their own namespace. With the `feature.eirini.enable:` true option set in the `values.yaml` of the KubeCF Helm chart a third namespace named `<kubecf-name>-eirini` will be created for all the Cloud Foundry apps to live in.

For each additional instance of KubeCF, you'll need an install of the cf-operator referencing an install of kubecf. Each pair of `cf-operator`+`kubecf` will get their own namespaces so if you intend on deploying a great number of KubeCF installs consistent naming will become important.

## Deploy the First KubeCF

A quick overview of installing a single instance of KubeCF is below. For complete instruction visit [the blog post](https://www.starkandwayne.com/blog/running-cloud-foundry-on-kubernetes-using-kubecf/).

 - Helm install the `cf-operator` configured to watch a particular namespace
 - Configure a `values.yaml` file for KubeCF, leave the default NodePort of 32123 for the Eirini service.
 - Helm install KubeCF

## Subsequent Installs

For each additional deployment of KubeCF there is a 1:1 install of the cf-operator required as well.

### Install `cf-operator`

The `cf-operator` configuration needs a few additional pieces:

 - A unique namespace
 - Name override so the helm chart creates all the cluster roles with unique names
 - Disable creating the CRDs again as they aren't namespaced
 - In the example below, a second cf-operater is deployed, extra points if you can guess the naming for the third one:

```
kubectl create namespace cf-operator2

helm install cf-operator2 \
  --namespace cf-operator2 \
  --set "global.operator.watchNamespace=kubecf2" \
  --set "fullnameOverride=cf-operator2" \
  --set "applyCRD=false" \
  https://s3.amazonaws.com/cf-operators/release/helm-charts/cf-operator-v2.0.0-0.g0142d1e9.tgz
```

### Configure `values.yaml` for KubeCF

There are two important pieces of information which must be unique between each of the installs:

 1. `system_domain` - Don't reuse the same system_domain for any existing Cloud Foundry deployment which is visible by your DNS, regardless of whether it is KubeCF, Tanzu Pivotal Platform (PCF), or `cf-deployment`-based. Debugging is hard enough without having to figure out which coconut we are talking to.
 2. `features.eirini.registry.service.nodePort` must be a unique number across the entire cluster. Verify the port you hard code is not in use before deploying.

```
system_domain: kubecf2.10.10.10.10.netip.cc   # Must be unique

...

features:
  eirini:
    enabled: true
    registry:
      service:
        nodePort: 32124  # Must be unique cluster-wide
```


### Install the Second KubeCF

There are a couple minor changes from the install of the first KubeCF:

 - Again make sure you have a unique helm chart name
 - The namespace name should match the `global.operator.watchNamespace` of the corresponding `cf-operator`
 - Reference the `values.yaml` file for this install of KubeCF

```
helm install kubecf2 \
  --namespace kubecf2 \
  --values /Users/chris/projects/kubecf/kubecf2/values.yaml \
  https://github.com/SUSE/kubecf/releases/download/v0.2.0/kubecf-0.2.0.tgz
```

## Repeat

At some point you will run yourself out of Kubernetes resources if you keep spinning additional KubeCF installs. Two concurrent installs run happily on 3 worker nodes with 2vCPU and 16GB of memory each.

# Update

Don't do this for production, please use 1 cluster per KubeCF install; this is an interesting science experiment but please don't put your mission-critical deployments in this configuration.  Thanks!

