---
layout: post
title: "Running the CrunchyData Postgres Operator on Minikube"
date: 2019-06-18
---

![map](https://raw.githubusercontent.com/cweibel/ghost_blog_pics/master/joshua-hoehne-0AzG_H6lOPo-unsplash.jpg)



Photo by [Joshua Hoehne](https://unsplash.com/@mrthetrain?utm_source=unsplash&utm_medium=referral&utm_content=creditCopyText) on [Unsplash](https://unsplash.com/s/photos/uri?utm_source=unsplash&utm_medium=referral&utm_content=creditCopyText)


Why are we interested in Postgres on Kubernetes?  I gave a talk last week on beginning the journey to getting PostgreSQL running on Kubernetes at the [Buffalo Web Developers Database Meetup](https://www.meetup.com/buffalowebdevelopers/events/262136467/).  There were examples on a simple deployment, configuring stateful sets, adding persistent volume claims and even a liveness probe.  What I wanted to show was the "next step" where all the tribal knowledge around living with Postgres was wrapped into a Kubernetes Operator.  Alas, I wasn't able to figure it out in time for the meetup so this blog post was created.

Versions shown in this example:

 - minikube v1.1.1 with Kubernetes v1.14.0
 - CrunchData Postgres Operator v4.0.0

I'm calling out these particular versions because there are other examples out on the interwebs which deal with older versions and many of the instructions no longer apply.

## Assumptions

 - `minikube` [is installed](https://kubernetes.io/docs/tasks/tools/install-minikube/), be sure to also have a hypervisor installed (VirtualBox, HyperKit, VMware Fusion)
 - You have golang installed (hint: `brew install go`)
 - Running on a Mac (only important for picking which flavor of `pgo` and `expenv` to grab)

 ## Basic Install

There are instructions for the general installation of the operator at [https://access.crunchydata.com/documentation/postgres-operator/4.0.0/installation/operator-install/](https://access.crunchydata.com/documentation/postgres-operator/4.0.0/installation/operator-install/).  What we'll be focusing on is instructions specifically for getting this to run on `minikube` so I'll pull out a subset of the commands from the instructions and add a few which maybe weren't the most obvious. Let's get started!

### Get the Project

This is a straight copy from the original instructions:

```
mkdir -p $HOME/odev/src/github.com/crunchydata $HOME/odev/bin $HOME/odev/pkg
cd $HOME/odev/src/github.com/crunchydata
git clone https://github.com/CrunchyData/postgres-operator.git
cd postgres-operator
git checkout 4.0.0
```

Before you move onto the next step there are a couple of bits to add, namely getting the expenv and pgo executables into your PATH:

```
echo "Getting expenv..."
wget -O $HOME/odev/bin/expenv \
   https://github.com/CrunchyData/postgres-operator/releases/download/4.0.0/expenv-mac
chmod +x $HOME/odev/bin/expenv

echo "Getting pgo..."
wget -O $HOME/odev/bin/pgo \
   https://github.com/CrunchyData/postgres-operator/releases/download/4.0.0/pgo-mac
chmod +x $HOME/odev/bin/pgo
```

If you aren't using osx switch `expenv-mac` and `pgo-mac` to use the binaries for your particular os.

Now back to the original instructions to set environment varialbes with a note, if you use `zsh` swap references to `.bashrc` below to `.zshrc`:

```
cat $HOME/odev/src/github.com/crunchydata/postgres-operator/examples/envs.sh >> $HOME/.bashrc
source $HOME/.bashrc
```

### Create the Namespaces

We'll just use the defaults, this will result in a total of 3 namespaces getting created:

```
export NAMESPACE=pgouser1,pgouser2
export PGO_OPERATOR_NAMESPACE=pgo
make setupnamespaces
```

You can run this command multiple times, subsequent runs should produce output similar to:

```
cd deploy && ./setupnamespaces.sh
creating namespaces to deploy the Operator into...
namespace pgo is already created

creating namespaces for the Operator to watch and create PG clusters into...
namespace pgouser1 is already created
namespace pgouser2 is already created
```

### Configure the Storage

One of the things we need to configure is persistent volumes claims for the Postgres data to live on.  It would be sad to delete a pod and have the database lose all of its data.

Since we are using `minikube` there is a storage class already defined for us:

```
➜  postgres-operator git:(7fb5f613) ✗ kubectl get storageclasses
NAME                 PROVISIONER                AGE
standard (default)   k8s.io/minikube-hostpath   58d
```

It is using `hostpath` which uses the underlying `minikube` vm as storage.  Let's leverage this knowledge and do a bit of tweaking, `vim conf/postgres-operator/pgo.yaml` and replace:

```
PrimaryStorage: storageos
BackupStorage: storageos
ReplicaStorage: storageos
BackrestStorage: storageos
```

with

```
PrimaryStorage: hostpathstorage
BackupStorage: hostpathstorage
ReplicaStorage: hostpathstorage
BackrestStorage: hostpathstorage
```

### Configure the Operator Security

Let's just keep it simple and use the original example:

```
cp ./conf/postgres-operator/pgouser $HOME/.pgouser
cp ./conf/postgres-operator/pgorole $HOME/.pgorole
```

### Default Installation

When the `pgo` operator deploys and creates a Kubernetes service it will be exposed with a ClusterIP and not the more useful `NodePort` which it will need to log in.  To make this change edit `deploy/service.json` and replace:

```
        "type": "ClusterIP",
```

with

```
        "type": "NodePort",
```

Now back to the original script:

```
make installrbac
make deployoperator
```

When this last operation is done you'll see the following pods if was successfully deployed:

```
➜  postgres-operator git:(7fb5f613) ✗ kubectl get pods -n pgo
NAME                                READY   STATUS    RESTARTS   AGE
postgres-operator-9944d4499-l8kjc   3/3     Running   3          2d22h
```

### Wire up pgo

`pgo` is the CLI used to manage Crunchy Data Postgres Clusters.  To connect `pgo` to the service running on the three pods created in the previous section you need 3 pieces of information:

 1. The IP address of the service
 1. The external port the service is bound to
 1. The certificates

The first one is easy, simply run `minikube status`, the last line shows the IP address `192.168.100.99`:

```
➜  minikube status
host: Running
kubelet: Running
apiserver: Running
kubectl: Correctly Configured: pointing to minikube-vm at 192.168.99.100
```

To get the port use the `kubectl get services` command to show the port the `postgres-operator` service is bound to, in our case `31854`:

```
➜  kubectl get services -n pgo
NAME                TYPE       CLUSTER-IP       EXTERNAL-IP   PORT(S)          AGE
postgres-operator   NodePort   10.107.127.184   <none>        8443:31854/TCP   3d1h
```

Using those two pieces of information we can now set the `pgo` API path:

```
export PGO_APISERVER_URL=https://192.168.99.100:31854
```

Now for the certs, back to the original instructions these are loaded by:

```
export PGO_CA_CERT=$PGOROOT/conf/postgres-operator/server.crt
export PGO_CLIENT_CERT=$PGOROOT/conf/postgres-operator/server.crt
export PGO_CLIENT_KEY=$PGOROOT/conf/postgres-operator/server.key
```

If everything went happily you can run `pgo version` and see a message similar to:

```
➜  pgo version
pgo client version 4.0.0
pgo-apiserver version 4.0.0
```

If you see a message like the following (with the Error: output) you missed one of the previous steps and will need to fix it before you proceed:

```
➜  pgo version
pgo client version 4.0.0
Error:
```

## It's Alive!

Great, how do I use it?  Good thinking. Let's start with the basics and create a Postgres instance:

```
➜  pgo create cluster pickles -n pgouser1
created Pgcluster pickles
workflow id 48391e42-2099-4d51-8055-ad79d045018f
```

What happened?  You created a postgres cluster in the `pgouser1` kubernetes namespace with the name `pickles`. Why pickles? I'm weird and like pickles. To see the cluster run:

```
➜  kubectl get pods -n pgouser1
NAME                                              READY   STATUS 
pickles-749d4bb77d-hw8z7                          1/1     Running
pickles-backrest-shared-repo-76df9cff78-cb58n     1/1     Running
pickles-stanza-create-kbmm7                       0/1     Completed
```

So the `pickles` database cluster is created and now you'll need to know what port is mapped to connect to it so run `kubectl get services`:

```
➜  kubectl get services -n pgouser1
NAME                             TYPE        CLUSTER-IP       PORT(S)
pickles                          ClusterIP   10.111.108.118   5432/TCP,9100/TCP,10000/TCP,2022/TCP,9187/TCP
pickles-backrest-shared-repo     ClusterIP   10.108.123.32    2022/TCP
```

Grrrr, there's that pesky `ClusterIP` again.  Without hacking the codebase we can modify the service to use `NodePort` instead.  To do that pull down the service definition for `pickles` service via:

```
➜  kubectl get service pickles -n pgouser1 -o yaml > pgouser1.pickles.service.yaml
```

Edit the file (`vim pgouser1.pickles.service.yaml`) and replace:

```
  type: ClusterIP
```

with

```
  type: NodePort
```

Apply the configuration change to the service with the following command:

```
➜  kubectl apply -n pgouser1 -f pgouser1.pickles.service.yaml
Warning: kubectl apply should be used on resource created by either kubectl create --save-config or kubectl apply
service/pickles configured
```

If you rerun the `kubectl get services -n pgouser1` command again you'll see ports now added:

```
➜  kubectl get services -n pgouser1
NAME                             TYPE        CLUSTER-IP       PORT(S)
pickles                          NodePort    10.111.108.118   5432:30814/TCP,9100:31207/TCP,10000:32264/TCP,2022:32022/TCP,9187:31960/TCP
pickles-backrest-shared-repo     ClusterIP   10.108.123.32    2022/TCP
```

From the previous command we can see the normal postgres 5432 port is bound to the external port of `30814` which means we now have all the information we need to connect to postgres, right?

```
➜  psql -h 192.168.99.100 -p 30814 -U postgres postgres
Password for user postgres:
```

Password?  You can try to guess, but you won't get it for a while.  The operator has set a password for the `postgres` user, to look up the users which were created by the operator use the `pgo show user` command:

```
➜  pgo show user pickles -n pgouser1

cluster : pickles

secret : pickles-postgres-secret
    username: postgres
    password: xaCkhNVGaJ

secret : pickles-primaryuser-secret
    username: primaryuser
    password: IorhwdDiVG

secret : pickles-testuser-secret
    username: testuser
    password: ZVKpDCGXOp
```

So the password for the `postgres` user is `xaCkhNVGaJ`, now we can log in:

```
➜  psql -h 192.168.99.100 -p 30814 -U postgres postgres
Password for user postgres: <type the password here>
psql (11.2, server 11.3)
Type "help" for help.

postgres=#
```

We're in!

## Leveraging pgo for Database Operations

Below is a handpicked overview of the tribal knowledge of operations included with the operator.  For a deeper dive refer to [https://access.crunchydata.com/documentation/postgres-operator/4.0.0/operatorcli/pgo-overview/](https://access.crunchydata.com/documentation/postgres-operator/4.0.0/operatorcli/pgo-overview/)

### Manual Backup & Restore

One of the first things we'll show is the backup AND restore functionality of the operator.

This operator utilizes a tool called [`pgBackRest`](https://pgbackrest.org/) which is an improvement over `pg_dump` since it can leverage multiple cores to perform the backup and compression.  The backups & restores can also be remote to the server and can perform full, incremental or differential backups.  The pgBackRest tool is deployed as a pod by default when you create a cluster, below it is named `pickles-backrest-shared-repo-76df9cff78-cb58n`:

```
➜  kubectl get pods -n pgouser1
NAME                                              READY   STATUS      RESTARTS   AGE
pickles-749d4bb77d-hw8z7                          1/1     Running     0          17h
pickles-backrest-shared-repo-76df9cff78-cb58n     1/1     Running     0          17h
pickles-stanza-create-kbmm7                       0/1     Completed   0          17h
```

Performing a manual full backup is simple using the `pgo backup` command:

```
➜  pgo backup pickles -n pgouser1
created Pgtask backrest-backup-pickles
```

To see the backups which are available the `pgo show backup` command can be used:

```
➜  pgo show backup pickles -n pgouser1

backrest : pickles

Storage Type: local
stanza: db
    status: ok
    cipher: none

    db (current)
        wal archive min/max (11-1): 000000010000000000000001/00000007000000000000000B

        full backup: 20190618-143358F
            timestamp start/stop: 2019-06-18 14:33:58 / 2019-06-18 14:34:12
            wal start/stop: 000000010000000000000004 / 000000010000000000000004
            database size: 30.3MB, backup size: 30.3MB
            repository size: 3.6MB, repository backup size: 3.6MB

        incr backup: 20190618-143358F_20190618-145825I
            timestamp start/stop: 2019-06-18 14:58:25 / 2019-06-18 14:58:29
            wal start/stop: 000000010000000000000006 / 000000010000000000000006
            database size: 30.4MB, backup size: 2.3MB
            repository size: 3.6MB, repository backup size: 229.9KB
            backup reference list: 20190618-143358F

        incr backup: 20190618-143358F_20190618-150535I
            timestamp start/stop: 2019-06-18 15:05:35 / 2019-06-18 15:05:44
            wal start/stop: 000000020000000000000009 / 000000020000000000000009
            database size: 30.5MB, backup size: 353.4KB
            repository size: 3.6MB, repository backup size: 21.2KB
            backup reference list: 20190618-143358F, 20190618-143358F_20190618-145825I
```

The above shows 3 backups.  The first one is a full backup, the next two are incremental backups that contain only the changes from the full backup.  Leveraging the full and two incremental backups the database can be restored to any point in time between `2019-06-18 14:33:58` and `2019-06-18 15:05:35`.

To manually kick off a point-in-time-recovery (PITR) use the `pgo restore` command:

```
➜  pgo restore pickles --pitr-target="2019-06-18 14:34:26.279693+00" --backup-opts="--type=time --log-level-console=info" -n pgouser1
Warning:  If currently running, the primary database in this cluster will be stopped and recreated as part of this workflow!
WARNING: Are you sure? (yes/no): yes
restore performed on pickles to pickles-ubsl opts=--type=time --log-level-console=info pitr-target=2019-06-18 14:34:26.279693+00
workflow id 463d5103-5d2b-4dd7-901f-f49f7f599166
```

While the restore is running you won't be able to log in (no surprise), when the restore is complete you'll be able to log back in.

### Scheduled Backups

Manual backups are a nice start, none of this counts until unless the full and incremental backups can be scheduled.  To do this, start with scheduling a full backup:

```
➜  pgo create schedule pickles --schedule="0 1 * * SUN" \
    --schedule-type=pgbackrest --pgbackrest-backup-type=full -n pgouser1
created schedule pickles-pgbackrest-full for cluster pickles
```

Then schedule more frequent incremental backups:

```
➜  pgo create schedule pickles --schedule="0 1 * * MON-SAT" \
    --schedule-type=pgbackrest --pgbackrest-backup-type=diff -n pgouser1
created schedule pickles-pgbackrest-diff for cluster pickles
```

To see the backup schedules run a `pgo show schedule` command:

```
➜  pgo show schedule pickles -n pgouser1
pickles-pgbackrest-diff:
    schedule: 0 1 * * MON-SAT
    schedule-type: pgbackrest
    backup-type: diff
pickles-pgbackrest-full:
    schedule: 0 1 * * SUN
    schedule-type: pgbackrest
    backup-type: full
```

### Create and Connect to a Replica

Adding a replica is almost trivial with the scale command:

```
➜  pgo scale pickles -n pgouser1
WARNING: Are you sure? (yes/no): yes
created Pgreplica pickles-ucsm
```

You can see an additional pod named `pickles-ucsm` has been created in the `pogouser1` namespace:

```
➜  kubectl get pods -n pgouser1
NAME                                              READY   STATUS      RESTARTS   AGE
pickles-749d4bb77d-hw8z7                          1/1     Running     0          17h
pickles-backrest-shared-repo-76df9cff78-cb58n     1/1     Running     0          17h
pickles-stanza-create-kbmm7                       0/1     Completed   0          17h
pickles-ucsm-595b968cdc-dvd57                     1/1     Running     0          27s
```

Looking at the services you'll see that the `pickles-replica` has been registered as a `ClusterIP`:

```
➜  kubectl get services -n pgouser1
NAME                             TYPE        CLUSTER-IP       PORT(S)
pickles                          NodePort    10.111.108.118   5432:30814/TCP,9100:31207/TCP,10000:32264/TCP,2022:32022/TCP,9187:31960/TCP
pickles-backrest-shared-repo     ClusterIP   10.108.123.32    2022/TCP
pickles-replica                  ClusterIP   10.101.52.114    5432/TCP,9100/TCP,10000/TCP,2022/TCP,9187/TCP
```

To connect to the replica remotely we'll need to download the service yaml and again switch the `pickles-replica` service from `ClusterIP` to `NodePort`, then apply the modified service yaml file:

```
➜  kubectl get service pickles-replica -n pgouser1 -o yaml > pgouser1-replica.pickles.service.yaml
➜  vim pgouser1-replica.pickles.service.yaml  # replacing ClusterIP with NodePort
➜  kubectl apply -n pgouser1 -f pgouser1-replica.pickles.service.yaml
Warning: kubectl apply should be used on resource created by either kubectl create --save-config or kubectl apply
service/pickles-replica configured
```

Taking a peek at the services again you can see the replica postgres port has been assigned port `31199`:

```
➜  kubectl get service pickles-replica -n pgouser1
NAME                             TYPE        CLUSTER-IP       PORT(S)
pickles-replica                  NodePort    10.101.52.114    5432:31199/TCP,9100:30839/TCP,10000:30365/TCP,2022:32174/TCP,9187:32043/TCP
```

Now you can connect to the replica with `psql` by using the minikube IP and `pickles-replica` exposed port.  Since it is a replica you can also see you cannot perform write operations:

```
➜  psql -h 192.168.99.100 -p 31199 -U postgres postgres
Password for user postgres:
psql (11.2, server 11.3)
Type "help" for help.

postgres=# create table hi(hello int);
ERROR:  cannot execute CREATE TABLE in a read-only transaction
```

### Other Commands

There are a bunch of other nifty tricks this operator can do:

 - Perform minor version upgrades of the cluster
 - Scale a cluster back down
 - Perform data loads
 - Policy based on labels, allowing you to perform SQL maintenance on all deployments with a matching label
 - A built-in `test` function to verify postgres is running.  This is less helpful in a `minikube` deployment because of the `NodePort` but useful elsewhere.
 - Ability to do a `pg_ctl reload` command from the `pgo` cli
 - The pgbench tool is available through `pgo benchmark` command

Please explore these and more [here](https://access.crunchydata.com/documentation/postgres-operator/4.0.0/operatorcli/pgo-overview/).

## Final Thoughts

This is the first Postgres Operator I've taken a look at.  There may be better ones which folks can chat about in the comments section below.  Whichever one you chose, make sure to test your backups and restores regularly!