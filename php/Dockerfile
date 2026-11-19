FROM php:8.5-apache

RUN docker-php-ext-install pdo_mysql

# Fuseau horaire = Paris
ENV TZ=Europe/Paris
COPY timezone.ini /usr/local/etc/php/conf.d/timezone.ini

# Activation Browscap
ADD https://browscap.org/stream?q=PHP_BrowsCapINI /usr/local/etc/php/conf.d/browscap.ini
COPY php-browscap.ini /usr/local/etc/php/conf.d/

# IP réelle du client derrière un reverse proxy (X-Forwarded-For)
COPY remoteip.conf /etc/apache2/conf-available/remoteip.conf
RUN a2enmod remoteip && a2enconf remoteip

COPY www/ /var/www/html
