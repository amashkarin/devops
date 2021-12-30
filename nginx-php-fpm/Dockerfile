FROM alpine:3.15.0

#######################
###   START NGINX   ###
#######################

ENV NGINX_VERSION=1.21.4
ENV NJS_VERSION=0.7.0
ENV PKG_RELEASE=1

RUN set -x \
&& addgroup -g 101 -S nginx \
&& adduser -S -D -H -u 101 -h /var/cache/nginx -s /sbin/nologin -G nginx -g nginx nginx \
&& apkArch="$(cat /etc/apk/arch)" \
&& nginxPackages=" \
    nginx=${NGINX_VERSION}-r${PKG_RELEASE} \
    nginx-module-xslt=${NGINX_VERSION}-r${PKG_RELEASE} \
    nginx-module-geoip=${NGINX_VERSION}-r${PKG_RELEASE} \
    nginx-module-image-filter=${NGINX_VERSION}-r${PKG_RELEASE} \
    nginx-module-njs=${NGINX_VERSION}.${NJS_VERSION}-r${PKG_RELEASE} \
#    nginx-module-perl=${NGINX_VERSION}-r${PKG_RELEASE} \
   " \
&& apk add --no-cache --virtual .checksum-deps \
    openssl \
&& case "$apkArch" in x86_64|aarch64) set -x \
&& KEY_SHA512="e7fa8303923d9b95db37a77ad46c68fd4755ff935d0a534d26eba83de193c76166c68bfe7f65471bf8881004ef4aa6df3e34689c305662750c0172fca5d8552a *stdin" \
&& wget -O /tmp/nginx_signing.rsa.pub https://nginx.org/keys/nginx_signing.rsa.pub \
&& if [ "$(openssl rsa -pubin -in /tmp/nginx_signing.rsa.pub -text -noout | openssl sha512 -r)" = "$KEY_SHA512" ]; then \
            echo "key verification succeeded!"; \
            mv /tmp/nginx_signing.rsa.pub /etc/apk/keys/; \
        else \
            echo "key verification failed!"; \
            exit 1; \
        fi \
&& apk add -X "https://nginx.org/packages/mainline/alpine/v$(egrep -o '^[0-9]+\.[0-9]+' /etc/alpine-release)/main" --no-cache $nginxPackages ;; *) \
set -x \
&& tempDir="$(mktemp -d)" \
&& chown nobody:nobody $tempDir \
&& apk add --no-cache --virtual .build-deps \
    gcc \
    libc-dev \
    make \
    openssl-dev \
    pcre-dev \
    zlib-dev \
    linux-headers \
    libxslt-dev \
    gd-dev \
    geoip-dev \
    perl-dev \
    libedit-dev \
    bash \
    alpine-sdk \
    findutils \
&& su nobody -s /bin/sh -c " \
    export HOME=${tempDir} \
    && cd ${tempDir} \
    && curl -f -O https://hg.nginx.org/pkg-oss/archive/${NGINX_VERSION}-${PKG_RELEASE}.tar.gz \
    && PKGOSSCHECKSUM=\"f917c27702aa89cda46878fc80d446839c592c43ce7f251b3f4ced60c7033d34496a92d283927225d458cbc4f2f89499e7fb16344923317cd7725ad722eaf93e *${NGINX_VERSION}-${PKG_RELEASE}.tar.gz\" \
    && if [ \"\$(openssl sha512 -r ${NGINX_VERSION}-${PKG_RELEASE}.tar.gz)\" = \"\$PKGOSSCHECKSUM\" ]; then \
       echo \"pkg-oss tarball checksum verification succeeded!\"; \
    else \
       echo \"pkg-oss tarball checksum verification failed!\"; \
       exit 1; \
    fi \
    && tar xzvf ${NGINX_VERSION}-${PKG_RELEASE}.tar.gz \
    && cd pkg-oss-${NGINX_VERSION}-${PKG_RELEASE} \
    && cd alpine \
#    https://nginx.org/ru/docs/configure.html \
    && configure \
    --prefix=/etc/nginx \
    --sbin-path=/usr/sbin/nginx \
    --modules-path=/usr/lib/nginx/modules \
    --conf-path=/etc/nginx/nginx.conf \
    --error-log-path=/var/log/nginx/error.log \
    --http-log-path=/var/log/nginx/access.log \
    --pid-path=/var/run/nginx.pid \
    --lock-path=/var/run/nginx.lock \
    --http-client-body-temp-path=/var/cache/nginx/client_temp \
    --http-proxy-temp-path=/var/cache/nginx/proxy_temp \
    --http-fastcgi-temp-path=/var/cache/nginx/fastcgi_temp \
    --http-uwsgi-temp-path=/var/cache/nginx/uwsgi_temp \
    --http-scgi-temp-path=/var/cache/nginx/scgi_temp \
    --with-perl_modules_path=/usr/lib/perl5/vendor_perl \
    --user=nginx \
    --group=nginx \
    --with-compat \
    --with-file-aio \
    --with-threads \
    --with-http_addition_module \
    --with-http_auth_request_module \
    --with-http_dav_module \
    --with-http_flv_module \
    --with-http_geoip_module \
    --with-http_gunzip_module \
    --with-http_gzip_static_module \
    --with-http_mp4_module \
    --with-http_random_index_module \
    --with-http_realip_module \
    --with-http_secure_link_module \
    --with-http_slice_module \
    --with-http_ssl_module \
    --with-http_stub_status_module \
    --with-http_sub_module \
    --with-http_v2_module \
    --with-mail \
    --with-mail_ssl_module \
#    --with-http_perl_module \
    --with-stream \
    --with-stream_geoip_module \
    --with-stream_realip_module \
    --with-stream_ssl_module \
    --with-stream_ssl_preread_module \
    --with-cc-opt='-Os -fomit-frame-pointer -g' \
    --with-ld-opt=-Wl,--as-needed,-O1,--sort-common \
    && make all \
    && apk index -o ${tempDir}/packages/alpine/${apkArch}/APKINDEX.tar.gz ${tempDir}/packages/alpine/${apkArch}/*.apk \
    && abuild-sign -k ${tempDir}/.abuild/abuild-key.rsa ${tempDir}/packages/alpine/${apkArch}/APKINDEX.tar.gz \
" \
&& cp ${tempDir}/.abuild/abuild-key.rsa.pub /etc/apk/keys/ \
&& apk del .build-deps \
&& apk add -X ${tempDir}/packages/alpine/ --no-cache $nginxPackages ;; esac \
&& apk del .checksum-deps \
&& if [ -n "$tempDir" ]; then rm -rf "$tempDir"; fi \
&& if [ -n "/etc/apk/keys/abuild-key.rsa.pub" ]; then rm -f /etc/apk/keys/abuild-key.rsa.pub; fi \
&& if [ -n "/etc/apk/keys/nginx_signing.rsa.pub" ]; then rm -f /etc/apk/keys/nginx_signing.rsa.pub; fi \
&& apk add --no-cache --virtual .gettext gettext \
&& mv /usr/bin/envsubst /tmp/ \
&& runDeps="$( \
    scanelf --needed --nobanner /tmp/envsubst \
    | awk '{ gsub(/,/, "\nso:", $2); print "so:" $2 }' \
    | sort -u \
    | xargs -r apk info --installed \
    | sort -u \
    )" \
