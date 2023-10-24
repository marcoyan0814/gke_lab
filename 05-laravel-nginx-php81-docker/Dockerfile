FROM php:8.1-fpm-alpine

WORKDIR /var/www/html

RUN mkdir -p /var/www/html/laravel/public

COPY ./supervisord.conf /etc/supervisord.conf
COPY ./nginx/nginx.conf /etc/nginx/nginx.conf
COPY ./nginx/conf.d/default.conf /etc/nginx/http.d/default.conf
COPY ./nginx/index.html /var/www/html/laravel/public/index.html

RUN apk update \
    # install pkg
    && apk add --no-cache git zip unzip dos2unix supervisor libpng-dev libzip-dev freetype-dev $PHPIZE_DEPS libjpeg-turbo-dev tzdata \
    # install composer
    && curl -sS https://getcomposer.org/installer | php -- --install-dir=/usr/local/bin --filename=composer \
    && composer --ansi --version --no-interaction \
    # install php extension
    && docker-php-ext-install gd pcntl bcmath mysqli pdo_mysql \
    # php extension setting
    && docker-php-ext-configure gd --with-freetype --with-jpeg \
    # use pecl install php extension
    && pecl install zip \
    && docker-php-ext-enable zip \
    && pecl install igbinary \
    && docker-php-ext-enable igbinary \
    # install imagemagick
    && apk add imagemagick imagemagick-dev \
    && printf "\n" | pecl install -o -f imagick \
    && docker-php-ext-enable imagick \
    && php -m | grep imagick \
    # install nginx & setting log
    && apk add nginx \
    && ln -sf /dev/stdout /var/log/nginx/access.log \
    && ln -sf /dev/stderr /var/log/nginx/error.log

# test crond
COPY production/test_cron.sh /tmp/test_cron.sh
COPY production/crontab /var/spool/cron/crontabs/www-data
RUN chown www-data:www-data /tmp/test_cron.sh

# install Laravel
ENV APP_PATH=/var/www/html/laravel
ENV TZ=Asia/Taipei

RUN rm -rf $APP_PATH
RUN composer create-project laravel/laravel laravel
RUN cd laravel \
    && cp .env.example .env \
    && php artisan key:generate \
    && composer install \
        --no-dev \
        --no-interaction \
        --prefer-dist \
        --ignore-platform-reqs \
        --optimize-autoloader \
        --apcu-autoloader \
        --ansi \
        --no-scripts \
        --audit \
    && composer dump-autoload -o \
    && chown www-data:www-data -R $APP_PATH \
    && chmod 775 -R $APP_PATH \
    && chmod o+w -R $APP_PATH/storage \
    && php artisan storage:link \
    && php artisan optimize:clear

RUN cd $APP_PATH && php artisan --version
RUN php -v
RUN php -m
RUN nginx -v

EXPOSE 80

# container run supervisor
CMD ["/usr/bin/supervisord", "-n", "-c", "/etc/supervisord.conf"]
# test nginx
# CMD ["/usr/sbin/nginx", "-g", "daemon off;"]
