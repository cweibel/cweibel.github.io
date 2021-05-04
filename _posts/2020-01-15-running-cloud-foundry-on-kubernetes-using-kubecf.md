---
layout: post
title: "Running Cloud Foundry on Kubernetes Using KubeCF"
date: 2020-01-15
---

![pic](https://raw.githubusercontent.com/cweibel/ghost_blog_pics/master/2344271164_56358f2397_k.jpg)

Photo by [Alex Gorzen](https://www.flickr.com/photos/zeusandhera/) on Flickr

At Stark & Wayne, we've spent a ton of time figuring out the best solutions to problems using the open source tools we have available. We've pondered problem spaces such as:

What if we could...

 - Put the lime in the coconut?
 - Put the peanut butter in the chocolate?
 - Put the Cloud Foundry in the Kubernetes?

This last one is the least tasty but potentially the most satisfying, taking the large virtual machine footprint of Cloud Foundry with its developer facing toolset and stuffing it into Kubernetes.

For those who've installed Cloud Foundry in the past, you know that BOSH is the only way to install and manage Cloud Foundry. Well, that is until the [cf-operator](https://github.com/cloudfoundry-incubator/cf-operator), [Cloud Foundry Quarks](https://www.cloudfoundry.org/project-quarks/), [Eirini](https://github.com/cloudfoundry-incubator/eirini), and `KubeCF` came along.

## Concepts

The **cf-operator** is a Kubernetes Operator deployed via a Helm Chart which installs a series of custom resource definitions that convert BOSH Releases into Kubernetes resources such as pods, deployments, and stateful sets. It alone does not result in a deployment of Cloud Foundry.

**KubeCF** is a version of Cloud Foundry deployed as a Helm Chart, mainly developed by SUSE, that leverages the `cf-operator`.

**Eirini** swaps the Diego backend for Kubernetes meaning when you cf push, your applications run as Kubernetes pods inside of a statefulset.

**Kubernetes** is the new kid in town for deploying platforms.

Using these tools together we can deploy Cloud Foundry's Control Plane (cloud controller, doppler, routers and the rest) and have the apps run as pods within Kubernetes.

## Making a fresh batch of Cloud Foundry on Kubernetes

Below, we'll cover all the moving pieces associated with sprinkling a bit of Cloud Foundry over a nice hot fresh batch of Kubernetes. Remember to add salt to taste!

![yummy fries](https://raw.githubusercontent.com/cweibel/ghost_blog_pics/master/emmy-smith-LEjEst7lLfU-unsplash.jpg)

*(Photo by Emmy Smith on Unsplash)*

There are a few layers to this process which include:

 - Deploying Kubernetes
 - Install Helm
 - Install the cf-operator
 - Create a configuration for KubeCF
 - Deploy KubeCF
 - Log into Cloud Foundry
 - Refill the ketchup bottle

## EKS

In our previous blog, [Getting Started with Amazon EKS](https://www.starkandwayne.com/blog/getting-started-with-amazons-eks), we create a Kubernetes cluster using the `eksctl` tool. This gives you time to ready a short story from Tolstoy and, in the end, a Kubernetes cluster is born. This allows you to deploy pods, deployments, and other exciting Kubernetes resources without having to manage the `master` nodes yourself.

Before continuing be sure you are targeting your `kubectl` CLI with the kubeconfig for this cluster. Run a `kubectl cluster-info dump | grep "cluster-name"` to verify that the name of the cluster in EKS matches what `kubectl` has targeted. This is important to check if you've been experimenting with other tools like `minikube` in the meantime since deploying the EKS cluster.

## Install Helm

Helm is a CLI tool for templating Kubernetes resources. Helm Charts bundle up a group of Kubernetes YAML files to deploy a particular piece of software. The [Bitnami PostgreSQL Helm Chart](https://hub.helm.sh/charts/bitnami/postgresql) installs an instance of the database with persistent storage and expose it via a service. The `cf-operator` and `kubecf` projects we use below are also Helm Charts.

To install `helm` on MacOS with Homebrew:

```
brew install helm
```

If you are using a different operating system, other means of installation are documented at [https://github.com/helm/helm#install](https://github.com/helm/helm#install).

This installs Helm v3.  All of the subsequent commands will assume you are using this newer version of Helm. Note that the instructions in the `cf-operator` and `kubecf` GitHub repos use Helm v2 style commands. A short guide to converting Helm commands is here on the [Stark & Wayne blog site](https://www.starkandwayne.com/blog/helm-3-how-do-i-do-helm-2-stuff/).

## Install the cf-operator Helm Chart

Since we are using Helm v3, we'll need to create a namespace for the cf-operator to use it (v2 would have done this for you automatically):

```
➜ kubectl create namespace cf-operator
```

Now you can install the `cf-operator`:

```
➜ helm install cf-operator \
  --namespace cf-operator \
  --set "global.operator.watchNamespace=kubecf" \
  https://s3.amazonaws.com/cf-operators/release/helm-charts/cf-operator-v2.0.0-0.g0142d1e9.tgz
```
When completed, you will have two pods in the `kubecf` namespace which look similar to:

```
➜ kubectl get pods -n kubecf
NAME                                      READY   STATUS    RESTARTS   AGE
cf-operator-5ff5684bb9-tsw2f              1/1     Running   2          8m4s
cf-operator-quarks-job-5dcc69584f-c2vnw   1/1     Running   2          8m3s
```

The pods may fail once or twice while initializing. This is ok as long as both report as "running" after a few minutes.

## Configure kubecf

Before installing the kubecf Helm Chart, you'll need to create a configuration file.  

The complete configuration file with all options is available at [https://github.com/SUSE/kubecf/blob/master/deploy/helm/kubecf/values.yaml](https://github.com/SUSE/kubecf/blob/master/deploy/helm/kubecf/values.yaml).  The examples below populate portions of this YAML file.

## Starting Configuration

The absolute minimum configuration of `values.yaml` file for KubeCF on EKS is:

 - `system_domain` - The URL Cloud Foundry will be accessed from
 - `kube.service_cluster_ip_range` - The /24 network block for services
 - `kube.pod_cluster_ip_range` - The /16 network block for the pods

[Project Eirini](https://github.com/cloudfoundry-incubator/eirini) swaps out using Diego Cells for Container Runtime and instead uses Kubernetes pods/statefulsets for each application instance. This feature is enabled by adding one more configuration to the `values.yaml` to the minimal configuration `features.eirini.enabled: true`.

```
system_domain: system.kubecf.lab.starkandwayne.com

kube:
  service_cluster_ip_range: 10.100.0.0/16
  pod_cluster_ip_range: 192.168.0.0/16

features:
  eirini:
    enabled: true
```

This configuration will get you:

 - A Cloud Foundry listening to the login at `https://api.<system_domain>`
 - Apps as Kubernetes Pods in the `kubecf-eirini` namespace
 - CF Control Plane with 1 pod/statefulset per `instance_group`
 - CF internal databases as a pod/statefulset
 - CF Blobstore as a pod/statefulset

## Production Configuration

There are many more configuration options available in the default `values.yaml` file, a more "Production Worthy" deployment would include:

### Scaling Control Plane

Having CF Control Plane with 1 pod per `instance_group` results in the deployment not being HA. If any of the pods stop, that part of the Cloud Foundry Control Plane stops functioning since there was only 1 instance.

There are a few ways of enabling multiple pods per instance group:

- Enable Multi AZ and HA Settings. Simply set the corresponding values to `true` in `values.yaml`:

```
multi_az: true
high_availability: true
```

- Manually set the instance group sizing in values.yaml:

```
sizing:
  adapter:
    instances: 3
  api:
    instances: 4
  ...  
  tcp_router:
    instances: 4
```

Note that setting `instance: sizes` overrides the default values of `high_availability: true`

### Using an external database (RDS) for the CF internal databases.

This needs to be created beforehand and the values populated in `values.yaml`:

```
features:
  external_database:
    enabled: true
    type: postgres
    host: postgresql-instance1.cg034hpkmmjt.us-east-1.rds.amazonaws.com
    port: 5432
    databases:
      uaa:
        name: uaa
        password: uaa-admin
        username: 698embi40dlb98403pbh
      cc:
        name: cloud_controller
        password: cloud-controller-admin
        username: 659ejkg84lf8uh8943kb
  ...
      credhub:
        name: credhub
        password: credhub-admin
        username: ffhl38d9ghs93jg023u7g
```

### Enable AutoScaler

[App-Autoscaler](https://github.com/cloudfoundry/app-autoscaler) is an add-on to Cloud Foundry to automatically scale the number of application instances based on CPU, memory, throughput, response time, and several other metrics. You can also add your own custom metrics as of v3.0.0. You decide which metrics you want to scale your app up and down by in a policy and then apply the policy to your application. Examples of usage can be found [here](https://www.starkandwayne.com/blog/setting-up-cf-autoscaler-using-genesis/).

Add the lines below to your `values.yaml` file to enable App Autoscaler:

```
features:
  autoscaler:
    enabled: true
```

### CF Blobstore

As of this writing, there is not a documented way to scale the `singleton-blobstore` or have it leverage S3. Let me know in the comments if you know of a way to do this!

### Install the KubeCF Helm Chart

Once you have assembled your `values.yaml` file with the configurations you want, the `kubecf` Helm Chart can be installed.

In the example below, an absolute path is used to the `values.yaml` file, you'll need to update the path to point to your file.

```
➜  helm install kubecf \
  --namespace kubecf \
  --values /Users/chris/projects/kubecf/values.yaml \
  https://github.com/SUSE/kubecf/releases/download/v0.2.0/kubecf-0.2.0.tgz
```
The install takes about 20 minutes to run and if you run a watch command, eventually you will see output similar to:

```
➜  watch -c "kubectl -n kubecf get pods"

NAME                                      READY   STATUS    RESTARTS   AGE
cf-operator-5ff5684bb9-tsw2f              1/1     Running   0          4h
cf-operator-quarks-job-5dcc69584f-c2vnw   1/1     Running   0          4h
kubecf-adapter-0                          4/4     Running   0          3h
kubecf-api-0                              15/15   Running   1          3h
kubecf-bits-0                             6/6     Running   0          3h
kubecf-bosh-dns-7787b4bb88-44fjf          1/1     Running   0          3h
kubecf-bosh-dns-7787b4bb88-rkjsr          1/1     Running   0          3h
kubecf-cc-worker-0                        4/4     Running   0          3h
kubecf-credhub-0                          5/5     Running   0          3h
kubecf-database-0                         2/2     Running   0          3h
kubecf-diego-api-0                        6/6     Running   2          3h
kubecf-doppler-0                          9/9     Running   0          3h
kubecf-eirini-0                           9/9     Running   9          3h
kubecf-log-api-0                          7/7     Running   0          3h
kubecf-nats-0                             4/4     Running   0          3h
kubecf-router-0                           5/5     Running   0          3h
kubecf-routing-api-0                      4/4     Running   0          3h
kubecf-scheduler-0                        8/8     Running   6          3h
kubecf-singleton-blobstore-0              6/6     Running   0          3h
kubecf-tcp-router-0                       5/5     Running   0          3h
kubecf-uaa-0                              6/6     Running   0          3h
```

At this point Cloud Foundry is alive, now we just need to a way to access it.

### Add a cname record

There are 3 load balancers which are created during the deployment and can be viewed by:

```
➜  cf-env git:(master) kubectl get services -n kubecf
NAME                           TYPE           CLUSTER-IP       EXTERNAL-IP                                                               PORT(S)

kubecf-router-public           LoadBalancer   10.100.193.124   a34d2e33633c511eaa0df0efe1a642cf-1224111110.us-west-2.elb.amazonaws.com   80:31870/TCP,443:31453/TCP                                                                                                                        3d
kubecf-ssh-proxy-public        LoadBalancer   10.100.221.190   a34cc18bf33c511eaa0df0efe1a642cf-1786911110.us-west-2.elb.amazonaws.com   2222:32293/TCP                                                                                                                                    3d
kubecf-tcp-router-public       LoadBalancer   10.100.203.79    a34d5ca4633c511eaa0df0efe1a642cf-1261111914.us-west-2.elb.amazonaws.com   20000:32715/TCP,20001:30059/TCP,20002:31403/TCP,20003:32130/TCP,20004:30255/TCP,20005:32727/TCP,20006:30913/TCP,20007:30725/TCP,20008:31713/TCP   3d
```

We need to associate the system_domain in `values.yaml` to the URL associated with the LoadBalancer named `kubecf-router-public`.

In CloudFlare, we add a `cname` record pointing the ELB to the system domain:

![cloudflare dns cname example](https://raw.githubusercontent.com/cweibel/ghost_blog_pics/master/kubecf-dns.png)

If you are using Amazon Route53, you can follow the instructions [here](https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/routing-to-elb-load-balancer.html).

### Log Into CF
Assuming you have the CF CLI already installed, ([see this if not](https://docs.cloudfoundry.org/cf-cli/install-go-cli.html)), you can target and authenticate to the Cloud Foundry deployment as seen below, remembering to update the system domain URL to the one registered in the previous step:

```
cf api --skip-ssl-validation "https://api.system.kubecf.lab.starkandwayne.com"

admin_pass=$(kubectl get secret \
        --namespace kubecf kubecf.var-cf-admin-password \
        -o jsonpath='{.data.password}' \
        | base64 --decode)

cf auth admin "${admin_pass}"
```

The full Kubecf documentation for logging in is [documented here](https://github.com/SUSE/kubecf/blob/002b49ad26109e175812c04a3764bd8712e54580/doc/dev/general.md#access). The "admin" password is stored as a Kubernetes Secret.

**That's it!**  You can now create all the drnic orgs and spaces you desire with the CF CLI and deploy the 465 copies of [Spring Music](https://github.com/cloudfoundry-samples/spring-music) to the platform you know and love!

## There be Dragons

I did not successfully deploy Kubecf on my first attempt, or my tenth. But most of these failed attempts were self inflicted and easily avoided. Below are a few snags encountered and their workarounds.

### Pods Evicting or HA never gets to HA Count

If your pods start to get evicted as you scale the components, you are likely running out of resources. If you run a `kubectl describe node <nameofnode>`, you'll see:

```
  Type     Reason                 Age                From                                                  
  ----     ------                 ----               ----                                              
  Warning  EvictionThresholdMet   13s (x6 over 22h)  kubelet, Attempting to reclaim ephemeral-storage
  Normal   NodeHasDiskPressure    8s (x6 over 22h)   kubelet, Node status is now: NodeHasDiskPressure
  Normal   NodeHasNoDiskPressure  8s (x17 over 15d)  kubelet, Node status is now: NodeHasNoDiskPressure
```

Either scale out the Kubernetes cluster with the `eksctl` tool or go easy on the `sizing.diego_cell.instances: 50bazillion` setting in the kubecf `values.yaml` file!

### Failed deployment, cannot curl system URL

There is more than one version of Kubecf Release v0.1.0 and some of them seemed to not work for me, your experience may vary. When I used the tarball `kubecf-0.1.0-002b49a.tgz` documented in the v0.1.0 release [here](https://github.com/SUSE/kubecf/releases/tag/v0.1.0), my router ELB had 0 of 2 members which I'm assuming was related to a [bad port configuration](https://github.com/SUSE/kubecf/pull/300). There is a [KubeCF Helm Chart S3 bucket](https://cf-operators.s3.amazonaws.com/helm-charts/index.html) which has additional tarballs, `cf-operator-v1.0.0-1.g424dd0b3.tgz` is the one used in the examples here.

A note: the original blog post had instructions for v0.1.0, the above is still true but you should not have the same issues with the v0.2.0 instructions that are now in this blog.

### Failed Deployment, kubecf-api-0 stuck

If you perform a `helm uninstall kubecf` and attempt to reinstall at a later time, there are a few PVC's in the `kubecf` namespace which you will need to delete after the first uninstall. If you don't, you'll wind up with an error similar to this on the `kubecf-api*` pod:

```
➜ kubectl logs kubecf-api-0 -c bosh-pre-start-cloud-controller-ng -f
...
redacted for brevity...
...
VCAP::CloudController::ValidateDatabaseKeys::EncryptionKeySentinelDecryptionMismatchError
```

To fix, list the PVC's and delete them:

```
➜ kubectl get pvc -n kubecf

NAME                                                          STATUS   
kubecf-database-pvc-kubecf-database-0                         Bound
kubecf-singleton-blobstore-pvc-kubecf-singleton-blobstore-0   Bound

➜ kubectl delete pvc kubecf-database-pvc-kubecf-database-0
➜ kubectl delete pvc kubecf-singleton-blobstore-pvc-kubecf-singleton-blobstore-0

➜ helm install kubecf #again
```
## Additional Reading

Since publishing this article, we're seeing more people succeeding deploying Cloud Foundry to Kubernetes.  This is very encouraging, keep it up!

João Pinto has written up a [tutorial to deploy Cloud Foundry](https://medium.com/@jmpinto/deploying-cloudfoundry-on-a-local-kubernetes-9103a57bf713) on [kind](https://github.com/kubernetes-sigs/kind) (Kubernetes IN Docker).


## Follow Up Reading

But wait, there's more!

Want to run multiple KubeCF installs on a single Kubernetes Cluster?  Each in their own namespace?  With a couple minor helm configurations you can have as many as you'd like!

See my new blog [Limes & Coconuts: Running Multiple KubeCF Deployments on One Kubernetes Cluster](https://cweibel.github.io/blog/2020/03/18/more-limes-running-multiple-kubecf-on-one-cluster) to discover how you can have an entire bucket of limes.


