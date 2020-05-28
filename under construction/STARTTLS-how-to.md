<img align="right" src="images/logo-internetnl-en.svg">

# UNDER CONSTRUCTION!!! 

# STARTTLS how-to
This how-to is created by the Dutch Internet Standards Platform (the organization behind [internet.nl](https://internet.nl)) and is meant to provide practical information and guidance on implementing STARTTLS.  

# Table of contents
To-Do

# What is STARTTLS?


# Why use STARTTLS?


# Tips, tricks and notices for implementation
* The sender address shown to the user ("RFC5322.From") is not used when authenticating. SPF uses the invisible "RFC5321.MailFrom" header. Combining SPF with DMARC removes this disadvantage. 
* E-mail forwarding is not supported, since the e-mail is often forwarded by another e-mail server.
* SPF does not work between domains that use the same e-mail server.
* Parked domains should be explicitly configured to not use e-mail. For SPF this is done with an empty policy (not mentioning any ip-adresses or hostnames which are allowed to send mail) and a hard fail: "v=spf1 –all".
* When processing incoming mail we advise to favor a DMARC policy over an SPF policy. Do not configure SPF rejection to go into effect early in handling, but take full advantage of the enhancements DMARC is offering. A message might still pass based on DKIM.
  * At the same time, be aware that some operaters still allow a hard fail (-all) to go into effect early in handling and skip DMARC operations. 



## Implementing STARTTLS in Postfix
**Specifics for this setup**
* Linux Debian 10 (Buster) 
* Postfix 3.4.5
* OpenSSL 1.1.1d

**Assumptions**
* Mail server is using DANE
* Software packages are already installed

### Configuring Postfix

    # use DANE
    smtp_dns_support_level = dnssec
    smtp_tls_security_level = dane
    smtp_host_lookup = dns
    smtp_tls_note_starttls_offer = yes
    
    # TLS protocol config
    smtpd_tls_mandatory_protocols = !SSLv2, !SSLv3, !TLSv1, !TLSv1.1
    smtpd_tls_protocols = !SSLv2, !SSLv3, !TLSv1, !TLSv1.1
    smtp_tls_mandatory_protocols = !SSLv2, !SSLv3, !TLSv1, !TLSv1.1
    smtp_tls_protocols = !SSLv2, !SSLv3, !TLSv1, !TLSv1.1
    lmtp_tls_mandatory_protocols = !SSLv2, !SSLv3, !TLSv1, !TLSv1.1
    lmtp_tls_protocols = !SSLv2, !SSLv3, !TLSv1, !TLSv1.1
	
	# TLS cipher config
    smtpd_tls_mandatory_ciphers=high
    smtpd_tls_ciphers=high
    tls_ssl_options = NO_COMPRESSION, 0x40000000
    smtpd_tls_exclude_ciphers = aNULL, eNULL, EXPORT, DES, RC4, MD5, PSK, EDH-DSS-DES-CBC3-SHA, EDH-RSA-DES-CBC3-SHA, KRB5-DES, CBC3-SHA, DHE-RSA-AES256-CCM8, AES256-CCM8, DHE-RSA-AES128-CCM8, AES128-CCM8
    smtp_tls_exclude_ciphers = aNULL, eNULL, EXPORT, DES, RC4, MD5, PSK, EDH-DSS-DES-CBC3-SHA, EDH-RSA-DES-CBC3-SHA, KRB5-DES, CBC3-SHA, DHE-RSA-AES256-CCM8, AES256-CCM8, DHE-RSA-AES128-CCM8, AES128-CCM8
	# Enable server cipher-suite preferences
    tls_preempt_cipherlist = yes
    # Forward secrecy (use the RFC 7919 defined DH group:https://raw.githubusercontent.com/internetstandards/dhe_groups/master/ffdhe4096.pem)
    smtpd_tls_eecdh_grade=ultra
    smtpd_tls_dh1024_param_file = /etc/postfix/ssl/ffdhe4096.pem
    # log the ciphers that are used
    smtp_tls_loglevel = 1
    smtpd_tls_loglevel = 1




