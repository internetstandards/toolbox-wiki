# Table of contents

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
* DMARC is about aligning the DKIM and/or SPF domain with the organizational domain in the From header.
* Parked domain: “DMARC p=reject”. Make sure to include rua and ruf addresses, since this allows monitoring of possible abuse attempts. Implement additional records (SPF, DKIM, NullMX) if possible, see also our [Parked domain how-to](https://github.com/internetstandards/toolbox-wiki/blob/master/parked-domain-how-to.md).
* RFC 7489 [states](https://tools.ietf.org/html/rfc7489#section-6.4) that the tags dmarc-version ("v=") and dmarc-request ("p=") should be on the first and second position of the DMARC record. The order of the other tags does not matter: "components other than dmarc-version and dmarc-request may appear in any order".
* [Errata 5440 of RFC 7489](https://www.rfc-editor.org/errata_search.php?rfc=7489) states that a semicolon should be included in the DMARC version tag. Correct: "v=DMARC1;". Incorrect: "v=DMARC1". 
* When using office 365, the forwarding of calendar appointments from a DMARC projected domain fails. This is a known issue. Read more on the [Office 365 UserVoice forum](https://office365.uservoice.com/forums/264636-general/suggestions/34012756-forwarding-of-calendar-appointments-from-a-dmarc-p) and don't forget to submit your vote! 
  * There is a workaround: Forward the appointment as an "iCalendar file" or as an attachment. 

# Creating a DMARC record
The DMARC policy is published by means of DNS TXT record.
Overview 

rua: aggregate reports
ruf: forensic reports

| DMARC configuration tag | Required? | Value(s) | Explanation |
| ---  | --- |  --- | --- |
| v | mandatory | DMARC1; | |
| p | mandatory | none<br>quarantine<br>reject | None: don't do anything if DMARC verification fails (used for testing)<br>quarantine: treat mail that fails DMARC check as suspicious<br>reject: reject mail that fail DMARC check |
| rua | optional | rua@example.nl | This field contains the email address used to send aggregate reports to |
| ruf | optional |ruf@example.nl | This field contains the email address used to send forensic reports to |
| fo | mandatory | <br>0<br>1<br>s<br>d | Reporting options for failure reports. Generates a report if:<br>- both SPF and DKIM tests fail (0)<br>- either SPF or DKIM test fail (1)<br>- SPF test fails (s)<br>- DKIM test fails (d) |
| adkim | optional | s<br>r | Controls how strict the result of DKIM verification should be intepreted. Strict or relaxed. |
| aspf | optional | s<br>r | Controls how strict the result of SPF verification should be intepreted. Strict or relaxed. |
| pct | optional | 0..100 | Determine percentage of mail from your domain to have the DMARC verificaton done by other mail providers. Default is 100. |
| rf | optional | | |
| ri | optional | | |
| sp | optional | | |

Be aware that implementing a DMARC record without a rua configuration is possible, this is not advised because the DMARC XML files that are received by implementing a rua email address can help with implementing DKIM or SPF to meet the DMARC requirements.

# Reporting
to-do

# Implementing DKIM with OpenDKIM for Postfix with SpamAssassin
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
to-do
