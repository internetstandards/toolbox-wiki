# Table of contents

# Introduction
This how-to is created by the Dutch Internet Standards Platform (the organization behind [internet.nl](https://internet.nl)) and is meant to provide practical information and guidance on implementing DMARC.

# What is DMARC?
DMARC is short for **D**omain based **M**essage **A**uthentication, **R**eporting and **C**onformance and is described in [RFC 7489](https://tools.ietf.org/html/rfc7489). With DMARC the owner of a domain can, by means of a DNS record, publish a  policy that states how to handle e-mail which is not properly authenticated using SPF and/or DKIM. At the same time it provides the means for receiving reports which allows a domain's administrator to detect whether their domainname is used for phishing or spam.
 
# Why use DMARC?
to-do.

# Tips, tricks and notices for implementation
* Interoperabily issues: https://tools.ietf.org/html/rfc7960
* DMARC does not require both DKIM or SPF. 
* 
