# A simple installation of Nginx with PHP, and https via Letsencrypt, based only on official Docker images
The only non-official Docker imaged used in this setup is the `Dockerfile` in `data/php-fpm` which is needed to be able to install custom php libraries. View it and make sure it only contains `apt install` commands, and is based of the official [PHP image](https://hub.docker.com/_/php).

Be up-and-running in 5 minutes, just follow the steps below to get started. 

## First time setup
The first thing you need to do is to manually request a certificate from letsencrypt.

1. `git clone https://github.com/dahlo/nginx_letsencrypt_php.git ; cd nginx_letsencrypt_php`
1. Rename and modify the nginx config file
   1. `Rename data/nginx/app.conf.dist` to `data/nginx/app.conf`
   1. Change the domain name `example.com` to your own domain everywhere in the file (4 places).
1. Get the certificate
   1. Temporarily start the web server, `docker-compose up -d`
   1. Run the following, and change email and domain name to match your, `docker-compose run --rm  certbot certonly --webroot --webroot-path /var/www/certbot/ --agree-tos --non-interactive --email user@example.com -d example.com` 
   1. Shutdown the web server, `docker-compose down`
1. Make nginx use the certificate
   1. Uncomment the whole ssl server section in `data/nginx/app.conf`
   1. Download certbot's recommended SSL settings
      1. `curl -s https://raw.githubusercontent.com/certbot/certbot/master/certbot-nginx/certbot_nginx/_internal/tls_configs/options-ssl-nginx.conf > "data/certbot/conf/options-ssl-nginx.conf"`  
      1. `curl -s https://raw.githubusercontent.com/certbot/certbot/master/certbot/certbot/ssl-dhparams.pem > "data/certbot/conf/ssl-dhparams.pem"`
   1. Start the web server, `docker-compose up -d`

Now the web server should be running and using the certificate you requested.

### Certificate renewal
The letsencrypt certificates has to be renewed every 3 months. To automate this process, create a cronjob that tries to renew the cert once every month. If renewal is not need it will not try to do it.

```bash
# edit crontab
crontab -e

# insert this entry
0 2 1 * * cd /path/to/repo ; docker-compose run --rm certbot renew
```

## PHP config
Add new modules to be install in `data/php-fpm/Dockerfile` and enable them in `data/php-fpm/php-ini-overrides.ini`

# Troubleshooting
## Docker containers not reaching the internet e.g. apt install servers

When running Docker in a virtual setting like OpenStack, it will prioritize the wrong network within the container if the network interface uses a non-standard MTU. Check which MTU the virtual network card has using `ip link`, then run the following command with the MTU you have (1400 in the example below).

```bash
cat << EOF > /etc/docker/daemon.json
{
  "mtu": 1400
}
EOF
```

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
