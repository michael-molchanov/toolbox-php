ARG VARIANT_VERSION=0.35.1
ARG UNICONF_VERSION=0.1.7
ARG GOMPLATE_VERSION=v3.5.0
ARG YQ_VERSION=2.4.0
ARG TOOLBOX_VERSION=0.2.1

FROM aroq/variant:$VARIANT_VERSION as variant
FROM aroq/uniconf:$UNICONF_VERSION as uniconf
FROM hairyhenderson/gomplate:$GOMPLATE_VERSION as gomplate
FROM mikefarah/yq:$YQ_VERSION as yq
FROM aroq/toolbox:$TOOLBOX_VERSION as toolbox
FROM golang:1-alpine as builder

FROM php:7.3-cli-alpine

LABEL maintainer "Michael Molchanov <mmolchanov@adyax.com>"

USER root

# SSH config.
RUN mkdir -p /root/.ssh
ADD config/ssh /root/.ssh/config
RUN chown root:root /root/.ssh/config && chmod 600 /root/.ssh/config

COPY --from=yq /usr/bin/yq /usr/bin/yq
COPY --from=variant /usr/bin/variant /usr/bin/
COPY --from=gomplate /gomplate /usr/bin/
COPY --from=uniconf /uniconf/uniconf /usr/bin/uniconf
COPY --from=toolbox /usr/bin/go-getter /usr/bin
COPY --from=toolbox /usr/local/bin/fd /usr/local/bin

RUN set -eux; apk add --no-cache --update \
  autoconf \
  automake \
  bash \
  build-base \
  bzip2-dev \
  ca-certificates \
  coreutils \
  curl \
  curl-dev \
  fftw-dev \
  freetype-dev \
  fuse \
  gawk \
  git \
  git-lfs \
  gmp-dev \
  grep \
  gzip \
  icu-dev \
  imagemagick-dev \
  imap-dev \
  jq \
  libbz2 \
  libde265-dev \
  libedit-dev \
  libffi-dev \
  libheif-dev \
  libimagequant-dev \
  libjpeg-turbo-dev \
  libmcrypt-dev \
  libpng-dev \
  libpq \
  libressl-dev \
  libstdc++ \
  libtool \
  libxml2-dev \
  libxslt-dev \
  libzip-dev \
  make \
  mercurial \
  mysql-client \
  nasm \
  oniguruma-dev \
  openldap-dev \
  openssh \
  openssh-client \
  patch \
  postgresql-client \
  postgresql-dev \
  procps \
  rsync \
  sqlite \
  sqlite-dev \
  strace \
  subversion \
  tar \
  tidyhtml-dev \
  tini \
  unzip \
  vips-dev \
  wget \
  x265-libs \
  zip \
  zlib-dev \
  && rm -rf /var/cache/apk/*


RUN apk add --update --no-cache --repository http://dl-cdn.alpinelinux.org/alpine/v3.13/community/ --allow-untrusted gnu-libiconv
ENV LD_PRELOAD /usr/lib/preloadable_libiconv.so php

# PHP modules.
RUN set -xeu \
  && apk update \
  && docker-php-ext-configure gd --with-freetype-dir=/usr/include/ --with-jpeg-dir=/usr/include/ --with-png-dir=/usr/include/ \
  && docker-php-ext-configure ldap --with-libdir=lib/ \
  && docker-php-ext-install -j$(nproc) \
    bcmath \
    bz2 \
    gd \
    gmp \
    imap \
    intl \
    ldap \
    mbstring \
    mysqli \
    opcache \
    pcntl \
    pdo \
    pdo_mysql \
    pdo_pgsql \
    pgsql \
    soap \
    sockets \
    tidy \
    xml \
    xsl \
    zip \
  && pecl install apcu \
  && pecl install channel://pecl.php.net/mcrypt-1.0.4 \
  && pecl install imagick \
  && pecl install redis \
  && docker-php-ext-enable apcu \
  && docker-php-ext-enable imagick \
  && docker-php-ext-enable mcrypt \
  && docker-php-ext-enable redis \
  && pecl clear-cache \
  && php -m \
  && runDeps="$( \
    scanelf --needed --nobanner --format '%n#p' --recursive /usr/local/lib/php/extensions \
      | tr ',' '\n' \
      | sort -u \
      | awk 'system("[ -e /usr/local/lib/" $1 " ]") == 0 { next } { print "so:" $1 }' \
    )" \
  && apk add --no-cache --update --virtual .composer-phpext-rundeps $runDeps \
  && rm -rf /var/cache/apk/* \
  && rm -rf /tmp/*


# Install MozJpeg.
ENV MOZJPEG_VERSION 3.3.1
RUN curl -fsSL -o mozjpeg.tar.gz "https://github.com/mozilla/mozjpeg/archive/v${MOZJPEG_VERSION}.tar.gz" \
  && tar -zxf mozjpeg.tar.gz \
  && cd "mozjpeg-${MOZJPEG_VERSION}" \
  && autoreconf -fiv \
  && ./configure --prefix=/usr \
  && make install\
  && cd .. \
  && rm -rf "mozjpeg-${MOZJPEG_VERSION}"

RUN printf "# composer php cli ini settings\n\
date.timezone=UTC\n\
memory_limit=-1\n\
" > $PHP_INI_DIR/php-cli.ini

ENV COMPOSER_ALLOW_SUPERUSER 1

# Install composer
RUN wget https://getcomposer.org/installer \
    && php installer --install-dir=/usr/local/bin/ --filename=composer --version=1.10.20 \
    && rm installer \
    && composer global --no-progress --no-interaction require hirak/prestissimo

# Set the Drush version.
ENV DRUSH_VERSION 8.4.8

# Install Drush 8 with the phar file.
RUN curl -fsSL -o /usr/local/bin/drush "https://github.com/drush-ops/drush/releases/download/$DRUSH_VERSION/drush.phar" && \
  chmod +x /usr/local/bin/drush

# Test your install.
RUN drush core-status

# Add git-secret package from edge testing
RUN apk add --no-cache -X http://dl-cdn.alpinelinux.org/alpine/edge/testing git-secret

# Install docman.
RUN apk add --update --no-cache ruby ruby-dev \
  && rm -rf /var/cache/apk/* \
  && gem install --no-doc etc \
  && gem install --no-doc -v 0.0.107 docman

RUN mkdir -p /toolbox
COPY --from=toolbox /toolbox/* /toolbox/
