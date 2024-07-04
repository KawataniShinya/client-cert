FROM php:8.3.7-fpm-bookworm

COPY --from=composer:2.6.6 /usr/bin/composer /usr/bin/composer

RUN apt update
RUN apt install -y procps vim less git zip unzip 
RUN docker-php-ext-install pdo_mysql

RUN pecl install xdebug-3.3.2
RUN docker-php-ext-enable xdebug

COPY ./xdebug/php-xdebug.ini /usr/local/etc/php/conf.d/docker-php-ext-xdebug.ini

WORKDIR "/var/www/app"