FROM debian:jessie
LABEL MAINTAINER="tiechuan.wang@gmail.com"

# persistent / runtime deps
RUN apt-get update && apt-get install -y --no-install-recommends \
      ca-certificates \
      curl \
      ssmtp \
      librecode0 \
      libmysqlclient-dev \
      libsqlite3-0 \
      libxml2 \
	  libfreetype6-dev \
    && apt-get clean \
    && rm -r /var/lib/apt/lists/*

# phpize deps
RUN apt-get update && apt-get install -y --no-install-recommends \
      autoconf \
      file \
      g++ \
      gcc \
      libc-dev \
      make \
      pkg-config \
      re2c \
    && apt-get clean \
    && rm -r /var/lib/apt/lists/*

##<apache2>##
RUN apt-get update && apt-get install -y apache2-bin apache2-dev apache2.2-common --no-install-recommends && rm -rf /var/lib/apt/lists/*

RUN rm -rf /var/www/html && mkdir -p /var/lock/apache2 /var/run/apache2 /var/log/apache2 /var/www/html && chown -R www-data:www-data /var/lock/apache2 /var/run/apache2 /var/log/apache2 /var/www/html

# Apache + PHP requires preforking Apache for best results
RUN a2dismod mpm_event && a2enmod mpm_prefork

##</apache2>##

ENV PHP_INI_DIR /etc/php5/apache2
RUN mkdir -p $PHP_INI_DIR/conf.d

# HKP, the "HTTP keyserver protocol", is normally spoken on a different port (11371/TCP) than regular HTTP. It could be that you're behind a firewall which blocks all but the most common ports.
# Many keyservers do accept HKP on the regular HTTP port, and some of them also work with HKPS (which just shares the HTTPS port). Therefore try one of the following keyservers:
# hkp://pgp.mit.edu:80
# hkps://pgp.mit.edu
# hkp://p80.pool.sks-keyservers.net:80
# hkps://hkps.pool.sks-keyservers.net

ENV GPG_KEYS 0B96609E270F565C13292B24C13C70B87267B52D 0A95E9A026542D53835E3F3A7DEC4E69FC9C83D7 0E604491
RUN set -xe \
  && for key in $GPG_KEYS; do \
    gpg --keyserver hkp://p80.pool.sks-keyservers.net:80 --recv-keys "$key"; \
  done

# fix --freetype 
RUN mkdir /usr/include/freetype2/freetype
RUN ln -s /usr/include/freetype2/freetype.h /usr/include/freetype2/freetype/freetype.h

# compile openssl, otherwise --with-openssl won't work
RUN CFLAGS="-fPIC" && OPENSSL_VERSION="1.0.2d" \
      && cd /tmp \
      && mkdir openssl \
      && curl -sL "https://www.openssl.org/source/openssl-$OPENSSL_VERSION.tar.gz" -o openssl.tar.gz \
      && curl -sL "https://www.openssl.org/source/openssl-$OPENSSL_VERSION.tar.gz.asc" -o openssl.tar.gz.asc \
      && gpg --verify openssl.tar.gz.asc \
      && tar -xzf openssl.tar.gz -C openssl --strip-components=1 \
      && cd /tmp/openssl \
      && ./config -fPIC && make && make install \
      && rm -rf /tmp/*

ENV PHP_VERSION 5.3.29

# php 5.3 needs older autoconf
# --enable-mysqlnd is included below because it's harder to compile after the fact the extensions are (since it's a plugin for several extensions, not an extension in itself)
RUN buildDeps=" \
                apache2-dev \
                autoconf2.13 \
                libcurl4-openssl-dev \
                libreadline6-dev \
                librecode-dev \
                libsqlite3-dev \
                libssl-dev \
                libxml2-dev \
                libpng-dev \
                xz-utils \
      " \
      && set -x \
      && apt-get update && apt-get install -y $buildDeps --no-install-recommends && rm -rf /var/lib/apt/lists/* \
      && curl -SL "http://php.net/get/php-$PHP_VERSION.tar.xz/from/this/mirror" -o php.tar.xz \
      && curl -SL "http://php.net/get/php-$PHP_VERSION.tar.xz.asc/from/this/mirror" -o php.tar.xz.asc \
      && gpg --verify php.tar.xz.asc \
      && mkdir -p /usr/src/php \
      && tar -xof php.tar.xz -C /usr/src/php --strip-components=1 \
      && rm php.tar.xz* \
      && cd /usr/src/php \
      && ./configure --disable-cgi \
            $(command -v apxs2 > /dev/null 2>&1 && echo '--with-apxs2=/usr/bin/apxs2' || true) \
            --with-config-file-path="$PHP_INI_DIR" \
            --with-config-file-scan-dir="$PHP_INI_DIR/conf.d" \
            --enable-ftp \
            --enable-mbstring \
            --enable-mysqlnd \
            --with-mysql \
            --with-mysqli \
            --with-pdo-mysql \
            --with-curl \
            --with-openssl=/usr/local/ssl \
            --enable-soap \
            --with-png \
            --with-gd \
            --with-readline \
            --with-recode \
            --with-zlib \
			--with-freetype-dir=/usr \
      && make -j"$(nproc)" \
      && make install \
      && { find /usr/local/bin /usr/local/sbin -type f -executable -exec strip --strip-all '{}' + || true; } \
      && apt-get purge -y --auto-remove -o APT::AutoRemove::RecommendsImportant=false -o APT::AutoRemove::SuggestsImportant=false $buildDeps \
      && make clean

RUN echo "default_charset = " > $PHP_INI_DIR/php.ini \
    && echo "date.timezone = Australia/Melbourne" >> $PHP_INI_DIR/php.ini

COPY docker-php-* /usr/local/bin/
COPY apache2-foreground /usr/local/bin/
RUN ["chmod", "+x", "/usr/local/bin/apache2-foreground"]
COPY apache2-startup /usr/local/bin/
RUN ["chmod", "+x", "/usr/local/bin/apache2-startup"]

RUN mkdir -p /etc/httpd/vhost  \
	&& mkdir -p /root/vhost \
	&& mkdir -p /root/config \
	&& mkdir -p /root/newrelic \
	&& mkdir -p /root/php \
	&& mkdir -p /root/ssmtp

# copy apache2.conf
RUN mv /etc/apache2/apache2.conf /etc/apache2/apache2.conf.dist
COPY apache2.conf /etc/apache2/apache2.conf

# copy mpm_prefork.conf
RUN mv /etc/apache2/mods-available/mpm_prefork.conf /etc/apache2/mods-available/mpm_prefork.conf.dist
COPY mpm_prefork.conf /etc/apache2/mods-available/mpm_prefork.conf

RUN mkdir -p /usr/local/lib/php/extensions/no-debug-non-zts-20090626

RUN \
  curl -L https://download.newrelic.com/php_agent/release/newrelic-php5-8.3.0.226-linux.tar.gz | tar -C /tmp -zx && \
	NR_INSTALL_PHPLIST=/usr/local/bin NR_INSTALL_USE_CP_NOT_LN=1 NR_INSTALL_SILENT=1 /tmp/newrelic-php5-*/newrelic-install install && \
      rm -rf /tmp/newrelic-php5-* /tmp/nrinstall*

RUN pecl channel-update pecl.php.net
RUN pecl install redis

RUN a2enmod headers \
	&& a2enmod rewrite \
	&& a2enmod ssl \
	&& a2enmod info

EXPOSE 80
#EXPOSE 443

WORKDIR /var/www/html

ENV APACHE_ENVIRONMENT=localhost

CMD apache2-foreground
