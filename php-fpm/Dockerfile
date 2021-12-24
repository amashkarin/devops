FROM alpine:3.15.0

ENV PHPIZE_DEPS="\
autoconf \
dpkg-dev dpkg \
file \
g++ \
gcc \
libc-dev \
make \
pkgconf \
re2c \
"

RUN apk add --no-cache \
ca-certificates \
curl \
tar \
xz \
openssl
RUN set -eux; \
adduser -u 82 -D -S -G www-data www-data


ENV PHP_INI_DIR=/usr/local/etc/php
RUN set -eux; \
mkdir -p "$PHP_INI_DIR/conf.d"; \
[ ! -d /var/www/html ]; \
mkdir -p /var/www/html; \
chown www-data:www-data /var/www/html; \
chmod 777 /var/www/html

ENV PHP_CFLAGS="-fstack-protector-strong -fpic -fpie -O2 -D_LARGEFILE_SOURCE -D_FILE_OFFSET_BITS=64"
ENV PHP_CPPFLAGS="-fstack-protector-strong -fpic -fpie -O2 -D_LARGEFILE_SOURCE -D_FILE_OFFSET_BITS=64"
ENV PHP_LDFLAGS="-Wl,-O1 -pie"
ENV GPG_KEYS="528995BFEDFBA7191D46839EF9BA0ADA31CBD89E 39B641343D8C104B2B146DC3F9C39DC0B9698544 F1F692238FBC1666E5A5CCD4199F9DFEF6FFBAFD"
ENV PHP_VERSION=8.1.1
ENV PHP_URL="https://www.php.net/distributions/php-8.1.1.tar.xz"
ENV PHP_ASC_URL="https://www.php.net/distributions/php-8.1.1.tar.xz.asc"
ENV PHP_SHA256="33c09d76d0a8bbb5dd930d9dd32e6bfd44e9efcf867563759eb5492c3aff8856"
RUN set -eux; \
    apk add --no-cache --virtual .fetch-deps gnupg; \
    mkdir -p /usr/src; \
    cd /usr/src; \
    curl -fsSL -o php.tar.xz "$PHP_URL"; \
    if [ -n "$PHP_SHA256" ]; then \
        echo "$PHP_SHA256 *php.tar.xz" | sha256sum -c -; \
    fi; \
    if [ -n "$PHP_ASC_URL" ]; then \
        curl -fsSL -o php.tar.xz.asc "$PHP_ASC_URL"; \
        export GNUPGHOME="$(mktemp -d)"; \
        for key in $GPG_KEYS; do \
            gpg --batch --keyserver keyserver.ubuntu.com --recv-keys "$key"; \
        done; \
        gpg --batch --verify php.tar.xz.asc php.tar.xz; \
        gpgconf --kill all; \
        rm -rf "$GNUPGHOME"; \
    fi; \
    apk del --no-network .fetch-deps

COPY bin/docker-php-source /usr/local/bin/
RUN set -eux; \
    apk add --no-cache --virtual .build-deps \
    $PHPIZE_DEPS \
    argon2-dev \
    coreutils \
    curl-dev \
    libsodium-dev \
    libxml2-dev \
    linux-headers \
    oniguruma-dev \
    openssl-dev \
    readline-dev \
    sqlite-dev \
    ; \
    export \
    CFLAGS="$PHP_CFLAGS" \
    CPPFLAGS="$PHP_CPPFLAGS" \
    LDFLAGS="$PHP_LDFLAGS" \
    ; \
    docker-php-source extract; \
    cd /usr/src/php; \
    gnuArch="$(dpkg-architecture --query DEB_BUILD_GNU_TYPE)"; \
    ./configure \
    --build="$gnuArch" \
    --with-config-file-path="$PHP_INI_DIR" \
    --with-config-file-scan-dir="$PHP_INI_DIR/conf.d" \
    --enable-option-checking=fatal \
    --with-mhash \
    --with-pic \
    --enable-ftp \
    --enable-mbstring \
    --enable-mysqlnd \
    --with-password-argon2 \
    --with-sodium=shared \
    --with-pdo-sqlite=/usr \
    --with-sqlite3=/usr \
    --with-curl \
    --with-openssl \
    --with-readline \
    --with-zlib \
    --with-pear \
    $(test "$gnuArch" = 's390x-linux-musl' && echo '--without-pcre-jit') \
    --disable-cgi \
    --enable-fpm \
    --with-fpm-user=www-data \
    --with-fpm-group=www-data \
    ; \
    make -j "$(nproc)"; \
    find -type f -name '*.a' -delete; \
    make install; \
    find /usr/local -type f -perm '/0111' -exec sh -euxc 'strip --strip-all "$@" || : ' -- '{}' + ; \
    make clean; \
    cp -v php.ini-* "$PHP_INI_DIR/"; \
    cd /; \
    docker-php-source delete; \
    runDeps="$( \
    scanelf --needed --nobanner --format '%n#p' --recursive /usr/local \
     | tr ',' '\n' \
     | sort -u \
     | awk 'system("[ -e /usr/local/lib/" $1 " ]") == 0 { next } { print "so:" $1 }' \
     )"; \
    apk add --no-cache $runDeps; \
    apk del --no-network .build-deps; \
    pecl update-channels; \
    rm -rf /tmp/pear ~/.pearrc; \
    php --version

COPY bin/docker-php-e* /usr/local/bin/
RUN docker-php-ext-enable sodium

ENTRYPOINT ["docker-php-entrypoint"]

WORKDIR /var/www/html
RUN set -eux; \
    cd /usr/local/etc; \
    if [ -d php-fpm.d ]; then \
        sed 's!=NONE/!=!g' php-fpm.conf.default | tee php-fpm.conf > /dev/null; \
        cp php-fpm.d/www.conf.default php-fpm.d/www.conf; \
    else \
        mkdir php-fpm.d; \
        cp php-fpm.conf.default php-fpm.d/www.conf; \
        { \
            echo '[global]'; \
            echo 'include=etc/php-fpm.d/*.conf'; \
        } | tee php-fpm.conf; \
    fi; \
    { \
        echo '[global]'; \
        echo 'error_log = /proc/self/fd/2'; \
        echo; \
        echo '; https://github.com/docker-library/php/pull/725#issuecomment-443540114'; \
        echo 'log_limit = 8192'; \
        echo; \
        echo '[www]'; \
        echo '; if we send this to /proc/self/fd/1, it never appears'; \
        echo 'access.log = /proc/self/fd/2'; \
        echo; \
        echo 'clear_env = no'; \
        echo; \
        echo '; Ensure worker stdout and stderr are sent to the main error log.'; \
        echo 'catch_workers_output = yes'; \
        echo 'decorate_workers_output = no'; \
    } | tee php-fpm.d/docker.conf; \
    { \
        echo '[global]'; \
        echo 'daemonize = no'; \
        echo; \
        echo '[www]'; \
        echo 'listen = 9000'; \
    } | tee php-fpm.d/zz-docker.conf

STOPSIGNAL SIGQUIT
EXPOSE 9000
CMD ["php-fpm"]
