---
layout: post
title: "Helm 3 - How Do I Do Helm 3 Stuff?"
date: 2020-01-08
---

![map](https://raw.githubusercontent.com/cweibel/ghost_blog_pics/master/n-RFId0_7kep4-unsplash-4.jpg)

Photo by [N.](https://unsplash.com/@ellladee?utm_source=unsplash&utm_medium=referral&utm_content=creditCopyText) on [Unsplash](https://unsplash.com/s/photos/filter?utm_source=unsplash&utm_medium=referral&utm_content=creditCopyText)

[Helm 3](https://helm.sh/) was [recently introduced](https://helm.sh/blog/helm-3-released/) which changed many of the internal bits within the CLI which are not fully backward compatible to those using Helm 2. Fear not, with a couple minor tweaks you can continue to use the Helm charts you know and love!

If you are a maintainer of Helm charts you should probably start with [https://helm.sh/docs/faq/#changes-since-helm-2](https://helm.sh/docs/faq/#changes-since-helm-2) to see how the changes impact you.

This post, however, focuses on the user experience of consuming charts other folks maintain.  Below we'll review how to install the PostgreSQL chart.

## Using Helm install is Now 2 Steps

### Helm 2 - Old Way

In the time of Lord Helm 2 installing a chart was as simple as:

```
helm install stable/postgresql
```

This assumed you would be using https://github.com/helm/charts as your chart repository.

### Helm 3 - The New Way

It is fairly common to deploy helm charts by specifying the absolute url to a chart. With Helm 3+ you'll need to do helm installs in two steps:

 1. Add the chart repo to Helm:
   
    ```
    helm repo add bitnami https://charts.bitnami.com
    ```
        
    Note that there is a newer chart repository at https://hub.helm.sh.

 2. Install the chart:

    ```
    helm install bitnami/postgresql --version 8.1.2
    ```
    
    
## Using the old charts with Helm 3

If you still wish to use the charts in `https://github.com/helm/charts` with Helm 3, download the repo and specify the file path:

```
git clone https://github.com/helm/charts.git
cd charts/stable

helm install ./postgresql
```

### Namespace is no longer automatically created

In Helm v2 you could specify a namespace during an install and if it didn't exist, the namespace would be created automatically:

```
➜  helm2 install postgresql --name postgres2 --namespace=purple
```

The `purple` namespace would be created automatically:

```
NAME              STATUS   AGE
default           Active   83m
kube-node-lease   Active   83m
kube-public       Active   83m
kube-system       Active   83m
purple            Active   6m27s
```

Under Helm v3+ the `orange` namespace is not created:

```
➜  helm3 install postgres3 ./postgresql --namespace=orange
Error: create: failed to create: namespaces "orange" not found
```

You'll need to remember to create the namespace first until that functionality is added to many of the older charts.

## No Tiller

You no longer need to do the `helm init` dance, no server side components should make Helm easier to manage for operators in the long run.  Hurray!

## Miscellaneous

There are a few additional changes that happened with Helm 3:

 - `helm del` is an alias for `helm uninstall` which no longer has a `--purge` option since the uninstall cleans up the references.
 - When performing an install, there is no longer a `--name` option, the usage is `helm install [NAME] [CHART] [flags]` so simply provide the name as the first parameter to the install.

Otherwise, the helm charts themselves should still work.  Enjoy!