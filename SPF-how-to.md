<img align="right" src="images/logo-internetnl-en.svg">

# SPF how-to
This how-to is created by the Dutch Internet Standards Platform (the organization behind [internet.nl](https://internet.nl)) and is meant to provide practical information and guidance on implementing SPF.  

# Table of contents
- [What is SPF?](#what-is-spf-)
- [Why use SPF?](#why-use-spf-)
- [Tips, tricks and notices for implementation](#tips--tricks-and-notices-for-implementation)
- [Outbound e-mail traffic (DNS records)](#outbound-e-mail-traffic--dns-records-)
- [Inbound e-mail traffic](#inbound-e-mail-traffic)
  * [Implementing SPF in Postfix with SpamAssassin](#implementing-spf-in-postfix-with-spamassassin)
    + [Configuring Postfix](#configuring-postfix)
      - [Postfix configuration for Python SPF policy agent](#postfix-configuration-for-python-spf-policy-agent)
      - [Postfix configuration for SpamAssassin](#postfix-configuration-for-spamassassin)
    + [Configuring Python SPF policy agent](#configuring-python-spf-policy-agent)
    + [Configuring SpamAssassin](#configuring-spamassassin)

<small><i><a href='http://ecotrust-canada.github.io/markdown-toc/'>Table of contents generated with markdown-toc</a></i></small>

# What is SPF?
SPF is short for "**S**ender **P**olicy **F**ramework" and is described in [RFC 7208](https://tools.ietf.org/html/rfc7208). It offers domain owners that use their domains for sending e-mail, the possibility to use the DNSSEC infrastructure to publish which hosts (mail servers) are authorized to use their domain names in the "MAIL FROM" and "HELO" identities. So basically SPF is a whitelist which lists all servers that are allowed to send e-mail on behalf of a specific domain. The receiving mail server may use the information (a SPF record) published in the DNS zone of a specific mail sending domain. 

# Why use SPF?
Our current e-mail infrastructure was originally designed for any mail sending host to use any DNS domain name it wants. The authenticity of the sending mail server cannot be deterimined, which makes it easy for random third parties to make use of a domain name with possibly a malicious intent. This increases the risk of processing e-mail since the intentions of the sender (host) are uncertain. SPF can help the fight against spam and other kinds of unwanted e-mail be offering a way of authenticating the sending mail server.  

# Tips, tricks and notices for implementation
* The sender address shown to the user ("RFC5322.From") is not used when authenticating. SPF uses the invisible "RFC5321.MailFrom" header. Combining SPF with DMARC removes this disadvantage. 
* E-mail forwarding is not supported, since the e-mail is often forwarded by another e-mail server.
* SPF does not work between domains that use the same e-mail server.
* Parked domains should be explicitly configured to not use e-mail. For SPF this is done with an empty policy (not mentioning any ip-adresses or hostnames which are allowed to send mail) and a hard fail: "v=spf1 –all".
* When processing incoming mail we advise to favor a DMARC policy over an SPF policy. Do not configure SPF rejection to go into effect early in handling, but take full advantage of the enhancements DMARC is offering. A message might still pass based on DKIM.
  * At the same time, be aware that some operaters still allow a hard fail (-all) to go into effect early in handling and skip DMARC operations. 
* As stated in [section 5.2 of the RFC](https://tools.ietf.org/html/rfc7208#section-5.2) the _include_ mechanism is not applicable to the _all_ mechanism within the referenced record. This means that an SPF record's default policy is not 'inherited' upon inclusion. When including one or more SPF records from other domains, a default policy (~all or -all) is still required. For fully 'inheriting' another domain's SPF record, consider using the [_redirect_ modifier](https://tools.ietf.org/html/rfc7208#section-6.1).

# Outbound e-mail traffic (DNS records)
SPF for outbound e-mail traffic is limited to publishing an SPF policy as a TXT-record in a domain name's DNS zone. This enables other parties to use SPF for validating the authenticity of e-mail servers sending e-mail on behalf of your domain name. 

The example below shows an SPF record with a **hard fail**.
> v=spf1 mx ip4:192.168.1.1/32 ip6:fd12:3456:789a:1::/64 a:mail.example.com a:mail2.example.com -all"

Although a soft fail (~all) is recommended in order to prevent false positives, a hard fail (-all) cloud decreases the load on the receiving party's e-mail infrastructure (depending on their configuration) since the e-mail can be dropped immediately when the e-mail is send by a host or IP-address that is not listed in the SPF record.

# Inbound e-mail traffic 
Ideally incoming e-mail is processed by making a **single decision** based on a collective evaluation of all relevant e-mail standards (SPF, DKIM, DMARC). Although this would be the most elegant way of processing incoming e-mail, most e-mail servers process e-mail standards in a sequential order. This should be taken into consideration when configuring your own e-mail environment; depending on a domain owner's preferences it is also possible to implement a "single decision" e-mail environment.

Thereafter, it is [stated in the DMARC RFC](https://tools.ietf.org/html/rfc7489#section-10.1) that some receiver architectures might implement SPF in advance of any DMARC operations. This means that a "-" prefix on a sender's SPF mechanism, such as "-all", could cause that rejection to go into effect early in handling, causing message rejection before any DMARC processing takes place. While operators choosing to use "-all" should be aware of this, we advise to favor a DMARC policy over an SPF policy. As also [stated in the DMARC RFC](https://tools.ietf.org/html/rfc7489#section-6.7), the final diposition of a message is always a matter of local policy. With this in mind we feel that operators should not configure SPF rejection to go into effect early in handling, and thus disregarding DMARC. At the cost of processing the entire message body, we advise to always evaluate all relevant standards before coming to a decision. If SPF fails, DKIM might still pass resulting in a satisfying DMARC evaluation. 

## Implementing SPF in Postfix with SpamAssassin
**Specifics for this setup**
* Linux Debian 9.8 (Stretch) 
* SpamAssassin version 3.4.2 (running on Perl version 5.28.1)
* Postfix 3.4.5
* BIND 9.10.3-P4-Debian
* Python SPF policy agent (postfix-policyd-spf-python) 2.0.1-1

**Assumptions**
* DNSSEC is used
* Mail server is operational
* Software packages are already installed

### Configuring Postfix
The [Postfix SMTP server](http://www.postfix.org/smtpd.8.html) seems to be processing e-mails in a sequential order by means of so-called [access restriction lists](http://www.postfix.org/SMTPD_ACCESS_README.html#lists). For each stage of the SMTP conversation Postfix can apply a specific set of restrictions. As repeatedly stated in the [main.cf man page](http://www.postfix.org/postconf.5.html) “Restrictions are applied in the order as specified; the first restriction that matches wins”. This should be taken into consideration when configuring your Postfix implementation. 

The follow table provides a schematic overview of an SMTP conversation and relates specific stages to Postfix' access restriction lists. 

| Part of the SMTP conversation | Associated Postfix access restriction list  | Comments
| ---  | --- | --- |
| Connected to 192.168.1.1 (192.168.1.1). | | |
| Escape character is '^]'. | | |
| 220 mail.example.com ESMTP Postfix | [smtpd_client_restrictions](http://www.postfix.org/postconf.5.html#smtpd_client_restrictions) | |
| HELO mail.example.com | [smtpd_helo_restrictions](http://www.postfix.org/postconf.5.html#smtpd_helo_restrictions) | |
| 250 mail.example.com | | |
| MAIL FROM:<sender@example.com> | [smtpd_sender_restrictions](http://www.postfix.org/postconf.5.html#smtpd_sender_restrictions) | Aliases: RFC5321.MailFrom, Return-Path, Envelope Sender, Envelope From <br>Used by **SPF** |
| 250 2.1.0 Ok | | |
| RCPT TO:<recipient@example.com> | [smtpd_recipient_restrictions](http://www.postfix.org/postconf.5.html#smtpd_recipient_restrictions) | |
| 250 2.1.5 Ok | | |
| DATA | [smtpd_data_restrictions](http://www.postfix.org/postconf.5.html#smtpd_data_restrictions) | | |
| 354 End data with <CR><LF>.<CR><LF> | | |  
| To:<recipient@example.com> | | header checks |
| From:<sender@example.com> | | Aliases: RFC5322.From, Header From, Message From<br>Used by DKIM |  
| Subject: Test message | | |  
| This is a test message | |  body checks |
| . | | |
| 250 2.0.0 Ok: queued as 534AB003427 | | |
| QUIT | | |
| 221 2.0.0 Bye | | |
| Connection closed by foreign host. | | |

#### Postfix configuration for Python SPF policy agent  
The implementation described in this how-to uses an external application to perform SPF checking: Python SPF policy agent (postfix-policyd-spf-python). In order for Postfix to be able to use this application, the following needs to be added to **/etc/postfix/master.cf**: 

`policy-spf unix - n n - - spawn user=nobody argv=/usr/bin/policyd-spf`

This creates the service "policy-spf" that can now be called upon by Postfix from within the access restriction lists _smtpd_client_restrictions_ and _smtpd_recipient_restrictions_. This requires the following additions to **/etc/postfix/main.cf**. Make sure to call for the policy-spf service before the "permit". 

```
smtpd_recipient_restrictions =
        check_policy_service unix:private/policy-spf,
        permit

smtpd_client_restrictions =
        check_policy_service unix:private/policy-spf,
        permit
```

Now also add the following to **/etc/postfix/main.cf**, outside of any section.

`policy-spf_time_limit = 3600s`

#### Postfix configuration for SpamAssassin
Because this implementation uses SpamAssassin for post-SMTP spam filtering, the following needs to be added to /etc/postfix/master.cf:

```
smtp inet n - - - - smtpd -o content_filter=spamassassin
spamassassin unix - n n - - pipe user=debian-spamd argv=/usr/bin/spamc -f -e /usr/sbin/sendmail -oi -f ${sender} ${recipient}
```
Finally, add the following to **/etc/postfix/main.cf** outside of any section to prevent Postfix from expecting SpamAssassin to parse multiple recipients at once (which is not supported).

`spamassassin_destination_recipient_limit = 1`

### Configuring Python SPF policy agent
The next step is to tell the Python SPF policy agent how to behave when checking SPF records. This behavior is determined by adding [configuration parameters](https://manpages.debian.org/stretch/postfix-policyd-spf-python/policyd-spf.conf.5.en.html) to **/etc/postfix-policyd-spf-python/policyd-spf.conf**. 

The default configuration of the Python SPF policy agent provides a binary "block" or "don't block" functionality. However, the implementation described in this how-to uses SpamAssassin as a post-SMTP spam filter. This means that Postfix should not reject e-mails coming from e-mail servers that are not listed in the SPF record. Instead an SPF header is appended to the e-mail. The information in the header is used by SpamAssassin to weigh whether an incoming e-mail should be considered spam. This specific setup requires the following non-default configuration parameters in **/etc/postfix-policyd-spf-python/policyd-spf.conf**:

```
HELO_reject = False
Mail_From_reject = False
```

### Configuring SpamAssassin
SpamAssassin uses a scoring mechanism in order to determine if an e-mail should be considered spam. By default SpamAssassin considers an e-mail to be spam if the score at least "5". An e-mail starts with a score of 0 and points are added based on the [tests](https://spamassassin.apache.org/old/tests_3_3_x.html) performed. The tests performed can be configured by adding specific [configuration parameters](https://spamassassin.apache.org/full/3.4.x/doc/Mail_SpamAssassin_Conf.html) in **/etc/spamassassin/local.cf**.

Now here's the tricky part. The points added to the score of an incoming e-mail based on the results of a specific test, is at its core a custom job. Many variables can be taken into consideration when scoring an e-mail (which is considered the strength of a post-SMTP spam filter) and the detailed scoring depends on a domain owner's specific wishes. For the sake of this how-to, the SPF scoring will be based on the assumption that the domain owner wants to consider an e-mail to be spam if the sending e-mail server's IP-address or host is not in the domain's SPF record. 

With SpamAssassin this can be configured by adding the following scoring configuration parameters to **/etc/spamassassin/local.cf**:

```
score SPF_FAIL 5.0
score SPF_SOFTFAIL 5.0
score SPF_HELO_FAIL 5.0
score SPF_HELO_SOFTFAIL 5.0
```

This means that either a soft fail (~all) or a hard fail (-all) policy will result in the e-mail being classified as spam if the IP-address or host is not in the domain's SPF record.
