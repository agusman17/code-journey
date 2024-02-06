# Docker Laravel

Example docker laravel use php, nginx and mysql

## Folder Structure
This is how i set folder

```
root project folder
├── .docker
│   ├── db
│   |── logs
|   |-- nginx
|   |__ php
|
├── src/
└── docker-compose.yml/
```

## Database

in this example use mysql version 8.1. include cnf file in folder 

#### .docker/sql/my.cnf
```
[mysqld]
character-set-server = utf8mb4
collation-server = utf8mb4_bin

default-authentication-plugin = mysql_native_password

log-error = /var/log/mysql/mysql-error.log

slow_query_log = 1
slow_query_log_file = /var/log/mysql/mysql-slow.log
long_query_time = 5.0
log_queries_not_using_indexes = 0

general_log = 1
general_log_file = /var/log/mysql/mysql-query.log

[mysql]
default-character-set = utf8mb4

[client]
default-character-set = utf8mb4
```

## Web Server

in this example use nginx. include default.conf and nginx.conf file in folder 

#### .docker/nginx/default.conf
```
server {
  listen 80;
  index index.php index.html;
  root /var/www/public;

  client_max_body_size 100M; # 413 Request Entity Too Large

  location / {
    root /var/www/public;
    index  index.html index.php;
    try_files $uri $uri/ /index.php?$query_string;
  }

  location ~ \.php$ {
    try_files $uri =404;
    fastcgi_split_path_info ^(.+\.php)(/.+)$;
    fastcgi_pass php:9000;
    fastcgi_read_timeout 3600;
    fastcgi_index index.php;
    include fastcgi_params;
    fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
    fastcgi_param PATH_INFO $fastcgi_path_info;
    send_timeout 3600;
    proxy_connect_timeout 3600;
    proxy_read_timeout    3600;
    proxy_send_timeout    3600;
  }
}
```

#### .docker/nginx/nginx.conf
```
user  nginx;
worker_processes  1;

error_log  /var/log/nginx/error.log warn;
pid        /var/run/nginx.pid;

events {
    worker_connections  1024;
}

http {
    include       /etc/nginx/mime.types;
    default_type  application/octet-stream;

    log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
                      '$status $body_bytes_sent "$http_referer" '
                      '"$http_user_agent" "$http_x_forwarded_for"';

    access_log  /var/log/nginx/access.log  main;

    sendfile        on;
    #tcp_nopush     on;

    keepalive_timeout  65;

    server_tokens off;
    chunked_transfer_encoding off;

    gzip on;
    gzip_types application/json;
    gzip_min_length 1000;

    include /etc/nginx/conf.d/*.conf;
}
```

## PHP

in this example use php 8.2. include configuration and Dockerfile
- .bashrc
- docker.conf
- Dockerfile
- php.ini

#### .docker/php/.bashrc

```
# ~/.bashrc: executed by bash(1) for non-login shells.

# Note: PS1 and umask are already set in /etc/profile. You should not
# need this unless you want different defaults for root.
PS1='${debian_chroot:+($debian_chroot)}\h:\w\$ '
umask 022

# You may uncomment the following lines if you want `ls' to be colorized:
export SHELL
export LS_OPTIONS='--color=auto'
eval $(dircolors ~/dircolors-solarized/dircolors.256dark)
alias ls='ls $LS_OPTIONS'
alias ll='ls $LS_OPTIONS -l'
alias l='ls $LS_OPTIONS -lA'

# Some more alias to avoid making mistakes:
alias rm='rm -i'
alias cp='cp -i'
alias mv='mv -i'

alias phpd='php -dzend_extension=xdebug.so -dxdebug.mode=debug -dxdebug.idekey=PHPSTORM -dxdebug.start_with_request=yes -dxdebug.client_host=host.docker.internal -dxdebug.client_port=9001'
```


#### docker.conf

```
[global]
error_log = /proc/self/fd/2
;request_terminate_timeout = 1h

[www]
; if we send this to /proc/self/fd/1, it never appears
access.log = /proc/self/fd/1

clear_env = no

; Ensure worker stdout and stderr are sent to the main error log.
catch_workers_output = yes
```

#### Dockerfile

```
FROM php:8.2-fpm

COPY php.ini /usr/local/etc/php/
COPY docker.conf /usr/local/etc/php-fpm.d/docker.conf
COPY .bashrc /root/

