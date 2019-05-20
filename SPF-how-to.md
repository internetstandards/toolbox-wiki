# Introduction
This how to is created by the Dutch Internet Standards Platform (the organization behind [internet.nl](https://internet.nl)) and is meant to provide practical information and guidance on implementing SPF.  

# What is SPF?
SPF is short for "**S**ender **P**olicy **F**ramework" and is described in [RFC 7208](https://tools.ietf.org/html/rfc7208). It offers domain owners that use their domains for sending e-mail, the possibility to use the DNSSEC infrastructure to publish which hosts (mail servers) are authorized to use their domain names in the "MAIL FROM" and "HELO" identities. So basically SPF is a whitelist which lists all servers that are allowed to send e-mail on behalf of a specific domain. The receiving mail server may use the information (a SPF record) published in the DNS zone of a specific mail sending domain. 

# Why use SPF?
Our current e-mail infrastructure was originally designed for any mail sending host to use any DNS domain name it wants. The authenticity of the sending mail server cannot be deterimined, which makes it easy for random third parties to make use of a domain name with possibly a malicious intent. This increases the risk of processing e-mail since the intentions of the sender (host) are uncertain. SPF can help the fight against spam and other kinds of unwanted e-mail be offering a way of authenticating the sending mail server.  

# Tips and tricks for implementation
* The sender address shown to the user ("5322.From header") is not used when authenticating. SPF uses the invisible "5321.From header". Combining SPF with DMARC removes this disadvantage. 

# DNS records (outbound e-mail traffic)
SPF for outbound e-mail traffic is limited to publishing an SPF policy as a TXT-record in a domain name's DNS zone. This enables other parties to use SPF for validating the authenticity of e-mail servers sending e-mail on behalf of your domain name. 

The example below shows an SPF record with a **hard fail**.
> v=spf1 mx ip4:192.168.1.1/32 ip6:fd12:3456:789a:1::/64 a:mail.example.com a:mail2.example.com -all"

Although a soft fail (~all) is recommended in order to prevent false positives.

# Inbound e-mail traffic 

## Implementing SPF on Debian Stretch
**Specifics for this setup**
* Linux Debian 9.8 (Stretch) 
* SpamAssassin version 3.4.2 (running on Perl version 5.28.1)
* Postfix 3.4.5
* BIND 9.10.3-P4-Debian
* postfix-policyd-spf-python 2.0.1-1

**Assumptions**
* DNSSEC is used
* Mail server is operational


