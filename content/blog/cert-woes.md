+++
title = "Cert Woes"
date = "2023-05-04T19:48:04-07:00"

#
# description is optional
#
# description = "An optional description for SEO. If not provided, an automatically created summary will be used."

tags = ["infra","tls","woes","rants"]
+++

I was bitten by TLS woes today. At work we use AWS ACM. For the most part, it's pretty much ok, except it has a few quirks.
However, I was bitten by what I believe to be a standardized format.

If you want some fun reading, check out the lenghty docs they have on the constraints for requesting a cert [here](https://docs.aws.amazon.com/acm/latest/APIReference/API_RequestCertificate.html).

TLS Certificates have a primary domain name that they can be configured with, which ends up being used as the CN (common name).
This is in the format of a FQDN (fully-qualified domain name).

The regex for this thing is pretty gnarly, and AWS specifies the length constraints as `1<=x>=253`.

```
^(\*\.)?(((?!-)[A-Za-z0-9-]{0,62}[A-Za-z0-9])\.)+((?!-)[A-Za-z0-9-]{1,62}[A-Za-z0-9])$
```

However, after squinting at the (below) paragraph, the actual length of the primary domain is 64 octets to remain in compliance with [RFC5280](https://datatracker.ietf.org/doc/html/rfc5280)!


> In compliance with RFC 5280, the length of the domain name (technically, the Common Name) that you provide cannot exceed 64 octets (characters), including periods. To add a longer domain name, specify it in the Subject Alternative Name field, which supports names up to 253 octets in length.

If the desire is to have a longer FQDN, that must be specified as the subject alternative name, which _finally_ has a FQDN length of 253 :).

It gets worse though, because each octet (the bits between each period. Commonly referred to as a subdomain) has a limitation of 63 characters, which combined (including the periods) must total be less than 253.

So what happens if the FQDN that you want to use ends up longer than what the CN allows? Well, now you're stuck figuring out some bogus common name that you need to come up with just so that you can stuff the _real_ FQDN that you want to use into the subject alt names list.

Why is this even an issue?

Well, building systems that automate cert generation based on user input generally wind you up with this problem.

For me, this manifests when allowing users to specify certain parts of the subdomain for their own use.


Oh btw, if you're generating a wildcard cert, that is _not_ included in the length limitations! So the url that is generated can be 64 characters, _plus_ `*.`!

Whew.