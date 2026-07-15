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

* Use the RFC 7919 defined DH groups: https://raw.githubusercontent.com/internetstandards/dhe_groups/main/ffdhe4096.pem)

## Implementing STARTTLS in Postfix
**Specifics for this setup**
* Ubuntu 24.04
* Postfix 3.10.x
* OpenSSL 3.0.3

**Assumptions**
* Mail server is using DANE
* Software packages are already installed

### Configuring Postfix

/etc/postfix/main.cf

    tls_high_cipherlist = ECDHE-ECDSA-CHACHA20-POLY1305:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-ECDSA-AES256-CCM:ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES128-CCM:ECDHE-RSA-CHACHA20-POLY1305:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-RSA-AES128-GCM-SHA256
    tls_config_name = postfix
    tls_preempt_cipherlist = yes

    smtpd_tls_cert_file=...
    smtpd_tls_key_file=...
    smtpd_tls_security_level=may
    smtpd_tls_auth_only=yes
    smtpd_tls_protocols = >=TLSv1.2
    smtpd_tls_mandatory_protocols = >=TLSv1.2
    smtpd_tls_ciphers = high
    smtpd_tls_mandatory_ciphers = high
    smtpd_tls_loglevel = 1
    smtpd_tls_session_cache_database = btree:/var/lib/postfix/smtpd_tls_session_cache
    smtpd_tls_received_header = yes

    smtp_tls_note_starttls_offer = yes
    smtp_tls_protocols = >=TLSv1.2
    smtp_tls_mandatory_protocols = >=TLSv1.2
    smtp_tls_ciphers = high
    smtp_tls_mandatory_ciphers = high
    smtp_tls_loglevel = 1
    smtp_tls_session_cache_database = btree:/var/lib/postfix/smtp_tls_session_cache
    smtp_dns_support_level = dnssec
    smtp_tls_security_level = dane


/etc/ssl/openssl.cnf

    postfix = postfix_settings
    ....
    [postfix_settings]
    ssl_conf = postfix_ssl_settings
    [postfix_ssl_settings]
    system_default = baseline_postfix_settings
    [baseline_postfix_settings]
    SignatureAlgorithms = ECDSA+SHA256:ECDSA+SHA384:ECDSA+SHA512:RSA-PSS+SHA256:RSA-PSS+SHA384:RSA-PSS+SHA512:RSA+SHA512:RSA+SHA256:RSA+SHA384


To enable PQC:

    tls_eecdh_auto_curves =
    tls_ffdhe_auto_groups =
