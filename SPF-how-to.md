# Introduction
This how to is created by the Dutch Internet Standards Platform (the organization behind [internet.nl](https://internet.nl)) and is meant to provide practical information and guidance on implementing SPF.  

# What is SPF?
SPF is short for "**S**ender **P**olicy **F**ramework" and is described in [RFC 7208](https://tools.ietf.org/html/rfc7208). It offers domain owners that use their domains for sending e-mail, the possibility to use the DNSSEC infrastructure to publish which hosts (mail servers) are authorized to use their domain names in the "MAIL FROM" and "HELO" identities. So basically SPF is a whitelist which lists all servers that are allowed to send e-mail on behalf of a specific domain. The receiving mail server may use the information (a SPF record) published in the DNS zone of a specific mail sending domain. 

# Why use SPF?
Our current e-mail infrastructure was originally designed for any mail sending host to use any DNS domain name it wants. The authenticity of the sending mail server cannot be deterimined, which makes it easy for random third parties to make use of a domain name with possibly a malicious intent. This increases the risk of processing e-mail since the intentions of the sender (host) are uncertain. SPF can help the fight against spam and other kinds of unwanted e-mail be offering a way of authenticating the sending mail server.  

