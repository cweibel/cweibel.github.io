---
layout: post
title: "Curl a CF API Endpoint"
date: 2017-03-03
---

![map](https://raw.githubusercontent.com/cweibel/ghost_blog_pics/master/steve-smith-qtpRoo_hZ_k-unsplash.jpg)



Photo by [Steve Smith](https://unsplash.com/@varrak?utm_source=unsplash&utm_medium=referral&utm_content=creditCopyText) on [Unsplash](https://unsplash.com/s/photos/uri?utm_source=unsplash&utm_medium=referral&utm_content=creditCopyText)

There may be times when an operator wants to test a Cloud Foundry API server for operations to list spaces, organizations, etc. There are many ways of doing this, below is one example.

Before starting you will need three pieces of information:

 - `client_id` with `cloud_controller.admin` privilege
 - `client_secret` for the `client_id`
 - api url

## Obtaining Your Bearer Token

Log into a server with the `uaac` client installed which has network access to your Cloud Foundry deployment:

```
uaac token client get my_client_defined_in_cf_uaa
```

You should be challenged to enter your `client_secret`, manually enter this and you should get a confirmation screen similar to:

```
Client secret:  ****************

Successfully fetched token via client credentials grant.
Target: https://login.system.aws-usw02-dev.ice.mycf.io
Context: my_client_defined_in_cf_uaa, from client my_client_defined_in_cf_uaa
```

Your bearer token will be written to `~/.uaac.yml`, view this file and locate the bearer token for this particular `client_id`:

```
...
my_client_defined_in_cf_uaa:
  current: true
  client_id: my_client_defined_in_cf_uaa
  access_token: eyJhbGciOiJSUzI1NiIsImtpZCI6ImxlZ2FjeS10b2tlbi1rZXkiLCJ0eXAiOiJKV1QifQ.eyJqdGkiOiI0M2FiNWJlOGQ2OTU0NTIyYmnotAholeTokenNiceTryF0aW9ucy53cml0ZSIsImNsb3VkX2NvbnRyb2xsZXIuYWRtaW4iXSwiY2xpZW50X2lkIjoiYXV0b3NjYWxpbmdfc2VydmljZSIsImNpZCI6ImF1dG9zY2FsaW5nX3NlcnZpY2UiLCJhenAiOiJhdXRvc2NhbGluZ19zZXJ2aWNlIiwiZ3JhbnRfdHlwZSI6ImNsaWVudF9jcmVkZW50aWFscyIsInJldl9zaWciOiIzZmY5NGJlZSIsImlhdCI6MTQ4ODU1NDUxNSwiZXhwIjoxNDg4NTk3NzE1LCJpc3MiOiJodHRwczovL3VhYS5zeXN0ZW0uYXdzLXVzdzAyLWRldi5pY2UucHJlZGl4LmlvL29hdXRoL3Rva2VuIiwiemlkIjoidWFhIiwiYXVkIjpbImVtYWlscyIsImNsb3VkX2NvbnRyb2xsZXIiLCJjcml0aWNhbF9ub3RpZmljYXRpb25zIiwiYXV0b3NjYWxpbmdfc2VydmljZSIsIm5vdGlmaWNhdGlvbnMiXX0.HTVyno0zcG0totvwkkWbqNsDoClCzDvibawrK8zPlG0B1iZXcyxiVMscIZZ44mDVnLHXvhoHeA4wFB2k6GyuPuGCOcCRZoGyjCgxhWzmLLaIL9HfTPhiWsvAldt36XxtPVsV7lfuGRYewC3kqdLYRWxX2xQRb0GzhUfRO3ToriylFlICt6btn480ecg4c_D-7tlN9E4Ivneqw0zT_sJXWMTBI5xu-9luFys0hwXEg8cUWg8xsX3VxbZZSxteD7E0FODewjiuxtDYXpBa8iQssftvMDbUJy8wv2gqj2iRbgiPOhxJ3w3UPRMohlYv96Ooh01uP4C7mAHR7YuiAzrl5g
  token_type: bearer
  expires_in: 43199
  scope:
  - emails.write
  - cloud_controller.read
  - cloud_controller.write
  - notifications.write
  - critical_notifications.write
  - cloud_controller.admin
...

```

## Now Hit the API

Using the bearer token and the API url you can curl one of the API endpoints:

```
curl -i -X GET -H "Authorization: bearer eyJhbGciOiJSUzI1NiIsImtpZCI6ImxlZ2FjeS10b2tlbi1rZXkiLCJ0eXAiOiJKV1QifQ.eyJqdGkiOiI0M2FiNWJlOGQ2OTU0NTIyYmnotAholeTokenNiceTryF0aW9ucy53cml0ZSIsImNsb3VkX2NvbnRyb2xsZXIuYWRtaW4iXSwiY2xpZW50X2lkIjoiYXV0b3NjYWxpbmdfc2VydmljZSIsImNpZCI6ImF1dG9zY2FsaW5nX3NlcnZpY2UiLCJhenAiOiJhdXRvc2NhbGluZ19zZXJ2aWNlIiwiZ3JhbnRfdHlwZSI6ImNsaWVudF9jcmVkZW50aWFscyIsInJldl9zaWciOiIzZmY5NGJlZSIsImlhdCI6MTQ4ODU1NDUxNSwiZXhwIjoxNDg4NTk3NzE1LCJpc3MiOiJodHRwczovL3VhYS5zeXN0ZW0uYXdzLXVzdzAyLWRldi5pY2UucHJlZGl4LmlvL29hdXRoL3Rva2VuIiwiemlkIjoidWFhIiwiYXVkIjpbImVtYWlscyIsImNsb3VkX2NvbnRyb2xsZXIiLCJjcml0aWNhbF9ub3RpZmljYXRpb25zIiwiYXV0b3NjYWxpbmdfc2VydmljZSIsIm5vdGlmaWNhdGlvbnMiXX0.HTVyno0zcG0totvwkkWbqNsDoClCzDvibawrK8zPlG0B1iZXcyxiVMscIZZ44mDVnLHXvhoHeA4wFB2k6GyuPuGCOcCRZoGyjCgxhWzmLLaIL9HfTPhiWsvAldt36XxtPVsV7lfuGRYewC3kqdLYRWxX2xQRb0GzhUfRO3ToriylFlICt6btn480ecg4c_D-7tlN9E4Ivneqw0zT_sJXWMTBI5xu-9luFys0hwXEg8cUWg8xsX3VxbZZSxteD7E0FODewjiuxtDYXpBa8iQssftvMDbUJy8wv2gqj2iRbgiPOhxJ3w3UPRMohlYv96Ooh01uP4C7mAHR7YuiAzrl5g"  https://api-reporting.system.mycf.io/v2/spaces
```

Why is this useful? You might be setting up a lone API server with a different URL then the rest of your API servers for a monitoring tool such as Prometheus to use. More on this to come!