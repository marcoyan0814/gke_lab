FROM alpine:latest

WORKDIR /var/www/html/

# Essentials
RUN apk add --no-cache zip unzip curl sqlite nginx \
    supervisor git tzdata mysql-client openssh \
    busybox-extras vim
RUN apk add --no-cache tzdata
ENV TZ=Asia/Taipei

# Installing bash
RUN apk add bash
RUN sed -i 's/bin\/ash/bin\/bash/g' /etc/passwd

RUN mkdir -p /var/www/html/web/public

# Installing PHP
RUN apk add --no-cache php81 \
    php81-common \
    php81-gd \
    php81-fpm \
    php81-pdo \
    php81-opcache \
    php81-zip \
    php81-phar \
    php81-iconv \
    php81-cli \
    php81-curl \
    php81-openssl \
    php81-mbstring \
    php81-tokenizer \
    php81-fileinfo \
    php81-json \
    php81-xml \
    php81-xmlwriter \
    php81-simplexml \
    php81-dom \
    php81-pdo_mysql \
    php81-pdo_sqlite \
    php81-tokenizer \
    php81-pecl-redis

# Installing composer
RUN curl -sS https://getcomposer.org/installer -o composer-setup.php
RUN php composer-setup.php --install-dir=/usr/local/bin --filename=composer
RUN rm -rf composer-setup.php

# Configure supervisor
RUN mkdir -p /etc/supervisor.d/
COPY conf/supervisord.ini /etc/supervisor.d/supervisord.ini

COPY conf/php-fpm.conf /etc/php81/php-fpm.conf
COPY conf/php.ini-production /etc/php81/php.ini

# Configure nginx
COPY conf/nginx.conf /etc/nginx/
COPY conf/http.d/default.conf /etc/nginx/http.d/
COPY conf/http.d/cdn.conf /etc/nginx/http.d/
COPY conf/index.html /var/www/html/web/public/index.html

RUN mkdir -p /run/nginx/
RUN touch /run/nginx/nginx.pid

RUN ln -sf /dev/stdout /var/log/nginx/access.log
RUN ln -sf /dev/stderr /var/log/nginx/error.log

# Building process
RUN chown nobody:nobody -R /var/lib/nginx
RUN chmod 777 -R /var/lib/nginx
RUN chown -R nobody:nobody /var/www/html/web
RUN chmod 775 -R /var/www/html/web

RUN php -v
RUN php -m
RUN nginx -v
RUN composer --version

EXPOSE 80
CMD ["supervisord", "-c", "/etc/supervisor.d/supervisord.ini"]
