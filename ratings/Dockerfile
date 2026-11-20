#
# Build the app
#
FROM docker.io/library/php:7.4-apache

# bullseye-security pool is inconsistent (404s); install from the main bullseye repo
RUN sed -i "/debian-security/d" /etc/apt/sources.list \
    && apt-get update && apt-get install -yqq unzip libzip-dev \
    && docker-php-ext-install pdo_mysql opcache zip

# Enable AutoProfile for PHP which is currently opt-in beta
RUN echo "instana.enable_auto_profile=1" > "/usr/local/etc/php/conf.d/zzz-instana-extras.ini"

# relax permissions on status
COPY status.conf /etc/apache2/mods-available/status.conf
# Enable Apache mod_rewrite and status
RUN a2enmod rewrite && a2enmod status

WORKDIR /var/www/html

COPY html/ /var/www/html

COPY --from=docker.io/library/composer:2 /usr/bin/composer /usr/bin/composer
RUN composer install

# This is important. Symfony needs write permissions and we
# dont know the context in which the container will run, i.e.
# which user will be forced from the outside so better play
# safe for this simple demo.
RUN rm -Rf /var/www/var/*
RUN chown -R www-data /var/www
RUN chmod -R 777 /var/www

