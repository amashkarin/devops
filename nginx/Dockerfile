FROM alpine:3.15.0
LABEL maintainer="NGINX Docker Maintainers <docker-maint@nginx.com>"
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
COPY docker-entrypoint.sh /
COPY docker-entrypoint.d/10-listen-on-ipv6-by-default.sh /docker-entrypoint.d/
COPY docker-entrypoint.d/20-envsubst-on-templates.sh /docker-entrypoint.d/
COPY docker-entrypoint.d/30-tune-worker-processes.sh /docker-entrypoint.d/

ENTRYPOINT ["/docker-entrypoint.sh"]
EXPOSE 80
STOPSIGNAL SIGQUIT
CMD ["nginx", "-g", "daemon off;"]
