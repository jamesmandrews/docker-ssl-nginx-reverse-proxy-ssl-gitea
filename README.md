# NGINX Reverse Proxy of Gitea with Let's Encrypt SSL via Docker

NGINX is powerful open-source software for web serving, reverse proxying, caching, load balancing, media streaming, and more.  Combined with Let’s Encrypt a non-profit certificate authority run by Internet Security Research Group, you can securely server websites, and web applications over an encrypted channel to protect your user’s privacy.

One of NGINX’s core functionalities is to act as a reverse proxy.  A reverse proxy allows you to run a service behind a firewall protecting the server that it is on from attackers on the outside of the firewall.  It protects the integrity and structure of the internal application by managing the connections to that service.  When used in conjunction with docker you can run many docker containers that contain different services without exposing their ports and have NGINX manage the routing of requests to these containers.

Today we’ll do just that using Gitea.  Gitea describes themselves as “A painless self-hosted Git service.”.  Since they offer a docker container on docker hub it is easy for us to make use of.



