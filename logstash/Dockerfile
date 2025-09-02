# This Dockerfile was generated from templates/Dockerfile.erb
  
          
# Build env2yaml
FROM golang:1.25 AS builder-env2yaml

COPY env2yaml/env2yaml.go env2yaml/go.mod env2yaml/go.sum /tmp/go/src/env2yaml/

WORKDIR /tmp/go/src/env2yaml

RUN go build -trimpath

# Build main image
# Minimal distributions do not ship with en language packs.
FROM redhat/ubi9-minimal:latest

ENV ELASTIC_CONTAINER=true
ENV PATH=/usr/share/logstash/bin:$PATH
ENV LANG=C.UTF-8 LC_ALL=C.UTF-8

WORKDIR /usr/share

# Install packages
RUN \
  microdnf install -y procps findutils tar gzip && \
  microdnf install -y openssl && \
  microdnf install -y which shadow-utils && \
  microdnf clean all

# Provide a non-root user to run the process
# Add Logstash itself and set permissions
RUN groupadd --gid 1000 logstash && \
  adduser --uid 1000 --gid 1000 \
  --home "/usr/share/logstash" \
  --no-create-home \
  logstash && \
  arch="$(rpm --query --queryformat='%{ARCH}' rpm)" && \
  curl -f -Lo logstash.tar.gz https://artifacts.elastic.co/downloads/logstash/logstash-9.0.6-linux-${arch}.tar.gz && \
  tar -zxf logstash.tar.gz -C /usr/share && \
  rm logstash.tar.gz && \
  mv /usr/share/logstash-9.0.6 /usr/share/logstash && \
  chown -R logstash:root /usr/share/logstash && \
  chmod -R g=u /usr/share/logstash && \
  mkdir /licenses && \
  mv /usr/share/logstash/NOTICE.TXT /licenses/NOTICE.TXT && \
  mv /usr/share/logstash/LICENSE.txt /licenses/LICENSE.txt && \
  find /usr/share/logstash -type d -exec chmod g+s {} \; && \
  ln -s /usr/share/logstash /opt/logstash

COPY --from=builder-env2yaml /tmp/go/src/env2yaml/env2yaml /usr/local/bin/env2yaml
COPY --chown=logstash:root config/pipelines.yml config/log4j2.properties config/log4j2.file.properties /usr/share/logstash/config/
COPY --chown=logstash:root config/logstash-full.yml /usr/share/logstash/config/logstash.yml
COPY --chown=logstash:root pipeline/default.conf /usr/share/logstash/pipeline/logstash.conf
COPY --chmod=0755 bin/docker-entrypoint /usr/local/bin/

WORKDIR /usr/share/logstash

USER 1000

EXPOSE 9600 5044

LABEL org.label-schema.build-date=2025-08-20T01:17:10+00:00 \
  org.label-schema.license="Elastic License" \
  org.label-schema.name="logstash" \
  org.label-schema.schema-version="1.0" \
  org.label-schema.url="https://www.elastic.co/products/logstash" \
  org.label-schema.vcs-url="https://github.com/elastic/logstash" \
  org.label-schema.vendor="Elastic" \
  org.label-schema.version="9.0.6" \
  org.opencontainers.image.created=2025-08-20T01:17:10+00:00 \
  org.opencontainers.image.description="Logstash is a free and open server-side data processing pipeline that ingests data from a multitude of sources, transforms it, and then sends it to your favorite 'stash.'" \
  org.opencontainers.image.licenses="Elastic License" \
  org.opencontainers.image.title="logstash" \
  org.opencontainers.image.vendor="Elastic" \
  org.opencontainers.image.version="9.0.6" \
  description="Logstash is a free and open server-side data processing pipeline that ingests data from a multitude of sources, transforms it, and then sends it to your favorite 'stash.'" \
  license="Elastic License" \
  maintainer="info@elastic.co" \
  name="logstash" \
  summary="Logstash is a free and open server-side data processing pipeline that ingests data from a multitude of sources, transforms it, and then sends it to your favorite 'stash.'" \
  vendor="Elastic"

ENTRYPOINT ["/usr/local/bin/docker-entrypoint"]
