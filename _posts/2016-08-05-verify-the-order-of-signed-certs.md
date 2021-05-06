---
layout: post
title: "Verify the Order of Signed Certificates for BOSH and UAA"
date: 2016-08-05
---

![map](https://raw.githubusercontent.com/cweibel/ghost_blog_pics/master/steve-smith-nY8_HU_DTaI-unsplash.jpg)



Photo by [Steve Smith](https://unsplash.com/@varrak?utm_source=unsplash&utm_medium=referral&utm_content=creditCopyText) on [Unsplash](https://unsplash.com/s/photos/uri?utm_source=unsplash&utm_medium=referral&utm_content=creditCopyText)


In a previous article [https://www.starkandwayne.com/blog/bosh-uaa-with-signed-certificates/](https://www.starkandwayne.com/blog/bosh-uaa-with-signed-certificates/) we discovered how to add a multiple/intermediate level signed certificates to UAA on BOSH. Recently I discovered one of my deployments had the certs in the wrong order and a kind gentleman named Thilak showed me how to verify the order of certificates is correct. While the `bosh_cli` didn't complain about the order other tools might so it's good to get them in the right order. We should always strive to have nice things!

Start by running the `openssl` and use the director url and port as seen below:

```
me@s1:~$ openssl s_client -showcerts -connect bosh1.starkandwayne.com:25555
```

Now look at the output. In the example below there are 4 levels of certificates labeled 0 through 3. 

**The certificate issued at a level should be signed from the previous level.**

![pic](https://github.com/cweibel/ghost_blog_pics/blob/master/verify_cert_order.jpg?raw=true)


`s:` is the subject line of the certificate and `i:` contains information about the issuing CA.

The ideal end result of a good openssl bingo: `Verify return code: 0 (ok)`















