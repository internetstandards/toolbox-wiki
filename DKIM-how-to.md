# Introduction
This how to is created by the Dutch Internet Standards Platform (the organization behind [internet.nl](https://internet.nl)) and is meant to provide practical information and guidance on implementing DKIM.

# What is DKIM?
DKIM stands for **D**omain**K**eys **I**dentified **M**ail and is described in RFC 6376](https://tools.ietf.org/html/rfc6376) with updates in [RFC 8301](https://tools.ietf.org/html/rfc8301) and {RFC 8463](https://tools.ietf.org/html/rfc8463). It is meant to provide the owner of a domain with the means to claim that a message has actually been send by the domain's e-mail server and should therefore be considered legitimate. It works by signing every individual e-mail message with a specific key (private key), so that the receiving party can use a corresponding key (public key) published in the sending domain's DNS record to validate the e-mail authenticity and to check whether the e-mail has not been tampered with. 

# Why use DKIM?
A common used technique used by spammers is to trick the receiving party into believing an e-mail is legitimate by using a forged sender address. This is also known as e-mail spoofing. DKIM has been designed to detect the use of spoofing. If an incoming e-mail does not have a DKIM signature or when it's DKIM signature does not validate, the receiving e-mail server should consider the e-mail to be SPAM.

# Tips, tricks and notices for implementation
* parked domain
* minimum key length

# Outbound e-mail traffic
DNS record

Signing in Postfix

## Implementing DKIM in Postfix with SpamAssassin
**Specifics for this setup**
* Linux Debian 9.8 (Stretch) 
* SpamAssassin version 3.4.2 (running on Perl version 5.28.1)
* Postfix 3.4.5
* BIND 9.10.3-P4-Debian
* OpenDKIM v2.11.0