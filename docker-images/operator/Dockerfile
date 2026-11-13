FROM strimzi/base:latest

LABEL org.opencontainers.image.source='https://github.com/strimzi/strimzi-kafka-operator'

# Hardening: the operator only needs the JRE and bash, so remove Python, dnf and curl/gnupg
RUN rpm -qa 'python3*' 'rf-python3*' 'dnf*' 'yum*' 'libdnf*' 'libcomps*' 'libmodulemd*' 'dbus-libs*' 2>/dev/null \
        | xargs -r rpm -e --nodeps \
    ; rpm -e --nodeps curl-minimal libcurl-minimal gnupg2-minimal 2>/dev/null || true; \
    rm -rf /var/cache/dnf /var/lib/dnf /etc/dnf /usr/lib/python3* \
    && java -version

# Add strimzi user with UID 1001
# The user is in the group 0 to have access to the mounted volumes and storage
RUN useradd -r -m -u 1001 -g 0 strimzi

ARG strimzi_version=1.0-SNAPSHOT

LABEL name='operator' \
    summary='Operator image of the Strimzi Kafka Operator project.' \
    description='Operator image used by the cluster and entity operator components of the Strimzi Kafka Operator.'

ENV STRIMZI_VERSION=${strimzi_version}
ENV STRIMZI_HOME=/opt/strimzi
RUN mkdir -p ${STRIMZI_HOME}/bin
WORKDIR ${STRIMZI_HOME}

COPY scripts/ tmp/bin/ bin/

COPY tmp/lib/ lib/

USER 1001
