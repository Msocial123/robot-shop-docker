FROM docker.io/fluent/fluentd:v1.17-1
USER root
RUN apk update && \
    apk add --virtual .build-dependencies build-base ruby-dev

RUN fluent-gem install fluent-plugin-elasticsearch && \
    fluent-gem install fluent-plugin-kubernetes_metadata_filter  && \
    fluent-gem install fluent-plugin-multi-format-parser