&& apk add --no-cache $runDeps \
&& apk del .gettext \
&& mv /tmp/envsubst /usr/local/bin/ \
&& apk add --no-cache tzdata \
&& apk add --no-cache curl ca-certificates \
&& ln -sf /dev/stdout /var/log/nginx/access.log \
&& ln -sf /dev/stderr /var/log/nginx/error.log \
&& mkdir /docker-entrypoint.d

#COPY nginx/docker-entrypoint.sh /
COPY nginx/docker-entrypoint.d/10-listen-on-ipv6-by-default.sh /docker-entrypoint.d/
COPY nginx/docker-entrypoint.d/20-envsubst-on-templates.sh /docker-entrypoint.d/
COPY nginx/docker-entrypoint.d/30-tune-worker-processes.sh /docker-entrypoint.d/
COPY nginx/default-site.conf /etc/nginx/conf.d/

#ENTRYPOINT ["/docker-entrypoint.sh"]
#EXPOSE 80
#STOPSIGNAL SIGQUIT
#CMD ["nginx", "-g", "daemon off;"]

#####################
###   END NGINX   ###
#####################


#####################
###   START PHP   ###
#####################

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
#RUN set -eux; \
#adduser -u 82 -D -S -G www-data www-data


ENV PHP_INI_DIR=/usr/local/etc/php
RUN set -eux; \
mkdir -p "$PHP_INI_DIR/conf.d"; \
[ ! -d /var/www/html ]; \
mkdir -p /var/www/html; \
chown nginx:nginx /var/www/html; \
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

COPY php-fpm/docker-php-source /usr/local/bin/
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
    --with-fpm-user=nginx \
    --with-fpm-group=nginx \
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

COPY php-fpm/docker-php-e* /usr/local/bin/
RUN docker-php-ext-enable sodium

#ENTRYPOINT ["docker-php-entrypoint"]

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

COPY ./php-fpm/www.conf /usr/local/etc/php-fpm.d/www.conf
RUN touch /var/run/php-fpm.sock \
    && chown nginx:nginx /var/run/php-fpm.sock \
    && chmod 0660 /var/run/php-fpm.sock

###################
###   END PHP   ###
###################



# todo add composer, xdebug

# Set supervisord configuration
RUN set -eux \
&& apk update \
&& apk add --no-cache \
    supervisor \
&& mkdir -p /etc/supervisor/conf.d/
COPY supervisord.conf /etc/supervisor/


COPY docker-entrypoint.sh /usr/local/bin/docker-entrypoint.sh
RUN set -eux \
&& chmod +x /usr/local/bin/docker-entrypoint.sh
ENTRYPOINT ["/usr/local/bin/docker-entrypoint.sh"]


STOPSIGNAL SIGQUIT
EXPOSE 80

CMD ["/usr/bin/supervisord","-n","-c","/etc/supervisor/supervisord.conf"]

#HEALTHCHECK --start-period=5s --interval=60s --timeout=1s --retries=3 \
#  CMD ps -C nginx || exit 1 && ps -C php-fpm || exit 1
