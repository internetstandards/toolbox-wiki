# Introduction
This howto is created by Platform Internetstandards (the organization behind [internet.nl](https://internet.nl)) and is meant to provide practical information on how to implement DANE for SMTP.  

# What is DANE?
DANE is short for "**D**NS-based **A**uthentication of **N**amed **E**ntities" and is described in [RFC 6698](https://tools.ietf.org/html/rfc6698). It offers the option to use the DNSSEC infrastructure to store and sign keys and certificates that are used by TLS. So basically DANE can be used to validate provided certificates based on information (a TLSA signature) published in the DNS zone of a specific domain. 

DANE can be used in a web context or in an e-mail context. DANE for web is unfortunately not supported by most browsers and therefore has little added value. DANE for SMTP (which is described in [RFC 7672](https://tools.ietf.org/html/rfc7672) on the other hand, is used increasingly and addresses a couple of shortcomings in SMTP using STARTTLS. DANE for SMTP uses the presence of DANE TLSA records to securely signal TLS support and to publish the means by which SMTP clients can successfully authenticate legitimate SMTP servers. This becomes "opportunistic DANE TLS" (which is better than regular "opportunistic TLS") and creates resistance to downgrade and man-in-the-middle (MITM) attacks.

# Why use DANE for SMTP?
The use of opportunistic TLS (via STARTTLS) is not without risks:
* Because forcing the use of TLS for all mail servers would break backwards compatibility, SMTP uses opportunistic TLS (via STARTTLS) as a mechanism to enable secure transport between mail servers. However, the fact that STARTTLS is opportunistic means that the initial connection from one mail server to another always starts unencrypted making it vulnerable to man in the middle attacks. If a mail server does not offer the 'STARTTLS capability' during the SMTP handshake (because it was stripped by an attacker), transport of mail occurs over an unencrypted connection. 
* By default mail servers do not validate the authenticity of another mail server's certificate; any random certificate is accepted (see [RFC 2487](https://tools.ietf.org/html/rfc2487). This was probably done because there is no user who can act on errors in case they occur. Unfortunately this default behavior introduces the risk of a man in the middle attack; an attacker can offer a false certificate enabling the attacker to decrypt encrypted traffic.

DANE addresses these shortcomings because:
* Sending mail servers can deduce a receiving mail server's ability to use TLS, by the presence of a TLSA record. This means that a connection does not have to start unencrypted (awaiting the STARTTLS capability) but can be encrypted from the start.
* TLSA records can be used to validate the certificate provided by the receiving mail server. This implies that the administrator of a domain 'guarantees' that the TLSA record is always correct and can be used to validate the certificate. Because of this guarantee the sending mail server does not have to fallback to unencrypted mail transport when the offered certificate does not match a single TLSA record. If this is the case the sending mail server can abort the transport and not send the e-mail.

# Guaranteeing a valid TLSA record
Because it is important that there is always a valid TLSA record to make sure mail transport can occur, DANE offers a roll-over mechanism. A roll-over is useful when certificates expire and need to be replaced. Since distributing information via DNS can be a bit slow (depending on the TTL settings), it's important to anticipate a certificate change from a DANE perspective. This can be done by applying a roll-over scheme of which there are two:
* current + next. 
* current + issuer

# Implementing DANE for SMTP on Debian Stretch
**Specifics for this setup**
* Linux Debian 9.8 (Stretch) 
* SpamAssassin version 3.4.2 (running on Perl version 5.28.1)
* Postfix 3.4.5
* BIND 9.10.3-P4-Debian
* Two certificates (for two mail servers) from Comodo / Sectigo

**Assumptions**
* DNSSEC is used
* Mail server is operational

## Outbound e-mail traffic
This part of the how to describes the steps that should be taken with regard to your outbound e-mail traffic. This enables other parties to use DANE for validating the certificates offered by your e-mail servers. 

### Generating DANE records
**Primary mail server (mail1.example.com)**

Generate the DANE SHA-256 hash with `openssl x509 -in /path/to/primary-mailserver.crt -noout -pubkey | openssl pkey -pubin -outform DER | openssl sha256`. This command results in the following output. 
> (stdin)= 29c8601cb562d00aa7190003b5c17e61a93dcbed3f61fd2f86bd35fbb461d084

**Secondary mail server (mail2.example.com)**

For the secondary mail server we generate the DANE SHA-256 hash using 
`openssl x509 -in /path/to/secondary-mailserver.crt -noout -pubkey | openssl pkey -pubin -outform DER | openssl sha256`. This command results in the following output. 
> (stdin)= 22c635348256dc53a2ba6efe56abfbe2f0ae70be2238a53472fef5064d9cf437

### Publishing DANE records
Now that we have the SHA-256 hashes, we can construct the DNS records. We make the following configuration choices:
* Usage field is "**3**"; we generated a DANE hash of the leaf certificate itself (DANE-EE: Domain Issued Certificate).
* Selector field is "**1**"; we used the certificates' public key to generate DANE hash/signature.
* Matching-type field is "**1**"; we use SHA-256.

With this information we can create the DNS record for DANE:

> _25._tcp.mail.example.com. IN TLSA 3 1 1 29c8601cb562d00aa7190003b5c17e61a93dcbed3f61fd2f86bd35fbb461d084  
> _25._tcp.mail2.example.com. IN TLSA 3 1 1 22c635348256dc53a2ba6efe56abfbe2f0ae70be2238a53472fef5064d9cf437

### Generating DANE roll-over records
We use the provided bundle file for generating the DANE hashes belonging to the root certificate. In order to do that, we first split the bundle file into multiple certificates using `cat ca-bundle-file.crt | awk 'BEGIN {c=0;} /BEGIN CERT/{c++} { print > "bundlecert." c ".crt"}'`. In this specific case this results in two files: _bundlecert.1.crt_ and _bundlecert.2.crt_.

For each file we view the **subject** and the **issuer**. We start with the first file using `openssl x509 -in bundlecert.1.crt -noout -subject -issuer`. This results in the following output.

> subject=C = GB, ST = Greater Manchester, L = Salford, O = Sectigo Limited, CN = Sectigo RSA Domain Validation Secure Server CA  
> issuer=C = US, ST = New Jersey, L = Jersey City, O = The USERTRUST Network, CN = USERTrust RSA Certification Authority

For the second file we use `openssl x509 -in bundlecert.2.crt -noout -subject -issuer`. This results in the following output.
> subject=C = US, ST = New Jersey, L = Jersey City, O = The USERTRUST Network, CN = USERTrust RSA Certification Authority  
> issuer=C = US, ST = New Jersey, L = Jersey City, O = The USERTRUST Network, CN = USERTrust RSA Certification Authority

Based on the information of these two certificates, we can conclude that the second certificate (bundlecert.2.crt) is the root certificate; since root certificates are self-signed the **subject** and the **issuer** are the same. The other certificate (bundlecert.1.crt) is an intermediate certificate which is (in this case) signed by the root certificate. 

### Publishing DANE roll-over records
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

## Implementing DANE for inbound e-mail traffic

### Configuring Postfix
Postfix plays an important role in using DANE for validating the when available.

Make sure the following entries are present in **/etc/postfix/main.cf**
> smtp_dns_support_level = dnssec  

This setting tells Postfix to perform DNS lookups using DNSSEC. This is an important prerequisite for DANE to be effective, since regular DNS lookups can be manipulated. Without DNSSEC support, Postfix cannot use DANE.

> smtp_tls_security_level = dane  

By default Postfix uses opportunistic TLS (smtp_tls_security_level = may) which is susceptible to man in the middle attacks. You could tell Postfix to use mandatory TLS (smtp_tls_security_level = encrypt) but this breaks backwards compatibility with mail servers that don't support TLS (and only work with plaintext delivery). However, when Postfix is configured to use the "dane" security level (smtp_tls_security_level = dane) it becomes resistant to man in the middle attacks, since Postfix will connect to other mail servers using "mandatory TLS" when TLSA records are found. If TLSA records are found but are unusable, Postfix won't fallback to plaintext or unauthenticated delivery. 

> smtp_host_lookup = dns  

This tells Postfix to perform lookups using DNS. Although this is default behavior it is important to make sure this is configured, since DANE won't be enabled if lookups are performed using a different mechanism.

> smtpd_tls_CAfile = /path/to/ca-bundle-file.crt  

When applying a DANE roll-over scheme using an "issuer certificate" (an intermediate or root certificate), Postfix must be able to provide the certificates of the used issuer in the chain of trust. Hence this setting.