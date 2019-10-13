# NGINX Reverse Proxy of Gitea with Let's Encrypt SSL via Docker

NGINX is powerful open-source software for web serving, reverse proxying, caching, load balancing, media streaming, and more.  Combined with Let’s Encrypt a non-profit certificate authority run by Internet Security Research Group, you can securely server websites, and web applications over an encrypted channel to protect your user’s privacy.

One of NGINX’s core functionalities is to act as a reverse proxy.  A reverse proxy allows you to run a service behind a firewall protecting the server that it is on from attackers on the outside of the firewall.  It protects the integrity and structure of the internal application by managing the connections to that service.  When used in conjunction with docker you can run many docker containers that contain different services without exposing their ports and have NGINX manage the routing of requests to these containers.

Today we’ll do just that using Gitea.  Gitea describes themselves as “A painless self-hosted Git service.”.  Since they offer a docker container on docker hub it is easy for us to make use of.


## Before you start.
In order to use these container you need to be running docker on a machine that has an IP address on the internet that has port 22, 80, and 443 open to the world.  This machine if already running ssh should have ssh reconfigured to a different port.  I typically use port 2222.  The reason for this is that Gitea needs access to port 22 for it's own ssh server.


## The docker-compose.yml at a glance. 
The docker compose is comprised of 5 pieces.

* NGINX - Which can be used to serve websites, or as a reverse proxy.
* CERBOT - Which is used to generate our secure certificates
* GITEA - For our example we'll use this as our back end application that we reverse proxy to
* MariaDB (GITEADB) - Database layer for Gitea.

The only things that we need to change here are the database settings in the GITEA container and the GITEADB container.  Choose a username and password and a password for the root user, save and your done with editing the docker compose file.

Do not change the service names as the initializing script will require the "nginx" service name, and NGINX uses the service names for routing purposes on the internal network.


## Configuring NGINX.

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


## Static Website.

Our first server declaration is pretty simple.  We have our domain ```ourwebsite.com``` on port 80 we add the acme-challenge location which Certbot uses to make sure we have a valid webserver up.  The main root, we'll do a redirect to the secure 443 layer if someone goes to port 80.

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


## Gitea reverse proxy.

The reverse proxy is a bit more complicated, but not that much.  First we create an upstream reference to the docker gitea installation.

```
    upstream docker-gitea {
        server gitea:3000;
    }
```

Then just like for the static website we'll create our acme-challenge location, and a redirect to the secure 443 port.

```

    server {
        listen 80;
        server_name  gitea.ourwebsite.com;

        location /.well-known/acme-challenge/ {
            root /var/www/certbot;
        }

        location / {
            return 301 https://$host$request_uri;
        }        
    }

```

The SSL server is also mostly the same, except instead fo defining a root directory for the website, you define proxy settings back to our upstream.

```

    server {
        listen 443 ssl;
        server_name gitea.ourwebsite.com;
        server_tokens off;

        ssl_certificate /etc/letsencrypt/live/websites/fullchain.pem;
        ssl_certificate_key /etc/letsencrypt/live/websites/privkey.pem;
        include /etc/letsencrypt/options-ssl-nginx.conf;
        ssl_dhparam /etc/letsencrypt/ssl-dhparams.pem;

        location / {
            proxy_pass         http://docker-gitea;
            proxy_redirect     off;
            proxy_set_header   Host $host;
            proxy_set_header   X-Real-IP $remote_addr;
            proxy_set_header   X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header   X-Forwarded-Host $server_name;
        }
    }

```

## Certbot and init-letsencrypt.sh

Once you have configured NGINX it is time to initialize your SSL certificates.  The problem is that in order for NGINX to start the SSL configuration we defined, we need to have actual SSL certificates.  We do that with the included script ```init-letsencrypt.sh```.  The original source for this script is from the [Nginx and Let’s Encrypt with Docker in Less Than 5 Minutes](https://medium.com/@pentacent/nginx-and-lets-encrypt-with-docker-in-less-than-5-minutes-b4b8a60d3a71) article.  I've made some minor modifications to fix a problem that was created when trying to do multiple servers.

Let's get to configuring the script so we can create the certificates.  

```
domains="ourwebsite.com,www.ourwebsite.com,gitea.ourwebsite.com"
cert_name="websites"
rsa_key_size=4096
email="" # Adding a valid address is strongly recommended
```

domains:  In our example we have 3 domains.  We want to pass them in as a single string.  Each domain separated by a comma, with NO SPACES.  This is how we'll pass them into Cerbot.   

cert_name: is the path in our ssl cert that we use as a common name so that we can load in the cert from all our servers.

rsa_key_size: Is how big if a key you want to create 4096 is a good place to start.

email: Define an email address, it's not manditory, but it's recommended.

With these 4 variables in the script set we are not ready to run it in staging mode.

```
./init-letsencrypt.sh --stage
```

We want to first run it in stage to make sure everything is configured correctly.  If we execute it in production mode let's encrypt may encounter rate limits which will prevent you from successfully getting your certificate.

If you run the command and are greeted with a the below message then you have successfully configured the servers.

```
IMPORTANT NOTES:
 - Congratulations! Your certificate and chain have been saved at:
   /etc/letsencrypt/live/websites/fullchain.pem
   Your key file has been saved at:
   /etc/letsencrypt/live/websites/privkey.pem
   Your cert will expire on 2020-01-11. To obtain a new or tweaked
   version of this certificate in the future, simply run certbot
   again. To non-interactively renew *all* of your certificates, run
   "certbot renew"
```

You should visit your URLs you'll get warnings about insecure certificates, but the fact that you have certificates is what you want to see.

Once you verify all your domains are available it's time to run it in production mode, but before running in production mode you should bring down the containers with the ```docker-compose down``` command.

Now execute the command in productino mode.

```
./init-letsencrypt.sh
```

You should be greated with the same message, but this time the certificates will be real and you should not be able to access the front page of Gitea, and of Your website.

## Gitea configuration.

Clicking on the login button of your Gitea installation will bring you to the install page.  I found that I had to use the root user and password for the database to get the database setup, but later was able to edit to the gitea user and password.  Don't forget to setup an administrator username and password. Follow the gitea instructions from here, to configure your installation.  As I get more comfortable with Gitea I'll write up more about it here.