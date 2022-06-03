# nginx_letsencrypt_php
A simple installation of [Nginx](https://hub.docker.com/_/nginx) with [PHP](https://hub.docker.com/_/php), and https via Letsencrypt and [Certbot](https://hub.docker.com/r/certbot/certbot/), based only on official Docker images.

The only non-official Docker imaged used in this setup is the `Dockerfile` in `data/php-fpm` which is needed to be able to install custom php libraries. Review it and make sure it doesn't contains any funny business, only `apk add` and `docker-php-ext-install` commands, and is based of the official PHP image (like `php:8.0xxxxx`).

Be up-and-running in 5 minutes, just follow the steps below to get started. 

## First time setup

### TLDR to set up a plain html server without a DNS name (development server)
```bash
# clone repo
git clone https://github.com/dahlo/nginx_letsencrypt_php.git ; cd nginx_letsencrypt_php

# copy nginx devel config
cp data/nginx/app.devel.conf.dist data/nginx/app.conf

# copy php dockerfile and modify if you need extra modules
cp data/php-fpm/Dockerfile.dist data/php-fpm/Dockerfile

# copy docker-compose.yml.dist and modify it if you need to change ports for the web server
cp docker-compose.yml.dist docker-compose.yml

# start finished server
docker-compose up -d
```


### TLDR to set up a SSL enabled server with a DNS name (production server)
```bash
# clone repo
git clone https://github.com/dahlo/nginx_letsencrypt_php.git ; cd nginx_letsencrypt_php

# copy nginx config and update example.com to your domain
cp data/nginx/app.conf.dist data/nginx/app.conf ; nano data/nginx/app.conf

# copy php dockerfile and modify if you need extra modules
cp data/php-fpm/Dockerfile.dist data/php-fpm/Dockerfile

# copy docker-compose.yml.dist
cp docker-compose.yml.dist docker-compose.yml

# get certificate, replace email and domain name
docker-compose up -d ; docker-compose exec certbot certbot certonly --webroot --webroot-path /var/www/certbot/ --agree-tos --non-interactive --email user@example.com -d example.com ; docker-compose down
 
# uncomment ssl server section in data/nginx/app.conf
nano data/nginx/app.conf

# download recommended ssl settings from certbot
curl -s https://raw.githubusercontent.com/certbot/certbot/master/certbot-nginx/certbot_nginx/_internal/tls_configs/options-ssl-nginx.conf > "data/certbot/conf/options-ssl-nginx.conf"
curl -s https://raw.githubusercontent.com/certbot/certbot/master/certbot/certbot/ssl-dhparams.pem > "data/certbot/conf/ssl-dhparams.pem"

# start finished server
docker-compose up -d --force-recreate
```

### Long version to set up a SSL enabled server with a DNS name (production server)

The first thing you need to do is to manually request a certificate from letsencrypt.

1. `git clone https://github.com/dahlo/nginx_letsencrypt_php.git ; cd nginx_letsencrypt_php`
1. Copy and modify the nginx config file
   1. Copy `data/nginx/app.conf.dist` to `data/nginx/app.conf`
   1. Change the domain name `example.com` to your own domain everywhere in the file (4 places).
1. Copy and modify the PHP Dockerfile
   1. Copy `data/php-fpm/Dockerfile.dist` to `data/php-fpm/Dockerfile`
   1. Modify `data/php-fpm/Dockerfile` to suit your needs, e.g. install and enable various PHP modules.
   1. Optionally, check which user id your user has (`id -u`) and modify the id given to the www-user inside the container, by editing the `usermod` command in the `Dockerfile`. This will minimize file ownership problems when you modify files in `data/html` from outside the container.
1. Get the certificate
   1. Temporarily start the web server, `docker-compose up -d`
   1. Run the following, and change email and domain name to match your, `docker-compose exec certbot certbot certonly --webroot --webroot-path /var/www/certbot/ --agree-tos --non-interactive --email user@example.com -d example.com`
   1. Shutdown the web server, `docker-compose down`
1. Make nginx use the certificate
   1. Uncomment the whole ssl server section in `data/nginx/app.conf`
   1. Download certbot's recommended SSL settings
      1. `curl -s https://raw.githubusercontent.com/certbot/certbot/master/certbot-nginx/certbot_nginx/_internal/tls_configs/options-ssl-nginx.conf > "data/certbot/conf/options-ssl-nginx.conf"`  
      1. `curl -s https://raw.githubusercontent.com/certbot/certbot/master/certbot/certbot/ssl-dhparams.pem > "data/certbot/conf/ssl-dhparams.pem"`
1. Start the web server, `docker-compose up -d --force-recreate`

Now the web server should be running and using the certificate you requested.

### Certificate renewal
The letsencrypt certificates has to be renewed every 3 months. The entrypoint command of the Certbot container in `docker-compose.yml` will have the container check every 30 days if renewal is needed and handle the renewal.

# Troubleshooting
## Docker containers not reaching the internet e.g. apt install servers

When running Docker in a virtual setting like OpenStack, there can be an issue with the network connectivity due to Dockers default value for MTU (Maximum Transmission Unit). The default value of the host computer is often 1500, and Docker will set its virtual network card to 1500 as well. Since Openstack has reserved 50 of the 1500 bytes, the Docker virtual network card will be larger than what Openstack can provide (1450). This breaks the internet connectivity of the container.

Check which MTU the host computer's network card has using `ip link`, then run the following command on the host computer with the MTU you have (1450 in the example below).

```bash
cat << EOF > /etc/docker/daemon.json
{
  "mtu": 1450
}
EOF
```

and restart the docker service.

From https://mlohr.com/docker-mtu/

> ## Docker MTU issues and solutions
> 
> October 16, 2018by Matthias Lohr
> 
> If you want to use Docker on servers or virtual machines, technical limitations can sometimes lead to a situation in which – even without intentional limitation – it is not possible to access the outer world from a docker container.
> 
> 
> 
> ## Docker MTU configuration
> 
> A common problem when operating dockers within a virtualization infrastructure is that the network cards provided to virtual machines do not have the default MTU of 1500. This is often the case, for example, when working in a cloud infrastructure (e.g. OpenStack). The Docker Daemon does not check the MTU of the outgoing connection at startup. Therefore, the value of the Docker MTU is set to 1500.
> 
> ### Solving the problem (docker daemon)
> 
> To solve the problem, you need to configure the Docker daemon in such a way that the virtual network card of newly created containers gets an MTU that is smaller than or equal to that of the outgoing network card. For this purpose create the file /etc/docker/daemon.json with the following content:
> 
> ```{
> 
>   "mtu": 1454
> 
> }
> ```
> In this example, I chose 1454 as the value, as this corresponds to the value of the outgoing network card (ens3). After restarting the Docker daemon, the MTU of new containers should be adapted accordingly. However, docker-compose create a new (bridge) network for every docker-compose environment by default.
> 
> ### Solving the problem (docker-compose)
> 
> ```
> networks:                                
>   default:                               
>     driver: bridge                       
>     driver_opts:                         
>       com.docker.network.driver.mtu: 1454
> ```
