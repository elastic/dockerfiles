################################################################################
# This Dockerfile was generated from the template at distribution/src/docker/Dockerfile
#
# Beginning of multi stage Dockerfile
################################################################################

################################################################################
# Step 1. Create a minimal root filesystem directory. This will form the basis
# for our image.
################################################################################
FROM centos:8 AS rootfs

ENV TINI_VERSION 0.19.0

# Start off with an up-to-date system
RUN yum update --setopt=tsflags=nodocs -y

# Create a directory into which we will install files
RUN mkdir /rootfs

# Create required devices
RUN mkdir -m 755 /rootfs/dev && \
    mknod -m 600 /rootfs/dev/console c 5 1 && \
    mknod -m 600 /rootfs/dev/initctl p && \
    mknod -m 666 /rootfs/dev/full c 1 7 && \
    mknod -m 666 /rootfs/dev/null c 1 3 && \
    mknod -m 666 /rootfs/dev/ptmx c 5 2 && \
    mknod -m 666 /rootfs/dev/random c 1 8 && \
    mknod -m 666 /rootfs/dev/tty c 5 0 && \
    mknod -m 666 /rootfs/dev/tty0 c 4 0 && \
    mknod -m 666 /rootfs/dev/urandom c 1 9 && \
    mknod -m 666 /rootfs/dev/zero c 1 5

# Install a minimal set of dependencies, and some for Elasticsearch
RUN yum --installroot=/rootfs --releasever=/ --setopt=tsflags=nodocs \
      --setopt=group_package_types=mandatory -y  \
      install bash curl findutils nc procps-ng shadow-utils tar unzip yum zip

# `tini` is a tiny but valid init for containers. This is used to cleanly
# control how ES and any child processes are shut down.
#
# The tini GitHub page gives instructions for verifying the binary using
# gpg, but the keyservers are slow to return the key and this can fail the
# build. Instead, we check the binary against the published checksum.
RUN set -e ; \
    TINI_BIN="" ; \
    case "$(arch)" in \
        aarch64) \
            TINI_BIN='tini-arm64' ; \
            ;; \
        x86_64) \
            TINI_BIN='tini-amd64' ; \
            ;; \
        *) echo >&2 "Unsupported architecture $(arch)" ; exit 1 ;; \
    esac ; \
    curl --retry 10 -S -L -O "https://github.com/krallin/tini/releases/download/v0.19.0/${TINI_BIN}" ; \
    curl --retry 10 -S -L -O "https://github.com/krallin/tini/releases/download/v0.19.0/${TINI_BIN}.sha256sum" ; \
    sha256sum -c "${TINI_BIN}.sha256sum" ; \
    rm "${TINI_BIN}.sha256sum" ; \
    mv "${TINI_BIN}" /rootfs/bin/tini ; \
    chmod 0555 /rootfs/bin/tini

RUN echo "NETWORKING=yes" > /rootfs/etc/sysconfig/network && \
    echo "HOSTNAME=localhost.localdomain" >> /rootfs/etc/sysconfig/network

# Cleanup the filesystem
RUN yum --installroot=/rootfs -y clean all && \
    cd /rootfs && \
    rm -rf \
        sbin/sln \
        usr/{{lib,share}/locale,{lib,lib64}/gconv,bin/localedef,sbin/build-locale-archive} \
        usr/share/{man,doc,info,gnome/help} \
        usr/share/cracklib \
        usr/share/i18n \
        var/cache/yum && \
    mkdir -p --mode=0755 var/cache/yum

# ldconfig
RUN rm -rf /rootfs/etc/ld.so.cache /rootfs/var/cache/ldconfig && \
    mkdir -p --mode=0755 /rootfs/var/cache/ldconfig

# Ensure that there are no files with setuid or setgid, in order to mitigate "stackclash" attacks.
RUN find /rootfs -xdev -perm -4000 -exec chmod ug-s {} +

################################################################################
# Step 2. Fetch the Elasticsearch distribution and configure it for Docker
################################################################################
FROM centos:8 AS builder

RUN mkdir /usr/share/elasticsearch
WORKDIR /usr/share/elasticsearch

RUN curl --retry 10 -S -L --output /opt/elasticsearch.tar.gz https://artifacts-no-kpi.elastic.co/downloads/elasticsearch/elasticsearch-7.15.2-linux-$(arch).tar.gz

RUN tar -zxf /opt/elasticsearch.tar.gz --strip-components=1

# The distribution includes a `config` directory, no need to create it
COPY config/elasticsearch.yml config/
COPY config/log4j2.properties config/log4j2.docker.properties

