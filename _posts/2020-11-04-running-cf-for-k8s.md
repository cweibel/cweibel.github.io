---
layout: post
title: "Running cf-for-k8s on MiniKube"
date: 2020-11-04
---

![yoda_koala](https://raw.githubusercontent.com/cweibel/ghost_blog_pics/master/henrique-felix-tWpWrLrdic8-unsplash-2.jpg)


Photo by [Henrique Félix](https://unsplash.com/@henriquefelix?utm_source=unsplash&utm_medium=referral&utm_content=creditCopyText) on [Unsplash](https://unsplash.com/?utm_source=unsplash&utm_medium=referral&utm_content=creditCopyText)

The [`cf-for-k8s`](https://github.com/cloudfoundry/cf-for-k8s) project is an interesting twist to running Cloud Foundry on Kubernetes using more "Kubernetes"-native projects like Istio, Eirini, and Fluentd.

If you are interested in giving `cf-for-k8s` a spin on your own laptop the instructions below will create a Kubernetes (Minikube) and install `cf-for-k8s`.

These instructions borrow **heavily** from [https://github.com/cloudfoundry/cf-for-k8s](https://github.com/cloudfoundry/cf-for-k8s) but I've attempted to streamline the process by making a number of assumptions for you including the use of MacOS and homebrew. Check out the link for additional instructions and options that were removed to streamline the install.

`<minikube ip>.netip.cc` will be used as the domain for the installation so no additional DNS solutions are required.

## Step 1 - Installing Tools

Install CLI Tools & Minikube

```
brew tap k14s/tap
brew install ytt kbld kapp imgpkg kwt vendir yq
brew install kubectl 
brew install cloudfoundry/tap/cf-cli@7
brew install cloudfoundry/tap/bosh-cli
brew install minikube
```

## Step 2 - Start Kubernetes

Start Minikube

```
minikube start --cpus=6 --memory=8g --kubernetes-version="1.19.2" --driver=docker
```

### Step 3 - Configure Minikube

Enable `metrics-server`:

```
minikube addons enable metrics-server
```

Obtain minikube IP that will be the basis for your domain URL.

```
minikube ip
MINIKUBE_IP=$(minikube ip)
```

Use minikube tunnel to expose the LoadBalancer service for the ingress gateway:

```
sudo minikube tunnel  # be sure to run in another session
```

This should be run in a separate terminal as this will block.
sudo give capabilities to the tunnel to open ports 80 and 443 for the gorouters.

The `kapp deploy` command will not exit successfully until this command is run to allow minikube to create the LoadBalancer service.

## Step 4 - Setup Config Files & Deploy Cloud Foundry

 - Clone and initialize this git repository:

   ```
   TMP_DIR="${PWD}/cf-for-k8s-tmp"
   mkdir -p ${TMP_DIR}
   git clone https://github.com/cloudfoundry/cf-for-k8s.git -b main
   cd cf-for-k8s
   ```

 - Create a "CF Installation Values" file and configure it `values.yml` file using the BOSH CLI to generate self signed certs and passwords.

   ```
   ./hack/generate-values.sh -d $MINIKUBE_IP.netip.cc > ${TMP_DIR}/cf-values.yml
   ```

 - Create an OCI-compliant app registry and configure the values file, [hub.docker.com](hub.docker.com) is pretty easy to get started:

   1. Create an account in [hub.docker.com](hub.docker.com). Note down the user name and password you used during signup.
   2. Create a repository in your account. Note down the repository name.
   3. Add the following registry config block to the end of `../cf-for-k8s-tmp/cf-values.yml` file, swapping in the `<my_username>` and `<my_password>` values from the previous two steps:

   ```
      app_registry:
        hostname: https://index.docker.io/v1/
        repository_prefix: "<my_username>"
        username: "<my_username>"
        password: "<my_password>"
      remove_resource_requirements: true
      enable_automount_service_account_token: true
      use_first_party_jwt_tokens: true
   ```

 - Run the following commands to install Cloud Foundry on your Kubernetes (Minikube) cluster:

   1. Render the final K8s template to raw K8s configuration:

      ```
      ytt -f config -f ${TMP_DIR}/cf-values.yml > ${TMP_DIR}/cf-for-k8s-rendered.yml
      ```
    
   2. Install using kapp and pass the above K8s configuration file
 
     ```
     kapp deploy -a cf -f ${TMP_DIR}/cf-for-k8s-rendered.yml -y
     ```

## Step 5 - Validate the Deployment

Target your CF CLI to point to the new CF instance and login using the admin credentials for key `cf_admin_password` in `${TMP_DIR}/cf-values.yml`:

```
cf api --skip-ssl-validation https://api.$MINIKUBE_IP.netip.cc
cf auth admin "$(yq read  ${TMP_DIR}/cf-values.yml 'cf_admin_password')"
```

Create an org/space for your app:

```
cf create-org test-org
cf create-space -o test-org test-space
cf target -o test-org -s test-space
```

Deploy a source code based app:

```
cf push test-node-app -p tests/smoke/assets/test-node-app
```

You should see the following output from the above command:

```
Pushing app test-node-app to org test-org / space test-space as admin...
Getting app info...
Creating app with these attributes...
... omitted for brevity ...
type: web
instances: 1/1
memory usage: 1024M
routes: test-node-app.<cf-domain>
state since cpu memory disk details
0 running 2020-03-18T02:24:51Z 0.0% 0 of 1G 0 of 1G
```

Validate the app is reachable over **https**:

```
curl -k https://test-node-app.$MINIKUBE_IP.netip.cc
```

You should see the following output:

> Hello World

Now do a little dance that you were able to get CF running on Kubernetes!

## Work Arounds

### Work Around 1

Older cf-cli was installed.  To overwrite:

```
brew link --overwrite cf-cli@7
```

### Work Around 2

An older Minikube was installed. If you don't need it anymore and getting weird errors, delete the cluster made by the older Minikube and create a new one:

```
minikube delete
minikube start
```

### Work Around 3

Error:

```
Exiting due to MK_USAGE: Docker Desktop has only 1998MB memory but you specified 8192MB
```

Go into `Docker Desktop` > `Preferences` > `Advanced` and move the slider on Memory over to at least 8GB, 16GB if you have it.

### Work Around 4

Error:

```
ytt: Error: Unknown comment syntax at line cf-values.yml:443: ' Below needed for Minikube':
```

Don't attempt to put comments starting with `#` into `cf-values.yml`, while valid yaml the parser errors out.

### Work Around 5

The instructions for creation of a Docker Hub account ask to create a repository, however there is nowhere to put the name of the repo. Attempting to use it in the `repository_prefix` results in a `unsupported status code 401` error for `builder/cf-default-builder (kpack.io/v1alpha1) namespace: cf-workloads-staging`


### Work Around 6

CF CLI cf auth command fails because it cannot pull the password. This may be due to differences in `yq`, the instructions say the command is:

```
cf auth admin "$(yq -r '.cf_admin_password' ${TMP_DIR}/cf-values.yml)"
```

But my copy of `yq` installed doesn't know `-r` (does have read) and even with the read command the next two parameters are reversed.


### Work Around 7

After minikube stop and subsequent minikube start, the UAA pod may not come up clean:

```
 NAMESPACE    NAME                   READY   STATUS                  RESTARTS   AGE
 cf-system    uaa-6bf497b5b9-snlg6   0/3     Init:CrashLoopBackOff   4          101m
```

Looking at the logs you'll see it failed to mount the secrets:

```
 Events:
 Type     Reason       Age                    From     Message
 ----     ------       ----                   ----     -------
 Warning  FailedMount  6m15s                  kubelet  MountVolume.SetUp failed for volume "cc-admin-client-credentials-file" : failed to sync secret cache: timed out waiting for the condition
 Warning  FailedMount  6m15s                  kubelet  MountVolume.SetUp failed for volume "ca-certs-files" : failed to sync secret cache: timed out waiting for the condition
 Warning  FailedMount  6m15s                  kubelet  MountVolume.SetUp failed for volume "istiod-ca-cert" : failed to sync configmap cache: timed out waiting for the condition
 Warning  FailedMount  6m15s                  kubelet  MountVolume.SetUp failed for volume "jwt-policy-signing-keys-file" : failed to sync secret cache: timed out waiting for the condition
 Warning  FailedMount  6m15s                  kubelet  MountVolume.SetUp failed for volume "uaa-token-gc8nw" : failed to sync secret cache: timed out waiting for the condition
 Warning  FailedMount  6m15s                  kubelet  MountVolume.SetUp failed for volume "admin-client-credentials-file" : failed to sync secret cache: timed out waiting for the condition
 Warning  FailedMount  6m14s (x2 over 6m15s)  kubelet  MountVolume.SetUp failed for volume "encryption-keys-file" : failed to sync secret cache: timed out waiting for the condition
 Warning  FailedMount  6m14s (x2 over 6m15s)  kubelet  MountVolume.SetUp failed for volume "cf-api-controllers-client-credentials-file" : failed to sync secret cache: timed out waiting for the condition
 Warning  FailedMount  6m14s (x2 over 6m15s)  kubelet  MountVolume.SetUp failed for volume "cf-admin-user-credentials-file" : failed to sync secret cache: timed out waiting for the condition
 Warning  FailedMount  6m14s                  kubelet  MountVolume.SetUp failed for volume "uaa-config" : failed to sync configmap cache: timed out waiting for the condition
 Warning  FailedMount  6m14s                  kubelet  MountVolume.SetUp failed for volume "saml-keys-file" : failed to sync secret cache: timed out waiting for the condition
 Warning  FailedMount  6m14s                  kubelet  MountVolume.SetUp failed for volume "smtp-credentials-file" : failed to sync secret cache: timed out waiting for the condition
 Warning  FailedMount  6m1s (x10 over 6m6s)   kubelet  (combined from similar events): MountVolume.SetUp failed for volume "cc-admin-client-credentials-file" : failed to sync secret cache: timed out...
```

The quick fix is to `kubectl delete pod uaa-6bf497b5b9-8klh9 -n cf-system` and let the scheduler build out a new pod and remount the secrets.

## What's Next?

You can also try out the other solution to running Cloud Foundry on Kubernetes called KubeCF. In [this blog post](https://cweibel.github.io/blog/2020/10/21/deploying-kubecf-to-eks-revisited) you can read up on how to deploy KubeCF on Amazon EKS using just a bit of Terraform.

Thanks!

PS: Thank you to Tyler Bird for pointing out some missing bits in the homebrew commands!

