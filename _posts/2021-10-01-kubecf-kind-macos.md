---
layout: post
title: "Running KubeCF using KIND on MacOS"
date: 2021-10-01
---

![pic](https://raw.githubusercontent.com/cweibel/ghost_blog_pics/master/2344271164_56358f2397_k.jpg)

Photo by [Alex Gorzen](https://www.flickr.com/photos/zeusandhera/) on Flickr


More Limes, More Coconuts

In previous blog posts I reviewed how to [deploy KubeCF on EKS](https://cweibel.github.io/blog/2020/01/15/running-cloud-foundry-on-kubernetes-using-kubecf), which gives you a nice stable deployment of `KubeCF`, the downside is this costs you money for every hour it is run on AWS.

I used to giving Amazon money, but I typically get a small cardboard box in exchange every few days. 

So, how do you run `KubeCF` on your Mac for free(ish)?  Tune in below.


## There Be Dragons Ahead

You need at least a 16GB of memory installed on your Apple MacOS device, the install will use around 11GB of the memory once it is fully spun up.

The install is fragile and frustrating at times, this is geared more towards operators who are trying out skunkworks on the platform such as testing custom buildpacks, hacking db queries and other potentially destructive activities.  The install does NOT survive reboots and become extra brittle after 24+ hours of running.  This is not KubeCF's fault, when run on EKS it will happily continue to run without issues.  You've been warned!

## Overview of Install

 - Install Homebrew and Docker Desktop
 - Install `tuntap` and start the shim
 - Install `kind` and deploy a cluster
 - Install and configure `metallb`
 - Install `cf-operator` and deploy `kubecf`
 - Log in and marvel at your creation


The next few sections are borrowed heavily from [https://www.thehumblelab.com/kind-and-metallb-on-mac/](https://www.thehumblelab.com/kind-and-metallb-on-mac/), I encourage you to skim this document to understand why the `tuntap` shim is needed and how to verify the configuration for `metallb`.


### Install Homebrew and Docker Desktop

I won't go into great detail as these tools are likely already installed:

 - Homebrew (full details are here: [https://docs.brew.sh/Installation](https://docs.brew.sh/Installation)) or just run:

   ```
   /bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"
   ```

 - Docker Desktop (full details for install are at: [https://docs.docker.com/desktop/mac/install/](https://docs.docker.com/desktop/mac/install/)) or run:

   ```
   brew install --cask docker
   # Then launch docker from Applications to complete the install and start docker
   ```

### Install `tuntap` and start the shim

Running `docker` on MacOS has some "deficiencies" that can be overcome by installing a networking shim, to perform this install:

```
brew install git
brew install --cask tuntap
git clone https://github.com/AlmirKadric-Published/docker-tuntap-osx.git
cd docker-tuntap-osx

./sbin/docker_tap_install.sh
./sbin/docker_tap_up.sh
```

### Install `kind` and deploy a cluster

Sweet'n'Simple:

```
brew install kind
kind create cluster
```

### Install and configure `metallb`

We'll be using `metallb` as a LoadBalancer resource and open up a local route so that MacOS can route traffic locally to the cluster.


```
sudo route -v add -net 172.18.0.1 -netmask 255.255.0.0 10.0.75.2

kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.9.3/manifests/namespace.yaml
kubectl create secret generic -n metallb-system memberlist --from-literal=secretkey="$(openssl rand -base64 128)"
kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.9.3/manifests/metallb.yaml

cat << EOF > metallb-config.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  namespace: metallb-system
  name: config
data:
  config: |
    address-pools:
    - name: default
      protocol: layer2
      addresses:
      - 172.18.0.150-172.18.0.200
EOF
kubectl create -f metallb-config.yaml
```


### Install `cf-operator` and deploy `kubecf`

```
brew install wget
brew install helm
brew install watch

wget https://github.com/cloudfoundry-incubator/kubecf/releases/download/v2.7.13/kubecf-bundle-v2.7.13.tgz
tar -xvzf kubecf-bundle-v2.7.13.tgz

kubectl create namespace cf-operator
helm install cf-operator \
  --namespace cf-operator \
  --set "global.singleNamespace.name=kubecf" \
  cf-operator.tgz \
 --wait

helm install kubecf \
  --namespace kubecf \
  --set system_domain=172.18.0.150.nip.io \
  --set features.eirini.enabled=false \
  --set features.ingress.enabled=false \
  --set services.router.externalIPs={172.18.0.150}\
  https://github.com/cloudfoundry-incubator/kubecf/releases/download/v2.7.13/kubecf-v2.7.13.tgz

watch kubectl get pods -A
```

Now, go take a walk, it will take 30-60 minutes for the `kubecf` helm chart to be fully picked up by the `cf-operator` CRDs and are scheduled for running.  When complete, you should see output similar to:


```
Every 2.0s: kubectl get pods -A

NAMESPACE            NAME                                         READY   STATUS      RESTARTS   AGE
cf-operator          quarks-cd9d4b96f-rtkbt                       1/1     Running     0          40m
cf-operator          quarks-job-6d8d744bc6-pmfnd                  1/1     Running     0          40m
cf-operator          quarks-secret-7d76f854dc-9wp2f               1/1     Running     0          40m
cf-operator          quarks-statefulset-f6dc85fb8-x6jfb           1/1     Running     0          40m
kube-system          coredns-558bd4d5db-ncmh5                     1/1     Running     0          41m
kube-system          coredns-558bd4d5db-zlpgg                     1/1     Running     0          41m
kube-system          etcd-kind-control-plane                      1/1     Running     0          41m
kube-system          kindnet-w4m9n                                1/1     Running     0          41m
kube-system          kube-apiserver-kind-control-plane            1/1     Running     0          41m
kube-system          kube-controller-manager-kind-control-plane   1/1     Running     0          41m
kube-system          kube-proxy-ln6hb                             1/1     Running     0          41m
kube-system          kube-scheduler-kind-control-plane            1/1     Running     0          41m
kubecf               api-0                                        17/17   Running     1          19m
kubecf               auctioneer-0                                 6/6     Running     2          20m
kubecf               cc-worker-0                                  6/6     Running     0          20m
kubecf               cf-apps-dns-76947f98b5-tbfql                 1/1     Running     0          39m
kubecf               coredns-quarks-7cf8f9f58d-msq9m              1/1     Running     0          38m
kubecf               coredns-quarks-7cf8f9f58d-pbkt9              1/1     Running     0          38m
kubecf               credhub-0                                    8/8     Running     0          20m
kubecf               database-0                                   2/2     Running     0          38m
kubecf               database-seeder-0bc49e7bcb1f9453-vnvjm       0/2     Completed   0          38m
kubecf               diego-api-0                                  9/9     Running     2          20m
kubecf               diego-cell-0                                 12/12   Running     2          20m
kubecf               doppler-0                                    6/6     Running     0          20m
kubecf               log-api-0                                    9/9     Running     0          20m
kubecf               log-cache-0                                  10/10   Running     0          19m
kubecf               nats-0                                       7/7     Running     0          20m
kubecf               router-0                                     7/7     Running     0          20m
kubecf               routing-api-0                                6/6     Running     1          20m
kubecf               scheduler-0                                  12/12   Running     2          19m
kubecf               singleton-blobstore-0                        8/8     Running     0          20m
kubecf               tcp-router-0                                 7/7     Running     0          20m
kubecf               uaa-0                                        9/9     Running     0          20m
local-path-storage   local-path-provisioner-547f784dff-r8trf      1/1     Running     1          41m
metallb-system       controller-fb659dc8-dhpnb                    1/1     Running     0          41m
metallb-system       speaker-h9lh9                                1/1     Running     0          41m
```


### Log in and marvel at your creation


To login with the `admin` uaa user account:

```
cf api --skip-ssl-validation "https://api.172.18.0.150.nip.io"

acp=$(kubectl get secret \
        --namespace kubecf var-cf-admin-password \
        -o jsonpath='{.data.password}' \
        | base64 --decode)

cf auth admin "${acp}"
cf create-space test -o system
cf target -o system -s test
```

Or, to use `smoke_tests` uaa client account (because you're a rebel or something):

```
cf api --skip-ssl-validation "https://api.172.18.0.150.nip.io"
myclient=$(kubectl get secret \
        --namespace kubecf var-uaa-clients-cf-smoke-tests-secret \
        -o jsonpath='{.data.password}' \
        | base64 --decode)

cf auth cf_smoke_tests "${myclient}" --client-credentials
cf create-space test -o system
cf target -o system -s test
```

## Cleaning Up and Putting Your Toys Away

If you are done with the deployment of KubeCF you have two options:

 1. Put your creation to sleep.  Start Docker Desktop > Dashboard > Select `kind-control-place` and click "Stop".  Go have fun, when you come back, click "Start".  After a few minutes the pods will recreate and become healthy
 2. Clean up.  To remove the cluster and custom routing:

   ```
   kind delete cluster
   sudo route delete 172.18.0.0
   ./sbin/docker_tap_uninstall.sh
   ```

## Debugging Issues

Get used to seeing:

> Request error: Get "https://api.172.18.0.150.nip.io": dial tcp 172.18.0.150:443: i/o timeout 

The api server is flaky, try whatever you were doing again after verifying all pods are running as seen in the `Install cf-operator and deploy kubecf` section.

## Follow up

Have questions?  There is an excellent community for KubeCF which can be found at [https://cloudfoundry.slack.com/archives/CQ2U3L6DC](https://cloudfoundry.slack.com/archives/CQ2U3L6DC) as `kubecf-dev` in Slack.  You can ping me there via `@cweibel`

I also have Terraform code which will spin up a VPC + EKS + KubeCF for a more permanent solution to running `KubeCF` not on a Mac, check out [https://github.com/cweibel/example-terraform-eks/tree/main/eks_for_kubecf_v2](https://github.com/cweibel/example-terraform-eks/tree/main/eks_for_kubecf_v2) for more details.

Enjoy!