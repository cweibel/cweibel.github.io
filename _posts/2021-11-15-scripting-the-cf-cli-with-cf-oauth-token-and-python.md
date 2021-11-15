---
layout: post
title: "Scripting the CF API with `cf oauth-token` and Python"
date: 2021-11-15
---

## The Rodney Dangerfield of the CF CLI, show it some respect!

![pic](https://raw.githubusercontent.com/cweibel/ghost_blog_pics/master/mk-s-WHf1wtNMMLU-unsplash-2.jpg)

Photo by [mk. s](https://unsplash.com/@mk__s?utm_source=unsplash&utm_medium=referral&utm_content=creditCopyText) on [Unsplash](https://unsplash.com)


I was assigned a task to take a look at all the environment variables being used for a foundation.  Normally I would put together a quick query against the Cloud Controller database but found this data to be encrypted.  However, you can pull the information from the CF CLI but you need to know how to login and loop through the results.  A bit of Python, a dancing gopher and proper course etiquette are all you need.

Grab your putter and head to the first hole!

## Hole 1 - Login

Quick and easy, use the `cf` CLI to log into your desired CF foundation:

```
cf login

# or 

cf login --sso
```

After successful login, the token and other information will stored in `~/.cf/config.json`

## Hole 2 - Create Python Script


The script below will use the token from the last step which is stored in `~/.cf/config.json`

```
#!/usr/bin/env python3
import requests
from requests.structures import CaseInsensitiveDict

import sys
import warnings


# Disable SSL Warnings - important for KubeCF
if not sys.warnoptions:
    warnings.simplefilter("ignore")

# Login
system_domain = sys.argv[1]
token = sys.argv[2]

headers = CaseInsensitiveDict()
headers["Accept"] = "application/json"
headers["Authorization"] = token

apps_url = "https://api." + system_domain + "/v3/apps/?per_page=100"

entries = requests.get(apps_url, headers=headers, verify=False).json()

total_results = entries["pagination"]["total_results"]
total_pages = entries["pagination"]["total_pages"]
current_page = 1
apps = {}

while True:
    print("Processing page " + str(current_page) + "/" + str(total_pages))

    for entry in entries["resources"]:

        env_vars_url = entry["links"]["environment_variables"]["href"]
        env_vars = requests.get(env_vars_url, headers=headers, verify=False).json()

        line_label = "app:" + entry["name"]+",env"
        apps[line_label] = []

        for key, value in env_vars["var"].items():
            apps[line_label].append(str(key) + "=" + str(value))


    current_page += 1

    if entries["pagination"]["next"] is None:
        break

    entries = requests.get(entries["pagination"]["next"]["href"], headers=headers, verify=False).json()


for key, value in apps.items():
    print(str(key) + ":" + str(value))
```

Save this file and call it `scrape.py`

## Hole 3 - Run the Script

This is where the magic of the `cf oauth-token` happens.  After you've logged in successfully, the bearer token is stored locally and can be retrieved on demand and is also refreshed as needed.  Cool, huh?  More details on the command can be found at [https://cli.cloudfoundry.org/en-US/v6/oauth-token.html](https://cli.cloudfoundry.org/en-US/v6/oauth-token.html)

If you clicked on that link and was underwhelmed, welcome to the club and can be excused for having never heard of the command before.  Keep going, you'll see how valuable it truly is.


Assuming you named the script `scrape.py`, you can now run it against your foundation, the first parameter is the system domain of your foundation and the second parameter is the command to retrieve the bearer token:

```
python3 scrape.py system_url.nip.io "$(cf oauth-token)"
```

The script will loop through the visible apps and pull back the list of environment variables.  Users with `cloud_controller.admin` or similar permissions will loop through all orgs and spaces, otherwise it will loop through just the orgs and spaces you have permissions to.

The output will look similar to:

```
Processing page 1/1
app:my-test-app,env:['MYENV=test']
```

If you'd like an example of also scraping org, space and alternate login combinations I've created [this example script](https://gist.github.com/cweibel/d9576c7f83011e1f73faba641cc0f900).  The notes at the top include instructions if you're attempting to run this against a locally deployed KubeCF.





Enjoy!  Remember to return your putter and score card back at the front desk.
