- [Introduction](#introduction)
- [What is a parked domain?](#what-is-a-parked-domain-)
- [Domain without e-mail](#domain-without-e-mail)
  * [Null MX](#null-mx)
  * [DMARC](#dmarc)
  * [DKIM](#dkim)
  * [SPF](#spf)
- [Domain without a website](#domain-without-a-website)

<small><i><a href='http://ecotrust-canada.github.io/markdown-toc/'>Table of contents generated with markdown-toc</a></i></small>

# Introduction
This how-to is created by the Dutch Internet Standards Platform (the organization behind [internet.nl](https://internet.nl)) and is meant to provide practical information and guidance on explicitly configuring a parked domain not to use e-mail.

# What is a parked domain?
[Domain parking](https://en.wikipedia.org/wiki/Domain_parking) is the registration of an Internet domain name without that domain being associated with any services such as e-mail or a website. 

# Domain without e-mail
If a domain is not using e-mail it is recommended to use the following settings.

## Null MX
Explicitly configure an 'empty' MX record according to [RFC7505 ](https://tools.ietf.org/html/rfc7505). 

`example.nl IN MX 0 .`

## DMARC
Set DMARC policy to reject mails, but allow reporting to take place. This helps detecting activity related to your domain.

`_dmarc IN TXT "v=DMARC1; p=reject; rua=mailto:rua@example.nl; ruf=mailto:ruf@example.nl`

## DKIM
When used with a wildcard selector, setting an empty public key indicates that all previously used keys are revoked and must be considered unreliable. The owner of a domain can also use this to explicitly signal that a domain is not configured to use e-mail. See our [DKIM how-to](https://github.com/internetstandards/toolbox-wiki/blob/master/DKIM-how-to.md) for more information. 

`*._domainkey IN TXT "v=DKIM1; p="`

## SPF
Set an an empty policy (not mentioning any ip-adresses or hostnames which are allowed to send mail) and a hard fail.

`example.nl IN TXT "v=spf1 –all"`
 
# Domain without a website
Apply the following settings to domains not using a website.

* Don't use an A or AAAA record for parked domains.
* Don't redirect from a parked domain to the used domain, since this encourages users to keep using the parked domain name. If a redirect is desirable, make sure to use the proper redirect order in order for HSTS headers to remain effective: 
    1. redirect from HTTP to HTTPS on the same (sub)domain.
    2. when using HTTPS, redirect to another (sub)domain.
