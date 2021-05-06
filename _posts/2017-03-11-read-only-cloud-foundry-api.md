---
layout: post
title: "Creating a Read-Only Cloud Foundry API Server for Reporting"
date: 2017-03-11
---

![map](https://raw.githubusercontent.com/cweibel/ghost_blog_pics/master/steve-smith-zIpjYcKsqbI-unsplash.jpg)



Photo by [Steve Smith](https://unsplash.com/@varrak?utm_source=unsplash&utm_medium=referral&utm_content=creditCopyText) on [Unsplash](https://unsplash.com/s/photos/uri?utm_source=unsplash&utm_medium=referral&utm_content=creditCopyText)


## Backstory

Cloud Foundry is a powerful PaaS allowing developers to easily deploy and scale their applications. Keeping an eye on the platform falls to the operators who can leverage various tools such as the CF Firehose and Prometheus to consume metrics and create dashboards to validate the health of the system.

One of the dashboards we have found useful in Prometheus is the `CF: Summary` dashboard which leverages data from an exporter called `cf_exporter` which connects to a Cloud Foundry API server and collects information about the number of:

 - Organizations
 - Services
 - Spaces
 - Stacks
 - Applications
 - Routes
 - Service Instances
 - Security Groups

 
With a few thousand applications running the `cf_exporter` takes over a minute to run, consuming resources on a Cloud Controller API server which is attached to the Cloud Controller database (MySQL or PostgreSQL) and likewise consuming database resources while the API commands are being processed.

Why is this potentially bad? The monitoring system to collect `cf_exporter` is competing for resources with the normal Cloud Foundry operations on the Cloud Controller API servers and the database server. This can cause delayed responses while the exporter is running.

## Solution

A typical deployment of Cloud Foundry has a load balancer, 1 or more routers, 1 or more Cloud Controller API servers, 1 master Cloud Controller database server (CCDB). When an API request is sent to Cloud Foundry it traverses the load balancer which connects to one of the GoRouters. The API servers register their endpoints with the GoRouters at regular intervals with a route_registrar job. Each API server also creates read/write database connections to the CCDB.

```
(Load Balancer) <---> (CF GoRouter) <---> (CC API) <---> (CCDB) 
```

With a few modifications you can mitigate the impact of monitoring the `cf_exporter` has on normal operations of Cloud Foundry. The key is to create a separate group of API servers for reporting and leverage a read-only db replica.

```
(Load Balancer) <--> (CF GoRouter) <--> (CC API) <--------> (CCDB) 
                            \/                                ||
                      (Reporting CC API) <---> (CCDB Read-only replica)
```

## Creating a Reporting CC API Server

Assuming that you can create a read-only replica, there are only a few properties you need to setup in a BOSH manifest for Cloud Foundry to create a separate CC API server you can use for reporting. What is important to understand is you need to register a different url from your original API servers and you can only perform API operations which do not write to the database server. You'll also need to disable the Reporting CC API server from running the database migrations at startup, the original API servers which have access to the write master will continue to perform this role.

Here is a portion of a Cloud Foundry deployment manifest which deploys a Reporting CC API server:

```
- instances: 1
  name: api_reporting_z1
  properties:
    cc:
      base_url: https://api-reporting.system.mycf.io
      consul:
        agent:
          services:
            cloud_controller_ng: {}
      run_prestart_migrations: false   #disables db migration at startup
      srv_api_uri: https://api-reporting.system.mycf.io
    ccdb:
      address: ccdb-readreplica-mycf-pr.fluffyrds.us-east-1.rds.amazonaws.com
    route_registrar:
      routes:
      - name: api
        port: 9022
        registration_interval: 20s
        tags:
          component: CloudController
        uris:
        - api-reporting.system.mycf.io
  templates:
  - name: consul_agent
    release: cf
  - name: cloud_controller_ng
    release: cf
  - name: metron_agent
    release: cf
  - name: route_registrar
    release: cf
```

Once the above server is deployed inside of a Cloud Foundry deployment you can curl the Reporting CC API server:

```
curl -i -X GET -H "Authorization: bearer $BEARER_TOKEN"  https://api-reporting.system.mycf.io/v2/spaces
```

For a full example of how to get the bearer token click here: 


You can now also use this new Reporting CC API endpoint when configuring the Prometheus cf_exporter exporter: [ https://github.com/cloudfoundry-community/prometheus-boshrelease/blob/master/jobs/cf_exporter/spec#L13]( https://github.com/cloudfoundry-community/prometheus-boshrelease/blob/master/jobs/cf_exporter/spec#L13)