RUN apt-get update \
  && apt-get install -y build-essential zlib1g-dev default-mysql-client curl gnupg procps vim git unzip libzip-dev libpq-dev \
  && docker-php-ext-install zip pdo_mysql pdo_pgsql pgsql

RUN apt-get install -y libicu-dev \
&& docker-php-ext-configure intl \
&& docker-php-ext-install intl

# pcov
RUN pecl install pcov && docker-php-ext-enable pcov

# Xdebug
# RUN pecl install xdebug \
# && docker-php-ext-enable xdebug \
# && echo ";zend_extension=xdebug" > /usr/local/etc/php/conf.d/docker-php-ext-xdebug.ini

# Node.js, NPM, Yarn
RUN curl -sL https://deb.nodesource.com/setup_18.x | bash -
RUN apt-get install -y nodejs
RUN npm install npm@latest -g
RUN npm install yarn -g

# Composer
RUN php -r "copy('https://getcomposer.org/installer', 'composer-setup.php');"
RUN php composer-setup.php
RUN php -r "unlink('composer-setup.php');"
RUN mv composer.phar /usr/local/bin/composer

ENV COMPOSER_ALLOW_SUPERUSER 1
ENV COMPOSER_HOME /composer
ENV PATH $PATH:/composer/vendor/bin
RUN composer config --global process-timeout 3600
RUN composer global require "laravel/installer"

WORKDIR /root
RUN git clone https://github.com/seebi/dircolors-solarized

EXPOSE 5173
WORKDIR /var/www
```

#### php.ini

```
; Example

; [Date]
; date.timezone = "Asia/Jakarta"

; [mbstring]
; mbstring.language = English
```

## Docker Compose
And finnaly this is docker-compose.yml

```
version: '3'
name: laravel-project

services:

    ####################################################################################################
    # PHP
    ####################################################################################################
    php:
        build: .docker/php
        ports:
            - 5173:5173
        volumes:
            - ./sources:/var/www:cached
        networks:
            - laravel

    ####################################################################################################
    # Nginx
    ####################################################################################################
    nginx:
        image: nginx
        ports:
            - 8082:80
        volumes:
            - ./sources:/var/www
            - .docker/nginx/default.conf:/etc/nginx/conf.d/default.conf
            - .docker/nginx/nginx.conf:/etc/nginx/nginx.conf
        depends_on:
            - php
        networks:
            - laravel

    ####################################################################################################
    # DATABASE (MySQL)
    ####################################################################################################
    db:
        image: mysql:8.1
        ports:
            - 3306:3306
        volumes:
            - .docker/db/data:/var/lib/mysql
            - .docker/logs:/var/log/mysql
            - .docker/db/my.cnf:/etc/mysql/conf.d/my.cnf
            - .docker/db/sql:/docker-entrypoint-initdb.d
        environment:
            MYSQL_ROOT_PASSWORD: root
            MYSQL_DATABASE: udsodara
            MYSQL_USER: user_dev
            MYSQL_PASSWORD: user_dev
        networks:
            - laravel

    # ####################################################################################################
    # # DATABASE (MariaDB)
    # ####################################################################################################
    # db:
    #     image: mariadb:10.11
    #     ports:
    #         - 3306:3306
    #     volumes:
    #         - .docker/db/data:/var/lib/mysql
    #         - .docker/logs:/var/log/mysql
    #         - .docker/db/my.cnf:/etc/mysql/conf.d/my.cnf
    #         - .docker/db/sql:/docker-entrypoint-initdb.d
    #     environment:
    #         MYSQL_ROOT_PASSWORD: root
    #         MYSQL_DATABASE: laravel_db_name
    #         MYSQL_USER: laravel_db_user
    #         MYSQL_PASSWORD: laravel_db_pass

    ####################################################################################################
    # phpMyAdmin
    ####################################################################################################
    # phpmyadmin:
    #     image: phpmyadmin/phpmyadmin
    #     ports:
    #         - 8080:80
    #     links:
    #         - db
    #     environment:
    #         PMA_HOST: db
    #         PMA_PORT: 3306
    #         PMA_ARBITRARY: 1
    #     volumes:
    #         - .docker/phpmyadmin/sessions:/sessions

    ####################################################################################################
    # Mailpit
    ####################################################################################################
    # mail:
    #     image: axllent/mailpit:latest
    #     ports:
    #     - 8025:8025
    #     - 1025:1025

networks:
    laravel:
        name: laravel
```