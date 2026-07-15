<img align="right" src="/images/logo-internetnl-en.svg">

# Webserver example configurations

## Common mistakes

For all webservers, we observe the following common mistakes:

* HTTPS redirect: our test demands an upgrade to HTTPS **first**, before redirecting. This means that when you want your website to be accessible over domain example.nl, we expect http://example.nl to redirect to https://example.nl, while http://www.example.nl should redirect first to https://www.example.nl, then to https://example.nl. The reason for this, is that HSTS will only work when the Strict-Transport-Security header is served over HTTPS. By redirecting directly, the initial domain is not protected by HSTS.
* SHA-1 as hash function for key exchange: this is a legacy hash function that is explicitly deprecated since 2021 by [RFC 9155](https://datatracker.ietf.org/doc/html/rfc9155) and insufficient according to the latest TLS guidelines. Note that this is a different setting than the accepted ciphers. Not all firmware (especially GUI firmware) allows you easily change this configuration.
* Internet.nl expects HTTP security headers to be set, even when a domain/URL is merely a redirect to another domain. This is considered a defense-in-depth measure and should not affect the redirect.

## nginx
The most actual example configuration can be found within the source code of Internet.nl itself. 

### Assumptions
* DNSSEC is enabled  
* IPv6 is used and working  
* RPKI is used for all IPs

### Configuring nginx

The configuration used by Internet.nl is to be found in [this folder](https://github.com/internetstandards/Internet.nl/blob/main/docker/webserver/).
Parts of the configuration can be reused by your project to set up a 100% compliant webserver.  

Most of the generic configuration can be found [here](https://github.com/internetstandards/Internet.nl/blob/main/docker/webserver/nginx_templates/default.conf.template). Most notable are
enabling HTTP3 and upgrading HTTP requests to HTTPS. Note that for the Internet.nl project specific configurations are made, for instance for the connection test.
You will not need this for an ordinary webserver configuration.  

For the TLS configuration, check [this file](https://github.com/internetstandards/Internet.nl/blob/main/docker/webserver/nginx_templates/tls.conf.template).
Note that this allows TLS 1.2 and TLS 1.3, with all sufficient/good ciphers. `ssl_conf_command SignatureAlgorithms` ensures only sufficient signature
algorithms are accepted by the server (in case you got a 'SHA-1' finding in this test). Make sure to uncomment `ssl_stapling` and `ssl_stapling_verify` if your
CA/certificate still supports OCSP to enable OCSP-stapling.  

For HTTP headers, [this configuration](https://github.com/internetstandards/Internet.nl/blob/main/docker/webserver/http.headers) includes general headers, [here](https://github.com/internetstandards/Internet.nl/blob/main/docker/webserver/hsts_h3.headers) HSTS (Strict-Transport-Security) and HTTP3 headers are included. [This file](https://github.com/internetstandards/Internet.nl/blob/main/docker/webserver/nginx_templates/csp.header.template) contains an example Content-Security-Policy (CSP) header, which should be adapted to your own website.  


## Apache 

We currently do not have a full example Apache configuration. Most settings are documented [here](https://httpd.apache.org/docs/current/mod/mod_ssl.html), for example the `SSLCipherSuite` setting to specify which cipher suites should be accepted.   

To make sure only the allowed hash functions for key exchange are used, make use of the following setting:  

`SSLOpenSSLConfCmd SignatureAlgorithms ECDSA+SHA256:ECDSA+SHA384:ECDSA+SHA512:ed25519:ed448:RSA-PSS+SHA256:RSA-PSS+SHA384:RSA-PSS+SHA256:RSA+SHA256:RSA+SHA384:RSA+SHA512`
