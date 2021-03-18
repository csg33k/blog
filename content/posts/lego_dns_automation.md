---
title: "LetsEncrypt SSL DNS automation with lego"
date: 2021-03-17T15:47:21-07:00
tags: ["docker", "ssl", "security", "automation", "ghost", "lego"]
draft: false
authors: ["csgeek"]
---
# Lego and LetsEncrypt

if you haven't heard about letsencrypt you should.  If you're still serving all your traffic over HTTP you should stop and move over the HTTPS everything.  Honestly at this point there shouldn't be any reason not to use SSL.  The CPU/server cost is minimal and there is tooling that makes this trivial.  I'm going to walk you through the process of setting SSL using nginx and we'll use docker for good measure.

## Dependencies/Tooling

Some familiarity with the following tools would be nice to have but not required.

- Docker
- nginx
- SSL

## SSL

[LetsEncrypt](https://letsencrypt.org/) provides free SSL/TLS certificate for the world at large as long as you can verify that you are who you are.  They have several tools that validate you, the easiest being running a web server, adding a DNS entry and so on.  [Certbot](https://certbot.eff.org/) is likely the most famous tool that generates SSL certificate

Personally I prefer using [Lego](https://github.com/go-acme/lego) which is a letsencrypt written in Go.  Partly I'm a big fan of Go as a language so I appreciate the developer's choice but also it lets me wildcard certificates and SAN very easily.

Although you can generate an SSL easily enough in order to get a wildcard DNS verification is much easier and simpler to use.  Though you do need a more legit DNS provider that has an API exposed.  Lego has a long list of supported [DNS Providers](https://go-acme.github.io/lego/dns/).  I have used Amazon Route53 and will likely move to a GCP one down the line, but honestly any of them will work fine.

What is very cool and special about SAN is that it creates a single certificate that supports multiple domains. 

You can run this anywhere but in my case I have it setup in `/etc/letsencrypt`.

The first step is obtaining the certificate the first time.

```sh
cd /etc/letsencrypt
AWS_ACCESS_KEY_ID=SECRET  AWS_SECRET_ACCESS_KEY="SECRET"  lego --dns route53 --domains="*.foobar.org" --domains="foobar.org"   --email valid@email run
```

As you can see I created a wildcard and a `foobar.org`.  It's a bit annoying but *.foobar.org does not cover the base domain.  You can enumerate all your domains if you don't want to have a wild card.  Example

```sh
cd /etc/letsencrypt
AWS_ACCESS_KEY_ID=SECRET  AWS_SECRET_ACCESS_KEY="SECRET"  lego --dns route53 --domains="www.foobar.org" --domains="foobar.org" --domains="mail.foobar.org" --domains="ftp.foobar.org" --email valid@email run
```

Once you have your certificate you simply need to ensure to renew it regularly. I created a simple bash scripts to do so.  

```sh
#!/usr/bin/env bash
cd /etc/letsencrypt/
DOMAINS="domain1 domain2 foobar.org"
AWS_KEY=<CHANGEME>
AWS_SECRET=<CHANGEME>

for domain in $DOMAINS;
do
        AWS_ACCESS_KEY_ID=$AWS_KEY  AWS_SECRET_ACCESS_KEY="$AWS_SECRET"  lego --dns route53 --domains="*.$domain" --domains="$domain"   --email user@gmail.com renew
done
```

Using AWS has a bit of a cost for using their DNS providers, but with the 3 domains I have I don't think my bill comes to more than maybe $3/month. That's well worth it to me.

## Ghost Web Site

```yaml
version: "3.7"
services:
  www:
    image: ghost:4.0.1
    ports:
      - "8890:2368"
    restart: always
    env_file: .env
    volumes:
      - ./content:/var/lib/ghost/content/
  mysql:
    container_name: shared_mysql
    image: mysql:5.7.29
    restart: always
    env_file: .env
    volumes:
      - ./data:/var/lib/mysql
```

At this point we expose a port locally 8890 that is serving HTTP traffic.

We need a .env file that is referenced in the MySQL and ghost instance.

```ini
## Database Ghost config
database__client=mysql
database__connection__host=shared_mysql
database__connection__user=root
database__connection__password=testing
database__connection__database=ghost

##MySQL config
MYSQL_ROOT_PASSWORD=testing
MYSQL_DATABASE=gbfest


## Domain config
#url=http://0.0.0.0:8890
url=https://www.foobar.org

## Email settings Production 
## FILL THESE IN as appropriate
#mail__from=donotreply@domain
#mail__transport=SMTP
#mail__options__port=587
#mail__options__host=
#mail_options_service=SMTP
#mail__options__auth__user=
#mail__options__auth__pass=

```

At this point we can bring up the website using `docker-compose up -d` but to earn brownie points, we'll use a systemd file.

### Systemd startup script

in `/etc/systemd/system` create the following file.  We'll assume you have a local user named docker_user that ideally has a shell of `/bin/nologin` that's part of the docker group.

ghost.service file:

```sh
[Unit]
Description = Ghost Website
After=docker.service
Requires=docker.service

[Service]
Type=simple
WorkingDirectory=/home/docker_user/ghost
ExecStart=/usr/bin/docker-compose up
ExecStop=/usr/bin/docker-compose stop
ExecReload =/usr/bin/docker-compose restart
User=docker_user
Group=docker
Restart=always
RestartSec=3


[Install]
WantedBy=multi-user.target
```

At this point we can start/stop/enable using systemd.

```sh
systemctl enable ghost.service
systemctl status ghost
```

In otherwords, if you restart your computer your website will still come up.  

### Final Notes on ghost

At this point we're listening to traffic on http://localhost:8890 that we are not exposing to the internet because like good little internet samaritans we have firewall rules that prevents bad actors from getting in.  But we do need to let people wanting to see our website access.


## Nginx SSL Wrapping

If you haven't you should open up port 80 and 443 on your webserver.  We won't be serving traffic on 80 but if someone comes on 80 we should redirect them and in order to do that we need 80 exposed.

in `/etc/nginx/sites-available` we're going to create the following ghost.conf.

```nginx
server {
	# SSL configuration
	#
	listen 443 ssl;
	listen [::]:443 ssl;

        access_log /var/log/nginx/ghost_access.log;
        error_log /var/log/nginx/ghost_error.log;

        ssl_certificate /etc/letsencrypt/.lego/certificates/_.foobar.org.crt;
        ssl_certificate_key /etc/letsencrypt/.lego/certificates/_.foobar.org.key;

        client_max_body_size 50M;


        root /var/www/html/ghost;
        index index.html index.htm;

        server_name www.foobar.org;
        large_client_header_buffers 4 32k;

	    location / {
	    	proxy_set_header        X-Real-IP $remote_addr;
	    	proxy_set_header        X-Forwarded-For $proxy_add_x_forwarded_for;
	    	proxy_set_header        X-Forwarded-Proto $scheme;
	    	proxy_pass http://0.0.0.0:8890;
	    	include proxy_params;
	    }

        location ~ /(data|conf|bin|inc)/ {
            deny all;
        }

        location = /robots.txt {
            allow all;
            log_not_found off;
            access_log off;
        }

        # Block access to "hidden" files and directories whose names begin with a
        # period. This includes directories used by version control systems such
        # as Subversion or Git to store control files.
        location ~ (^|/)\. {
            return 403;
        }


}

server {
	listen 80;
	listen [::]:80;
    return 301 https://$host$request_uri;

    server_name www.foobar.org;
}
```

Before this is finalized we need to enable the site.

```sh
cd /etc/nginx/sites-enabled
ln -s ../sites-available/ghost.conf 01_ghost.conf
nginx -t ## Validates config
systemclt restart nginx
```

We're essentially redirecting port 80 to 443 and creating a proxy that wraps all traffic around SSL and services traffic for www.foobar.org.  That also means if we connect to our server by IP we'll get the default page rather then a valid website.

### Backup Strategies

At this point you should be taking a regular SQL dump of your database, and backup the content of your ghost website.  The ghost lab has a export for all your content which you should use regularly before doing any major operations.  

## Final Notes

I personally like using Google Storage for all my images content.  I'm currently maintaining a copy of the ghost docker image with additional plugins so support GS with plans of adding S3 soon. You can find the source [here](https://github.com/OSAlt/gb-docker-ghost) and docker images can be found [here](https://hub.docker.com/r/geekbeacon/gb-ghost).  The tags are matched to the official ghost docker container.

