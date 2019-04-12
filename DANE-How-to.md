# Introduction

# What is DANE?

# Why use DANE?

# Implementing DANE for outbound e-mail traffic
This part of the How-to describes the steps that should be taken with regard to your outbound e-mail traffic. This enables other parties to use DANE for validating the certificate offered by your e-mail server. 

## Debian Stretch
**Specifics for this setup**
* Linux Debian 9.7 (Stretch) 
* SpamAssassin version 3.4.2 (running on Perl version 5.24.1)
* Postfix 3.1.8
* BIND 9.10.3-P4-Debian
* Two certficates (for two mailservers) were purchased from Comodo/Sectigo

**Assumptions**
* DNSSEC is used
* Mailserver is operational

## Generating DANE records
**primairy mailserver (mail1.example.com)**

Generate the DANE SHA-256 hash with `openssl x509 -in /path/to/primairy-mailserver.crt -noout -pubkey | openssl pkey -pubin -outform DER | openssl sha256`. This command results in the following output. 
> (stdin)= 29c8601cb562d00aa7190003b5c17e61a93dcbed3f61fd2f86bd35fbb461d084

**secundairy mailserver (mail2.example.com)**
For the secundairy mailserver we generate the DANE SHA-256 hash using 
`openssl x509 -in /path/to/secundairy-mailserver.crt -noout -pubkey | openssl pkey -pubin -outform DER | openssl sha256`. This command results in the following output. 
> (stdin)= 22c635348256dc53a2ba6efe56abfbe2f0ae70be2238a53472fef5064d9cf437

## Publishing DANE records
Configuration options
* Selector field is "1" because we used the certificates' public key to generate DANE hash/signature
* Usage is "3". In this case we generated a DANE hash of the leaf certificate itself. Therefore we use usage field "3" (DANE-EE: Domain Issued Certificate) 
* Matching-type is "1" because we use SHA-256.

With this information we can create a DNS record for DANE:

`_25._tcp.mail.example.com. IN TLSA 3 1 1 29c8601cb562d00aa7190003b5c17e61a93dcbed3f61fd2f86bd35fbb461d084`
`_25._tcp.mail2.example.com. IN TLSA 3 1 1 22c635348256dc53a2ba6efe56abfbe2f0ae70be2238a53472fef5064d9cf437`

## Generating DANE roll-over records
First we split the bundle file into multiple certificates. 
`cat ca-bundle-file.crt | awk 'BEGIN {c=0;} /BEGIN CERT/{c++} { print > "bundlecert." c ".crt"}'`
In this case this results in two files: _bundlecert.1.crt_ and _bundlecert.2.crt_.

For each file we view the subject and the issuer.
Command
`openssl x509 -in bundlecert.1.crt -noout -subject -issuer`

Output
`subject=C = GB, ST = Greater Manchester, L = Salford, O = Sectigo Limited, CN = Sectigo RSA Domain Validation Secure Server CA`
`issuer=C = US, ST = New Jersey, L = Jersey City, O = The USERTRUST Network, CN = USERTrust RSA Certification Authority`

Command
openssl x509 -in bundlecert.2.crt -noout -subject -issuer
Output
subject=C = US, ST = New Jersey, L = Jersey City, O = The USERTRUST Network, CN = USERTrust RSA Certification Authority
issuer=C = US, ST = New Jersey, L = Jersey City, O = The USERTRUST Network, CN = USERTrust RSA Certification Authority

The second certificate (bundlecert.2.crt) is the root certficate as the subject and the issuer are the same. Root certificates are self-signed, while intermediate certificates are signed by another certificate (being a root certificate, of another intermediate certificate).

## Publishing DANE roll-over records
In this case we select the root certificate as a roll-over anchor. 

Command
`openssl x509 -in bundlecert.2.crt -noout -pubkey | openssl pkey -pubin -outform DER | openssl sha256`

Output
`(stdin)= c784333d20bcd742b9fdc3236f4e509b8937070e73067e254dd3bf9c45bf4dde`

Configuration options
* Selector field is "1" because we use the certificate public key to generate DANE hash/signature
* Usage is "2". In this case I generated a DANE hash of a certificate in the chain the chain of trust, instead of the certificate itself. Therefore we use usage field "2" (DANE-TA: Trust Anchor Assertion) 
* Matching-type is "1" because I use SHA-256.

With this information we can create a rollover DNS record for DANE:
`_25._tcp.mail.traxotic.net. IN TLSA 2 1 1 c784333d20bcd742b9fdc3236f4e509b8937070e73067e254dd3bf9c45bf4dde`
`_25._tcp.mail2.traxotic.net. IN TLSA 2 1 1 c784333d20bcd742b9fdc3236f4e509b8937070e73067e254dd3bf9c45bf4dde`


## Configuring mailserver

# Implementing DANE for inbound e-mail traffic
