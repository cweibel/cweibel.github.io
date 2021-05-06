---
layout: post
title: "BOSH + UAA With Signed Certificates - Part I"
date: 2016-07-24
---

![map](https://raw.githubusercontent.com/cweibel/ghost_blog_pics/master/steve-smith-V1aIVHwbO3o-unsplash.jpg)



Photo by [Steve Smith](https://unsplash.com/@varrak?utm_source=unsplash&utm_medium=referral&utm_content=creditCopyText) on [Unsplash](https://unsplash.com/s/photos/uri?utm_source=unsplash&utm_medium=referral&utm_content=creditCopyText)


Pivotal has done a great job with documenting adding UAA as the authentication and authorization for BOSH instead of relying on local BOSH accounts. This allows you to later integrate with LDAP or SAML later on.

The instructions have you generate a series of unsigned certs which works great except now you have to use the `--ca-cert` parameter and paste your `rootCA.pem` file constantly. But what if you got your hands on some signed certificates and keys? For one, you won't need to specify the `--ca-cert` parameter everywhere.


## Single Level Signed Cert

We'll assume the following:

 - You already had a microbosh deployed at `10.1.2.3`
 - You've assigned DNS to this address to `bosh1.starkandwayne.com`
 - You have a single level signed certificate file called `rootCA.pem` in a `certs/` folder
 - You have a `ssl cert` and key called `ssl.crt` and `ssl.key` in a `certs/` folder

Below are the modifications to the tutorial found at [https://bosh.io/docs/director-users-uaa.html](https://bosh.io/docs/director-users-uaa.html) if you have a signed single level root and key

### Add uaa section to the deployments manifest:

```
properties:
  uaa:
    url: "https://bosh1.starkandwayne.com:8443"
```

### Property Changes

Change Director configuration to specify how to contact the UAA server and how to verify an access token. Since UAA will be on the same server we can use the same IP as the one used for the Director.

```
properties:
  director:
    user_management:
      provider: uaa
      uaa:
        url: "https://bosh1.starkandwayne.com:8443"
```

Be sure to comment out your existing local user accounts in case it wasn't obvious from the instructions:

```
director:
  user_management:
#    local:
#        provider: local
#        local:
#          users:
#          - name:      admin
#            password:  myLocalBoshPassword
```

### Configure Certificates and Keys

The first part references the generation of self signed cert here [http://bosh.io/docs/director-certs.html](https://bosh.io/docs/director-certs.html), you do not need to run the script at the top but instead skip down to the mapping of the generated files making the following substitutions (we assume you have ssl.key, ssl.crt and rootCA.pem as a single level signed certs in a folder named `certs/`):

Update the Director deployment manifest:

 - `director.ssl.key`
    - Private key for the Director (e.g. `certs/ssl.key`)
 - `director.ssl.cert`
    - Associated certificate for the Director (e.g. `certs/ssl.crt`)
    - Include all intermediate certificates if necessary
 - `hm.director_account.ca_cert`
    - CA certificate used by the HM to verify the Director's certificate (e.g. `certs/rootCA.pem`)

If you are using the UAA for user management, additionally put certificates in these properties:

 - `uaa.sslPrivateKey`
    - Private key for the UAA (e.g. `certs/ssl.key`)
 - `uaa.sslCertificate`
    - Associated certificate for the UAA (e.g. `certs/ssl.crt`)
    - Include all intermediate certificates if necessary
 - `login.saml.serviceProviderKey`
    - Private key for the UAA (e.g. `certs/ssl.key`)
 - `login.saml.serviceProviderCertificate`
    - Associated certificate for the UAA (e.g. `certs/ssl.crt`)

That's it, continue with the rest of step 7 and all the subsequent steps. When done you should be able to log in using uaa accounts.

## Multiple/Intermediate Level Signed Certs

Let's assume you have a multiple level signed cert like the following example `rootCA.pem`:

```
-----BEGIN CERTIFICATE-----
MyTopLevelCert123MyTopLevelCert123MyTopLevelCert123MyTopLevelCe
rt123MyTopLevelCert123MyTopLevelCert123MyTopLevelCert123MyTopLe
velCert123MyTopLevelCert123MyTopLevelCert123MyTopLevelCert123My
TopLevelCert123=
-----END CERTIFICATE-----
-----BEGIN CERTIFICATE-----
MyMiddleLevelCertMyMiddleLevelCertMyMiddleLevelCertMyMiddleLevel
CertMyMiddleLevelCertMyMiddleLevelCertMyMiddleLevelCertMyMiddleL
evelCertMyMiddleLevelCertMyMiddleLevelCertMyMiddleLevelCertMyMid
dleLevelCert==
-----END CERTIFICATE-----
```

These need to then be appended anywhere you are using `certs/ssl.crt`

 - `director.ssl.cert`
    - Associated certificate for the Director (e.g. `certs/ssl.crt` + `certs/rootCA.pem`)
    - Include all intermediate certificates if necessary
 - `uaa.sslCertificate`
    - Associated certificate for the UAA (e.g. `certs/ssl.crt` + `certs/rootCA.pem`)
    - Include all intermediate certificates if necessary
 - `login.saml.serviceProviderCertificate`
    - Associated certificate for the UAA (e.g. `certs/ssl.crt` + `certs/rootCA.pem`)

This is why there is the note about "Include all intermediate certs", if you fail to do this you will wind up with an error message when you perform a bosh target similar to:

> Invalid SSL Cert. Use --ca-cert option when setting target to specify SSL certificate'


## Verify Certificate Order

Once you've deployed BOSH+UAA you can verify the order of your certificates. There is a blog post here by a wonderful author which shows you how this is done: [https://cweibel.github.io/blog/2016/08/05/verify-the-order-of-signed-certs](https://cweibel.github.io/blog/2016/08/05/verify-the-order-of-signed-certs)

## Configure Health Manager's Connections
You will also need to configure Health Manager on the director to login with client credentials instead of local BOSH logins, see this blog post for more information: [https://cweibel.github.io/blog/2016/08/09/bosh-and-uaa-with-signed-certs-part-2](https://cweibel.github.io/blog/2016/08/09/bosh-and-uaa-with-signed-certs-part-2)