---
layout: post
title: "Enabling SSL for Stratos PostgreSQL Connections"
date: 2022-04-04
---

## SSL - Adding a bit of security

![pic](https://raw.githubusercontent.com/cweibel/ghost_blog_pics/master/ricardo-gomez-angel-Q06y764UY3A-unsplash2.png)

Photo by [Ricardo Gomez Angel](https://unsplash.com/@rgaleria?utm_source=unsplash&utm_medium=referral&utm_content=creditCopyText) on [Unsplash](https://unsplash.com)


Adding the requirement for SSL to [Stratos](https://stratos.app/docs/deploy/cloud-foundry/cloud-foundry) is a fairly easy process.  This configuration is highly recommended for production deployments of Stratos on Cloud Foundry.


In the example manifest below, this option is enabled by adding `DB_SSL_MODE: "verify-ca"` to the bottom of the environment variables:

```
applications:
  - name: console
    memory: 1512M
    disk_quota: 1024M
    host: console
    timeout: 180
    buildpack: https://github.com/cloudfoundry-incubator/stratos-buildpack#v4.0
    health-check-type: port
    services:
    - console_db
    env:
      CF_API_URL: https://api.bosh-lite.com
      CF_CLIENT: stratos_client
      CF_CLIENT_SECRET: sssshhhitsasecret
      SSO_OPTIONS: "logout, nosplash"
      SSO_WHITELIST: "https://console.bosh-lite.com"
      SSO_LOGIN: true
      DB_SSL_MODE: "verify-ca"
```

## Why this works

The example above relies on a CUPS service instance called `console_db` which points to a RDS PostgreSQL instance created manually.  Creating the CUPS service is as easy as:

```
cf cups console_db -p '{"uri": "postgres://", "username":"myuser", "password":"mypass", "hostname":"something.xyx.us-west-2.rds.amazon.com", "port":"5432", "dbname":"console_db"}'
```

Once executed, you can use the `console_db` as the name of the service in `manifest.yml` for Stratos.

Also take note that I'm using a RDS instance which means I need the RDS CA in the trusted store of the CF app container which Stratos is running in.  This is done by configuring the following ops file to be deployed against Cloud Foundry:

```
- type: replace
  path: /instance_groups/name=diego-cell/jobs/name=rep/properties/containers/trusted_ca_certificates/-
  value: &rds-uswest2-ca |-
    -----BEGIN CERTIFICATE-----
    MIIEBjCCAu6gAwIBAgIJAMc0ZzaSUK51MA0GCSqGSIb3DQEBCwUAMIGPMQswCQYD
    VQQGEwJVUzEQMA4GA1UEBwwHU2VhdHRsZTETMBEGA1UECAwKV2FzaGluZ3RvbjEi
    MCAGA1UECgwZQW1hem9uIFdlYiBTZXJ2aWNlcywgSW5jLjETMBEGA1UECwwKQW1h
    em9uIFJEUzEgMB4GA1UEAwwXQW1hem9uIFJEUyBSb290IDIwMTkgQ0EwHhcNMTkw
    ODIyMTcwODUwWhcNMjQwODIyMTcwODUwWjCBjzELMAkGA1UEBhMCVVMxEDAOBgNV
    BAcMB1NlYXR0bGUxEzARBgNVBAgMCldhc2hpbmd0b24xIjAgBgNVBAoMGUFtYXpv
    biBXZWIgU2VydmljZXMsIEluYy4xEzARBgNVBAsMCkFtYXpvbiBSRFMxIDAeBgNV
    BAMMF0FtYXpvbiBSRFMgUm9vdCAyMDE5IENBMIIBIjANBgkqhkiG9w0BAQEFAAOC
    AQ8AMIIBCgKCAQEArXnF/E6/Qh+ku3hQTSKPMhQQlCpoWvnIthzX6MK3p5a0eXKZ
    oWIjYcNNG6UwJjp4fUXl6glp53Jobn+tWNX88dNH2n8DVbppSwScVE2LpuL+94vY
    0EYE/XxN7svKea8YvlrqkUBKyxLxTjh+U/KrGOaHxz9v0l6ZNlDbuaZw3qIWdD/I
    6aNbGeRUVtpM6P+bWIoxVl/caQylQS6CEYUk+CpVyJSkopwJlzXT07tMoDL5WgX9
    O08KVgDNz9qP/IGtAcRduRcNioH3E9v981QO1zt/Gpb2f8NqAjUUCUZzOnij6mx9
    McZ+9cWX88CRzR0vQODWuZscgI08NvM69Fn2SQIDAQABo2MwYTAOBgNVHQ8BAf8E
    BAMCAQYwDwYDVR0TAQH/BAUwAwEB/zAdBgNVHQ4EFgQUc19g2LzLA5j0Kxc0LjZa
    pmD/vB8wHwYDVR0jBBgwFoAUc19g2LzLA5j0Kxc0LjZapmD/vB8wDQYJKoZIhvcN
    AQELBQADggEBAHAG7WTmyjzPRIM85rVj+fWHsLIvqpw6DObIjMWokpliCeMINZFV
    ynfgBKsf1ExwbvJNzYFXW6dihnguDG9VMPpi2up/ctQTN8tm9nDKOy08uNZoofMc
    NUZxKCEkVKZv+IL4oHoeayt8egtv3ujJM6V14AstMQ6SwvwvA93EP/Ug2e4WAXHu
    cbI1NAbUgVDqp+DRdfvZkgYKryjTWd/0+1fS8X1bBZVWzl7eirNVnHbSH2ZDpNuY
    0SBd8dj5F6ld3t58ydZbrTHze7JJOd8ijySAp4/kiu9UfZWuTPABzDa/DSdz9Dk/
    zPW4CXXvhLmE02TA9/HeCw3KEHIwicNuEfw=
    -----END CERTIFICATE-----
- type: replace
  path: /instance_groups/name=diego-cell/jobs/name=cflinuxfs3-rootfs-setup/properties/cflinuxfs3-rootfs/trusted_certs/-
  value: *rds-uswest2-ca
```

The RDS certs for other AWS regions are documented at [https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/UsingWithRDS.SSL.html](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/UsingWithRDS.SSL.html) 



## Verifying that Database Connections is using SSL 

Trust but verify.  By making a `psql` connection to the RDS instance you can verify the connections from Stratos are indeed leveraging SSL.  Run the following:

```
SELECT 
  datid.datname,
  pg_stat_ssl.pid,
  usesysid,
  usename,
  application_name,
  client_addr,
  client_hostname,
  client_port,
  ssl,
  cipher,
  bits,
  compression
FROM
  pg_stat_activity,
  pg_stat_ssl
WHERE
  pg_stat_activity.pid = pg_stat_ssl.pid
  AND pg_stat_activity.usename = 'myuser';  # Name of the user you configured in CUPS


 dataid |  datname   |  pid  | usesysid | username | application_name | client_addr  | client_hostname | client_port | ssl |           cipher            | bits | compression
 -------+------------+-------+----------+----------+------------------+--------------+-----------------+-------------+-----+-----------------------------+------+------------
  16104 | console_db |  3518 |    16939 | myuser   |                  | 10.244.0.20  |                 |       43104 | t   | ECDHE-RSA-AES256-GCM-SHA384 |  256 | f
  16104 | console_db | 22334 |    16939 | myuser   |                  | 10.244.0.20  |                 |       56321 | t   | ECDHE-RSA-AES256-GCM-SHA384 |  256 | f
  16104 | console_db | 25259 |    16939 | myuser   | psql             | 10.244.0.99  |                 |       58990 | t   | ECDHE-RSA-AES256-GCM-SHA384 |  256 | f

```


In the example above, the third connection is the `psql` client we are running this query from, the other two connections are coming from the Stratos app on the Diego cell.

## What doesn't work

If you are attempting to set the SSL Mode via the URI, while a valid assumption, configuring the CUPS connection will be ignored:

```
cf cups console_db -p '{"uri": "postgres://", "username":"myuser", "password":"mypass", "hostname":"something.xyx.us-west-2.rds.amazon.com", "port":"5432", "dbname":"console_db", "sslmode":"verify-ca" }'
```

This is because the Stratos configuration is specifically looking for an environment variable:

> db.SSLMode = env.String("DB_SSL_MODE", "disable")

From [https://github.com/cloudfoundry/stratos/blob/master/src/jetstream/datastore/database_cf_config.go#L81](https://github.com/cloudfoundry/stratos/blob/master/src/jetstream/datastore/database_cf_config.go#L81)



Enjoy!  
