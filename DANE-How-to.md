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
**Primairy mailserver (mail1.example.com)**

Generate the DANE SHA-256 hash with `openssl x509 -in /path/to/primairy-mailserver.crt -noout -pubkey | openssl pkey -pubin -outform DER | openssl sha256`. This command results in the following output. 
> (stdin)= 29c8601cb562d00aa7190003b5c17e61a93dcbed3f61fd2f86bd35fbb461d084

**Secundairy mailserver (mail2.example.com)**

For the secundairy mailserver we generate the DANE SHA-256 hash using 
`openssl x509 -in /path/to/secundairy-mailserver.crt -noout -pubkey | openssl pkey -pubin -outform DER | openssl sha256`. This command results in the following output. 
> (stdin)= 22c635348256dc53a2ba6efe56abfbe2f0ae70be2238a53472fef5064d9cf437

## Publishing DANE records
Now that we have the SHA-256 hashes, we can construct the DNS records. We make the following configuration choices:
* Usage field is "**3**"; we generated a DANE hash of the leaf certificate itself (DANE-EE: Domain Issued Certificate).
* Selector field is "**1**"; we used the certificates' public key to generate DANE hash/signature.
* Matching-type field is "**1**"; we use SHA-256.

With this information we can create the DNS record for DANE:

> _25._tcp.mail.example.com. IN TLSA 3 1 1 29c8601cb562d00aa7190003b5c17e61a93dcbed3f61fd2f86bd35fbb461d084
> _25._tcp.mail2.example.com. IN TLSA 3 1 1 22c635348256dc53a2ba6efe56abfbe2f0ae70be2238a53472fef5064d9cf437

## Generating DANE roll-over records
We use the provided bundle file for generating the DANE hashes belonging to the root certificate. In order to do that, we first split the bundle file into multiple certificates using `cat ca-bundle-file.crt | awk 'BEGIN {c=0;} /BEGIN CERT/{c++} { print > "bundlecert." c ".crt"}'`. In this specific case this results in two files: _bundlecert.1.crt_ and _bundlecert.2.crt_.

For each file we view the **subject** and the **issuer**. We start with the first file using `openssl x509 -in bundlecert.1.crt -noout -subject -issuer`. This results in the following output.

> subject=C = GB, ST = Greater Manchester, L = Salford, O = Sectigo Limited, CN = Sectigo RSA Domain Validation Secure Server CA

> issuer=C = US, ST = New Jersey, L = Jersey City, O = The USERTRUST Network, CN = USERTrust RSA Certification Authority

For the second file we use `openssl x509 -in bundlecert.2.crt -noout -subject -issuer`. This results in the following output.
> subject=C = US, ST = New Jersey, L = Jersey City, O = The USERTRUST Network, CN = USERTrust RSA Certification Authority

> issuer=C = US, ST = New Jersey, L = Jersey City, O = The USERTRUST Network, CN = USERTrust RSA Certification Authority

Based on the information of these two certificates, we can conclude that the second certificate (bundlecert.2.crt) is the root certificate; since root certificates are self-signed the **subject** and the **issuer** are the same. The other certificate (bundlecert.1.crt) is an intermediate certificate which is (in this case) signed by the root certificate. 

## Publishing DANE roll-over records
Because we prefer the root certificate to be our roll-over anchor, we generate the DANE SHA-256 hash using `openssl x509 -in bundlecert.2.crt -noout -pubkey | openssl pkey -pubin -outform DER | openssl sha256`. This results in the following output.

> (stdin)= c784333d20bcd742b9fdc3236f4e509b8937070e73067e254dd3bf9c45bf4dde

Since both certificates for the primary and secondary come from the same Certificate Authority, they both have the same root certificate. So we don't have to repeat this with a different bundle file.

Now that we have the SHA-256 hash, we can construct the DANE roll-over DNS records. We make the following configuration choices:
* Usage field is "**2**"; we generated a DANE hash of the root certificate which is in the chain the chain of trust of the actual leaf certificate (DANE-TA: Trust Anchor Assertion). 
* Selector field is "**1**"; because we use the root certificate's public key to generate a DANE hash.
* Matching-type field is "**1**"; because we use SHA-256.

With this information we can create a rollover DNS record for DANE:
> _25._tcp.mail.example.com. IN TLSA 2 1 1 c784333d20bcd742b9fdc3236f4e509b8937070e73067e254dd3bf9c45bf4dde
> _25._tcp.mail2.example.com. IN TLSA 2 1 1 c784333d20bcd742b9fdc3236f4e509b8937070e73067e254dd3bf9c45bf4dde


## Configuring mailserver

# Implementing DANE for inbound e-mail traffic