#  1. Configure the distribution for Docker
#  2. Create required directory
#  3. Move the distribution's default logging config aside
#  4. Move the generated docker logging config so that it is the default
#  5. Reset permissions on all directories
#  6. Reset permissions on all files
#  7. Make CLI tools executable
#  8. Make some directories writable. `bin` must be writable because
#     plugins can install their own CLI utilities.
#  9. Make some files writable
RUN sed -i -e 's/ES_DISTRIBUTION_TYPE=tar/ES_DISTRIBUTION_TYPE=docker/' bin/elasticsearch-env && \
    mkdir data && \
    mv config/log4j2.properties config/log4j2.file.properties && \
    mv config/log4j2.docker.properties config/log4j2.properties && \
    find . -type d -exec chmod 0555 {} + && \
    find . -type f -exec chmod 0444 {} + && \
    chmod 0555 bin/* jdk/bin/* jdk/lib/jspawnhelper modules/x-pack-ml/platform/linux-*/bin/* && \
    chmod 0775 bin config config/jvm.options.d data logs plugins && \
    find config -type f -exec chmod 0664 {} +

################################################################################
# Stage 3. Build the final image, using the rootfs above as the basis, and
# copying in the Elasticsearch distribution
################################################################################
FROM scratch

# Setup the initial filesystem.
COPY --from=rootfs /rootfs /

RUN groupadd -g 1000 elasticsearch && \
    adduser -u 1000 -g 1000 -G 0 -d /usr/share/elasticsearch elasticsearch && \
    chmod 0755 /usr/share/elasticsearch && \
    chown -R 0:0 /usr/share/elasticsearch

ENV ELASTIC_CONTAINER true

WORKDIR /usr/share/elasticsearch
COPY --from=builder --chown=0:0 /usr/share/elasticsearch /usr/share/elasticsearch

ENV PATH /usr/share/elasticsearch/bin:$PATH

COPY bin/docker-entrypoint.sh /usr/local/bin/docker-entrypoint.sh

# 1. Sync the user and group permissions of /etc/passwd
# 2. Set correct permissions of the entrypoint
# 3. Ensure that there are no files with setuid or setgid, in order to mitigate "stackclash" attacks.
#    We've already run this in previous layers so it ought to be a no-op.
# 4. Replace OpenJDK's built-in CA certificate keystore with the one from the OS
#    vendor. The latter is superior in several ways.
#    REF: https://github.com/elastic/elasticsearch-docker/issues/171
# 5. Tighten up permissions on the ES home dir (the permissions of the contents are handled earlier)
# 6. You can't install plugins that include configuration when running as `elasticsearch` and the `config`
#    dir is owned by `root`, because the installed tries to manipulate the permissions on the plugin's
#    config directory.
RUN chmod g=u /etc/passwd && \
    chmod 0555 /usr/local/bin/docker-entrypoint.sh && \
    find / -xdev -perm -4000 -exec chmod ug-s {} + && \
    ln -sf /etc/pki/ca-trust/extracted/java/cacerts /usr/share/elasticsearch/jdk/lib/security/cacerts && \
    chmod 0775 /usr/share/elasticsearch && \
    chown elasticsearch bin config config/jvm.options.d data logs plugins

EXPOSE 9200 9300

LABEL org.label-schema.build-date="2021-11-04T14:04:42.515624022Z" \
  org.label-schema.license="Elastic-License-2.0" \
  org.label-schema.name="Elasticsearch" \
  org.label-schema.schema-version="1.0" \
  org.label-schema.url="https://www.elastic.co/products/elasticsearch" \
  org.label-schema.usage="https://www.elastic.co/guide/en/elasticsearch/reference/index.html" \
  org.label-schema.vcs-ref="93d5a7f6192e8a1a12e154a2b81bf6fa7309da0c" \
  org.label-schema.vcs-url="https://github.com/elastic/elasticsearch" \
  org.label-schema.vendor="Elastic" \
  org.label-schema.version="7.15.2" \
  org.opencontainers.image.created="2021-11-04T14:04:42.515624022Z" \
  org.opencontainers.image.documentation="https://www.elastic.co/guide/en/elasticsearch/reference/index.html" \
  org.opencontainers.image.licenses="Elastic-License-2.0" \
  org.opencontainers.image.revision="93d5a7f6192e8a1a12e154a2b81bf6fa7309da0c" \
  org.opencontainers.image.source="https://github.com/elastic/elasticsearch" \
  org.opencontainers.image.title="Elasticsearch" \
  org.opencontainers.image.url="https://www.elastic.co/products/elasticsearch" \
  org.opencontainers.image.vendor="Elastic" \
  org.opencontainers.image.version="7.15.2"

# Our actual entrypoint is `tini`, a minimal but functional init program. It
# calls the entrypoint we provide, while correctly forwarding signals.
ENTRYPOINT ["/bin/tini", "--", "/usr/local/bin/docker-entrypoint.sh"]

# Dummy overridable parameter parsed by entrypoint
CMD ["eswrapper"]

################################################################################
# End of multi-stage Dockerfile
################################################################################
