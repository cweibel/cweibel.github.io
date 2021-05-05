---
layout: post
title: "Modifying the Default PostgreSQL Helm Chart to Emit Logging"
date: 2019-08-14
---

![map](https://raw.githubusercontent.com/cweibel/ghost_blog_pics/master/aleksandar-radovanovic-mXKXJI98aTE-unsplash.jpg)



Photo by [Aleksandar Radovanovic](https://unsplash.com/@aleksandarr) on [Unsplash](https://unsplash.com/s/photos/uri?utm_source=unsplash&utm_medium=referral&utm_content=creditCopyText)

While using the [PostgreSQL Helm Chart](https://github.com/helm/charts/tree/master/stable/postgresql) I wanted to take a look at the queries which were running.  I quickly realized I needed to enable the logging to see all the DML goodness to later feed into [pgBadger](https://github.com/darold/pgbadger) to review the usage patterns of the queries.  Below are three different methods for enabling the logging of all the queries to local logs:

## Solution 1 - Custom `values.yaml` file

Copy the default `values.yaml` file from [here](https://github.com/bitnami/charts/blob/master/upstreamed/postgresql/values.yaml).

In the file uncomment `postgresqlExtendedConf` and add the following lines:

```
postgresqlExtendedConf:
  loggingCollector: on
  logDirectory: mylogs4pgbadger
  logMinDurationStatement: 0
```

A quick note: don't use `postgresqlConfiguration`, this will simply wipe the ENTIRE `postgresql.conf` file and replace it with the 3 configurations above and is not what we want.

If you are running `minikube` take a moment and update the `service:` section of the yaml

```
service:
  ## PosgresSQL service type
  type: ClusterIP
  clusterIP: None
  port: 5432
```

to

```
service:
  ## PosgresSQL service type
  type: NodePort
  port: 5432
```

Now to deploy an instance of this, from the same folder as your customized `values.yaml` file run:

```
helm install --name pg88c -f ./values.yaml stable/postgresql
```

Specify a name other than `pg88c` which makes sense to you.  From the output of the `helm create` command will be the command to retrieve the password.  Run it so you have the password available to you later on:

```
kubectl get secret --namespace default pg88c-postgresql -o \
  jsonpath="{.data.postgresql-password}" | base64 --decode
```

To view the contents of the log files, connect to the container and tail the logs:

```
kubectl exec -it pg88c-postgresql-0 -- bash
cd /bitnami/postgresql/data/mylogs4pgbadger
tail -f *.log
```

To view the contents of the configuration file which will be processed AFTER the `postgresql.conf` file, connect to the container and output the file:

```
kubectl exec -it pg88c-postgresql-0 -- bash
cd /opt/bitnami/postgresql/conf/conf.d
cat override.conf
```

with the contents of the file being:

```
log_directory=mylogs4pgbadger
log_min_duration_statement=0
logging_collector=true
```

Note that the parameters are converted from camel case to the snake case PostgreSQL needs.

## Solution 2 - Supply the Configs as set Parameters

This is the shorter version that accomplishes the same thing as `Solution 1` without having to manually edit and populate a `values.yaml` file.  Simply provide the extended PostgreSQL configurations as `set` commands when calling `helm install`:

```
helm install --name pg88g \
--set postgresqlExtendedConf.logDirectory=mylogs4pgbadger \
--set postgresqlExtendedConf.logMinDurationStatement=0 \
--set postgresqlExtendedConf.loggingCollector=true \
--set service.type=NodePort \
stable/postgresql
```

A couple notes:

 - You still have to provide the options in camel case
 - This will populate the `override.conf` file just like the first solution
 - Remove the `service.type=NodePort` if you are running somewhere other than `minikube`

 
## Solution 3 - Use a ConfigMap to for the extendedConfigMap parameter

Create a file called `override.conf` in the current working folder with these contents: 

```
log_directory=mylogs4pgbadger
log_min_duration_statement=0
logging_collector=true
```

Create a ConfigMap called `pg-log-configs` which references the `override.conf` file:

```
kubectl create configmap pg-log-configs \
  --from-file=override.conf
```

Now deploy the helm chart with the ConfigMap:

```
helm install --name pg88i \
  --set extendedConfConfigMap=pg-log-configs \
  --set service.type=NodePort \
  stable/postgresql  
```

To view the ConfigMap:

```
kubectl get configmaps pg-log-configs -o yaml
```

To view the contents of the configuration file which will be processed AFTER the `postgresql.conf` file, connect to the container and output the file:

```
kubectl exec -it pg88c-postgresql-0 -- bash
cd /opt/bitnami/postgresql/conf/conf.d
cat override.conf
```

with the contents of the file being:

```
log_directory=mylogs4pgbadger
log_min_duration_statement=0
logging_collector=true
```

## Summary

If you need a single one line `helm` command without the need to create and manage separate yaml files, solution #2 is the way to go.  This is most useful if you only need a handful of customizations.

If you have a longer list of customizations either of the other solutions are viable.  Solution #3 took a bit of trial and error with the trick being the name of the ConfigMap file must be `override.conf`, other file names did not work in my testing.

Happy Postgresifying!
 
 