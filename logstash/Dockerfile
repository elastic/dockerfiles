# This Dockerfile was generated from templates/Dockerfile.erb
            
# Build env2yaml
FROM golang:1.23 as builder-env2yaml

COPY env2yaml/env2yaml.go /tmp/go/src/env2yaml/env2yaml.go
COPY env2yaml/go.mod      /tmp/go/src/env2yaml/go.mod
COPY env2yaml/go.sum      /tmp/go/src/env2yaml/go.sum

WORKDIR /tmp/go/src/env2yaml

RUN go build

# Build main image
# Minimal distributions do not ship with en language packs.
FROM docker.elastic.co/ubi9/ubi-minimal

ENV ELASTIC_CONTAINER true
ENV PATH=/usr/share/logstash/bin:$PATH
ENV LANG=C.UTF-8 LC_ALL=C.UTF-8

WORKDIR /usr/share

COPY --from=builder-env2yaml /tmp/go/src/env2yaml/env2yaml /usr/local/bin/env2yaml
COPY config/pipelines.yml config/log4j2.properties config/log4j2.file.properties config/
COPY config/logstash-full.yml config/logstash.yml
COPY pipeline/default.conf pipeline/logstash.conf
COPY bin/docker-entrypoint /usr/local/bin/

# Install packages
RUN for iter in {1..10}; do \
  microdnf update -y && \
  microdnf upgrade -y && \
  microdnf install -y procps findutils tar gzip && \
  microdnf install -y openssl && \
  microdnf install -y which shadow-utils && \
  microdnf clean all && \
  exit_code=0 && break || \
  exit_code=$? && echo "packaging error: retry $iter in 10s" && \
  microdnf clean all && sleep 10; \
  done; \
  (exit $exit_code)

# Provide a non-root user to run the process
# Add Logstash itself and set permissions
RUN groupadd --gid 1000 logstash && \
  adduser --uid 1000 --gid 1000 \
  --home "/usr/share/logstash" \
  --no-create-home \
  logstash && \
  curl -Lo - https://artifacts.elastic.co/downloads/logstash/logstash-9.0.0-rc1-linux-x86_64.tar.gz | \
  tar zxf - -C /usr/share && \
  mv /usr/share/logstash-9.0.0-rc1 /usr/share/logstash && \
  chown -R logstash:root /usr/share/logstash config/ pipeline/ && \
  chmod -R g=u /usr/share/logstash && \
  mv config/* /usr/share/logstash/config && \
  mv pipeline /usr/share/logstash/pipeline && \
  mkdir /licenses && \
  mv /usr/share/logstash/NOTICE.TXT /licenses/NOTICE.TXT && \
  mv /usr/share/logstash/LICENSE.txt /licenses/LICENSE.txt && \
  find /usr/share/logstash -type d -exec chmod g+s {} \; && \
  ln -s /usr/share/logstash /opt/logstash && \
  chmod 0755 /usr/local/bin/docker-entrypoint && \
  rmdir config

WORKDIR /usr/share/logstash

USER 1000

EXPOSE 9600 5044

LABEL org.label-schema.build-date=2025-03-18T02:18:47+00:00 \
  org.label-schema.license="Elastic License" \
  org.label-schema.name="logstash" \
  org.label-schema.schema-version="1.0" \
  org.label-schema.url="https://www.elastic.co/products/logstash" \
  org.label-schema.vcs-url="https://github.com/elastic/logstash" \
  org.label-schema.vendor="Elastic" \
  org.label-schema.version="9.0.0-rc1" \
  org.opencontainers.image.created=2025-03-18T02:18:47+00:00 \
  org.opencontainers.image.description="Logstash is a free and open server-side data processing pipeline that ingests data from a multitude of sources, transforms it, and then sends it to your favorite 'stash.'" \
  org.opencontainers.image.licenses="Elastic License" \
  org.opencontainers.image.title="logstash" \
  org.opencontainers.image.vendor="Elastic" \
  org.opencontainers.image.version="9.0.0-rc1" \
  description="Logstash is a free and open server-side data processing pipeline that ingests data from a multitude of sources, transforms it, and then sends it to your favorite 'stash.'" \
  license="Elastic License" \
  maintainer="info@elastic.co" \
  name="logstash" \
  summary="Logstash is a free and open server-side data processing pipeline that ingests data from a multitude of sources, transforms it, and then sends it to your favorite 'stash.'" \
  vendor="Elastic"

ENTRYPOINT ["/usr/local/bin/docker-entrypoint"]
