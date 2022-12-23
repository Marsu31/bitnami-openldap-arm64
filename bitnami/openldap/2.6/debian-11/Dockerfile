FROM --platform=linux/arm64 docker.io/bitnami/minideb:bullseye AS build

ARG BITNAMI_VERSION=2.6.3-8
ARG OPENLDAP_DIR=/opt/bitnami/openldap

USER root

RUN install_packages ca-certificates curl build-essential git groff-base libsodium-dev libltdl-dev libssl-dev libsasl2-dev libwrap0-dev
WORKDIR /build
# clone main branch
RUN git clone -b OPENLDAP_REL_ENG_2_6 https://git.openldap.org/openldap/openldap.git openldap
WORKDIR openldap
RUN ./configure \
      --prefix=${OPENLDAP_DIR} \
      --libexecdir=\${prefix}/lib \
      --sysconfdir=\${prefix}/conf \
      CPPFLAGS="-I${OPENLDAP_DIR}/include" LDFLAGS="-L${OPENLDAP_DIR}/lib -Wl,-rpath,${OPENLDAP_DIR}/lib" \
      # Optional Features: \
      --enable-debug --disable-ipv6 \
      # SLAPD (Standalone LDAP Daemon) Options: \
      --enable-slapd --enable-dynacl --enable-aci --enable-crypt --enable-spasswd --enable-modules --enable-rlookups --enable-slapi --enable-wrappers \
      --disable-cleartext --disable-slp \
      # SLAPD Backend Options: \
      --enable-backends=mod --enable-mdb \
      # deprecated since 2.6 (enabled in bitnami's distri)
      --disable-perl --disable-sock --disable-sql --disable-wt \
      # SLAPD Overlay Options: \
      --enable-overlays=mod \
      --disable-autoca \
      # SLAPD Password Module Options: \
      --enable-argon2 \
      # Library Generation & Linking Options \
      --enable-static=no \
      # Optional Packages: \
      --with-cyrus-sasl --without-systemd --with-tls=openssl --with-argon2=libsodium \
      # Optimization \
      --disable-dependency-tracking --disable-maintainer-mode
RUN make -j$(getconf _NPROCESSORS_ONLN) depend
RUN make -j$(getconf _NPROCESSORS_ONLN)
RUN make -j$(getconf _NPROCESSORS_ONLN) install

# Password modules
RUN make -j$(getconf _NPROCESSORS_ONLN) DESTDIR="" prefix=${OPENLDAP_DIR} -C contrib/slapd-modules/passwd/pbkdf2 install-lib
RUN make -j$(getconf _NPROCESSORS_ONLN) DESTDIR="" prefix=${OPENLDAP_DIR} -C contrib/slapd-modules/passwd/sha2 install-lib
RUN make -j$(getconf _NPROCESSORS_ONLN) DESTDIR="" prefix=${OPENLDAP_DIR} -C contrib/slapd-modules/passwd/totp install-lib

WORKDIR ${OPENLDAP_DIR}
# delete man pages
RUN rm -rf share/man
# create etc as expected
RUN mkdir etc
RUN mv conf/openldap/* etc
RUN rm -rf conf
# deprecated, unused, bad conf files
RUN rm etc/slapd.conf*
RUN rm etc/slapd.ldif*
# missing directories
RUN mkdir etc/certs
# slapd binary at expected place (/sbin)
WORKDIR ${OPENLDAP_DIR}/sbin
RUN ln -s ${OPENLDAP_DIR}/lib/slapd

# slapd.ldif from bitnami
# TODO : link needs to be kept up to date (algo to find last version, remove build argument)
RUN curl -SsLf "https://downloads.bitnami.com/files/stacksmith/openldap-${BITNAMI_VERSION}-linux-amd64-debian-11.tar.gz" | \
      tar zx -C ${OPENLDAP_DIR}/share/ --wildcards --no-anchored */files/openldap/share/slapd.ldif --strip-components 4
RUN chown root:root ${OPENLDAP_DIR}/share/slapd.ldif

#-------------------------------------------------------------------------------

FROM --platform=linux/arm64 docker.io/bitnami/minideb:bullseye

ARG TARGETARCH
ARG GOSU_VERSION=1.16

LABEL org.opencontainers.image.authors="https://github.com/Marsu31" \
      org.opencontainers.image.description="Application packaged by Marsu31 (from Bitnami's)" \
      org.opencontainers.image.ref.name="2.6-debian-11" \
      org.opencontainers.image.source="https://github.com/Marsu31/bitnami-openldap-arm64/tree/main/bitnami/openldap" \
      org.opencontainers.image.title="openldap" \
      org.opencontainers.image.version="2.6"

ENV HOME="/" \
    OS_ARCH="${TARGETARCH:-arm64}" \
    OS_FLAVOUR="debian-11" \
    OS_NAME="linux"

COPY prebuildfs /
SHELL ["/bin/bash", "-o", "pipefail", "-c"]
# Install required system packages and dependencies
RUN install_packages ca-certificates curl libargon2-1 libcom-err2 libcrypt1 libgssapi-krb5-2 libk5crypto3 libkeyutils1 libkrb5-3 libkrb5support0 libltdl7 libnsl2 libnss3-tools libodbc1 libperl5.32 libsasl2-2 libssl1.1 libtirpc3 libwrap0 mdbtools procps psmisc \
      # libsodium23 -> argon2, netcat -> healthchecks \
      libsodium23 netcat
#----------
# gosu (in /common/bin)
RUN mkdir -p /opt/bitnami/common/bin
RUN curl -SsLf https://github.com/tianon/gosu/releases/download/${GOSU_VERSION}/gosu-arm64 -o /opt/bitnami/common/bin/gosu
RUN chmod +x /opt/bitnami/common/bin/gosu
# openldap
COPY --from=build /opt/bitnami/openldap /opt/bitnami/openldap
#----------
RUN apt-get autoremove --purge -y curl && \
    apt-get update && apt-get upgrade -y && \
    apt-get clean && rm -rf /var/lib/apt/lists /var/cache/apt/archives
RUN chmod g+rwX /opt/bitnami

COPY rootfs /
RUN /opt/bitnami/scripts/openldap/postunpack.sh
ENV APP_VERSION="2.6" \
    BITNAMI_APP_NAME="openldap" \
    PATH="/opt/bitnami/openldap/bin:/opt/bitnami/openldap/sbin:/opt/bitnami/common/bin:$PATH"

EXPOSE 1389 1636

USER 1001
ENTRYPOINT [ "/opt/bitnami/scripts/openldap/entrypoint.sh" ]
CMD [ "/opt/bitnami/scripts/openldap/run.sh" ]

# basic health chech: port is opened
HEALTHCHECK --start-period=15s --interval=10s --timeout=5s --retries=3 CMD nc -z localhost ${LDAP_PORT_NUMBER:-1389} || exit 1
