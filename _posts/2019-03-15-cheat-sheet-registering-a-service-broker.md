---
layout: post
title: "Registering (Someone Else's) Service Broker in Cloud Foundry"
date: 2019-03-15
---

![map](https://raw.githubusercontent.com/cweibel/ghost_blog_pics/master/cytonn-photography-n95VMLxqM2I-unsplash.jpg)



Photo by [Cytonn Photography](https://unsplash.com/@cytonn_photography?utm_source=unsplash&utm_medium=referral&utm_content=creditCopyText) on [Unsplash](https://unsplash.com/s/photos/uri?utm_source=unsplash&utm_medium=referral&utm_content=creditCopyText)


Below is a quick summary for registering your (or someone else's) service broker into CF.  Be sure to verify the username and password, especially if someone else pushed the broker application.

## Step 1 - Push the Broker & Verify

Push the broker app so it registers a route.  To verify you have the correct url, username and password, you can curl the catalog endpoint. For example:

```
curl https://username:supersecretpassword@pickle-servicebroker.lab1.tonyandbruce/v2/catalog
```

Example output:


> {"services":[{"id":"pickle-service","name":"pickle-service","description":"Securely store and manage jars of pickles.","bindable":true,"plans":[{"id":"bc3e-56a0-11a1-19780901a22::pickle-service","name":"Gerkin","description":"The best kind of pickle","free":true}]}]}%


## Step 2 - Register the Broker

The syntax is:

```
cf create-service-broker SERVICE_BROKER USERNAME PASSWORD URL
```

For example:

```
cf create-service-broker pickle-service username supersecretpassword https://pickle-servicebroker.lab1.tonyandbruce.com
```

### Step 3 - Get Service Name

To get the name of the service, run `cf curl /v2/services`, in the output you’ll see the label you’ll need for the service name parameter below:

```
...
    {
         "metadata": {
            "guid": "715febce-473d-11e9-b210-d663bd873d93",
            "url": "/v2/services/715febce-473d-11e9-b210-d663bd873d93",
            "created_at": "2019-03-15T19:26:33Z",
            "updated_at": "2019-03-15T19:26:33Z"
         },
         "entity": {
            "label": "pickle-service",          ## <=== Looking for this value
            "provider": null,
            "url": null,
            "description": "The best kind of pickle.",
          ...
            "service_broker_name": "pickle-service",
         }
      }
```

## Step 4 - Enable the Broker

To enable the service broker for all orgs:

```
cf enable-service-access pickle-service
```

To enable for a particular org:

```
cf enable-service-access pickle-service -o my_cool_org
```

Enjoy!!