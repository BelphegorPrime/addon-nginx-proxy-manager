ARG BUILD_FROM=ghcr.io/hassio-addons/base:17.2.3
# hadolint ignore=DL3006
FROM ${BUILD_FROM}

# Required for webpack config of NPM
ENV NODE_OPTIONS="--openssl-legacy-provider"

# Set shell
SHELL ["/bin/bash", "-o", "pipefail", "-c"]

# Copy Python requirements file & patches
COPY requirements.txt /tmp/
COPY patches/*.patch /usr/src/

# Setup base
ARG NGINX_PROXY_MANAGER_VERSION="v2.10.4"
# hadolint ignore=DL3003,DL3042
RUN apk add --no-cache --virtual .build-dependencies \
        build-base=0.5-r3 \
        git=2.47.2-r0 \
        libffi-dev=3.4.7-r0 \
        openssl-dev=3.3.3-r0 \
        patch=2.7.6-r10 \
        python3-dev=3.12.9-r0
RUN apk add --no-cache \
        apache2-utils=2.4.62-r0 \
        certbot=3.0.1-r0 \
        libcap=2.71-r0 \
        logrotate=3.21.0-r1 \
        nginx-mod-stream=1.26.3-r0 \
        nginx=1.26.3-r0 \
        openssl=3.3.3-r0 \
        py3-pip=24.3.1-r0 \
        python3=3.12.9-r0 \
        yarn=1.22.22-r1
RUN pip3 install -r /tmp/requirements.txt

RUN yarn global add modclean
RUN curl -fSL -o /tmp/nginxproxymanager.tar.gz "https://github.com/NginxProxyManager/nginx-proxy-manager/archive/${NGINX_PROXY_MANAGER_VERSION}.tar.gz"
RUN tar -tzf /tmp/nginxproxymanager.tar.gz >/dev/null
RUN tar -xzf /tmp/nginxproxymanager.tar.gz --strip 1 -C /app
RUN mkdir /app
RUN if ls /usr/src/*.patch 1> /dev/null 2>&1; then \
     for i in /usr/src/*.patch; do patch -d /app -p 1 < "${i}"; done; \
    fi
RUN cd /app/frontend
RUN yarn install
RUN yarn build
RUN rm -rf node_modules

RUN mkdir -p /opt/nginx-proxy-manager/frontend
RUN cp -r /app/frontend/dist/. /opt/nginx-proxy-manager/frontend/

RUN cd /app/backend
RUN yarn install
RUN rm -rf node_modules
RUN cp -r /app/backend/. /opt/nginx-proxy-manager/

RUN cp -R /app/global/. /opt/nginx-proxy-manager/global/

RUN cd /opt/nginx-proxy-manager
RUN yarn install
RUN rm -rf /etc/services.d/frontend
RUN rm -rf /opt/nginx-proxy-manager/config

RUN rm -f -r /etc/nginx
RUN cp -r /app/docker/rootfs/etc/nginx /etc/nginx 
RUN rm -f /etc/nginx/conf.d/dev.conf

RUN cp /app/docker/rootfs/etc/letsencrypt.ini /etc 
RUN  cp /app/docker/rootfs/etc/logrotate.d/nginx-proxy-manager /etc/logrotate.d 

RUN  sed -i "s#user npm;#user root;#" /etc/nginx/nginx.conf 
RUN  sed -i "s#root /app/frontend;#root /opt/nginx-proxy-manager/frontend;#" /etc/nginx/conf.d/production.conf 
RUN  sed -i "s#table.string('id').notNull().primary();#table.string('id', 32).notNull().primary();#" /opt/nginx-proxy-manager/migrations/20190227065017_settings.js 

RUN  mkdir -p  /run/nginx 

RUN  modclean \
    --path /opt/nginx-proxy-manager \
    --no-progress \
    --keep-empty \
    --run 

RUN yarn global remove modclean 
RUN yarn cache clean
    
RUN apk del --purge .build-dependencies
    
RUN rm -f -r \
        /app \
        /etc/init.d/nginx \
        /etc/logrotate.d/nginx \
        /opt/nginx-proxy-manager/node_modules/bcrypt/build \
        /root/.cache \
        /root/.node-gyp \
        /tmp/.[!.]* \
        /tmp/* \
        /usr/lib/node_modules \
        /usr/local/share/.cache \
        /var/lib/nginx \
        /var/log/nginx \
        /var/tmp/nginx \
        /var/www

# Copy root filesystem
COPY rootfs /

# Build arguments
ARG BUILD_ARCH
ARG BUILD_DATE
ARG BUILD_DESCRIPTION
ARG BUILD_NAME
ARG BUILD_REF
ARG BUILD_REPOSITORY
ARG BUILD_VERSION

# Labels
LABEL \
    io.hass.name="${BUILD_NAME}" \
    io.hass.description="${BUILD_DESCRIPTION}" \
    io.hass.arch="${BUILD_ARCH}" \
    io.hass.type="addon" \
    io.hass.version=${BUILD_VERSION} \
    maintainer="Franck Nijhof <frenck@addons.community>" \
    org.opencontainers.image.title="${BUILD_NAME}" \
    org.opencontainers.image.description="${BUILD_DESCRIPTION}" \
    org.opencontainers.image.vendor="Home Assistant Community Add-ons" \
    org.opencontainers.image.authors="Franck Nijhof <frenck@addons.community>" \
    org.opencontainers.image.licenses="MIT" \
    org.opencontainers.image.url="https://addons.community" \
    org.opencontainers.image.source="https://github.com/${BUILD_REPOSITORY}" \
    org.opencontainers.image.documentation="https://github.com/${BUILD_REPOSITORY}/blob/main/README.md" \
    org.opencontainers.image.created=${BUILD_DATE} \
    org.opencontainers.image.revision=${BUILD_REF} \
    org.opencontainers.image.version=${BUILD_VERSION}
