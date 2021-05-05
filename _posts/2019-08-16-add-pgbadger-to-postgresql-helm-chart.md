---
layout: post
title: "Adding pgBadger to the PostgreSQL Helm Chart"
date: 2019-08-16
---

![map](https://raw.githubusercontent.com/cweibel/ghost_blog_pics/master/vincent-van-zalinge-xX4VG7mHuSE-unsplash.jpg)


Photo by [Vincent van Zalinge](https://unsplash.com/@vincentvanzalinge?utm_source=unsplash&utm_medium=referral&utm_content=creditCopyText) on [Unsplash](https://unsplash.com/s/photos/uri?utm_source=unsplash&utm_medium=referral&utm_content=creditCopyText)


[pgBadger](https://github.com/darold/pgbadger) is a helpful tool which will generate reports and diagrams about the type and pattern of sql queries over time that are submitted to PostgreSQL.  The tool works by reading the PostgreSQL logs (not transaction logs) which are not typically enabled by default in the helm chart.

Below are a few methods for leveraging pgBadger on a PostgreSQL instance deployed via the default PostgreSQL Helm Chart.  Each approach has pros and cons, pick the one which is appropriate for your situation!

## Prerequisites

There are a few configurations which must be performed to allow pgBadger to read logs:

 - Configure PostgreSQL to emit logs
 - Know the `log_destination` used
 - Know the `log_line_prefix` used

### Configure PostgreSQL

To configure PostgreSQL to emit logs see [this](https://cweibel.github.io/blog/2019/08/14/modify-postgresql-helm-chart-to-emit-logging) blog post. The author is fantastic.  Basically you'll need to enable logging and select which queries are emitted to the logs.  The example in the blog post assumes you want ALL queries since any query taking 0 or longer seconds is logged.

### Retrieve Log Destination
There are a number of ways to define the type of logs to emit.  You will need this value as a parameter to the `pgbadger` script. To retrieve what the current value used is query the database instance:

```
postgres=# select name, setting from pg_settings where name='log_destination';
      name       | setting 
-----------------+---------
 log_destination | stderr  
```

The typical values are the following with the options defined [here](https://www.postgresql.org/docs/current/runtime-config-logging.html):

 - stderr
 - cvslog
 - syslog

### Retrieve Log Line Prefix

You will need this value as a parameter to the `pgbadger` script. To retrieve the current value of `log_line_prefix` again query the database:

```
postgres=# select name, setting from pg_settings where name='log_line_prefix';
      name       | setting  
-----------------+----------
 log_line_prefix | %m [%p]  
```

More information on logging prefixes for PostgreSQL: [https://severalnines.com/blog/postgresql-log-analysis-pgbadger](https://severalnines.com/blog/postgresql-log-analysis-pgbadger), there are some better settings which you may want to leverage over the default `%m [%p]`.

## Method 1: Hack an existing Helm Chart deployed PostgreSQL statefulset

This method is not... probably the way you want to make the changes in production but does work.  Basically, deploy the PostgreSQL helm chart, download the statefulset spec, add the pgBadger container and apply the change.  It seems to survive `helm upgrade` commands but I wouldn't rely on that behavior.

Assuming that the deployment was performed similar to:

```
helm install --name pg88g \
--set postgresqlExtendedConf.logDirectory=mylogs4pgbadger \
--set postgresqlExtendedConf.logMinDurationStatement=0 \
--set postgresqlExtendedConf.loggingCollector=true \
--set service.type=NodePort \
stable/postgresql
```

Download the existing `statefulset` :

```
kubectl get statefulset pg88g-postgresql \
-o yaml > pg88g-postgresql.statefulset.yaml
```

Modify it to include the following:

```
...
      containers:
      - name: pgbadger
        image: dalibo/pgbadger
        command: 
        - sh
        - -c
        - |
          pgbadger /data/data/mylogs4pgbadger/*.log -f stderr \
            --prefix='%m [%p]' --jobs 4 -o \
            /data/pgbadger$(date +"%Y_%m_%d_at_%H_%M").html
          sleep 10000
        volumeMounts:
        - mountPath: /data
          name: data
...
```

Note that:

 - `--prefix='%m [%p]'` is the value retrieved in the `log_line_prefix` Prerequisites section

 - `-f stderr` comes from the `log_line_prefix` in the Prerequisites section

 - the `sleep 10000` is a terrible solution which give you enough time to exec onto the container to see the files created and allow for debugging issues.

 - the `volumeMounts` section is copied from the volume used by the original postgresql container

   ```
     volumeMounts:
     - mountPath: /data
       name: data
   ```

 - both of the containers (postgres and pgbadger) will be sharing the same volume but mounted to different places within each container. The `mylogs4pgbadger` was specified as the folder logged to when the helm chart was deployed (`--set postgresqlExtendedConf.logDirectory=mylogs4pgbadger \`).

 - the `/data/pgbadger$(date +"%Y_%m_%d_at_%H_%M").html` will generate the files in a format similar to `pgbadger2019_08_15_at_18_47.html` in the `/data` directory.

Apply the changes to the `statefulset` we've manually updated:

```
kubectl apply -f pg88g-postgresql.statefulset.yaml
```

Once deployed you'll be able to log into the new `pgbadger` container and look at the file created:

```
kubectl exec pg88g-postgresql-0 --container=pgbadger -it -- sh
cd /data
ls -alh *.html
```

If you are connected to the container you can be manually run the `pgbadger` command again:

```
pgbadger /data/data/mylogs4pgbadger/*.log \
  -f stderr --prefix='%m [%p]' --jobs 4 \
  -o /data/pgbadger$(date +"%Y_%m_%d_at_%H_%M").html
```

## Method 2: Deploy pgBadger as a separate Job

Heavily borrowed from [https://stackoverflow.com/questions/41192053/cron-jobs-in-kubernetes-connect-to-existing-pod-execute-script](https://stackoverflow.com/questions/41192053/cron-jobs-in-kubernetes-connect-to-existing-pod-execute-script) we can deploy a separate Job which will connect to the same volume as the PostgreSQL helm chart instance, run the `pgbadger` command and then exit.

Start by deploying another chart:

```
helm install --name pg88h \
--set postgresqlExtendedConf.logDirectory=mylogs4pgbadger \
--set postgresqlExtendedConf.logMinDurationStatement=0 \
--set postgresqlExtendedConf.loggingCollector=true \
--set service.type=NodePort \
stable/postgresql
```

Create a file named `pgbadger.job.yaml` and populate it with:

```
apiVersion: batch/v1
kind: Job
metadata:
  name: pg88h-postgresql-pgbadger
spec:
  ttlSecondsAfterFinished: 300
  template:
    metadata:
      name: pg88h-postgresql-pgbadger
      labels:
        app: postgresql
        chart: postgresql-6.2.1
        heritage: Tiller
        release: pg88h
        role: master
    spec:
      containers:
      - name: pgbadger
        image: dalibo/pgbadger
        command: 
        - sh
        - -c
        - |
          pgbadger /data/data/mylogs4pgbadger/*.log -f stderr --prefix='%m [%p]' --jobs 4 -o /data/pgbadger$(date +"%Y_%m_%d_at_%H_%M").html

        volumeMounts:
        - mountPath: /data
          name: data
      volumes:
      - name:  data
        persistentVolumeClaim:
          claimName: data-pg88h-postgresql-0
      restartPolicy: OnFailure
```

Then apply:

```
kubectl apply -f pgbadger.job.yaml
```
If the job is successful you will see something similar to:

```
kubectl get pods

NAME                              READY   STATUS      RESTARTS   AGE
pg88h-postgresql-0                1/1     Running     0          41m
pg88h-postgresql-pgbadger-cxdp2   0/1     Completed   0          8m37s
```

If you connect back to either the postgres or pgbadger containers in their respective pods you'll see the file written out from the pgBadger run:

```
$ kubectl exec pg88h-postgresql-0 -it -- sh
$ cd /bitnami/postgresql/data
$ ls -alh
total 1.6M
drwxrwxrwx  4 root root 4.0K Aug 16 17:18 .
drwxr-xr-x  3 root root 4.0K Jul  1 11:10 ..
drwxr-xr-x  3 root root 4.0K Aug 16 16:45 conf
drwx------ 20 1001 1001 4.0K Aug 16 16:45 data
-rw-r--r--  1 root root 772K Aug 16 17:18 pgbadger2019_08_16_at_17_18.html
```

A couple notes about the yaml file:

 - The contents of `containers:` is the same as Method #1, refer to that section for the breakdown of the `command:` portion of the spec.
 - Similar to Method #1, both containers (postgresql and pgbadger) have access to the persistent volume but in this solution the containers are in different pods
 - The `ttlSecondsAfterFinished: 300` allows the scheduler to remove the pod after 5 minutes giving you some time to debug on the container if something has gone wrong.


Downfalls:

 - If the PVC is defined as `ReadWriteOnce` then the job pod has to be located on the same worker as the postgres pod.  If you are on `minikube`, this isn't an issue but in other Kubernetes deployments with multiple workers it can be a problem.
 - If `ReadWriteMany` is used so you aren't forced to locate the pgbadger job on the same instance, not all IaaS providers support this, the list is documented here: [https://kubernetes.io/docs/concepts/storage/persistent-volumes/#access-modes](https://kubernetes.io/docs/concepts/storage/persistent-volumes/#access-modes).

## Method 3: Cron Job

This is almost the same as Method #2 except we'll use a `kind: CronJob` instead of `kind: Job` resource so we can schedule the job to repeat.

Start by deploying another chart:
 
```
 helm install --name pg88i \
--set postgresqlExtendedConf.logDirectory=mylogs4pgbadger \
--set postgresqlExtendedConf.logMinDurationStatement=0 \
--set postgresqlExtendedConf.loggingCollector=true \
--set service.type=NodePort \
stable/postgresql
```

Create a file named `pgbadger.cronjob.yaml` and populate it with:

```
apiVersion: batch/v1beta1
kind: CronJob
metadata:
  name: pg88i-postgresql-pgbadger
spec:
  schedule: "*/1 * * * *"
  jobTemplate:
    spec:
      template:
        metadata:
          name: pg88i-postgresql-pgbadger
          labels:
            app: postgresql
            chart: postgresql-6.2.1
            heritage: Tiller
            release: pg88i
            role: master
        spec:
          containers:
          - name: pgbadger
            image: dalibo/pgbadger
            command: 
            - sh
            - -c
            - |
              pgbadger /data/data/mylogs4pgbadger/*.log -f stderr --prefix='%m [%p]' --jobs 4 -o /data/pgbadger$(date +"%Y_%m_%d_at_%H_%M").html

            volumeMounts:
            - mountPath: /data
              name: data
          volumes:
          - name:  data
            persistentVolumeClaim:
              claimName: data-pg88i-postgresql-0
          restartPolicy: OnFailure
```

Then apply:

```
kubectl apply -f pgbadger.cronjob.yaml
```

If the job is successful you will see something similar to:

```
kubectl get pods

NAME                                         READY   STATUS      RESTARTS   AGE
pg88i-postgresql-0                           1/1     Running     0          10m
pg88i-postgresql-pgbadger-1565977800-ppfn8   0/1     Completed   0          2m50s
pg88i-postgresql-pgbadger-1565977860-c86mz   0/1     Completed   0          110s
pg88i-postgresql-pgbadger-1565977920-fqmsq   0/1     Completed   0          50s
```

The last 3 successful and last 1 unsuccessful containers will be kept and older ones will be deleted automatically.

To see all the cronjobs currently available to your namespace:

```
kubectl get cronjobs

NAME                        SCHEDULE      SUSPEND   ACTIVE   LAST SCHEDULE   AGE
pg88i-postgresql-pgbadger   */1 * * * *   False     0        25s             11m
```

Notes about this method:

 - This method has all the downfalls of Method #2 with respect to `ReadWriteOnce` across multiple nodes.
 - The schedule of `schedule: "*/1 * * * *"` means it will run every minute.  This is good to show for a demo, but you'll likely want a larger time range. [https://kubernetes.io/docs/tasks/job/automated-tasks-with-cron-jobs/](https://kubernetes.io/docs/tasks/job/automated-tasks-with-cron-jobs/) has more details.

 
## Method 4: Patches

This one is also nifty and functions very much like Method #1 but without manually downloading and editing the statefulset file.  The idea with this one is to "patch" the statefulset and merge an additional container definition to the pod following [https://kubernetes.io/docs/tasks/run-application/update-api-object-kubectl-patch/](https://kubernetes.io/docs/tasks/run-application/update-api-object-kubectl-patch/).

In this example, we'll deploy the helm chart, then merge with `kubectl patch` command which will add the pgbadger container spec to the deployed statefulset.

Start by deploying another chart:

```
helm install --name pg88j \
--set postgresqlExtendedConf.logDirectory=mylogs4pgbadger \
--set postgresqlExtendedConf.logMinDurationStatement=0 \
--set postgresqlExtendedConf.loggingCollector=true \
--set service.type=NodePort \
stable/postgresql
```

If you take a look at the pods there will be a single container with PostgreSQL running happily:

```
kubectl get pods

NAME                 READY   STATUS    RESTARTS   AGE
pg88j-postgresql-0   1/1     Running   0          151m
```

Create a file named `pgbadger.patch.yaml` and populate it with:

```
spec:
  template:
    spec:
      containers:
      - name: pgbadger
        image: dalibo/pgbadger
        command: 
        - sh
        - -c
        - |
          pgbadger /data/data/mylogs4pgbadger/*.log -f stderr --prefix='%m [%p]' --jobs 4 -o /data/pgbadger$(date +"%Y_%m_%d_at_%H_%M").html
          sleep 10000
        
        volumeMounts:
        - mountPath: /data
          name: data
```

Then apply the patch to the statefulset:

```
kubectl patch statefulset pg88j-postgresql --patch "$(cat pgbadger.patch.yaml)"
```

If the job is successful you will see something similar to the following with now two containers in the pod:

```
kubectl get pods

NAME                 READY   STATUS    RESTARTS   AGE
pg88j-postgresql-0   2/2     Running   0          5m35s
```

What is nice with this solution is you can leave the original helm chart definition alone and "sidecar" on an additional container inside the pod after the initial deployment.  Since there is nothing specific in the patch to a particular deployment the patch can be used over and over. This solution, however, doesn't allow us to repeatedly schedule pgBadger to run.

## Conclusions

Above are a few possible solutions to leveraging pgBadger with the existing PostgreSQL Helm Chart without trying to make changes to the chart.  None of them are perfect for my situation but hopefully one might be to you!