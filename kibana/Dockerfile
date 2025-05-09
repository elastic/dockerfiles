################################################################################
# This Dockerfile was generated from the template at:
#   src/dev/build/tasks/os_packages/docker_generator/templates/Dockerfile
#
# Beginning of multi stage Dockerfile
################################################################################

################################################################################
# Build stage 0 `builder`:
# Extract Kibana artifact
################################################################################
FROM redhat/ubi9-minimal:latest AS builder

RUN microdnf install -y findutils tar gzip

RUN cd /tmp && \
  arch="$(rpm --query --queryformat='%{ARCH}' rpm)" && \
  curl -f --retry 8 -s -L \
    --output kibana.tar.gz \
     https://artifacts.elastic.co/downloads/kibana/kibana-9.0.1-linux-${arch}.tar.gz && \
  cd -

RUN mkdir /usr/share/kibana
WORKDIR /usr/share/kibana
RUN tar \
  --strip-components=1 \
  -zxf /tmp/kibana.tar.gz

# Ensure that group permissions are the same as user permissions.
# This will help when relying on GID-0 to run Kibana, rather than UID-1000.
# OpenShift does this, for example.
# REF: https://docs.openshift.org/latest/creating_images/guidelines.html
RUN chmod -R g=u /usr/share/kibana

# Add an init process, check the checksum to make sure it's a match
RUN set -e ; \
    TINI_BIN="" ; \
    arch="$(rpm --query --queryformat='%{ARCH}' rpm)"; \
    case "$arch" in \
        aarch64) \
            TINI_BIN='tini-arm64' ; \
            TINI_CHECKSUM='07952557df20bfd2a95f9bef198b445e006171969499a1d361bd9e6f8e5e0e81' ; \
            ;; \
        x86_64) \
            TINI_BIN='tini-amd64' ; \
            TINI_CHECKSUM='93dcc18adc78c65a028a84799ecf8ad40c936fdfc5f2a57b1acda5a8117fa82c' ; \
            ;; \
        *) echo >&2 "Unsupported architecture $arch" ; exit 1 ;; \
    esac ; \
  TINI_VERSION='v0.19.0' ; \
  curl -f --retry 8 -S -L -O "https://github.com/krallin/tini/releases/download/${TINI_VERSION}/${TINI_BIN}" ; \
  echo "${TINI_CHECKSUM} ${TINI_BIN}" | sha256sum -c - ; \
  mv "${TINI_BIN}" /bin/tini ; \
  chmod +x /bin/tini
RUN mkdir -p /usr/share/fonts/local && \
  curl --retry 8 -S -L -o /usr/share/fonts/local/NotoSansCJK-Regular.ttc https://github.com/googlefonts/noto-cjk/raw/NotoSansV2.001/NotoSansCJK-Regular.ttc && \
  echo "5dcd1c336cc9344cb77c03a0cd8982ca8a7dc97d620fd6c9c434e02dcb1ceeb3  /usr/share/fonts/local/NotoSansCJK-Regular.ttc" | sha256sum -c -


################################################################################
# Build stage 1 (the actual Kibana image):
#
# Copy kibana from stage 0
# Add entrypoint
################################################################################
FROM redhat/ubi9-minimal:latest
EXPOSE 5601

RUN microdnf install --setopt=tsflags=nodocs -y \
      fontconfig liberation-fonts-common freetype shadow-utils nss findutils && \
      microdnf clean all

# Bring in Kibana from the initial stage.
COPY --from=builder --chown=1000:0 /usr/share/kibana /usr/share/kibana
COPY --from=builder --chown=0:0 /bin/tini /bin/tini
# Load reporting fonts
COPY --from=builder --chown=0:0 /usr/share/fonts/local/NotoSansCJK-Regular.ttc /usr/share/fonts/local/NotoSansCJK-Regular.ttc
RUN fc-cache -v
WORKDIR /usr/share/kibana

RUN ln -s /usr/share/kibana /opt/kibana

ENV ELASTIC_CONTAINER=true
ENV PATH=/usr/share/kibana/bin:$PATH

# Set some Kibana configuration defaults.
COPY --chown=1000:0 config/kibana.yml /usr/share/kibana/config/kibana.yml

# Add the launcher/wrapper script. It knows how to interpret environment
# variables and translate them to Kibana CLI options.
COPY bin/kibana-docker /usr/local/bin/

# Ensure gid 0 write permissions for OpenShift.
RUN chmod g+ws /usr/share/kibana && \
    find /usr/share/kibana -gid 0 -and -not -perm /g+w -exec chmod g+w {} \;

# Remove the suid bit everywhere to mitigate "Stack Clash"
RUN find / -xdev -perm -4000 -exec chmod u-s {} +

# Provide a non-root user to run the process.
RUN groupadd --gid 1000 kibana && \
    useradd --uid 1000 --gid 1000 -G 0 \
      --home-dir /usr/share/kibana --no-create-home \
      kibana

LABEL org.label-schema.build-date="2025-04-30T12:17:16.628Z" \
  org.label-schema.license="Elastic License" \
  org.label-schema.name="Kibana" \
  org.label-schema.schema-version="1.0" \
  org.label-schema.url="https://www.elastic.co/products/kibana" \
  org.label-schema.usage="https://www.elastic.co/guide/en/kibana/reference/index.html" \
  org.label-schema.vcs-ref="0df588b1a307954a3103991180df23c70184d34c" \
  org.label-schema.vcs-url="https://github.com/elastic/kibana" \
  org.label-schema.vendor="Elastic" \
  org.label-schema.version="9.0.1" \
  org.opencontainers.image.created="2025-04-30T12:17:16.628Z" \
  org.opencontainers.image.documentation="https://www.elastic.co/guide/en/kibana/reference/index.html" \
  org.opencontainers.image.licenses="Elastic License" \
  org.opencontainers.image.revision="0df588b1a307954a3103991180df23c70184d34c" \
  org.opencontainers.image.source="https://github.com/elastic/kibana" \
  org.opencontainers.image.title="Kibana" \
  org.opencontainers.image.url="https://www.elastic.co/products/kibana" \
  org.opencontainers.image.vendor="Elastic" \
  org.opencontainers.image.version="9.0.1"

LABEL name="Kibana" \
  maintainer="infra@elastic.co" \
  vendor="Elastic" \
  version="9.0.1" \
  release="1" \
  summary="Kibana" \
  description="Your window into the Elastic Stack."

RUN mkdir /licenses && ln LICENSE.txt /licenses/LICENSE

ENTRYPOINT ["/bin/tini", "--"]


CMD ["/usr/local/bin/kibana-docker"]


USER 1000
