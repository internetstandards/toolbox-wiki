# Table of contents
- [Introduction](#introduction)
- [What is DMARC?](#what-is-dmarc-)
- [Why use DMARC?](#why-use-dmarc-)
- [Tips, tricks and notices for implementation](#tips--tricks-and-notices-for-implementation)
- [Overview of DMARC configuration tags](#overview-of-dmarc-configuration-tags)
- [Implementing DMARC with OpenDMARC for Postfix with SpamAssassin](#implementing-dmarc-with-opendmarc-for-postfix-with-spamassassin)
  * [Outbound e-mail traffic](#outbound-e-mail-traffic)
    + [Setting up a DMARC record](#setting-up-a-dmarc-record)
  * [Inbound e-mail traffic](#inbound-e-mail-traffic)
    + [Set up OpenDMARC](#set-up-opendmarc)
    + [Integrate with Postfix](#integrate-with-postfix)
    + [Set up reporting](#set-up-reporting)
    + [Configuring SpamAssassin](#configuring-spamassassin)
- [Special thanks](#special-thanks)

<small><i><a href='http://ecotrust-canada.github.io/markdown-toc/'>Table of contents generated with markdown-toc</a></i></small>

# Introduction
This how-to is created by the Dutch Internet Standards Platform (the organization behind [internet.nl](https://internet.nl)) and is meant to provide practical information and guidance on implementing DMARC.

# What is DMARC?
DMARC is short for **D**omain based **M**essage **A**uthentication, **R**eporting and **C**onformance and is described in [RFC 7489](https://tools.ietf.org/html/rfc7489). With DMARC the owner of a domain can, by means of a DNS record, publish a  policy that states how to handle e-mail (deliver, quarantine, reject) which is not properly authenticated using SPF and/or DKIM. 

At the same time DMARC also provides the means for receiving reports which allows a domain's administrator to detect whether their domainname is used for phishing or spam. 
 
# Why use DMARC?
Before DMARC, organizations already took several measures to determine the authenticity of an e-mail (like SPF and DKIM) to reduce the received amount of SPAM to a minimum. This is basically a good thing, but if these measures fail to choose whether or not an email is SPAM with a high level of certainty, the choice is redirected to the addressee (receiving party). This methodology is prone to abuse, since users are generally not equiped with the knowledge and/or means to classify incoming emails. 
 
DMARC addresses this problem and enables the owner of a domain to take explicit responsiblity with regard to the actions taken by the sending party when the validity of an incoming email cannot be determined.  

# Tips, tricks and notices for implementation
* Interoperabily issues: https://tools.ietf.org/html/rfc7960
* DMARC does not require both DKIM or SPF. But implementation of both is strongly advised.
* DMARC is about aligning the domain in the DKIM header (the "d=" value) and SPF header (*a.k.a. RFC5321.MailFrom, Return-Path, Envelope Sender, Envelope From*) with the organizational domain in the message From header (*a.k.a. RFC5322.From, Header From, Message From*) which is visible to the user.
  * If these values do not align this could mean for example, that an attacker placed a valid DKIM signature header in an email with a "d=" value that points to a domain the attacker controls, allowing DKIM to pass while still spoofing the From address to the user.
* Parked domain: “DMARC p=reject”. Make sure to include rua and ruf addresses, since this allows monitoring of possible abuse attempts. Implement additional records (SPF, DKIM, NullMX) if possible, see also our [Parked domain how-to](https://github.com/internetstandards/toolbox-wiki/blob/master/parked-domain-how-to.md).
* RFC 7489 [states](https://tools.ietf.org/html/rfc7489#section-6.4) that the tags dmarc-version ("v=") and dmarc-request ("p=") should be on the first and second position of the DMARC record. The order of the other tags does not matter: "components other than dmarc-version and dmarc-request may appear in any order".
* [Errata 5440 of RFC 7489](https://www.rfc-editor.org/errata_search.php?rfc=7489) states that a semicolon should be included in the DMARC version tag. Correct: "v=DMARC1;". Incorrect: "v=DMARC1". 
* When using office 365, the forwarding of calendar appointments from a DMARC projected domain fails. This is a known issue. Read more on the [Office 365 UserVoice forum](https://office365.uservoice.com/forums/264636-general/suggestions/34012756-forwarding-of-calendar-appointments-from-a-dmarc-p) and don't forget to submit your vote! 
  * There is a workaround: Forward the appointment as an "iCalendar file" or as an attachment. 

# Overview of DMARC configuration tags
The DMARC policy is published by means of a DNS TXT record. A DMARC record can contain several configuration tags. The table below will list all configuration tags and explain their purpose.

| DMARC configuration tag | Required? | Value(s) | Explanation |
| ---  | --- |  --- | --- |
| v | mandatory | DMARC1; | |
| p | mandatory | none<br>quarantine<br>reject | None: don't do anything if DMARC verification fails (used for testing)<br>quarantine: treat mail that fails DMARC check as suspicious<br>reject: reject mail that fail DMARC check |
| rua | optional | rua@example.nl | This field contains the email address used to send **aggregate** reports to |
| ruf | optional |ruf@example.nl | This field contains the email address used to send **forensic** reports to |
| fo | mandatory | <br>0<br>1<br>s<br>d | Reporting options for failure reports. Generates a report if:<br>- both SPF and DKIM tests fail (0)<br>- either SPF or DKIM test fail (1)<br>- SPF test fails (s)<br>- DKIM test fails (d) |
| adkim | optional | s<br>r | Controls how strict the result of DKIM alignment checks should be intepreted. Strict or relaxed. |
| aspf | optional | s<br>r | Controls how strict the result of SPF alignment checks should be intepreted. Strict or relaxed. |
| pct | optional | 0..100 | Determine percentage of mail from your domain to have the DMARC verificaton done by other mail providers. Default is 100. |
| rf | optional | afrf<br>aodef | Preferred reporting format for forensic reports. Default is afrf. |
| ri | optional | 60..86400 | Interval (in seconds) at which you want to receive DMARC reports. The DMARC RFC specifies that organizations should be able to send reports at least once every day (86400 seconds) and also states a minimum interval of one hour (60 seconds). Default is 86400. |
| sp | optional | none<br>quarantine<br>reject |  Subdomain policy. If not set the policy defined under "p=" will apply to all your subdomains. |

Be aware that implementing a DMARC record without a rua configuration is possible, this is not advised because the DMARC XML files that are received by implementing a rua email address can help with implementing DKIM or SPF to meet the DMARC requirements.

# Implementing DMARC with OpenDMARC for Postfix with SpamAssassin
**Specifics for this setup**
* Linux Debian 9.8 (Stretch) 
* Postfix 3.4.5
* BIND 9.10.3-P4-Debian
* OpenDMARC v1.3.2

**Assumptions**
* DNSSEC is used
* Mail server is operational
* Software packages are already installed

## Outbound e-mail traffic
DMARC for outbound e-mail traffic can be accomplished by publishing a DMARC policy as a TXT record in a domain name's DNS zone.

### Setting up a DMARC record
Depending on your preferences and needs, you can determine the value of the configuration tags. The values below seem like a good starting point when setting up DMARC.

    _dmarc IN TXT "v=DMARC1; p=quarantine; rua=mailto:dmarc@example.nl; ruf=mailto:dmarc@example.nl; fo=0; adkim=r; aspf=r; pct=100; rf=afrf; ri=86400; sp=quarantine"
    
Because this specific setup uses SpamAssassin for classifying e-mail to be SPAM or legitimate (HAM), the DMARC policy used is quarantine. This is done to prevent OpenDMARC from blocking the e-mail and, as a result, not enabling SpamAssassin to do its job.

## Inbound e-mail traffic
DMARC for inbound e-mail traffic can be accomplished by setting up OpenDMARC and integrate it with Postfix.

### Set up OpenDMARC
Make sure the file **/etc/opendmarc.conf** has a least the following configuration parameters.

    AuthservID mail.example.nl
    PidFile /var/run/opendmarc/opendmarc.pid
    RejectFailures false
    Syslog true
    TrustedAuthservIDs mail.example.nl,mail2.example.nl,localhost,127.0.0.1
    UMask 0002
    UserID opendmarc:opendmarc
    IgnoreAuthenticatedClients true
    IgnoreHosts /etc/opendmarc/ignore.hosts
    HistoryFile /var/run/opendmarc/opendmarc.dat
    Socket inet:54321@localhost

For more information about these configuration parameters, take a look at [its man page](https://manpages.debian.org/unstable/opendmarc/opendmarc.conf.5.en.html).

Make sure the file **/etc/opendmarc/ignore.hosts** contains all hosts that you trust. The e-mail coming from these hosts will not be checked by OpenDMARC:

    127.0.0.1
    localhost

Make sure the default file **/etc/default/opendmarc** contains:

    RUNDIR=/var/run/opendmarc
    SOCKET=inet:54321@localhost
    USER=opendmarc
    GROUP=opendmarc
    PIDFILE=$RUNDIR/opendmarc.pid

### Integrate with Postfix
Now we need to tell Postfix to use OpenDMARC as a mail filter in order to use its functionality. This is done by making sure that **/etc/postfix/main.cf** contains the configuration parameters as listed below. Notice that the DKIM check (localhost:12301) is done _before_ DMARC (localhost:54321) since DMARC relies on the DKIM results.

    smtpd_milters = inet:localhost:12301,inet:localhost:54321
    non_smtpd_milters = inet:localhost:12301,inet:localhost:54321

### Set up reporting
In order to become DMARC compliant, we need to start sending aggregate DMARC reports. Let's start by creating an OpenDMARC database using the provided MySQL script (/usr/share/doc/opendmarc/schema.mysql) in the OpenDMARC package.  

In the example below the user root is used to create the new "opendmarc" database. If you would also like to create a user with access to the database, then uncomment the last two lines in the MySQL script and make sure to change the default 'changeme' password into something strong.

    mysql -u root -p < /usr/share/doc/opendmarc/schema.mysql

After creation of the database, you need to create an executable Bash script in order to be able to send out reports. Create the file **/etc/opendmarc/report_script** and use the following content:

```
#!/bin/bash
 
DB_SERVER='database.example.nl'
DB_USER='opendmarc'
DB_PASS='somethingstrong'
DB_NAME='opendmarc'
WORK_DIR='/var/run/opendmarc'
REPORT_EMAIL='dmarc@example.nl'
REPORT_ORG='Example Company (example.nl)'
 
mv ${WORK_DIR}/opendmarc.dat ${WORK_DIR}/opendmarc_import.dat -f
cat /dev/null > ${WORK_DIR}/opendmarc.dat
 
/usr/sbin/opendmarc-import --dbhost=${DB_SERVER} --dbuser=${DB_USER} --dbpasswd=${DB_PASS} --dbname=${DB_NAME} --verbose < ${WORK_DIR}/opendmarc_import.dat
/usr/sbin/opendmarc-reports --dbhost=${DB_SERVER} --dbuser=${DB_USER} --dbpasswd=${DB_PASS} --dbname=${DB_NAME} --verbose --interval=86400 --report-email $REPORT_EMAIL --report-org $REPORT_ORG
```

Make sure the script is executable

    chmod +x /etc/opendmarc/report_script
    
and run it every day under the user opendmarc by adding the following to **/etc/crontab**:

    1 0 * * * opendmarc /etc/opendmarc/report_script
    
### Configuring SpamAssassin
SpamAssassin uses a scoring mechanism in order to determine if an e-mail should be considered spam. By default SpamAssassin considers an e-mail to be spam if the score at least "5". An e-mail starts with a score of 0 and points are added based on the [tests](https://spamassassin.apache.org/old/tests_3_3_x.html) performed. The tests performed can be configured by adding specific [configuration parameters](https://spamassassin.apache.org/full/3.4.x/doc/Mail_SpamAssassin_Conf.html) in **/etc/spamassassin/local.cf**.

Now here's the tricky part. The points added to the score of an incoming e-mail based on the results of a specific test, is at its core a custom job. Many variables can be taken into consideration when scoring an e-mail (which is considered the strength of a post-SMTP spam filter) and the detailed scoring depends on a domain owner's specific wishes. For the sake of this how-to, the DMARC scoring will be based on the assumption that the domain owner wants to consider an e-mail to be spam if the sending e-mail server's DMARC validation did fail. 

With SpamAssassin this can be configured by adding the following scoring configuration parameters to **/etc/spamassassin/local.cf**:

```
#dmarc fail
header CUST_DMARC_FAIL Authentication-Results =~ /mail\.example\.nl; dmarc=fail/
score CUST_DMARC_FAIL 5.0

#dmarc pass
header CUST_DMARC_PASS Authentication-Results =~ /mail\.example\.nl; dmarc=pass/
score CUST_DMARC_PASS -1.0
```

This means that when the "Authentication-Results" header of your e-mail contains "mail.example.nl; dmarc=fail" 5 points will be added to the score; instantly classifying this e-mail as SPAM. On the other hand, if the "Authentication-Results" header of your e-mail contains "mail.example.nl; dmarc=pass" -1 points will be added to the score; classifying this e-mail as legitimate.

# Special thanks
Our infinite gratitude goes out to the following people for their support in building this how-to for DANE.

Alwin de Bruin  
