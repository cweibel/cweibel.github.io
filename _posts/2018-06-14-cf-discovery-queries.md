---
layout: post
title: "Cloud Foundry Foundation Discover Queries"
date: 2018-06-14
---

![map](https://raw.githubusercontent.com/cweibel/ghost_blog_pics/master/steve-smith-EDLKpRS118o-unsplash.jpg)



Photo by [Steve Smith](https://unsplash.com/@varrak?utm_source=unsplash&utm_medium=referral&utm_content=creditCopyText) on [Unsplash](https://unsplash.com/s/photos/uri?utm_source=unsplash&utm_medium=referral&utm_content=creditCopyText)


Use these queries to get an overview on a CF/PCF/TCF/SCF/`*`CF environment from Cloud Controller's perspective.

You'll need to connect to your ccdb PostgreSQL or MySQL instance. Quick hint: all the `db:` info you need is in `/var/vcap/jobs/cloud_controller_ng/config/cloud_controller_ng.yml` on you `api/0` vm.

## Show Service Broker Utilization

For each service broker, show the service instances and app/space/org information about the bound service. Note that unbound services are not included.

```
select
	organizations.name as org_name,
	spaces.name as space_name,
	apps.name as app_name,
	service_brokers.name as service_broker_name,
	service_instances.name as service_instance_name,
	service_plans.name as service_plan_name,
	processes.state as process_state
from
	service_bindings,
	service_instances,
	service_plans,
	services,
	service_brokers,
	apps, 
	spaces, 
	organizations, 
	processes
where
	service_bindings.service_instance_guid = service_instances.guid
	and service_instances.service_plan_id = service_plans.id
	and service_plans.service_id = services.id
	and services.service_broker_id = service_brokers.id
	and apps.space_guid = spaces.guid 
	and spaces.organization_id=organizations.id 
	and processes.app_guid = apps.guid
	and service_bindings.app_guid = apps.guid
order by
	1,2,3,4,5;
```
Output:

```
 org_name | space_name |  app_name   | service_broker_name | service_instance_name | service_plan_name | process_state
----------+------------+-------------+---------------------+-----------------------+-------------------+---------------
 system   | system     | cf-apigen   | vault               | secrets               | shared            | STARTED
 system   | system     | cf-vault-ui | vault               | secrets               | shared            | STARTED
 system   | system     | web-app     | blacksmith          | postgresql            | standalone        | STARTED
 system   | system     | web-app     | shield-broker       | shield                | shared            | STARTED
 system   | system     | web-app     | vault               | secrets               | shared            | STARTED
```

## Cell Resources Consumed

Use this to determine the amount of memory and disk currently allocated to apps:

```
select 
	sum(instances) as app_instances,  
	sum(memory * instances) as allocated_memory, 
	sum( disk_quota * instances) as allocated_disk, 
	state 
from 
	processes 
group by 
	state;
```

Output:

```
 app_instances | allocated_memory | allocated_disk |  state
---------------+------------------+----------------+---------
             1 |             1024 |           1024 | STOPPED
             9 |             1984 |           3600 | STARTED
```

## List all Apps, Orgs, Spaces

This is useful for knowing a point-in-time the list of apps and their state.

```
SELECT 
	organizations.name as org_name, 
	spaces.name as space_name, 
	apps.name as app_name, 
	processes.instances as instance_count, 
	processes.state as state 
from 
	apps, 
	spaces, 
	organizations, 
	processes 
where 
	apps.space_guid = spaces.guid 
	and spaces.organization_id=organizations.id 
	and processes.app_guid = apps.guid 
order by 1,2,3;
```

Output:

```
 org_name | space_name |    app_name    | instance_count |  state
----------+------------+----------------+----------------+---------
 system   | system     | bolo-cf-nozzle |              4 | STARTED
 system   | system     | cachefire      |              1 | STOPPED
 system   | system     | cf-apigen      |              1 | STARTED
 system   | system     | cfseeker       |              1 | STARTED
 system   | system     | cf-vault-ui    |              1 | STARTED
 system   | system     | vault-broker   |              1 | STARTED
 system   | system     | web-app        |              1 | STARTED
```

## Show Orgs Exceeding Quota

Use this query to show the Orgs which are exceeding their current memory quotas which will prevent restarting the existing apps in that org. This is usually the result of a change to a quota associated to an org.

The `memory_buffer` column will show by how many mb of RAM is left. Any orgs with negative numbers in this field have a total memory which exceeds the quota.

```
SELECT a.org_name,
       a.actual_app_instances,
       (CASE WHEN a.max_app_instances > 0 THEN a.max_app_instances
             ELSE NULL END) AS max_app_instances,
       (CASE WHEN a.max_app_instances > 0 THEN a.max_app_instances - a.actual_app_instances
             ELSE 99999999 END) AS instances_buffer,
       a.allocated_memory, a.max_memory,
       a.max_memory - a.allocated_memory AS memory_buffer
FROM (
    SELECT
      o.name AS org_name,
      sum(p.instances) as actual_app_instances,
      max(q.app_instance_limit) AS max_app_instances,
      sum(p.memory * p.instances) as allocated_memory,
      max(q.memory_limit) AS max_memory
    FROM 
        processes p, apps a, spaces s, organizations o, quota_definitions q
    WHERE
          p.app_guid = a.guid
      AND s.guid = a.space_guid
      AND o.id = s.organization_id
      AND q.id = o.quota_definition_id
      AND p.state = 'STARTED'
    GROUP BY 
        o.id) a
ORDER BY 7,1;
```

Output:

```
+------------+----------------------+-------------------+------------------+------------------+------------+---------------+
| org_name   | actual_app_instances | max_app_instances | instances_buffer | allocated_memory | max_memory | memory_buffer |
+------------+----------------------+-------------------+------------------+------------------+------------+---------------+
| org1       |                    2 |              1000 |              998 |             2048 |       2048 |             0 |
| org2       |                    3 |              1000 |              997 |             2024 |       2048 |            24 |
| org3       |                    2 |              1000 |              998 |             2000 |       2048 |            48 |
+------------+----------------------+-------------------+------------------+------------------+------------+---------------+
```

## Summary Queries

### Org Count

> SELECT COUNT(*) FROM organizations;

### Spaces Count

> SELECT COUNT(*) FROM spaces;

### App Count

> SELECT COUNT(*) FROM apps;

### Quota Definitions

> SELECT COUNT(*) FROM quota_definitions;

If you combine this output with the [cf inventory scraper](https://github.com/cloudfoundry-community/cloudfoundry-utils/blob/master/bin/cf-inventory) you'll get a full overview of your CF deployment.

## List of Organizations and the number of application instances they've started in the last 7 days:

```
SELECT 
  sum(processes.instances) as app_instance_count,
  organizations.name
FROM 
  processes, 
  apps, 
  spaces, 
  organizations 
WHERE 
  processes.app_guid=apps.guid and 
  apps.space_guid=spaces.guid and
  spaces.organization_id=organizations.id and
  processes.state='STARTED' and 
  processes.created_at > DATE(NOW()) - INTERVAL 7 DAY 
GROUP BY 
  organizations.id, 
  organizations.name 
ORDER BY 1 desc;
```

## List of Organizations and how much RAM their running application instances have reserved:

```
SELECT
  sum(processes.memory * processes.instances) AS memory_reserved,  
  organizations.name 
FROM 
  processes, 
  apps, 
  spaces, 
  organizations 
WHERE 
  processes.app_guid=apps.guid and 
  apps.space_guid=spaces.guid and
  spaces.organization_id=organizations.id and
  processes.state='STARTED' 
GROUP BY 
  organizations.id, 
  organizations.name 
ORDER BY 1 desc;
```