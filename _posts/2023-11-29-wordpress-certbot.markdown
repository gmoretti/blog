---
layout: post_with_comments
title:  "Wordpress, Nginx Reverse Proxy and Certbot"
description: 
date:   2023-11-29 23:07:00 +0200
---

![configuring]({{ site.baseurl }}/assets/images/computer_confused_guy.jpeg "configuring")
*this is what AI thinks of my problem

Welcome, everyone! In this post, I'll walk you through the process of setting up a Wordpress installation behind Docker, under a NGINX reverse proxy (to route different domains to different services), and Certbot to secure with SSL the site. Throughout this journey, I encountered and and somewhat successfully addressed various challenges.

## Introduction

To provide a bit of context, I manage a Virtual Private Server (VPS) on Digital Ocean. On this VPS, I have Docker installed, and I utilize an NGINX reverse proxy to serve not only this Wordpress installation but also my personal website and other services I need. My personal domain is pointing to this VPS, creating a cohesive and centralized environment for all my online endeavors.

## Configuring Docker and WordPress

I kicked off the process by installing WordPress through Docker. I have all my docker configs organized in folders depending on the service.
My docker compose file for all Wordpress needs looks like this:

```yaml
version: '3.4'
services:
    database:
        image: mysql:5.7
        command:
            - "--character-set-server=utf8"
            - "--collation-server=utf8_unicode_ci"
        ports:
            - "3306:3306" # (*)
        restart: on-failure:5
        environment: 
            MYSQL_USER: wordpress
            MYSQL_DATABASE: wordpress
            MYSQL_PASSWORD: wordpress
            MYSQL_ROOT_PASSWORD: wordpress
    wordpress:
        depends_on:
            - database
        image: wordpress:latest
        ports:
            - "8008:80" # (*)
        restart: on-failure:5
        volumes:
            - ./public:/var/www/html
        environment:
            WORDPRESS_DB_HOST: database:3306
            WORDPRESS_DB_PASSWORD: wordpress
            WORDPRESS_DB_USER: wordpress
            WORDPRESS_DB_NAME: wordpress
    phpmyadmin:
        depends_on:
            - database
        image: phpmyadmin/phpmyadmin
        ports:
            - 8084:80 # (*)
        restart: on-failure:5
        environment:
            PMA_HOST: database
    wordmove:
        tty: true
        depends_on:
            - wordpress
        image: mfuezesi/wordmove
        restart: on-failure:5
        container_name: luablu_wordmove  # (+)
        volumes:
            - ./config:/home
            - ./public:/var/www/html
            - ~/.ssh:/root/.ssh      # (!)

networks:
  default:
    external:
      name: nginx-reverse_default
```
The most important part of this config is define the network and make all defined services belong to that network.
 
## Configuring NGINX
I had to be able to route the new domain to wordpress and I wasn't able to find the host and port that contained the wordpress intallation. I ended up with this config.

```
server {
        server_name domain.com;
        listen 80 ;
        # Do not HTTPS redirect Let'sEncrypt ACME challenge
        
        location /.well-known/acme-challenge/ {
                root /home/static-websites/letsencrypt;
                auth_basic off;
                allow all;
                try_files $uri =404;
                break;
        }



        location / {
                return 301 https://$server_name$request_uri;
        }
}

server {
        server_name domain.com;
        listen 443 ssl http2 ;
        ssl_session_timeout 5m;
        ssl_session_cache shared:SSL:50m;
        ssl_session_tickets off;
        ssl_certificate /etc/nginx/letsencrypt/live/domain.com/fullchain.pem;
        ssl_certificate_key /etc/nginx/letsencrypt/live/domain.com/privkey.pem;
        add_header Strict-Transport-Security "max-age=31536000" always;

        location / {
        proxy_pass http://wordpress:80/;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        }
}
```

The acme challenge will be relevant for later CERTBOT config. the firts block redirects to HTTPS. For now the important part is the proxy_pass. Which is pointing to the CONTAINER name and the exposed port. I actually didnt know you could use container's name to point to it. Non of Ips or other known host names worked for me.

With these adjustments, I achieved accessibility to the WordPress site from any external location.

## SSL Certificate Configuration with Certbot

Moving on, I installed Certbot (from official docs)[https://certbot.eff.org/instructions?ws=other&os=ubuntufocal] on my Digital Ocean VPS to obtain an SSL certificate. This step presented 2 main hurdles:

1. **Incorrect DNS redirect configuration**: My domain's IPV6 was not pointing to the correct IP from DO, this meant the Certbot challenges were failing

2. **Utilizing the Certbot web-root plugin**: Overcoming Let's Encrypt challenges involved using the web-root plugin, with specific configuration for the challenge location pointing to the directory created by Certbot.

Successfully navigating these challenges, I generated the SSL certificates necessary to enhance the security of the site.

3. **Nginx reverse proxy config**: My Nginx is another container in the same system so I had to create volume to expose the certificates created by certbot to it. The folder wew certbot may vary. I exposed the whole thing to be able to navigate through everything.
My new volume looked like this:

```
      - /etc/letsencrypt/:/etc/nginx/letsencrypt
```

## Automating the Renewal Process

Recognizing that security extends beyond the initial certificate installation, I created a script that automates the renewal process. This script accesses the container and triggers a reload to apply the new certificates. This automation is seamlessly integrated into the Certbot renewal process, executed as a convenient hook.

To streamline the entire process, I added the `certbot renewal` command with the hook to a cronjob on my Digital Ocean VPS. This ensures that the certificates are renewed at regular intervals, maintaining the site's security and constant accessibility.

Since the certificate are only loaded once at boot by Nginx, after the cert is regenerated we need to restart nginx. I did thig by attaching a POST RENEWAL script to certbot with the following content.

```bash
#!/bin/bash

docker exec -it nginx-proxy service nginx reload
```
nginx-proxy is the name of the container that has my nginx reverser proxy, and the rest the command to execute inside the machine.
Now to include the trigger we execute like this:

```bash
certbot renew --deploy-hook /root/scripts/letsencrypt/post-deploy-certbot-renewal.sh
```

Make sure you add the deploy hook for the auto renewal. It can be by placing it in the deploy hook folder inside etc/letsencrypt or defining it when you create the cert. When reniewing, certbot will remember.

Thats ALL!
