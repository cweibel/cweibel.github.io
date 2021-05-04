---
layout: post
title: "Genesis CF Kit + LDAP Example"
date: 2019-11-18
---

![map](https://raw.githubusercontent.com/cweibel/ghost_blog_pics/master/christopher-paul-high-5pyd9392uNU-unsplash.jpg)

Photo by [Christopher Paul High](https://unsplash.com/@victormuunoz) on [Unsplash](https://unsplash.com/s/photos/uri?utm_source=unsplash&utm_medium=referral&utm_content=creditCopyText)


Below is an example of using LDAP to back UAA for the [Cloud Foundry Kit](https://github.com/genesis-community/cf-genesis-kit) in [Genesis](https://genesisproject.io/).  Comments have been left on each of the `params` to note where these values come from or to simply set-and-forget the values:

```
# UAA LDAP configuration
params:
  ldap_spring_profiles: ldap
  ldap_ssl_certificate: (( vault "secret/starkandwayne/vs/cf/ldap:certificate" ))  # AD Cert from AD Server if LDAPS is used (secure LDAP)
  ldap_host: ldaps://amer-prod.starkandwayne.io                                    # or ldap:// if non-ssl
  ldap_bind_user_Dn: "CN= _svc_CloudFoundry,OU=Service Accounts,OU=Servers & Applications,DC=amer,DC=starkandwayne,DC=io"     #Full conconical format of the user to use for query
  ldap_bind_user_passwd: (( vault "secret/starkandwayne/vs/cf/ldap:password" ))    # search user pwd
  ldap_user_search_base: "DC=amer,DC=starkandwayne,DC=io"                          # Base AD domain in conconical format
  ldap_user_search_filter: "(&(objectclass=user)(sAMAccountName={0}))"             # Leave alone
  ldap_group_search_base: "OU=Groups,DC=amer,DC=starkandwayne,DC=io"               # Base AD level where AD groups are stored in concicial format
  ldap_group_search_filter: "member={0}"                                           # Leave alone
  ldap_email_domain: "starkandwayne.io"                                            # Email domain used
  ldap_attributeMappings_given_name: givenName                                     # Leave alone
  ldap_attributeMappings_family_name: sn                                           # Leave alone
  ldap_attributeMappings_email: mail                                               # Leave alone
  login_user_prompt: "User ID"                                                     # Prompt label for user name in ui

instance_groups:
- name: uaa
  jobs:
  - name: uaa
    properties:
      login:
        prompt:
          username:
            text: (( grab params.login_user_prompt ))
        links:
          passwd:                                                                  # Removes links on gui sign page to pwd reset
          signup:                                                                  # Removes links on gui sign page to self sign up

      uaa:
        spring_profiles: (( grab params.ldap_spring_profiles ))
        ldap:
          sslCertificate: (( grab params.ldap_ssl_certificate))
          mailAttributeName: mail
          url: (( grab params.ldap_host ))
          searchBase: (( grab params.ldap_user_search_base ))
          searchFilter: (( grab params.ldap_user_search_filter ))
          enabled: true
          emailDomain:
            - (( grab params.ldap_email_domain ))
          referral: follow
          groups:
            profile_type: groups-map-to-scopes
            searchBase: (( grab params.ldap_group_search_base ))
            groupSearchFilter: (( grab params.ldap_group_search_filter ))
          userDN: (( grab params.ldap_bind_user_Dn ))
          userPassword: (( grab params.ldap_bind_user_passwd ))
          sslCertificateAlias:
        clients:
          cf:
            refresh-token-validity: 604800
        scim:
          external_groups:
            ldap:
              CN=GG_CloudFoundry_Admins,OU=Application Access Groups,OU=Groups,DC=amer,DC=starkandwayne,DC=io:  #this is the full conconical format of a AD users group to map to uaa groups - this is array style in manifest
              - cloud_controller.admin
```

If you are not using Genesis but want to use LDAP for UAA in `cf-deployment` simply swap out the `((grab blah...))` references with Credhub variables and a vars file.  As of this writing there isn't an ops file for enabling LDAP for UAA in this deployment repo so you'll need the entire populated `/instance_groups/name=uaa` ops variable.

I also found an interesting way to work in a flying penguin into a blog post. Hurray for me!