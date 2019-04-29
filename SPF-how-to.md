# Introduction
This how to is created by Platform Internetstandards (the organization behind [internet.nl](https://internet.nl)) and is meant to provide practical information and guidance on implementing SPF.  

# What is SPF?
DANE is short for "**D**NS-based **A**uthentication of **N**amed **E**ntities" and is described in [RFC 6698](https://tools.ietf.org/html/rfc6698). It offers the option to use the DNSSEC infrastructure to store and sign keys and certificates that are used by TLS. So basically DANE can be used to validate provided certificates based on information (a TLSA signature) published in the DNS zone of a specific domain. 
