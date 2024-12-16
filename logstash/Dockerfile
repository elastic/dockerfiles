# This Dockerfile was generated from templates/Dockerfile.erb


                                        
FROM ubuntu:20.04

RUN for iter in {1..10}; do \
      export DEBIAN_FRONTEND=noninteractive && \
    apt-get update -y && \
  apt-get upgrade -y && \
  apt-get install -y procps findutils tar gzip && \
        apt-get install -y locales && \
        apt-get install -y curl && \
    apt-get clean all && \
      locale-gen 'en_US.UTF-8' && \
    apt-get clean metadata && \
  exit_code=0 && break || exit_code=$? && \
echo "packaging error: retry $iter in 10s" && \
apt-get clean all && \
  apt-get clean metadata && \
sleep 10; done; \
(exit $exit_code)

# Provide a non-root user to run the process.
RUN groupadd --gid 1000 logstash && \
  adduser --uid 1000 --gid 1000 --home /usr/share/logstash --no-create-home logstash

# Add Logstash itself.
RUN curl -Lo - https://artifacts.elastic.co/downloads/logstash/logstash-8.17.0-linux-$(arch).tar.gz | \
  tar zxf - -C /usr/share && \
  mv /usr/share/logstash-8.17.0 /usr/share/logstash && \
  chown --recursive logstash:logstash /usr/share/logstash/ && \
  chown -R logstash:root /usr/share/logstash && \
  chmod -R g=u /usr/share/logstash && \
  mkdir /licenses/ && \
  mv /usr/share/logstash/NOTICE.TXT /licenses/NOTICE.TXT && \
  mv /usr/share/logstash/LICENSE.txt /licenses/LICENSE.txt && \
  find /usr/share/logstash -type d -exec chmod g+s {} \; && \
  ln -s /usr/share/logstash /opt/logstash

WORKDIR /usr/share/logstash
ENV ELASTIC_CONTAINER true
ENV PATH=/usr/share/logstash/bin:$PATH

# Provide a minimal configuration, so that simple invocations will provide
# a good experience.
  COPY config/logstash-full.yml config/logstash.yml
COPY config/pipelines.yml config/log4j2.properties config/log4j2.file.properties config/
COPY pipeline/default.conf pipeline/logstash.conf

RUN chown --recursive logstash:root config/ pipeline/
# Ensure Logstash gets the correct locale by default.
ENV LANG=en_US.UTF-8 LC_ALL=en_US.UTF-8

COPY env2yaml/env2yaml-amd64 env2yaml/env2yaml-arm64 env2yaml/
# Copy over the appropriate env2yaml artifact
RUN env2yamlarch="$(dpkg --print-architecture)"; \
  case "${env2yamlarch}" in \
    'x86_64'|'amd64') \
      env2yamlarch=amd64; \
      ;; \
    'aarch64'|'arm64') \
      env2yamlarch=arm64; \
      ;; \
    *) echo >&2 "error: unsupported architecture '$env2yamlarch'"; exit 1 ;; \
  esac; \
  mkdir -p /usr/local/bin; \
  cp env2yaml/env2yaml-${env2yamlarch} /usr/local/bin/env2yaml; \
  rm -rf env2yaml
# Place the startup wrapper script.
COPY bin/docker-entrypoint /usr/local/bin/

RUN chmod 0755 /usr/local/bin/docker-entrypoint


USER 1000

EXPOSE 9600 5044

LABEL  org.label-schema.schema-version="1.0" \
  org.label-schema.vendor="Elastic" \
  org.opencontainers.image.vendor="Elastic" \
  org.label-schema.name="logstash" \
  org.opencontainers.image.title="logstash" \
  org.label-schema.version="8.17.0" \
  org.opencontainers.image.version="8.17.0" \
  org.label-schema.url="https://www.elastic.co/products/logstash" \
  org.label-schema.vcs-url="https://github.com/elastic/logstash" \
  org.label-schema.license="Elastic License" \
  org.opencontainers.image.licenses="Elastic License" \
  org.opencontainers.image.description="Logstash is a free and open server-side data processing pipeline that ingests data from a multitude of sources, transforms it, and then sends it to your favorite 'stash.'" \
  org.label-schema.build-date=2024-12-05T00:55:38+00:00 \
  org.opencontainers.image.created=2024-12-05T00:55:38+00:00

ENTRYPOINT ["/usr/local/bin/docker-entrypoint"]
