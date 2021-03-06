FROM ubuntu:trusty as builder

ARG APT_FLAGS_COMMON="-y"
ARG APT_FLAGS_DEV="${APT_FLAGS_COMMON} --no-install-recommends"

ARG MAJOR_VERSION=3.4
ARG ZBX_VERSION=${MAJOR_VERSION}.10
ARG ZBX_SOURCES=svn://svn.zabbix.com/tags/${ZBX_VERSION}/
ENV ZBX_VERSION=${ZBX_VERSION} ZBX_SOURCES=${ZBX_SOURCES} \
    LANG=en_US.UTF-8 LANGUAGE=en_US:en LC_ALL=en_US.UTF-8 DEBIAN_FRONTEND=noninteractive TERM=xterm \
    MIBDIRS=/var/lib/mibs/iana:/var/lib/mibs/ietf:/usr/share/snmp/mibs:/var/lib/zabbix/mibs MIBS=+ALL \
    ZBX_TYPE=server ZBX_DB_TYPE=mysql ZBX_OPT_TYPE=nginx \
    MYSQL_ALLOW_EMPTY_PASSWORD=true ZBX_ADD_SERVER=true ZBX_ADD_WEB=true DB_SERVER_HOST=localhost MYSQL_USER=zabbix ZBX_ADD_JAVA_GATEWAY=true ZBX_JAVAGATEWAY_ENABLE=true ZBX_JAVAGATEWAY=localhost

COPY ["conf/tmp/font-config", "/tmp/font-config"]

