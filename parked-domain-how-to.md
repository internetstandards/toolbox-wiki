# Introduction
This how-to is created by the Dutch Internet Standards Platform (the organization behind [internet.nl](https://internet.nl)) and is meant to provide practical information and guidance on explicitly configuring a parked domain not to use e-mail.

# Null MX
Explicitly configure an 'empty' MX record according to [RFC7505 ](https://tools.ietf.org/html/rfc7505). 

`example.nl IN MX 0 .`

# DMARC

`DMARC p=reject`

# DKIM

`selector._domainkey IN TXT "v=DKIM1; p="`

# SPF

`example.nl IN TXT "v=spf1 –all"`
   
# Other tips and tricks
* Don't use an A or AAAA record for parked domains.
* Don't redirect from a parked domain to the used domain, since this encourages users to keep using the parked domain name.
  * If a redirect is used, make sure to use the proper redirect order in order for HSTS headers to remain effective: 
    1. redirect from HTTP to HTTPS on the same (sub)domain.
    2. when using HTTPS, redirect to another (sub)domain.
