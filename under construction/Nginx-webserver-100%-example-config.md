<img align="right" src="/images/logo-internetnl-en.svg">

# Ngnix example configuration
This example configuration was created by SIDN Labs and show how to configure Ngnix in order to score 100% in the Website test on Internet.nl. 

# Assumptions
* DNSSEC is used
* IPv6 is used

# Configuring Ngnix

    server_tokens off;
    add_header X-Frame-Options SAMEORIGIN;
    add_header X-Content-Type-Options nosniff;
    add_header X-XSS-Protection "1; mode=block";
    add_header Strict-Transport-Security "max-age=31536000" always;
    add_header Content-Security-Policy "default-src 'self'; frame-ancestors 'none'" always;
    add_header Referrer-Policy "strict-origin" always;
     
    ssl_stapling on;
    ssl_stapling_verify on;
    ssl_trusted_certificate /etc/letsencrypt/live/example.nl/fullchain.pem;
 
    # Rate limiting 20 requests/s
    limit_req_zone $binary_remote_addr zone=mylimit:10m rate=20r/s;
     
    # http://example.nl -> https://example.nl
    server {
        listen 80;
        listen [::]:80;
        server_name example.nl;
     
        location /.well-known/acme-challenge/ {
            root /var/www/certbot;
        }
     
        location / {
            return 301 https://example.nl$request_uri;
        }
    }
     
    # http://(www|api).example.nl -> https://(www|api).example.nl
    server {
        listen 80;
        listen [::]:80;
        server_name www.example.nl api.example.nl;
     
        return 301 https://$host$request_uri;
    }
     
    # https://example.nl -> https://www.example.nl
    server {
        listen 443 ssl http2;
        listen [::]:443 ssl http2;
        server_name example.nl;
     
        ssl_certificate /etc/letsencrypt/live/example.nl/fullchain.pem;
        ssl_certificate_key /etc/letsencrypt/live/example.nl/privkey.pem;
     
        ssl_session_cache shared:SSL:50m;
        ssl_session_timeout 1d;
        ssl_session_tickets on;
     
        ssl_dhparam /etc/letsencrypt/ssl-dhparams.pem;
     
        ssl_protocols TLSv1.2 TLSv1.3;
        ssl_ciphers ECDH+AESGCM:ECDH+AES256:ECDH+AES128:DH+3DES:!ADH:!AECDH:!MD5;
        ssl_prefer_server_ciphers on;
     
        return 301 https://www.$host$request_uri;
    }
     
    # serve https://(www|api).example.nl
    server {
        listen 443 ssl http2;
        listen [::]:443 ssl http2;
        server_name www.example.nl api.example.nl;
     
        ssl_certificate /etc/letsencrypt/live/example.nl/fullchain.pem;
        ssl_certificate_key /etc/letsencrypt/live/example.nl/privkey.pem;
     
        ssl_session_cache shared:SSL:50m;
        ssl_session_timeout 1d;
        ssl_session_tickets on;
     
        ssl_dhparam /etc/letsencrypt/ssl-dhparams.pem;
     
        ssl_protocols TLSv1.2 TLSv1.3;
        ssl_ciphers ECDH+AESGCM:ECDH+AES256:ECDH+AES128:DH+3DES:!ADH:!AECDH:!MD5;
        ssl_prefer_server_ciphers on;
     
        # Example location for a Flask application running on port 8080
        location / {
            limit_req zone=mylimit burst=100 nodelay;
     
            include uwsgi_params;
            uwsgi_pass flask:8080;
            uwsgi_ignore_client_abort on;
        }
    }





