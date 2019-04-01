# build this
# docker build -t rbphp72-apache-baseimage .

FROM php:7.2.15-apache-stretch

RUN apt-get update && apt-get install -y ssmtp

RUN docker-php-ext-install mysqli && docker-php-ext-enable mysqli

RUN apt-get update && apt-get install -y \
        libfreetype6-dev \
        libjpeg62-turbo-dev \
        libpng-dev \
    && docker-php-ext-install -j$(nproc) iconv \
    && docker-php-ext-configure gd --with-freetype-dir=/usr/include/ --with-jpeg-dir=/usr/include/ \
    && docker-php-ext-install -j$(nproc) gd

RUN pecl channel-update pecl.php.net

RUN pecl install redis-4.2.0 \
	&& docker-php-ext-enable redis
	
RUN \
  curl -L https://download.newrelic.com/php_agent/release/newrelic-php5-8.5.0.235-linux.tar.gz | tar -C /tmp -zx && \
	NR_INSTALL_PHPLIST=/usr/local/bin NR_INSTALL_USE_CP_NOT_LN=1 NR_INSTALL_SILENT=1 /tmp/newrelic-php5-*/newrelic-install install && \
      rm -rf /tmp/newrelic-php5-* /tmp/nrinstall*

RUN a2enmod headers \
	&& a2enmod rewrite \
	&& a2enmod info

COPY apache2-startup /usr/local/bin/
RUN ["chmod", "+x", "/usr/local/bin/apache2-startup"]

# copy apache2.conf
RUN mv /etc/apache2/apache2.conf /etc/apache2/apache2.conf.dist
COPY apache2.conf /etc/apache2/apache2.conf

# copy mpm_prefork.conf
RUN mv /etc/apache2/mods-available/mpm_prefork.conf /etc/apache2/mods-available/mpm_prefork.conf.dist
COPY mpm_prefork.conf /etc/apache2/mods-available/mpm_prefork.conf

RUN mkdir -p /etc/apache2/vhost  \
	&& mkdir -p /root/vhost \
	&& mkdir -p /root/config \
	&& mkdir -p /root/newrelic \
	&& mkdir -p /root/php \
	&& mkdir -p /root/ssmtp
