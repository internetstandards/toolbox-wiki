# Introduction
This how-to is created by the Dutch Internet Standards Platform (the organization behind [internet.nl](https://internet.nl)) and is meant to provide practical information and guidance on implementing DKIM.

# What is DKIM?
DKIM stands for **D**omain**K**eys **I**dentified **M**ail and is described in [RFC 6376](https://tools.ietf.org/html/rfc6376) with updates in [RFC 8301](https://tools.ietf.org/html/rfc8301) and [RFC 8463](https://tools.ietf.org/html/rfc8463). It is meant to provide the owner of a domain with the means to claim that a message has actually been send by the domain's e-mail server and should therefore be considered legitimate. It works by signing every individual e-mail message with a specific key (private key), so that the receiving party can use a corresponding key (public key) published in the sending domain's DNS record to validate the e-mail authenticity and to check whether the e-mail has not been tampered with. 

# Why use DKIM?
A common used technique used by spammers is to trick the receiving party into believing an e-mail is legitimate by using a forged sender address. This is also known as e-mail spoofing. DKIM has been designed to protect against spoofing. If an incoming e-mail does not have a DKIM signature or when it's DKIM signature does not validate, the receiving e-mail server should consider the e-mail to be SPAM.

# Tips, tricks and notices for implementation
* Use a DKIM key (RSA) of [at least 1024 bits](https://tools.ietf.org/html/rfc6376#section-3.3.3) to minimize the successrate of offline attacks. Don't go beyond a key size of 2048 bits since this is not mandatory according to the RFC.
* Make sure you to change your DKIM keys regularly. A rotation scheme of 6 months is recommended. 
* Parked domains should be explicitly configured to not use e-mail. For DKIM this is done with an empty policy: "v=DKIM1; p=".

# Implementing DKIM with OpenDKIM for Postfix with SpamAssassin
**Specifics for this setup**
* Linux Debian 9.8 (Stretch) 
* SpamAssassin version 3.4.2 (running on Perl version 5.28.1)
* Postfix 3.4.5
* BIND 9.10.3-P4-Debian
* OpenDKIM v2.11.0

**Assumptions**
* DNSSEC is used
* Mail server is operational
* Software packages are already installed

## Outbound e-mail traffic
DKIM for outbound e-mail traffic can be accomplished by publishing a DKIM policy as a TXT record in a domain name's DNS zone, and by configuring the e-mail server to sign outbound e-mails.

### Set up OpenDKIM and created key pair for your domain
Make sure the file ***/etc/opendkim.conf** has a least the following configuration options.

    UMask                   002
    Canonicalization        relaxed/simple
    Mode                    sv
    AutoRestart             Yes
    AutoRestartRate         10/1h
    ExternalIgnoreList      refile:/etc/opendkim/trusted_hosts
    InternalHosts           refile:/etc/opendkim/trusted_hosts
    KeyTable                refile:/etc/opendkim/key_table
    SigningTable            refile:/etc/opendkim/signing_table
    PidFile                 /var/run/opendkim/opendkim.pid
    SignatureAlgorithm      rsa-sha256
    UserID                  opendkim:opendkim
    Socket                  inet:12301@localhost

Create the the file **/etc/opendkim/trusted_hosts** and make sure it contains the following:

    127.0.0.1
    localhost
    example.nl
    mail.example.nl

Now create the directory **/etc/opendkim/keys/example.nl** and execute the following command with this directory and make sure to replace 'YYYYMM' with the number of the current year and month. For example: "selector201906". This makes it easier to determine the age of a specific key in a later stage. 

`opendkim-genkey -s selectorYYYYMM -d example.nl`

There are now 2 files in **/etc/opendkim/keys/example.nl** (the key pair): 
* selector201906.txt: this file contains DNS complete DKIM DNS record including the public key.
> selector201906._domainkey IN TXT "v=DKIM1; k=rsa; p=MIGfMA0GCSqGSIb3DQEBAQUAA4GNADCBiQKBgQCooJQftNOg3wOqVW5wOpr1PhhzgeP1IE9dTOtpUOCENP+z1HwP+8fFp9aGo/EKHoDQRhDUxXlVfocmRjb0lyjHD5ax16BBKLAd8+AgHZt1er8fmm2cL+7nurv0vU5YBG9LGUklD9qO/zJrIz+Lp+YO7D2rt0qYAgGzUOLJBWLBNQIDAQAB"  ; ----- DKIM key selector201906 for example.nl

* selector201906.private: this file contains the private key which is going to be used by Postfix to sign all outbound e-mails.

> -----BEGIN RSA PRIVATE KEY-----  
> MIICXAIBAAKBgQCooJQftNOg3wOqVW5wOpr1PhhzgeP1IE9dTOtpUOCENP+z1HwP  
> +8fFp9aGo/EKHoDQRhDUxXlVfocmRjb0lyjHD5ax16BBKLAd8+AgHZt1er8fmm2c  
> L+7nurv0vU5YBG9LGUklD9qO/zJrIz+Lp+YO7D2rt0qYAgGzUOLJBWLBNQIDAQAB  
> AoGASy+V+/Efbxogw0DmRgoLb4+pTU87+d7XJC2YxVN3V9tdq6vxSRslPr8QCuZs  
> Ievp2XN0K7qE2BbbYbhq5nHDjwzPJ7vCZzN3JI8eOC9gKP++Te6AAcDjP+G3LND4  
> Np2AWsn6JwGeM0QYI5Ehrxrw5HlqNb620N6wOEyd/7s4Px0CQQDVT3LhDzUOkbAW  
> J/jUHdV4WYozRcBGFFJvH85ASJAbK9OSrF3tfJZj9e78xP4Z5EZ8jp9iKgajt5zl  
> fYtAYjZfAkEAyl/gascX17nxO3rH/8hr8dPS6hY0KYTKXCZfuvYaSG7AZ3oaSQSc  
> mz3Rz67cm14DZNc01aBE7PwiRjq9TsQo6wJAQkppijXeqENwdMJBWzJWWAuDnoGL  
> ynugTraUs3eZiUgqfUeh/R8d4bzZY6aYzVUa7rSoJaqn25NBaDSG5SBggwJBAKrp  
> VepXwjcafjSxeP74ENHnBxVTMzJtR0mTzv1iosfRYQUDBffswSYKi4tOLlm4iD09  
> 0w0nkY5jUb7mFMLUv4kCQDjgGWNO8AeAohYF47fmYXeMMvS29rKtdLiR7D41WtOo  
> +zM7YTQa9kGRihEK+iT8v1x7ZX3mt0WZ5eoupeGauio=  
> -----END RSA PRIVATE KEY-----

Now make sure that the private key can only be read by the user opendkim by executing the following command:

`chown opendkim:opendkim selector201906.private`

The next step is to create the key table file **/etc/opendkim/key_table**. This file will tell opendkim about the domains that have been configured and where to find their keys. Add the following to configure example.nl:

> selector201906._domainkey.example.nl example.nl:selector201906:/etc/opendkim/keys/example.nl/selector201906.private

Create the file **/etc/opendkim/signing_table** and add the following line:

> *@example.nl selector201906._domainkey.example.nl

This concludes the configuration of OpenDKIM. Start OpenDKIM and check your logfiles for possible errors.

### Publish the DNS record

Make sure to add the following lines to you domain's zone file:
> selector201906._domainkey IN TXT "v=DKIM1; k=rsa; p=MIGfMA0GCSqGSIb3DQEBAQUAA4GNADCBiQKBgQCooJQftNOg3wOqVW5wOpr1PhhzgeP1IE9dTOtpUOCENP+z1HwP+8fFp9aGo/EKHoDQRhDUxXlVfocmRjb0lyjHD5ax16BBKLAd8+AgHZt1er8fmm2cL+7nurv0vU5YBG9LGUklD9qO/zJrIz+Lp+YO7D2rt0qYAgGzUOLJBWLBNQIDAQAB"
> _adsp._domainkey IN TXT "dkim=all"

The first line publishes the selector and the associated public key. The second line tells receiving mail server that all e-mail coming from the domain example.nl are DKIM signed.

### Configure Postfix
The final step is to configure Postfix to actually sign outbound e-mail using OpenDKIM. In order to do this add the following to **/etc/postfix/main.cf**:

    milter_protocol = 6
    milter_default_action = accept
    smtpd_milters = inet:localhost:12301
    non_smtpd_milters = inet:localhost:12301

When you are ready to start using DKIM restart Postfix, but make sure you waited long enough for the DKIM DNS record to succesfully propagate.

## Inbound e-mail

### Configuring SpamAssassin
SpamAssassin uses a scoring mechanism in order to determine if an e-mail should be considered spam. By default SpamAssassin considers an e-mail to be spam if the score at least "5". An e-mail starts with a score of 0 and points are added based on the [tests](https://spamassassin.apache.org/old/tests_3_3_x.html) performed. The tests performed can be configured by adding specific [configuration parameters](https://spamassassin.apache.org/full/3.4.x/doc/Mail_SpamAssassin_Conf.html) in **/etc/spamassassin/local.cf**.

Now here's the tricky part. The points added to the score of an incoming e-mail based on the results of a specific test, is at its core a custom job. Many variables can be taken into consideration when scoring an e-mail (which is considered the strength of a post-SMTP spam filter) and the detailed scoring depends on a domain owner's specific wishes. For the sake of this how-to, the DKIM scoring will be based on the assumption that the domain owner wants to consider an e-mail to be spam if the sending e-mail server's IP-address or host is not in the domain's SPF record. 

With SpamAssassin this can be configured by adding the following scoring configuration parameters to **/etc/spamassassin/local.cf**:

```
score DKIM_ADSP_ALL 5.0
# No valid author signature, domain signs all mail

score DKIM_ADSP_DISCARD 5.0
# No valid author signature, domain signs all mail and suggests discarding the rest 

score DKIM_ADSP_NXDOMAIN 5.0
# No valid author signature and domain not in DNS 
```

This means that incoming e-mail is instantly classificied as spam if there is not a valid DKIM signature in the mail header and:
* the sending domain's DKIM ADSP record states that all e-mail should be signed and all unsigned mails should be discarded (DISCARD).
* the sending domain's DKIM ADSP record states that all e-mail should be signed (ALL). 
* the sending domain used in the "From"-header (a.k.a. RFC5322.From, Header From, Message From) does not exist. 
