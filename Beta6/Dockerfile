FROM alpine:latest
MAINTAINER David Lockwood

ARG OSP_VERSION="beta6d"

ARG NGINX_VERSION=1.17.3
ARG NGINX_RTMP_VERSION=1.2.1

ARG DEFAULT_DB_URL="sqlite:///db/database.db"
ARG DEFAULT_REDIS_HOST="localhost"
ARG DEFAULT_REDIS_PORT=6379
ARG DEFAULT_REDIS_PASSWORD=""
ARG DEFAULT_FLASK_SECRET="CHANGEME"
ARG DEFAULT_FLASK_SALT="CHANGEME"
ARG DEFAULT_OSP_ALLOWREGISTRATION="True"
ARG DEFAULT_OSP_REQUIREVERIFICATION="True"
ARG DEFAULT_TZ="ETC/UTC"

ENV DB_URL=$DEFAULT_DB_URL
ENV REDIS_HOST=$DEFAULT_REDIS_HOST
ENV REDIS_PORT=$DEFAULT_REDIS_PORT
ENV REDIS_PASSWORD=$DEFAULT_REDIS_PASSWORD
ENV FLASK_SECRET=$DEFAULT_FLASK_SECRET
ENV FLASK_SALT=$DEFAULT_FLASK_SALT
ENV OSP_ALLOWREGISTRATION=$DEFAULT_OSP_ALLOWREGISTRATION
ENV OSP_REQUIREVERIFICATION=$DEFAULT_OSP_REQUIREVERIFICATION

EXPOSE 80/tcp
EXPOSE 443/tcp
EXPOSE 1935/tcp


# Get initial dependancies
RUN apk update
RUN apk add alpine-sdk \
  pcre-dev \
  libressl-dev \
  openssl-dev \
  libffi-dev \
  wget \
  git \
  linux-headers \
  zlib-dev \
  postgresql-dev \
  gcc \
  libgcc \
  musl-dev \
  jpeg-dev \
  zlib-dev

RUN apk add --no-cache tzdata

ENV TZ=$DEFAULT_TZ

RUN apk add --no-cache bash

# Download OSP from Repo
RUN cd /tmp && \
  wget "https://gitlab.com/Deamos/flask-nginx-rtmp-manager/-/archive/${OSP_VERSION}/flask-nginx-rtmp-manager-${OSP_VERSION}.tar.gz" && \
  tar zxf flask-nginx-rtmp-manager-${OSP_VERSION}.tar.gz && \
  rm flask-nginx-rtmp-manager-${OSP_VERSION}.tar.gz

# Make OSP Install Directory
RUN mv /tmp/flask-nginx-rtmp-manager-${OSP_VERSION} /opt/osp/

# Transfer OSP Docker Files
COPY entrypoint.sh /opt/osp/setup/
COPY env_initial_setup.py /opt/osp/setup
COPY supervisord.conf /opt/osp/setup
COPY wait-for-it.sh /opt/osp/setup

# Create the www-data user
RUN set -x ; \
  addgroup -g 82 -S www-data ; \
  adduser -u 82 -D -S -G www-data www-data && exit 0 ; exit 1

# Set the OSP directory to www-data
RUN chown -R www-data:www-data /opt/osp

# Download NGINX
RUN cd /tmp && \
  wget https://nginx.org/download/nginx-${NGINX_VERSION}.tar.gz && \
  tar zxf nginx-${NGINX_VERSION}.tar.gz && \
  rm nginx-${NGINX_VERSION}.tar.gz

# Download the NGINX-RTMP Module
RUN cd /tmp && \
  wget https://github.com/arut/nginx-rtmp-module/archive/v${NGINX_RTMP_VERSION}.tar.gz && \
  tar zxf v${NGINX_RTMP_VERSION}.tar.gz && rm v${NGINX_RTMP_VERSION}.tar.gz

RUN cd /tmp && \
  wget "https://bitbucket.org/nginx-goodies/nginx-sticky-module-ng/get/master.tar.gz" && \
  tar xxf master.tar.gz

# Compile NGINX with the NGINX-RTMP Module
RUN cd /tmp/nginx-${NGINX_VERSION} && \
  ./configure \
  --with-http_ssl_module \
  --with-http_v2_module \
  --with-http_auth_request_module \
  --with-cc-opt="-Wimplicit-fallthrough=0" \
  --add-module=../nginx-rtmp-module-${NGINX_RTMP_VERSION} \
  --add-module=../nginx-goodies-nginx-sticky-module-ng-08a395c66e42 && \
  cd /tmp/nginx-${NGINX_VERSION} && make && make install

RUN rm -rf /tmp/nginx-${NGINX_VERSION}

# Configure NGINX
RUN cp /opt/osp/setup/nginx/*.conf /usr/local/nginx/conf/
RUN cp /opt/osp/setup/nginx/mime.types /usr/local/nginx/conf/

# Send NGINX logs to stdout
RUN mkdir -p /usr/local/nginx/logs
RUN ln -sf /dev/stdout /usr/local/nginx/logs/access.log
RUN ln -sf /dev/stderr /usr/local/nginx/logs/error.log

# Install Python, Gunicorn, and uWSGI
RUN apk add python3 \
  py3-pip \
  py3-setuptools \
  python3-dev \
  py3-gunicorn \
  uwsgi-python3

# Upgrade PIP
RUN pip3 install --upgrade pip

# Install OSP Dependancies
RUN pip3 install -r /opt/osp/setup/requirements.txt
RUN pip3 install cryptography

# Setup FFMPEG for recordings and Thumbnails
RUN apk add ffmpeg

# Add Dialog (used in osp-config.sh)
RUN apk add dialog

# Setup Wait-For-It Script
RUN chmod +x /opt/osp/setup/wait-for-it.sh

# Install Supervisor
RUN apk add supervisor
RUN mkdir -p /var/log/supervisor

VOLUME ["/var/www", "/usr/local/nginx/conf", "/opt/osp/db", "/opt/osp/conf"]

RUN chmod +x /opt/osp/osp-config.sh
RUN chmod +x /opt/osp/setup/entrypoint.sh
ENTRYPOINT ["/bin/sh","-c", "/opt/osp/setup/entrypoint.sh"]
