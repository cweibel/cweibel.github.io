---
layout: post
title: "Setting up Cloud Foundry App Autoscaler user Genesis"
date: 2019-11-25
---

![map](https://raw.githubusercontent.com/cweibel/ghost_blog_pics/master/victor-munoz-nDJeJye6y_A-unsplash.jpg)

Photo by [Victor Muñoz](https://unsplash.com/@victormuunoz) on [Unsplash](https://unsplash.com/s/photos/uri?utm_source=unsplash&utm_medium=referral&utm_content=creditCopyText)


[App Autoscaler](https://github.com/cloudfoundry/app-autoscaler) is an add-on to Cloud Foundry to automatically scale the number of application instances based on CPU, memory, throughput, response time, and several other metrics.  You can even add your own custom metrics as of v3.0.0.  You decide which metrics you want to scale your app up and down by in a policy and then apply the policy to your application.

Adding the Autoscaler feature to the Cloud Foundry Genesis kit is pretty simple.  Start by adding the feature flag, register the service broker, create a service instance and bind it and a policy to your app.  Below we'll run through the steps of adding and configuring the various components:

## Add the Feature

In the env file add the feature flag:

```
kit:
  name:    cf
  version: 1.6.0
  features:
    - local-db
    ...
    - autoscaler   # <<< Add this
```

Under `params` reuse the `cf-core` network defined in the cloud-config:

```
params:
  ...
  autoscaler_network: cf-core
```

Perform a `genesis deploy NAME_OF_ENV` and autoscaler will be added to your CF footprint.

## Configure service broker

Retrieve the credentials for the service broker for autoscaler from safe:

```
safe get secret/snw/someone/lab2/cf/autoscaler/servicebroker_account
```
This will retrieve the `username` and `password`.  The url will be `autoscalerservicebroker.SYSTEM_URL` and with these 3 pieces of information you can validate that the broker is alive:

```
curl http://username:mypassword@autoscalerservicebroker.system.schwifty.lab.starkandwayne.com/v2/catalog
```

Which should output something similar to:

```
{"services":[{"bindable":true,"description":"Automatically increase or decrease the number of 
application instances based on a policy you define.","id":"autoscaler-
guid","name":"autoscaler","plans":[{"description":"This is the example service plan for the Auto-
Scaling service.","id":"autoscaler-example-plan-id","name":"autoscaler-example-plan","schemas":

...
Example truncated for readability...
```

Now that we know the broker is alive we can register it with CF:

```
cf create-service-broker autoscaler user mypassword \
  http://autoscalerservicebroker.system.schwifty.lab.starkandwayne.com
```

Finally enable the broker for all orgs:

```
cf enable-service-access autoscaler
```

## Apply a Policy

If you have an app already deployed, you can create a Autoscaler Policy and apply it to the applications.

Start by creating an instance of the service:

```
cf create-service autoscaler autoscaler-example-plan myas
```

Create a policy file on the os named `as_policy1.json` with the contents:

```
{
    "instance_min_count": 1,
    "instance_max_count": 2,
    "scaling_rules": [
        {
            "metric_type": "cpu",
            "breach_duration_secs": 60,
            "threshold": 10,
            "operator": "<=",
            "cool_down_secs": 60,
            "adjustment": "-1"
        },
        {
            "metric_type": "cpu",
            "breach_duration_secs": 60,
            "threshold": 10,
            "operator": "<",
            "cool_down_secs": 60,
            "adjustment": "+1"
        }
    ]
}
```

Now bind the service instance to the app and provide the policy to be used:

```
cf bind-service cf-env myas -c as_policy1.json
```

## Install the Autoscaler CLI

Installing the CLI is pretty simple, other methods are described [here](https://github.com/cloudfoundry/app-autoscaler-cli-plugin#install-plugin) in case the command below does not work for you:

```
cf install-plugin -r CF-Community app-autoscaler-plugin
```

## Show the current policy for an app

If you don't know what the current policy is for an application you can retrieve it with the `cf autoscaling-policy` command:

```
➜  cf-env git:(master) ✗ cf autoscaling-policy cf-env
Retrieving policy for app cf-env...
{
	"instance_min_count": 1,
	"instance_max_count": 2,
	"scaling_rules": [
		{
			"metric_type": "cpu",
			"breach_duration_secs": 60,
			"threshold": 10,
			"operator": "<=",
			"cool_down_secs": 60,
			"adjustment": "+1"
		},
		{
			"metric_type": "cpu",
			"breach_duration_secs": 60,
			"threshold": 10,
			"operator": "<",
			"cool_down_secs": 60,
			"adjustment": "-1"
		}
	]
}
```

## Set the url for the CLI

Note that the naming scheme is typically `https://autoscaler + CF_SYSTEM_URL`

```
cf autoscaling-api http://autoscaler.system.schwifty.lab.starkandwayne.com
```

## View the Metrics

To see the metrics Autoscaler is using for your application you can run the `cf autoscaling-metrics` command.  Note that you will only see metrics AFTER your application is bound with a policy to a service instance of the Autoscaler service.  You will also only see metrics referenced in `metric_type` of the policy for an application being enforced.

> cf autoscaling-metrics cf-env cpu

```
➜  cf-env git:(master) ✗ cf autoscaling-metrics cf-env cpu
Retrieving aggregated cpu metrics for app cf-env...
Metrics Name     	Value     	Timestamp
cpu              	0%        	2019-11-21T10:21:56-05:00
cpu              	0%        	2019-11-21T10:21:16-05:00
cpu              	0%        	2019-11-21T10:20:36-05:00
```

Below is an example of the history command.  The number of application instances was manually scaled to 4 using the `cf scale -i 4 cf-env` command.  Once the policy was enforced it was scaled to 2 instances since the `"instance_max_count": 2`, was defined.  Later it was scaled to a single instance because the CPU was below the threshold of 10% for 60 seconds.

```
➜  cf-env git:(master) ✗ cf autoscaling-history cf-env
Retrieving scaling event history for app cf-env...
Scaling Type     	Status        	Instance Changes     	Time                          	Action                                                	Error
dynamic          	succeeded     	2->1                 	2019-11-21T10:16:36-05:00     	-1 instance(s) because cpu <= 10% for 60 seconds
dynamic          	succeeded     	4->2                 	2019-11-21T10:15:16-05:00     	-2 instance(s) because limited by max instances 2
```

## Final Thoughts

One of the primary reasons to use Cloud Foundry is the ease of which you can scale your application.  The developer can run a single CLI command to scale up or scale down based on ... does anyone know why developers scale their apps?

With the presence of autoscaler you can take the guess-work of manually scaling away and let it happen automatically.  Especially if you are paying per application instance!





