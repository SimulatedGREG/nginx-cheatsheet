# nginx-cheatsheet
> A quick reference to common server configurations from serving static files to using in congruency with Node.js applications.

Each configuration below is written with minimum requirements for their described function. Please know that real world applications will most likely use a combination of these settings. This cheatsheet is meant to provide a general overview of how to setup specific features of nginx.

These configurations are meant to be used as **Name-Based Virtual Hosts**, saved within `/etc/nginx/sites-enabled`.

#### Table of Configurations
* [General Settings](#general-settings)
	* [Port (`listen`)](#port-listen)
	* [Domain name (`server_name`)](#domain-name-server_name)
	* [Access Logging (`access_log`)](#access-logging-access_log)
	* [Miscellaneous (`gzip`, `client_max_body_size`)](#miscellaneous-gzip-client_max_body_size)
* [Serving Files](#serving-files)
	* [Static assets](#static-assets)
	* [Static assets with HTML5 History Mode](#static-assets-with-html5-history-mode)
* [Redirects](#redirects)
	*  [`301` Permanent](#301-permanent)
	*  [`302` Temporary](#302-temporary)
	*  [Redirect on specific URL](#redirect-on-specific-url)
*  [Reverse Proxy](#reverse-proxy)
	*  [Basic](#basic)
	*  [Basic+](#basic-1)
	*  [Upgraded Connection (Recommended for Node.js Applications)](#upgraded-connection-recommended-for-nodejs-applications)
* [TLS/SSL (HTTPS)](#tlsssl-https)
    * [Basic](#basic-2)
*  [Large Scale Applications](#large-scale-applications)
	*  [Load Balancing](#load-balancing)

## General Settings
#### Port (`listen`)
```nginx
server {
  # standard HTTP protocol
  listen 80;
  
  # standard HTTPS protocol
  listen 443 ssl;
  
  # listen on 80 using IPv6
  listen [::]:80;
  
  # listen only on IPv6
  listen [::]:80 ipv6only=on;
}
```
#### Domain name (`server_name`)
```nginx
server {
  # Listen to yourdomain.com
  server_name yourdomain.com;
  
  # Listen to multiple domains
  server_name yourdomain.com www.yourdomain.com;
  
  # Listen to all sub-domains
  server_name *.yourdomain.com;
  
  # Listen to all top-level domains
  server_name yourdomain.*;
  
  # Listen to unspecified hostnames (listens to IP address itself)
  server_name "";
}
```
#### Access Logging (`access_log`)
```nginx
server {
  # Relative or full path to log file
  access_log /path/to/file.log;
  
  # Turn 'on' or 'off'
  access_log on;
}
```
#### Miscellaneous (`gzip`, `client_max_body_size`)
```nginx
server {
  # Turn gzip compression 'on' or 'off'
  gzip on;
  
  # Limit client body size to 10mb
  client_max_body_size 10M;
}
```
## Serving Files
#### Static assets
The traditional web server.
```nginx
server {
  listen 80;
  server_name yourdomain.com;
  
  location / {
  	root /path/to/website;
  }
}
```

#### Static assets with HTML5 History Mode
Useful for Single-Page Applications like Vue, React, Angular, etc.
```nginx
server {
  listen 80;
  server_name yourdomain.com;
  root /path/to/website;
  
  location / {
  	try_files $uri $uri/ /index.html;
  }
}
```

## Redirects
#### `301` Permanent
Useful for handling `www.yourdomain.com` vs. `yourdomain.com` or redirecting `http` to `https`. In this case we will redirect `www.yourdomain.com` to `yourdomain.com`.
```nginx
server {
  listen 80;
  server_name www.yourdomain.com;
  return 301 http://yourdomain.com$request_uri;
}
```
#### `302` Temporary
```nginx
server {
  listen 80;
  server_name yourdomain.com;
  return 302 http://otherdomain.com;
}
```
#### Redirect on specific URL
Can be permanent (`301`) or temporary (`302`).
```nginx
server {
  listen 80;
  server_name yourdomain.com;
  
  location /redirect-url {
	return 301 http://otherdomain.com;  
  }
}
```
## Reverse Proxy
Useful for Node.js applications like express.

#### Basic
```nginx
server {
  listen 80;
  server_name yourdomain.com;
  
  location / {
    proxy_pass http://0.0.0.0:3000;
    # where 0.0.0.0:3000 is your Node.js Server bound on 0.0.0.0 listing on port 3000
  }
}
```

#### Basic+
```nginx
upstream node_js {
  server 0.0.0.0:3000;
  # where 0.0.0.0:3000 is your Node.js Server bound on 0.0.0.0 listing on port 3000
}

server {
  listen 80;
  server_name yourdomain.com;
  
  location / {
    proxy_pass http://node_js;
  }
}
```

#### Upgraded Connection (Recommended for Node.js Applications)
Useful for Node.js applications with support for WebSockets like socket.io.
```nginx
upstream node_js {
  server 0.0.0.0:3000;
}

server {
  listen 80;
  server_name yourdomain.com;
  
  location / {
    proxy_pass http://node_js;
    proxy_redirect off;
    proxy_http_version 1.1;
    proxy_set_header Upgrade $http_upgrade;
    proxy_set_header Connection "upgrade";
    proxy_set_header Host $host;
	
    # not required but useful for applications with heavy WebSocket usage
    # as it increases the default timeout configuration of 60
    proxy_read_timeout 80;
  }
}
```
## TLS/SSL (HTTPS)
#### Basic
**The below configuration is only an example of what a TLS/SSL setup should look like. Please do not take these settings as the perfect secure solution for your applications. Please do research the proper settings that best fit with your Certificate Authority.**

If you are looking for free SSL certificates, [**Let's Encrypt**](https://letsencrypt.org/) is a free, automated, and open Certificate Authority. Also, here is a wonderful [step-by-step guide from Digital Ocean](https://www.digitalocean.com/community/tutorials/how-to-secure-nginx-with-let-s-encrypt-on-ubuntu-16-04) on how to setup TLS/SSL on Ubuntu 16.04.
```nginx
server {
  listen 443 ssl;
  server_name yourdomain.com;
  
  ssl on;
  
  ssl_certificate /path/to/cert.pem;
  ssl_certificate_key /path/to/privkey.pem;
  
  ssl_stapling on;
  ssl_stapling_verify on;
  ssl_trusted_certificate /path/to/fullchain.pem;

  ssl_protocols TLSv1 TLSv1.1 TLSv1.2;
  ssl_session_timeout 1d;
  ssl_session_cache shared:SSL:50m;
  add_header Strict-Transport-Security max-age=15768000;
}

# Permanent redirect for HTTP to HTTPS
server {
  listen 80;
  server_name yourdomain.com;
  return 301 https://$host$request_uri;
}
```
## Large Scale Applications
#### Load Balancing
Useful for large applications running multiple instances.
```nginx
upstream node_js {
  server 0.0.0.0:3000;
  server 0.0.0.0:4000;
  server 123.131.121.122;
}

server {
  listen 80;
  server_name yourdomain.com;
  
  location / {
    proxy_pass http://node_js;
  }
}
```
