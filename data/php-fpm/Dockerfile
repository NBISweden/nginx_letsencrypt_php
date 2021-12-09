FROM php:8.0-fpm
WORKDIR "/www"

RUN apt-get update; \
    apt-get -y --no-install-recommends install \
        php8.0-bz2 \ 
        php8.0-gd \ 
        php8.0-memcached \
        git; \
    apt-get clean; \
    rm -rf /var/lib/apt/lists/* /tmp/* /var/tmp/* /usr/share/doc/*
