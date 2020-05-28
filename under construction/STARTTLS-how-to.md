<img align="right" src="/images/logo-internetnl-en.svg">

# UNDER CONSTRUCTION!!! 

# STARTTLS how-to
This how-to is created by the Dutch Internet Standards Platform (the organization behind [internet.nl](https://internet.nl)) and is meant to provide practical information and guidance on implementing STARTTLS.  

# Table of contents
Under construction

# What is STARTTLS?
Under construction

# Why use STARTTLS?
Under construction

# Tips, tricks and notices for implementation
* http://postfix.1071664.n5.nabble.com/Disable-SSL-TLS-renegotiation-td96864.html#a96871
* Use the RFC 7919 defined DH groups: https://raw.githubusercontent.com/internetstandards/dhe_groups/master/ffdhe4096.pem)

## Implementing STARTTLS in Postfix
**Specifics for this setup**
* Linux Debian 10 (Buster) 
* Postfix 3.4.5
* OpenSSL 1.1.1d

**Assumptions**
* Mail server is using DANE
* Software packages are already installed

### Configuring Postfix

    # use DANE (when acting as a client)
    smtp_dns_support_level = dnssec
    smtp_tls_security_level = dane
    smtp_host_lookup = dns
    smtp_tls_note_starttls_offer = yes

    # --- TLS settings ---
    smtpd_tls_security_level = may
    smtpd_tls_key_file = /etc/postfix/ssl/example.nl.key
    smtpd_tls_cert_file = /etc/postfix/ssl/example.nl.crt
    smtpd_tls_CAfile = /etc/postfix/ssl/example.nl-cabundle.crt
    smtpd_tls_received_header = yes
    smtpd_tls_session_cache_timeout = 3600s
      
    # --- TLS protocol config ---
    smtpd_tls_mandatory_protocols = !SSLv2, !SSLv3, !TLSv1, !TLSv1.1
    smtpd_tls_protocols = !SSLv2, !SSLv3, !TLSv1, !TLSv1.1
    smtp_tls_mandatory_protocols = !SSLv2, !SSLv3, !TLSv1, !TLSv1.1
    smtp_tls_protocols = !SSLv2, !SSLv3, !TLSv1, !TLSv1.1
    lmtp_tls_mandatory_protocols = !SSLv2, !SSLv3, !TLSv1, !TLSv1.1
    lmtp_tls_protocols = !SSLv2, !SSLv3, !TLSv1, !TLSv1.1
	
	# --- TLS cipher config ---
    smtpd_tls_mandatory_ciphers=high
    smtpd_tls_ciphers=high
	# disable compression and client-initiated renegotiation
    tls_ssl_options = NO_COMPRESSION, 0x40000000
	# disable unsecure ciphers
    smtpd_tls_exclude_ciphers = aNULL, eNULL, EXPORT, DES, RC4, MD5, PSK, EDH-DSS-DES-CBC3-SHA, EDH-RSA-DES-CBC3-SHA, KRB5-DES, CBC3-SHA, DHE-RSA-AES256-CCM8, AES256-CCM8, DHE-RSA-AES128-CCM8, AES128-CCM8
    smtp_tls_exclude_ciphers = aNULL, eNULL, EXPORT, DES, RC4, MD5, PSK, EDH-DSS-DES-CBC3-SHA, EDH-RSA-DES-CBC3-SHA, KRB5-DES, CBC3-SHA, DHE-RSA-AES256-CCM8, AES256-CCM8, DHE-RSA-AES128-CCM8, AES128-CCM8
	# Enable server cipher-suite preferences
    tls_preempt_cipherlist = yes
    # Forward secrecy
    smtpd_tls_eecdh_grade=ultra
    smtpd_tls_dh1024_param_file = /etc/postfix/ssl/ffdhe4096.pem
	
	# --- TLS logging ---
    smtp_tls_loglevel = 1
    smtpd_tls_loglevel = 1




