---
layout: post
title: "Cloud Foundry TCP Routing - The Rest of the Story"
date: 2022-05-12
---

## The story is real, but what is the part of TCP Routing you need to know about still?

![pic](https://raw.githubusercontent.com/cweibel/ghost_blog_pics/master/gary-bendig-6GMq7AGxNbE-unsplash.jpg)

Photo by [Gary Bendig](https://unsplash.com/@kris_ricepees?utm_source=unsplash&utm_medium=referral&utm_content=creditCopyText) on [Unsplash](https://unsplash.com)


The [documentation to configure Cloud Foundry for TCP Routing](https://docs.cloudfoundry.org/adminguide/enabling-tcp-routing.html) is a great reference for getting started on your journey to implementation but there a few missing pieces which I think I can help fill in if you are deploying on AWS. 



## Assumptions

 - I need an ELB to listen on tcp ports 40000-50000 and forward the traffic to the tcp-routers.  The default range of 1024-1033 is fine for some folks, others, not so much.
 - I want to use a `vm_extension` to add `tcp-router`'s as they are recreated without manual intervention.
 - I have an AWS account with an ELB listeners quota of 100 and I want to use all 100 ips
 - I want to use Terraform to create the ELB and any necessary supporting resources.

 
#### A quick side bar: why an ELB instead of a NLB?

I'm glad you asked.  My goal is to have as many tcp ports as possible with a single load balancer.  As of this writing, NLB's have a default quota of 50 target groups, each target group can manage a single port.  A classic ELB has a default quota of 100 listeners.  100 > 50, therefore the ELB wins!

The link to the soft quota limit for ELB listeners is at [https://docs.aws.amazon.com/servicequotas/latest/userguide/request-quota-increase.html](https://docs.aws.amazon.com/servicequotas/latest/userguide/request-quota-increase.html)




## Steps to Implement

1. Create the ELB with Terraform 
2. Use the ELB name to modify the Cloud Config
3. Create the ops file
4. Set router groups with `cf curl`
5. Use `cf` cli to create shared domain
6. Test with an app

### Step 1: Create the ELB with Terraform

This is one of the places where the documentation isn't 100% helpful, however the good people who have been maintaining BBL help us out.  In particular this
chunk of Terraform is a great place to start: [https://github.com/cloudfoundry/bosh-bootloader/blob/main/terraform/aws/templates/cf_lb.tf#L244-L1041](https://github.com/cloudfoundry/bosh-bootloader/blob/main/terraform/aws/templates/cf_lb.tf#L244-L1041)

To support a different range of ips requires a few easy changes.  Start by replacing the `ingress` block in the two security group definitions:

```
  ingress {
    security_groups = ["${aws_security_group.cf_tcp_lb_security_group.id}"]
    protocol        = "tcp"
    from_port       = 1024
    to_port         = 1123
  }
```


with

```
  ingress {
    security_groups = ["${aws_security_group.cf_tcp_lb_security_group.id}"]
    protocol        = "tcp"
    from_port       = 40000
    to_port         = 40099
  }
```

You'll also need to replace the block of `listeners` defined in the resource `aws_elb.cf_tcp_lb`:

```
  listener {
    instance_port     = 1024
    instance_protocol = "tcp"
    lb_port           = 1024
    lb_protocol       = "tcp"
  }
  ...
  98 bottles of listeners on the wall, 98 bottles of listeners...
  ...
  listener {
    instance_port     = 1123
    instance_protocol = "tcp"
    lb_port           = 1123
    lb_protocol       = "tcp"
  }
```

With something like:


```
  listener {
    instance_port     = 40000
    instance_protocol = "tcp"
    lb_port           = 40000
    lb_protocol       = "tcp"
  }
  ...
  98 bottles of listeners on the wall, 98 bottles of listeners...
  ...
  listener {
    instance_port     = 40099
    instance_protocol = "tcp"
    lb_port           = 40099
    lb_protocol       = "tcp"
  }
```

Don't feel like copy/paste/modifying the same 6 lines of code 99 times?  Here's a quick `python` script that you can run, then copy/paste the results into the Terraform file:

```
start_port = int(input("Enter starting port (40000):") or "40000")
end_port   = int(input("Enter ending port (40099):") or 40099) + 1

for x in range(start_port, end_port):
    print("  listener {")
    print('    instance_port     =', x)
    print('    instance_protocol = "tcp"')
    print("    lb_port           =", x)
    print('    lb_protocol       = "tcp"')
    print("  }")
```

Cute, right?  Anyway, I called this `listeners.py` which I can run with `python3 listeners.py`, copy in the output and enjoy.


If you are going to just use the section of BBL code highlighted with the few changes above you'll need to provide a couple more values for your terraform:

 - `subnets` - No guidance here other than to pick two subnets in your VPC
 - `var.env_id` - When in doubt, `variable "env_id"  { default = "starkandwayne"}`
 - `short_env_id` - When in doubt, `variable "short_env_id"  { default = "sw"}`.  Shameless plug, I know.


After your terraform run is complete, you'll see output like:


```
Outputs:

cf_tcp_lb_internal_security_group = sg-0f9b6a5c6d63f1375
cf_tcp_lb_name = sw-cf-tcp-lb
cf_tcp_lb_security_group = sg-0e5cd4f4f262a8d87
cf_tcp_lb_url = sw-cf-tcp-lb-1943122948.us-west-2.elb.amazonaws.com
```


Register the ELB CNAME with your DNS provider to point to `tcp.APP_DOMAIN`, in my case:

 - My apps are in `*.apps.codex.starkandwayne.com`
 - Therefore I'm using `tcp.apps.codex.starkandwayne.com` as my TCP url I need to register with DNS
 - So `tcp.apps.codex.starkandwayne.com` has a CNAME record added for `sw-cf-tcp-lb-1943122948.us-west-2.elb.amazonaws.com`



### Step 2 - Configure Cloud Config

Add to cloud config:

```
vm_extensions:
  - name: cf-tcp-router-network-properties
    cloud_properties:
      elbs:
        - sw-cf-tcp-lb  # Your name will be in the terraform output as `cf_tcp_lb_name`
```

A quick update to the bosh director:

```
$ bosh -e dev update-config --type cloud --name dev dev.yml
Using environment 'https://10.4.16.4:25555' as user 'admin'

  vm_extensions:
  - name: cf-tcp-router-network-properties
+   cloud_properties:
+     elbs:
+     - sw-cf-tcp-lb

Continue? [yN]:
```

If you take a peek at `cf-deployment` you'll see that the `tcp-router` is looking for a `vm_extension` called `cf-tcp-router-network-properties` here: [https://github.com/cloudfoundry/cf-deployment/blob/v20.2.0/cf-deployment.yml#L1433-L1434](https://github.com/cloudfoundry/cf-deployment/blob/v20.2.0/cf-deployment.yml#L1433-L1434) so once you configure the cloud config, `cf-deployment` is already configured to use the extension.  What this means is whenever a `tcp-router` instance is created, BOSH will automatically add it back to the ELB once it passes the health check.

### Step 3 - Create the ops file

Since I need a custom port range, some of the properties in `cf-deployment.yml` need to be changed.

An example ops file to change ports for the routing release:

```
- path: /instance_groups/name=api/jobs/name=routing-api/properties/routing_api/router_groups/name=default-tcp?
  type: replace
  value:
    name: default-tcp
    reservable_ports: 40000-40099
    type: tcp

```

When you include this new ops file in your deployment you'll see the change with:

```
Task 4856 done
  instance_groups:
  - name: api
    jobs:
    - name: routing-api
      properties:
        routing_api:
          router_groups:
          - name: default-tcp
-           reservable_ports: 1024-1033
+           reservable_ports: 40000-40099
```

### Step 4 - Set router groups via `cf curl`

Post deployment however CAPI still has the old ports:

```
$ cf curl /routing/v1/router_groups
[
   {
      "guid": "abe622af-2246-43a2-73f8-79bcb8e0cbb4",
      "name": "default-tcp",
      "type": "tcp",
      "reservable_ports": "1024-1033"
   }
]
```

To configure the Cloud Controller with the range of ips to use:

```
$ cf curl -X PUT -d '{"reservable_ports":"40000-40099"}' /routing/v1/router_groups/abe622af-2246-43a2-73f8-79bcb8e0cbb4
```


### Step 5 - Create shared domain

The DNS is configured for `tcp.apps.codex.starkandwayne.com` and the name of the router group from the ops file is `default-tcp`.  Using the `cf` cli Cloud Foundry can then be configured to map these two togther into a shared domain:

```
cf create-shared-domain tcp.apps.codex.starkandwayne.com --router-group default-tcp
```

If you run the `cf domains` command you'll see the new tcp domain added with `type = tcp`

```
$ cf domains
Getting domains in org system as admin...
name                                status   type   details
apps.codex.starkandwayne.com        shared          
tcp.apps.codex.starkandwayne.com    shared   tcp    
system.codex.starkandwayne.com      owned    
```


### Step 6 - Push an App

To create an app using tcp, there are a few options:

1. With the `cf` cli v6: Push the app with the domain specified and a random port

   ```
   cf push myapp -d tcp.apps.codex.starkandwayne.com --random-route
   ```

2. Create a route for a space, push an app, then map the route to the space. 

   ```
   $ cf create-route tcp.apps.codex.starkandwayne.com --port 40001
   $ cf push myapp --no-route                    # see next section for example app
   $ cf map-route myapp tcp.apps.codex.starkandwayne.com --port 40001
   ```
   
3. Create an app manifest which contains `routes:`, then specify the app manifest in the cf push (`cf push -f manifest.yml`) with the contents of `manifest.yml` being:

   ```
   applications:
   - name: cf-env
     memory: 256M
     routes:
     - route: tcp.apps.codex.starkandwayne.com
   ```
   
In the previous examples, swap `--port` with `--random-route` for the app push to pick any available port instead of a bespoke one.  This will help developers from having to guess which ports are still available. 


#### Testing an App

Once the application is pushed, for instance with `cf push myapp -d tcp.apps.codex.starkandwayne.com --random-route` which uses [`cf-env`](https://github.com/cloudfoundry-community/cf-env.git), you can use `curl` to test the access:

```
$ curl http://tcp.apps.codex.starkandwayne.com:40001

<html><body style="margin:0px auto; width:80%; font-family:monospace"><head><title>Cloud Foundry Environment</title><meta name="viewport" content="width=device-width"></head><h2>Cloud Foundry Environment</h2><div><table><tr><td><strong>BUNDLER_ORIG_BUNDLER_VERSION</strong></td><td>BUNDLER_ENVIRONMENT_PRESERVER_INTENTIONALLY_NIL</tr><tr><td><strong>BUNDLER_ORIG_BUNDLE_BIN_PATH</strong></td><td>BUNDLER_ENVIRONMENT_PRESERVER_INTENTIONALLY_NIL</tr><tr><td><strong>BUNDLER_ORIG_BUNDLE_GEMFILE</strong></td><td>/home/vcap/app/Gemfile</tr><tr><td><strong>BUNDLER_ORIG_GEM_HOME</strong></td><td>/home/vcap/deps/0/gem_home</tr><tr><td><strong>BUNDLER_ORIG_GEM_PATH</strong></td><td>/home/vcap/deps/0/vendor_bundle/ruby/2.7.0:/home/vcap/deps/0/gem_home:/home/vcap/deps/0/bundler</tr><tr><td><strong>BUNDLER_ORIG_MANPATH</strong></td><td>BUNDLER_ENVIRONMENT_PRESERVER_INTENTIONALLY_NIL</tr><tr><td><strong>BUNDLER_ORIG_PATH</strong></td><td>/home/vcap/deps/0/bin:/usr/local/bin:/usr/bin:/bin</tr><tr><td><strong>BUNDLER_ORIG_RB_USER_INSTALL</strong></td><td>BUNDLER_ENVIRONMENT_PRESERVER_INTENTIONALLY_NIL</tr><tr><td><strong>BUNDLER_ORIG_RUBYLIB</strong></td><td>BUNDLER_ENVIRONMENT_PRESERVER_INTENTIONALLY_NIL</tr><tr><td><strong>BUNDLER_ORIG_RUBYOPT</strong></td><td>BUNDLER_ENVIRONMENT_PRESERVER_INTENTIONALLY_NIL</tr><tr><td><strong>BUNDLER_VERSION</strong></td><td>2.2.28</tr><tr><td><strong>BUNDLE_BIN</strong></td><td>/home/vcap/deps/0/binstubs</tr><tr><td><strong>BUNDLE_BIN_PATH</strong></td><td>/home/vcap/deps/0/bundler/gems/bundler-2.2.28/exe/bundle</tr><tr><td><strong>BUNDLE_GEMFILE</strong></td><td>/home/vcap/app/Gemfile</tr><tr><td><strong>BUNDLE_PATH</strong></td><td>/home/vcap/deps/0/vendor_bundle</tr><tr><td><strong>CF_INSTANCE_ADDR</strong></td><td>10.4.23.17:61020</tr><tr><td><strong>CF_INSTANCE_CERT</strong></td><td>/etc/cf-instance-credentials/instance.crt</tr><tr><td><strong>CF_INSTANCE_GUID</strong></td><td>32064364-6709-44b9-4a91-a1f3</tr><tr><td><strong>CF_INSTANCE_INDEX</strong></td><td><pre>0</pre></tr><tr><td><strong>CF_INSTANCE_INTERNAL_IP</strong></td><td>10.255.103.15</tr><tr><td><strong>CF_INSTANCE_IP</strong></td><td>10.4.23.17</tr><tr><td><strong>CF_INSTANCE_KEY</strong></td><td>/etc/cf-instance-credentials/instance.key</tr><tr><td><strong>CF_INSTANCE_PORT</strong></td><td><pre>61020</pre></tr><tr><td><strong>CF_INSTANCE_PORTS</strong></td><td><pre>[
  {
    "external": 61020,
    "internal": 8080,
    "external_tls_proxy": 61022,
    "internal_tls_proxy": 61001
  },
  {
    "external": 61021,
    "internal": 2222,
    "external_tls_proxy": 61023,
    "internal_tls_proxy": 61002
  }
]</pre></tr><tr><td><strong>CF_SYSTEM_CERT_PATH</strong></td><td>/etc/cf-system-certificates</tr><tr><td><strong>DEPS_DIR</strong></td><td>/home/vcap/deps</tr><tr><td><strong>GEM_HOME</strong></td><td>/home/vcap/deps/0/vendor_bundle/ruby/2.7.0</tr><tr><td><strong>GEM_PATH</strong></td><td></tr><tr><td><strong>HOME</strong></td><td>/home/vcap/app</tr><tr><td><strong>INSTANCE_GUID</strong></td><td>32064364-6709-44b9-4a91-a1f3</tr><tr><td><strong>INSTANCE_INDEX</strong></td><td><pre>0</pre></tr><tr><td><strong>LANG</strong></td><td>en_US.UTF-8</tr><tr><td><strong>MEMORY_LIMIT</strong></td><td>256m</tr><tr><td><strong>OLDPWD</strong></td><td>/home/vcap</tr><tr><td><strong>PATH</strong></td><td>/home/vcap/deps/0/vendor_bundle/ruby/2.7.0/bin:/home/vcap/deps/0/bin:/usr/local/bin:/usr/bin:/bin</tr><tr><td><strong>PORT</strong></td><td><pre>8080</pre></tr><tr><td><strong>PWD</strong></td><td>/home/vcap/app</tr><tr><td><strong>RACK_ENV</strong></td><td>production</tr><tr><td><strong>RAILS_ENV</strong></td><td>production</tr><tr><td><strong>RAILS_LOG_TO_STDOUT</strong></td><td>enabled</tr><tr><td><strong>RAILS_SERVE_STATIC_FILES</strong></td><td>enabled</tr><tr><td><strong>RUBYLIB</strong></td><td>/home/vcap/deps/0/bundler/gems/bundler-2.2.28/lib</tr><tr><td><strong>RUBYOPT</strong></td><td>-r/home/vcap/deps/0/bundler/gems/bundler-2.2.28/lib/bundler/setup</tr><tr><td><strong>SHLVL</strong></td><td><pre>1</pre></tr><tr><td><strong>TMPDIR</strong></td><td>/home/vcap/tmp</tr><tr><td><strong>USER</strong></td><td>vcap</tr><tr><td><strong>VCAP_APPLICATION</strong></td><td><pre>{
  "application_id": "2d19faba-0cae-4cb7-8078-67c092cfcc33",
  "application_name": "test",
  "application_uris": [
    "tcp.apps.codex.starkandwayne.com:40001"
  ],
  "application_version": "16d2c062-932b-4902-b874-0ea519e01dd8",
  "cf_api": "https://api.system.codex.starkandwayne.com",
  "host": "0.0.0.0",
  "instance_id": "32064364-6709-44b9-4a91-a1f3",
  "instance_index": 0,
  "limits": {
    "disk": 1024,
    "fds": 16384,
    "mem": 256
  },
  "name": "test",
  "organization_id": "d396b0c6-872f-46a2-a752-bdea51819c06",
  "organization_name": "system",
  "port": 8080,
  "process_id": "2d19faba-0cae-4cb7-8078-67c092cfcc33",
  "process_type": "web",
  "space_id": "4e081328-2ac1-4509-8f51-ffcbfc012165",
  "space_name": "ops",
  "uris": [
    "tcp.apps.codex.starkandwayne.com:40001"
  ],
  "version": "16d2c062-932b-4902-b874-0ea519e01dd8"
}</pre></tr><tr><td><strong>VCAP_APP_HOST</strong></td><td>0.0.0.0</tr><tr><td><strong>VCAP_APP_PORT</strong></td><td><pre>8080</pre></tr><tr><td><strong>VCAP_SERVICES</strong></td><td><pre>{
}</pre></tr><tr><td><strong>_</strong></td><td>/home/vcap/deps/0/bin/bundle</tr></table></div><h2>HTTP Request Headers</h2><div><table><tr><td><strong>accept</strong></td><td>*/*</tr><tr><td><strong>host</strong></td><td>tcp.apps.codex.starkandwayne.com:40001</tr><tr><td><strong>user_agent</strong></td><td>curl/7.79.1</tr><tr><td><strong>version</strong></td><td>HTTP/1.1</tr></table></div></body></html>%
```




## Additional Reading

These links are to documentation used to put this guide together

 - Primary document on how to configure deploy CF with TCP Routing [here](https://docs.cloudfoundry.org/adminguide/enabling-tcp-routing.html)
 - Configuring routes/ports for apps after tcp routing is setup [here](https://docs.cloudfoundry.org/devguide/deploy-apps/routes-domains.html#create-route-with-port)
 - bbl terraform to create the tcp load balancer [here](https://github.com/cloudfoundry/bosh-bootloader/blob/main/terraform/aws/templates/cf_lb.tf#L244-L1041)
 - Trouble shoot tcp app issues [here](https://docs.cloudfoundry.org/adminguide/troubleshooting_tcp_routing.html)

Good Day!

PS: I grew up listening to Paul Harvey on the radio in my parent's station wagon.  You are missed good sir!