RUN locale-gen $LC_ALL && \
    apt-get ${APT_FLAGS_COMMON} update && \
    apt-get ${APT_FLAGS_DEV} install \
            autoconf \
            automake \
            gcc \
            gettext \
            libc6-dev \
            libcurl4-openssl-dev \
            libevent-dev \
            libiksemel-dev \
            libldap2-dev \
            libmysqlclient-dev \
            libopenipmi-dev \
            libpcre3-dev \
            libsnmp-dev \
            libssh2-1-dev \
            libxml2-dev \
            make \
            openjdk-7-jdk \
            patch \
            pkg-config \
            subversion \
            unixodbc-dev && \
    cd /tmp/ && \
    svn --quiet export ${ZBX_SOURCES} zabbix-${ZBX_VERSION} && \
    cd /tmp/zabbix-${ZBX_VERSION} && \
    zabbix_revision=`svn info ${ZBX_SOURCES} | grep "Last Changed Rev"|awk '{print $4;}'` && \
    sed -i "s/{ZABBIX_REVISION}/$zabbix_revision/g" include/version.h && \
    sed -i "s/{ZABBIX_REVISION}/$zabbix_revision/g" include/version.h && \
    sed -i "s/{ZABBIX_REVISION}/$zabbix_revision/g" src/zabbix_java/src/com/zabbix/gateway/GeneralInformation.java && \
    ./bootstrap.sh && \
    export CFLAGS="-fPIC -pie -Wl,-z,relro -Wl,-z,now" && \
    ./configure \
            --datadir=/usr/lib \
            --libdir=/usr/lib/zabbix \
            --prefix=/usr \
            --sysconfdir=/etc/zabbix \
            --enable-agent \
            --enable-${ZBX_TYPE} \
            --with-${ZBX_DB_TYPE} \
            --with-jabber \
            --with-ldap \
            --with-libcurl \
            --with-libxml2 \
            --enable-java \
            --with-net-snmp \
            --with-openipmi \
            --with-openssl \
            --with-ssh2 \
            --with-unixodbc \
            --enable-ipv6 \
            --silent && \
    make -j"$(nproc)" -s dbschema && \
    make -j"$(nproc)" -s && \
    cat database/${ZBX_DB_TYPE}/schema.sql > database/${ZBX_DB_TYPE}/create.sql && \
    cat database/${ZBX_DB_TYPE}/images.sql >> database/${ZBX_DB_TYPE}/create.sql && \
    cat database/${ZBX_DB_TYPE}/data.sql >> database/${ZBX_DB_TYPE}/create.sql && \
    gzip database/${ZBX_DB_TYPE}/create.sql && \
    rm -rf /tmp/zabbix-${ZBX_VERSION}/src/zabbix_java/lib/*.xml && \
    cd frontends/php/ && \
    patch -p3 < /tmp/font-config && \
    rm -f conf/zabbix.conf.php && \
    rm -rf tests && \
    rm /tmp/zabbix-${ZBX_VERSION}/frontends/php/fonts/DejaVuSans.ttf && \
    ./locale/make_mo.sh

FROM ubuntu:trusty
LABEL maintainer="Alexey Pustovalov <alexey.pustovalov@zabbix.com>"

ARG BUILD_DATE
ARG VCS_REF

ARG APT_FLAGS_COMMON="-y"
ARG APT_FLAGS_PERSISTENT="${APT_FLAGS_COMMON} --no-install-recommends"
ARG MAJOR_VERSION=3.4
ARG ZBX_VERSION=${MAJOR_VERSION}.10
ARG ZBX_SOURCES=svn://svn.zabbix.com/tags/${ZBX_VERSION}/
ENV ZBX_VERSION=${ZBX_VERSION} ZBX_SOURCES=${ZBX_SOURCES} \
    LANG=en_US.UTF-8 LANGUAGE=en_US:en LC_ALL=en_US.UTF-8 DEBIAN_FRONTEND=noninteractive TERM=xterm \
    MIBDIRS=/var/lib/mibs/iana:/var/lib/mibs/ietf:/usr/share/snmp/mibs:/var/lib/zabbix/mibs MIBS=+ALL \
    ZBX_TYPE=server ZBX_DB_TYPE=mysql ZBX_OPT_TYPE=nginx \
    MYSQL_ALLOW_EMPTY_PASSWORD=true ZBX_ADD_SERVER=true ZBX_ADD_WEB=true DB_SERVER_HOST=localhost MYSQL_USER=zabbix ZBX_ADD_JAVA_GATEWAY=true ZBX_JAVAGATEWAY_ENABLE=true ZBX_JAVAGATEWAY=localhost

LABEL org.label-schema.name="zabbix-${ZBX_TYPE}-${ZBX_DB_TYPE}-ubuntu" \
      org.label-schema.vendor="Zabbix LLC" \
      org.label-schema.url="https://zabbix.com/" \
      org.label-schema.description="Zabbix appliance with MySQL database support and ${ZBX_OPT_TYPE} web-server" \
      org.label-schema.vcs-ref="${VCS_REF}" \
      org.label-schema.build-date="${BUILD_DATE}" \
      org.label-schema.schema-version="1.0" \
      org.label-schema.license="GPL v2.0" \
      org.label-schema.usage="https://www.zabbix.com/documentation/${MAJOR_VERSION}/manual/installation/containers" \
      org.label-schema.version="${ZBX_VERSION}" \
      org.label-schema.vcs-url="${ZBX_SOURCES}" \
      org.label-schema.docker.cmd="docker run --name zabbix-appliance -p 80:80 -p 10051:10051 -d zabbix-appliance:ubuntu-${ZBX_VERSION}"

STOPSIGNAL SIGTERM

COPY --from=builder /tmp/zabbix-${ZBX_VERSION}/src/zabbix_${ZBX_TYPE}/zabbix_${ZBX_TYPE} /usr/sbin/zabbix_${ZBX_TYPE}
COPY --from=builder /tmp/zabbix-${ZBX_VERSION}/src/zabbix_get/zabbix_get /usr/bin/zabbix_get
COPY --from=builder /tmp/zabbix-${ZBX_VERSION}/src/zabbix_sender/zabbix_sender /usr/bin/zabbix_sender
COPY --from=builder /tmp/zabbix-${ZBX_VERSION}/conf/zabbix_${ZBX_TYPE}.conf /etc/zabbix/zabbix_${ZBX_TYPE}.conf
COPY --from=builder /tmp/zabbix-${ZBX_VERSION}/database/${ZBX_DB_TYPE}/create.sql.gz /usr/share/doc/zabbix-${ZBX_TYPE}-${ZBX_DB_TYPE}/create.sql.gz
            
COPY --from=builder /tmp/zabbix-${ZBX_VERSION}/src/zabbix_java/bin/ /usr/sbin/zabbix_java/bin/
COPY --from=builder /tmp/zabbix-${ZBX_VERSION}/src/zabbix_java/lib/ /usr/sbin/zabbix_java/lib/
            
COPY --from=builder /tmp/zabbix-${ZBX_VERSION}/frontends/php/ /usr/share/zabbix/

RUN locale-gen $LC_ALL && \
    echo "#!/bin/sh\nexit 0" > /usr/sbin/policy-rc.d && \
    addgroup --system --quiet zabbix && \
    adduser --quiet \
            --system --disabled-login \
            --ingroup zabbix \
            --home /var/lib/zabbix/ \
            --no-create-home \
        zabbix && \
    mkdir -p /etc/zabbix && \
    mkdir -p /var/lib/zabbix && \
    mkdir -p /usr/lib/zabbix/alertscripts && \
    mkdir -p /var/lib/zabbix/enc && \
    mkdir -p /usr/lib/zabbix/externalscripts && \
    mkdir -p /var/lib/zabbix/mibs && \
    mkdir -p /var/lib/zabbix/modules && \
    mkdir -p /var/lib/zabbix/snmptraps && \
    mkdir -p /var/lib/zabbix/ssh_keys && \
    mkdir -p /var/lib/zabbix/ssl && \
    mkdir -p /var/lib/zabbix/ssl/certs && \
    mkdir -p /var/lib/zabbix/ssl/keys && \
    mkdir -p /var/lib/zabbix/ssl/ssl_ca && \
    chown --quiet -R zabbix:root /var/lib/zabbix && \
    mkdir -p /usr/share/doc/zabbix-${ZBX_TYPE}-${ZBX_DB_TYPE}/ && \
    apt-get ${APT_FLAGS_COMMON} update && \
    apt-get ${APT_FLAGS_COMMON} install \
            wget && \
    DISTRIB_CODENAME=$(/bin/bash -c 'source /etc/lsb-release && echo $DISTRIB_CODENAME') && \
    wget -q https://nginx.org/keys/nginx_signing.key -O- | apt-key add - && \
    echo "deb http://nginx.org/packages/ubuntu/ $DISTRIB_CODENAME nginx" >> /etc/apt/sources.list.d/nginx.list && \
    apt-get ${APT_FLAGS_COMMON} update && \
    apt-get ${APT_FLAGS_PERSISTENT} install \
            fping \
            libcurl3 \
            libevent-2.0 \
            libiksemel3 \
            libmysqlclient18 \
            libopenipmi0 \
            libpcre3 \
            libsnmp30 \
            libssh2-1 \
            libssl1.0.0 \
            libxml2 \
            mysql-client \
            mysql-server \
            nginx \
            openjdk-7-jre-headless \
            php5-fpm \
            php5-gd \
            php5-json \
            php5-ldap \
            php5-mysql \
            snmp-mibs-downloader \
            supervisor \
            ttf-dejavu-core \
            unixodbc && \
    update-alternatives --install /usr/share/zabbix/fonts/graphfont.ttf \
            zabbix-frontend-font /usr/share/fonts/truetype/ttf-dejavu/DejaVuSans.ttf 10 && \
    mkdir -p /var/lib/locales/supported.d/ && \
    rm -f /var/lib/locales/supported.d/local && \
    cat /usr/share/zabbix/include/locales.inc.php | grep display | grep true | awk '{$1=$1};1' | \
                cut -d"'" -f 2 | sort | \
                xargs -I '{}' bash -c 'echo "{}.UTF-8 UTF-8" >> /var/lib/locales/supported.d/local' && \
    dpkg-reconfigure locales && \
    apt-get ${APT_FLAGS_COMMON} purge \
            wget && \
    apt-get ${APT_FLAGS_COMMON} autoremove && \
    apt-get ${APT_FLAGS_COMMON} clean && \
    mkdir -p /var/lib/php5 && \
    chown --quiet -R www-data:www-data /var/lib/php5 && \
    rm -rf /var/cache/nginx/* && \
    rm -rf /var/lib/apt/lists/*

EXPOSE 80/TCP 443/TCP 10051/TCP

WORKDIR /var/lib/zabbix

VOLUME ["/etc/ssl/nginx"]
VOLUME ["/usr/lib/zabbix/alertscripts", "/usr/lib/zabbix/externalscripts", "/var/lib/zabbix/enc", "/var/lib/zabbix/mibs", "/var/lib/zabbix/modules"]
VOLUME ["/var/lib/zabbix/snmptraps", "/var/lib/zabbix/ssh_keys", "/var/lib/zabbix/ssl/certs", "/var/lib/zabbix/ssl/keys", "/var/lib/zabbix/ssl/ssl_ca"]

COPY ["conf/etc/supervisor/", "/etc/supervisor/"]
COPY ["conf/etc/zabbix/nginx.conf", "/etc/zabbix/"]
COPY ["conf/etc/zabbix/nginx_ssl.conf", "/etc/zabbix/"]
COPY ["conf/etc/zabbix/web/zabbix.conf.php", "/etc/zabbix/web/"]
COPY ["conf/etc/nginx/nginx.conf", "/etc/nginx/"]
COPY ["conf/etc/php5/fpm/conf.d/99-zabbix.ini", "/etc/php5/fpm/conf.d/"]
COPY ["conf/etc/zabbix/zabbix_java_gateway_logback.xml", "/etc/zabbix/"]
COPY ["conf/usr/sbin/zabbix_java_gateway", "/usr/sbin/"]
COPY ["docker-entrypoint.sh", "/usr/bin/"]

ENV ZBX_TYPE=appliance

ENTRYPOINT ["docker-entrypoint.sh"]
