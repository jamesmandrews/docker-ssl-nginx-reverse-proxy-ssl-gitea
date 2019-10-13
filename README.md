# NGINX Reverse Proxy of Gitea with Let's Encrypt SSL via Docker

NGINX is powerful open-source software for web serving, reverse proxying, caching, load balancing, media streaming, and more.  Combined with Let’s Encrypt a non-profit certificate authority run by Internet Security Research Group, you can securely server websites, and web applications over an encrypted channel to protect your user’s privacy.

One of NGINX’s core functionalities is to act as a reverse proxy.  A reverse proxy allows you to run a service behind a firewall protecting the server that it is on from attackers on the outside of the firewall.  It protects the integrity and structure of the internal application by managing the connections to that service.  When used in conjunction with docker you can run many docker containers that contain different services without exposing their ports and have NGINX manage the routing of requests to these containers.

Today we’ll do just that using Gitea.  Gitea describes themselves as “A painless self-hosted Git service.”.  Since they offer a docker container on docker hub it is easy for us to make use of.


# The docker-compose.yml at a glance. 
The docker compose is comprised of 5 pieces.

* NGINX - Which can be used to serve websites, or as a reverse proxy.
* CERBOT - Which is used to generate our secure certificates
* GITEA - For our example we'll use this as our back end application that we reverse proxy to
* MariaDB (GITEADB) - Database layer for Gitea.

# Configuring NGINX.

We're going to configure 2 servers.  One will be a static website, and the other will be our reverse proxy to gitea.  We include the entire nginx configuration directory so that we can make manage it without baking it into the NGINX container itself

```
├── conf
│   └── nginx
│       ├── conf.d
│       │   ├── website.conf
│       │   └── gitea.conf
│       ├── fastcgi_params
│       ├── koi-utf
│       ├── koi-win
│       ├── mime.types
│       ├── nginx.conf
│       ├── scgi_params
│       ├── uwsgi_params
│       └── win-utf

```

For this tutorial we only really care about the files in ```conf/nginx/conf.d```  there are 2 of them, one of them is for our website that will serve static pages, the second is for gitea reverse proxy.


# Static Website.

Our first server declaration is pretty simple.  We have our domain ```ourwebsite.com``` on port 80 we add the acme-challenge location which Certbot uses to make sure we have a valid webserver up.  The main root, we'll do a redirect to the secure 443 layer.

```
    server {
        listen 80;
        server_name ourwebsite.com
                    www.ourwebsite.com;

        location /.well-known/acme-challenge/ {
            root /var/www/certbot;
        }

        location / {
            return 301 https://$host$request_uri;
        }        
    }
```

The SSL layer We define our webroot, and our index file type.  We'll also define where we'll keep our certificate files.  In our setup all certificates for all domains will go into one file.  

```

    server {
        listen 443 ssl;
        server_name ourwebsite.com
                    www.ourwebsite.com;
        root /var/www/html;
        index index.html
        server_tokens off;

        ssl_certificate /etc/letsencrypt/live/websites/fullchain.pem;
        ssl_certificate_key /etc/letsencrypt/live/websites/privkey.pem;
        include /etc/letsencrypt/options-ssl-nginx.conf;
        ssl_dhparam /etc/letsencrypt/ssl-dhparams.pem;

        location / {
          root /var/www/html;
          index index.html;
        }
    }

```

