---
layout: post
title: "Configuring SHIELD v8 to use UAA"
date: 2017-12-14
---

![map](https://raw.githubusercontent.com/cweibel/ghost_blog_pics/master/steve-smith-9gJsKvfbvgE-unsplash.jpg)



Photo by [Steve Smith](https://unsplash.com/@varrak?utm_source=unsplash&utm_medium=referral&utm_content=creditCopyText) on [Unsplash](https://unsplash.com/s/photos/uri?utm_source=unsplash&utm_medium=referral&utm_content=creditCopyText)

## OVERVIEW

The upcoming release of SHIELD v8 adds support for GitHub and UAA authentication providers. Previous versions of SHIELD only supported basic auth. Many organizations already heavily rely on GitHub and UAA to organize users and permissions within an organization. You will be able to leverage these existing authentication providers to grant access to SHIELD.

[Multi-tenancy](OVERVIEW

The upcoming release of SHIELD v8 adds support for GitHub and UAA authentication providers. Previous versions of SHIELD only supported basic auth. Many organizations already heavily rely on GitHub and UAA to organize users and permissions within an organization. You will be able to leverage these existing authentication providers to grant access to SHIELD.

Multi-tenancy is also being added. A tenant is a single group that defines the context for interaction with resources in a SHIELD configuration. All retention policies, jobs, backup targets, storage endpoints, and archives belong to a single tenant. Each tenant creates and manages its own target and storage configurations, and tenants are prevented from accessing or viewing another tenant's configuration.

The rest of this blog explores leveraging groups defined in UAA and mapping these groups to tenants in SHIELD. There are many different ways to mapping tenant roles to UAA groups, you are encouraged to understand these roles and map them to your organization as you see fit.) is also being added. A tenant is a single group that defines the context for interaction with resources in a SHIELD configuration. All retention policies, jobs, backup targets, storage endpoints, and archives belong to a single tenant. Each tenant creates and manages its own target and storage configurations, and tenants are prevented from accessing or viewing another tenant's configuration.

The rest of this blog explores leveraging groups defined in UAA and mapping these groups to tenants in SHIELD. There are many different ways to mapping [tenant roles](https://github.com/starkandwayne/shield/blob/v8/docs/auth/uaa.md#mappings) to UAA groups, you are encouraged to understand these roles and map them to your organization as you see fit.

## Use Case

Assume we work at a company where UAA is used to control access to BOSH with the following groups of people:

 - Admins are defined in the BOSH deployment manifest's UAA properties to control the cloud-config and platform level maintenance such as deploying BOSH, Cloud Foundry & SHIELD.
 - There are also [BOSH Teams](https://bosh.io/docs/director-users-uaa-perms.html#team-admin) defined where members of each of these teams can deploy without having access to the platform deployments or deployments from other BOSH teams.
 - There are three BOSH Teams defined with this sandboxed access: Postgres Team, RabbitMQ Team, and Vault Team.
Our goal is for each of these teams to control their own BOSH deployments and also control their own backups and restores with SHIELD without impacting each other. The overall use case looks like:

![chart](https://raw.githubusercontent.com/cweibel/ghost_blog_pics/master/shield.v8.uaa.png)

## Implementation

Configuring SHIELD to use UAA is done in two steps:

 - Configure UAA groups and membership in BOSH
 - Configure the UAA group mappings to SHIELD tenants and roles

## Step 1 - Configuring BOSH

We've identified a number of groups in our organization which have different access requirements to both BOSH and SHIELD. Below we break down how to turn pieces of the use case into configuration points in the BOSH deployment manifest.

### Define BOSH Teams / UAA Groups

In the manifest for BOSH we can define the three teams, one each for the Postgres Team, RabbitMQ Team, and Vault Team:

```
properties:
  uaa:
    scim:
      groups:
        bosh.teams.postgres.admin: Postgres Admin Group
        bosh.teams.rabbitmq.admin: RabbitMQ Admin Group
        bosh.teams.vault.admin:    Vault Admin Group
```
 
So that each team can also upload their own stemcells and releases, we also add the groups `bosh.read`, `bosh.releases.upload`, and `bosh.stemcell.upload` as described in the [BOSH Teams documentation](https://bosh.io/docs/director-users-uaa-perms.html#team-admin):

```
properties:
  uaa:
    scim:
      groups:
        bosh.read: BOSH Read Only Access
        bosh.releases.upload: BOSH Releases Upload Group
        bosh.stemcells.upload: BOSH Stemcell Upload Group
``` 

We'll also add three SHIELD specific UAA groups to make mapping a bit easier to each of the SYSTEM tenant roles understood by SHIELD (admin, manager, engineer):

```
properties:
  uaa:
    scim:
      groups:
        shield.admin: SHIELD System Admin Group
        shield.manager: SHIELD System Manager Group
        shield.engineer: SHIELD System Engineering Group
```

### Define UAA Users

Now that all the UAA groups have been defined for our use case, the UAA users can be added. UAA users are defined with the usernames and passwords used for authentication. UAA users are associated with one or more UAA groups.

We'll start with a typical BOSH Admin. These individuals have full access to BOSH and are responsible for deploying and maintaining BOSH, SHIELD, and Cloud Foundry. Normally, this account would be configured with `bosh.admin`, `uaa.admin`, `scim.read` and `scim.write` access but we're also adding `shield.admin` to make the mapping clearer in Step 2. Replace admin with a real person's name or identifier for your own deployment: 
 
```
properties:
  uaa:
    scim:
      users:
      - name: admin
        groups: [shield.admin, bosh.admin, uaa.admin, scim.read, scim.write]
        password: admin2
```

To add a user to the Postgres Team, they will need to be placed in their own BOSH team and be able to upload BOSH releases and stemcells. The same needs to be done for the RabbitMQ and Vault team members and is omitted for brevity. Again, replace `postgres.admin` with a real person's name or identifier. 

```
properties:
  uaa:
    scim:
      users:
      - name: postgres.admin
        groups: [bosh.teams.postgres.admin, bosh.read, bosh.releases.upload, bosh.stemcells.upload]
        password: psq5432
```

### Define a UAA Client

The SHIELD daemon also needs to communicate with UAA to retrieve information about group membership. This is done by configuring a UAA Client, an alternate set of instructions using the `uaac` client are described [here](https://github.com/starkandwayne/shield/blob/v8/docs/auth/uaa.md#registering-a-client-with-uaa). The only change you need to make for your own deployment is to the `redirect-uri` which will be the URL to the SHIELD daemon:

```
properties:
  uaa:
    clients:
      shield-dev:
        name: S.H.I.E.L.D.
        override: true
        authorized-grant-types: authorization_code
        scope: openid
        authorities: uaa.none
        access-token-validity: 180
        refresh-token-validity: 180
        secret: "s.h.i.e.l.d."
        redirect-uri: http://localhost:8181/auth/uaa1/redir
```

Above, the `redirect-uri` also contains the name of the UAA instance, called the `identifier`, which in this example has the value `uaa1` which will be needed later in Step 2. The "name" of the client `shield-dev` and the "secret" `s.h.i.e.l.d.` will also be needed in Step 2.

### Complete BOSH Manifest Snippet

Putting this all together, here is the full snippet we'll be adding to the BOSH manifest:

```
properties:
  uaa:
    scim:
      groups:
        bosh.read: BOSH Read Only Access
        bosh.releases.upload: BOSH Releases Upload Group
        bosh.stemcells.upload: BOSH Stemcell Upload Group
        bosh.teams.postgres.admin: Postgres Admin Group
        bosh.teams.rabbitmq.admin: RabbitMQ Admin Group
        bosh.teams.vault.admin: Vault Admin Group
        shield.admin: SHIELD System Admin Group
        shield.manager: SHIELD System Manager Group
        shield.engineer: SHIELD System Engineering Group
      users:
      - name: admin
        groups: [shield.admin, bosh.admin, uaa.admin, scim.read, scim.write]
        password: admin2
      - name: a.security.engineer
        groups: [shield.engineer, bosh.read, scim.read]
        password: admin2
      - name: postgres.admin
        groups: [bosh.teams.postgres.admin, bosh.read, bosh.releases.upload, bosh.stemcells.upload]
        password: psq5432
      - name: rabbitmq.admin
        groups: [bosh.teams.rabbitmq.admin, bosh.read, bosh.releases.upload, bosh.stemcells.upload]
        password: rabbit5672
      - name: vault.admin
        groups: [bosh.teams.vault.admin, bosh.read, bosh.releases.upload, bosh.stemcells.upload]
        password: hashi8200
    clients:
      shield-dev:
        name: S.H.I.E.L.D.
        override: true
        authorized-grant-types: authorization_code
        scope: openid
        authorities: uaa.none
        access-token-validity: 180
        refresh-token-validity: 180
        secret: "s.h.i.e.l.d."
        redirect-uri: http://localhost:8181/auth/uaa1/redir
```

Now redeploy your BOSH Director so the changes are pushed to the UAA instance on the director.

Note that all of this could have been done using `uaac` commands however if you have multiple environments, having the groups, users, and clients defined in the BOSH manifest makes it easier to repeat this configuration. The passwords are also easily cracked in this example, use more secure passwords or let [credhub automatically create them for you](https://docs.cloudfoundry.org/credhub/setup-credhub-bosh.html).

## Step 2 - Configure UAA Groups Mapping to SHIELD Tenants

This portion of the configuration is responsible for controlling which authentication providers will be used by SHIELD. This is also where the mapping between UAA Groups, SHIELD Tenants, and Roles are defined. This is done by modifying the deployment manifest for SHIELD.

### Define UAA Authentication Provider

Start with the configuring SHIELD to allow `uaa` to be one of the authentication providers. At the end of Step 1 we configured a UAA Client for the SHIELD daemon. From this we can extract the `client_id`, `client_secret` and `identifier`.

```
properties:
  uaa:
    clients:
      shield-dev:                                            #<== Match to client_id:
        secret: "s.h.i.e.l.d."                               #<== Match to client_secret:
        redirect-uri: http://localhost:8181/auth/uaa1/redir  #<== `uaa1` matches to the identifier:
```

Now we can start to configure the manifest for SHIELD, the source of information is commented on each line:

```
auth:
  - name:       Prod BOSH UAA                     #<== This text shows up in the SHIELD login page 
    identifier: uaa1                              #<== From the BOSH Manifest
    backend:    uaa                               #<== Must be `uaa`, indicates provider type
    properties:
      client_id:       shield-dev                 #<== From the BOSH Manifest
      client_secret:   s.h.i.e.l.d.               #<== From the BOSH Manifest
      uaa_endpoint:    https://192.168.50.6:8443  #<== Points to the UAA instance on BOSH
      skip_verify_tls: true
```

### Define Mappings - BOSH Admin & Security Admin

First we will start with the mappings for the BOSH Admin and the Security Admin. There is a special tenant, called the SYSTEM tenant, that exists solely to allow SHIELD site operators to assign system-level rights and roles to UAA group members, based on the same rules as tenant-level role assignment.

The SYSTEM tenant has its own set of assignable roles:

 - admin - Full control over all of SHIELD.
 - manager - Control over tenants and manual role assigments.
 - engineer - Control over shared resources like global storage definitions and retention policy templates.

These rights rules are processed until one matches; subsequent rules are skipped.

Here is our use case mapping:

![pic](https://raw.githubusercontent.com/cweibel/ghost_blog_pics/master/shield.v8.uaa.bosh.admin.security.admin.png)

We'll use the `System` tenant and map the UAA group `BOSH Admins` are mapped to `SHIELD System Admin` role in the use case. We are using both `bosh.admin` and `shield.admin` in case there were other users defined as `bosh.admin` manually through `uaac`.

We also will map the UAA group shield.engineer, which the `Security Admin` is a part of to the `SHIELD System Engineer`, that will have a SHIELD role of `engineer`:

```
auth:
  - name:       Prod BOSH UAA
    properties:
      mapping:
        - tenant: SYSTEM
          rights:
            - { scim: bosh.admin,      role: admin }
            - { scim: shield.admin,    role: admin }
            - { scim: shield.engineer, role: engineer }
```

### Define Mappings - BOSH Admin for the BOSH and CF Tenant

It is also desired to have BOSH Admins control the backups for the BOSH Director database, the SHIELD database, and Cloud Foundry's databases.

![pic](https://raw.githubusercontent.com/cweibel/ghost_blog_pics/master/shield.v8.uaa.bosh.ccdb.png)

For non-SYSTEM tenants valid values for the role field are:

 - admin - Full control over the tenant.
 - engineer - Control over the configuration of stores, targets, retention policies, and jobs.
 - operator - Control over running jobs, pausing and unpausing scheduled jobs, and performing restore operations.


In our case we want anyone who is a BOSH Admin to be a BOSH & CF Tenant Admin. To do this, define a SHIELD tenant and associate the BOSH Admins to be a SHIELD admin for this tenant:

```
auth:
  - name:       Prod BOSH UAA
    properties:
      mapping:
        - tenant: BOSH and CF
          rights:
            - { scim: bosh.admin, role: admin }
```

### Define Mappings - Postgres Admin

Now we can map the tenant for the Postgres Admin. The Postgres Admin is identified by the UAA group `bosh.teams.postgres.admin` and we want this person to have complete control of their backups.

![pic](https://raw.githubusercontent.com/cweibel/ghost_blog_pics/master/shield.v8.uaa.postgres.admin.png)

To do this define, a SHIELD tenant named `Postgres Team`, and associate the UAA Group `bosh.teams.postgres.admin` for the Postgres Admin with the SHIELD tenant role `admin` for the Postgres Team tenant:

```
auth:
  - name:       Prod BOSH UAA
    properties:
      mapping:
        - tenant: Postgres Team
          rights:
            - { scim: bosh.teams.postgres.admin, role: admin }
```


### Define Mappings - RabbitMQ Admin

Now we can map the tenant for the RabbitMQ Admin. The RabbitMQ Admin is identified by the UAA group `bosh.teams.rabbitmq.admin` and we want this person to have complete control of their backups.

![use cases](https://raw.githubusercontent.com/cweibel/ghost_blog_pics/master/shield.v8.uaa.rabbitmq.admin.png)

To do this, define a SHIELD tenant named `RabbitMQ Team` and associate the UAA Group `bosh.teams.rabbitmq.admin` for the RabbitMQ Admin with the SHIELD tenant role `admin` for the RabbitMQ Team tenant:

```
auth:
  - name:       Prod BOSH UAA
    properties:
      mapping:
        - tenant: RabbitMQ Team
          rights:
            - { scim: bosh.teams.rabbitmq.admin, role: admin }
```

### Define Mappings - Vault Admin

Now we can map the tenant for the Vault Admin. The Vault Admin is identified by the UAA group `bosh.teams.vault.admin`, and we want this person to have complete control of their backups.

![use cases](https://raw.githubusercontent.com/cweibel/ghost_blog_pics/master/shield.v8.uaa.vault.admin.png)

To do this define a SHIELD tenant named `Vault Team` and associate the UAA Group `bosh.teams.vault.admin` for the Vault Admin with the SHIELD tenant role `admin` for the Vault Team tenant:

```
auth:
  - name:       Prod BOSH UAA
    properties:
      mapping:
        - tenant: Vault Team
          rights:
            - scim: bosh.teams.vault.admin
              role: admin
```

### The Complete Mapping

The full snippet for the defining the UAA authentication provider and mappings is below. Members of the mapped UAA groups will be able to connect to the SHIELD tenants they are mapped to.

```
auth:
  - name:       Prod BOSH UAA
    identifier: uaa1
    backend:    uaa
    properties:
      client_id:       shield-dev
      client_secret:   s.h.i.e.l.d.
      uaa_endpoint:    https://192.168.50.6:8443
      skip_verify_tls: true
      mapping:
        - tenant: SYSTEM                       
          rights:
            - { scim: bosh.admin,      role: admin }
            - { scim: shield.admin,    role: admin }
            - { scim: shield.engineer, role: engineer }

        - tenant: BOSH and CF
          rights:
            - { scim: bosh.admin, role: admin }

        - tenant: Postgres Team               
          rights:
            - { scim: bosh.teams.postgres.admin, role: admin }                     

        - tenant: RabbitMQ Team               
          rights:
            - { scim: bosh.teams.rabbitmq.admin, role: admin }                
                                
        - tenant: Vault Team     
          rights:
            - { scim: bosh.teams.vault.admin, role: admin }
```

### Selecting UAA Authentication Provider for Login

Once SHIELD is deployed with the configuration for the UAA this authentication provider will be available via the SHIELD UI. The authentication provider defined in `auth.name` of the SHIELD manifest is what will appear on the login screen, clicking on this will start the UAA interaction:

![pic](https://raw.githubusercontent.com/cweibel/ghost_blog_pics/master/shield_v8_login.png)

### Login to UAA

Now the interaction with UAA is initiated. Enter the username and password of one of the UAA user accounts that was defined in the manifest for BOSH. Note the URL in the browser navigates to the UAA instance on the BOSH Director as defined in `auth.properties.uaa_endpoint` of the SHIELD manifest. In the screenshot below, we'll login as BOSH Admin:

![pic](https://raw.githubusercontent.com/cweibel/ghost_blog_pics/master/shield_v8_uaa_login.png)

### System Tenant & CF and BOSH Tenant

We are now logged in as the BOSH Admin. The BOSH Admin belongs to two SHIELD tenants: `SYSTEM` and `BOSH and CF`. Membership to SYSTEM gives the user access to the `Admin` tab in the UI. Membership to the `BOSH and CF` tenant allows the other `Systems`, `Storage`, and `Retention` tabs to manage backups for BOSH, CF, and SHIELD database backups:

![pic](https://raw.githubusercontent.com/cweibel/ghost_blog_pics/master/shield_v8_bosh_admin_homepage.png)

### Postgres Tenant

If instead we had logged in as `Postgres Admin` we would have access to the Postgres Team tenant but not access to other tenants or the `Admin` tab:

![pic](https://raw.githubusercontent.com/cweibel/ghost_blog_pics/master/shield_v8_postgres_admin_homepage.png)

## Next Steps

Another supported SHIELD Authentication Provider is Github; it also has the concept of mapping. This involves mapping Github orgs to SHIELD tenants and roles. More about this is [here](https://github.com/starkandwayne/shield/blob/v8/docs/auth/github.md).

You can also use a stand-alone deployment of UAA, or even the UAA instance that is included in Cloud Foundry. The example in this blog is based on a real world use case where there are BOSH Teams already leveraged to sandbox BOSH access. There are also tenant roles which were not used because each of the teams have a few people all with the same responsibilities to their teams. The additional tenant roles can be used for teams where team members have different roles and responsibilities.

Everyone deserves nice things, if there is another use case you would like to discuss, please contact us! There is a Slack channel `#help` in [shieldproject.slack.com](https://shieldproject.slack.com/) or simply reply to this blog post.

Hint, tag `#tpol` if no one answers right away!