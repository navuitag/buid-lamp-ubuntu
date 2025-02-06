# Install New Ubuntu Server (Version 20)

## 1. AWS New Instance Launch script

~~~bash
  #! /bin/bash

  sudo apt update -y
  sudo apt-get update -y
  sudo apt install -y curl unzip
  sudo apt install -y nginx nginx-extras
  sudo apt install -y php-cli php-common php-curl php-fpm php-gd php-intl php-json php-mbstring php-mysql php-opcache php-readline php-soap php-xml php-xmlrpc php-zip
  sudo apt install -y nodejs npm
~~~

## 2. Server configuration

### 1. PHP

~~~bash
  # max_execution_time = 1800
  # memory_limit = 2G
  # upload_max_filesize = 20M
  # zlib.output_compression = On

  sudo vi /etc/php/7.4/fpm/php.ini
  sudo vi /etc/php/7.4/cli/php.ini

  # request_terminate_timeout = 1800
  sudo vi /etc/php/7.4/fpm/pool.d/www.conf

  sudo systemctl stop php7.4-fpm && sudo systemctl start php7.4-fpm && sudo systemctl enable php7.4-fpm
~~~

### 2. Nginx

* Modify nginx config

~~~bash
  sudo vi /etc/nginx/nginx.conf
~~~

~~~conf
        sendfile on;
        tcp_nopush on;
        tcp_nodelay on;
        keepalive_timeout 600s;
        types_hash_max_size 2048;
        client_max_body_size 100M;
        client_header_timeout 3000;
        client_body_timeout 3000;
        server_tokens off;

        fastcgi_read_timeout 600s;

        proxy_connect_timeout       600;
        proxy_send_timeout          600;
        proxy_read_timeout          600;
        send_timeout                600;
~~~

* Create header config

~~~bash
  sudo vi /etc/nginx/headers.conf
~~~

~~~conf
# Security Header

add_header Strict-Transport-Security 'max-age=31536000; includeSubDomains; preload';
add_header X-Frame-Options "DENY";
add_header X-Content-Type-Options nosniff;
add_header X-XSS-Protection "1; mode=block";
add_header 'Access-Control-Allow-Origin' '*';
#add_header Content-Security-Policy "script-src 'self' 'unsafe-inline' 'unsafe-eval' www.googletagmanager.com connect.facebook.net www.googleadservices.com www.google-analytics.com googleads.g.doubleclick.net onesignal.com tpc.googlesyndication.com;"; # use for WordPress
add_header Content-Security-Policy "default-src 'self'; font-src *; img-src *; script-src 'self' 'unsafe-inline'; style-src *";
add_header Referrer-Policy strict-origin;
add_header X-Permitted-Cross-Domain-Policies master-only;
add_header Expect-CT 'max-age=60, report-uri="https://support.euda.com"';
add_header Permissions-Policy "geolocation=(self),midi=(self),sync-xhr=(*),microphone=(self),camera=(self),magnetometer=(),gyroscope=(),fullscreen=(self),payment=()";
add_header Clear-Site-Data '"cache", "cookies", "storage"';
more_clear_headers Server;
~~~

* Create errors config

~~~bash
  sudo vi /etc/nginx/errors.conf
~~~

~~~conf
#site-wide error pages
error_page 401 402 403 404 /40x.html;
location = /40x.html {
       root   /var/www/html;
       allow all;
       internal;
}

error_page 500 502 503 504 /50x.html;
location = /50x.html {
       root   /var/www/html;
       allow all;
       internal;
}
~~~

* Create new host

~~~bash
  sudo vi /etc/nginx/sites-available/alice
~~~

~~~conf
upstream php-fpm {
    server unix:/var/run/php/php7.4-fpm.sock;
}

server {
    listen 80;
    listen [::]:80;

    server_name pentest.kentridge.health;
    root /var/www/alice-web2/public;
    #return 301 https://$host$request_uri;

    error_log /var/log/nginx/error.log;
    access_log /var/log/nginx/access.log;

    include /etc/nginx/headers.conf;
    include /etc/nginx/errors.conf;
    index index.php index.html index.htm;

    location / {
        try_files $uri $uri/ /index.php$is_args$args;
    }

    location ~ \.(php|phar)(/.*)?$ {
        fastcgi_split_path_info ^(.+\.(?:php|phar))(/.*)$;
        try_files $uri /index.php =404;

        fastcgi_intercept_errors on;
        fastcgi_pass   php-fpm;
        fastcgi_index  index.php;
        fastcgi_buffers 16 16k;
        fastcgi_buffer_size 32k;
        fastcgi_param  SCRIPT_FILENAME  $document_root$fastcgi_script_name;
        fastcgi_param  PATH_INFO $fastcgi_path_info;
        include fastcgi_params;
    }

    gzip on;
    gzip_disable "msie6";

    gzip_comp_level 6;
    gzip_min_length 1100;
    gzip_buffers 16 8k;
    gzip_proxied any;
    gzip_types
        text/plain
        text/css
        text/js
        text/xml
        text/javascript
        application/javascript
        application/x-javascript
        application/json
        application/xml
        application/xml+rss
        image/svg+xml;
    gzip_vary on;

    # Deny access to .htaccess, .git files
    location ~* (\.htaccess$|\.git) {
        deny all;
    }
}
~~~

* Restart nginx

~~~bash
  sudo systemctl stop nginx && sudo systemctl start nginx && sudo systemctl enable nginx
~~~

### 3. Install Composer

~~~bash
  curl -sS https://getcomposer.org/installer | sudo php -- --install-dir=/usr/local/bin --filename=composer
~~~

### 4. Git

~~~bash
  ssh-keygen -t ed25519 -C "vu@kentridgehealth.com"
  eval "$(ssh-agent -s)"
  ssh-add ~/.ssh/id_ed25519
  cat ~/.ssh/id_ed25519.pub
~~~

### 5. Clean Linux

~~~bash
sudo apt-get clean
sudo apt-get autoclean
sudo apt-get autoremove
sudo journalctl --vacuum-size 10M
~~~

### 6. Add new user

~~~bash
adduser alice
cat /etc/passwd | grep alice
groups alice
usermod -aG sudo alice
id alice
groups alice
~~~

### 7. MySQL

- Create new database

~~~bash
CREATE DATABASE `DB name` CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
~~~

- Create a new user and grant permission

~~~bash
USE `DB name`;
CREATE USER 'DB user'@'%' IDENTIFIED WITH mysql_native_password BY 'P@ssword!123#';
GRANT ALL ON `DB name`.* TO 'DB user'@'%';
FLUSH PRIVILEGES;
~~~
