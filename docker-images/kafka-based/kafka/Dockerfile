#####
# Build Kafka Exporter from source to get a patched Go standard library
#####
ARG RF_GOLANG_IMAGE=396420358689.dkr.ecr.us-east-1.amazonaws.com/rfcurated:golang-1.27.3-alpine3.24-fips-rfcurated-20261110
FROM ${RF_GOLANG_IMAGE} AS exporter-build

USER root

ARG KAFKA_EXPORTER_VERSION=1.11.0
ARG KAFKA_EXPORTER_SRC_SHA256=e3366dba6da40559c36f9becc45075c594265815b96da181710bc00c0b0f29a9
# The RapidFort FIPS Go image (Microsoft build of Go 1.27.3) only allows its bundled toolchain
ARG GO_TOOLCHAIN=local
# Patched klauspost/compress (zstd) to fix a CVE in the version pinned by kafka_exporter
ARG ZSTD_VERSION=1.20.2

RUN set -ex; \
    wget -q -O /tmp/ke.tgz https://github.com/danielqsj/kafka_exporter/archive/refs/tags/v${KAFKA_EXPORTER_VERSION}.tar.gz; \
    echo "${KAFKA_EXPORTER_SRC_SHA256}  /tmp/ke.tgz" | sha256sum -c -; \
    mkdir /src && tar xzf /tmp/ke.tgz -C /src --strip-components=1; cd /src; \
    export GOTOOLCHAIN=${GO_TOOLCHAIN} GOFLAGS=-mod=mod; \
    go get github.com/klauspost/compress@v${ZSTD_VERSION}; \
    MS_GO_NOSYSTEMCRYPTO=1 CGO_ENABLED=0 go build -trimpath -ldflags="-s -w" -o /out/kafka_exporter .; \
    cp LICENSE /out/LICENSE

#####
# Build the helper container image for downloading prerequisites
#####
FROM registry.access.redhat.com/ubi9/ubi-minimal:latest AS downloader

ARG TARGETOS
ARG TARGETARCH

RUN microdnf --setopt=install_weak_deps=0 --setopt=tsflags=nodocs install -y \
            # Tar is used to unpack the downloaded binaries here in the Dockerfile \
            tar \
            # gzip is used to unpack the downloaded binaries here in the Dockerfile \
            gzip

#####
# Get Prometheus JMX Exporter
#####
ENV JMX_EXPORTER_HOME=/opt/prometheus-jmx-exporter
ENV JMX_EXPORTER_VERSION=1.6.0
ENV JMX_EXPORTER_CHECKSUM="2b1278f76480c0d5053ca0c7622716403de0e6f6f3ee70acda8157cf97cba2c5eeb2234025e5d66b57951429397c38a6770e934e6e8ffc5a75b0337c719530de  jmx_prometheus_javaagent-${JMX_EXPORTER_VERSION}.jar"

RUN set -ex; \
    curl -LO https://github.com/prometheus/jmx_exporter/releases/download/${JMX_EXPORTER_VERSION}/jmx_prometheus_javaagent-${JMX_EXPORTER_VERSION}.jar; \
    echo $JMX_EXPORTER_CHECKSUM > jmx_prometheus_javaagent-${JMX_EXPORTER_VERSION}.jar.sha512; \
    sha512sum -c jmx_prometheus_javaagent-${JMX_EXPORTER_VERSION}.jar.sha512; \
    mkdir $JMX_EXPORTER_HOME; \
    mv jmx_prometheus_javaagent-${JMX_EXPORTER_VERSION}.jar $JMX_EXPORTER_HOME/; \
    rm -f jmx_prometheus_javaagent-${JMX_EXPORTER_VERSION}.jar.sha512;

#####
# Build the main container image
#####
FROM strimzi/base:latest

LABEL org.opencontainers.image.source='https://github.com/strimzi/strimzi-kafka-operator'

ARG KAFKA_DIST_DIR
ARG KAFKA_VERSION
ARG THIRD_PARTY_LIBS
ARG strimzi_version
ARG TARGETOS
ARG TARGETARCH

LABEL name='kafka' \
    version="${strimzi_version}-${KAFKA_VERSION}" \
    release="${strimzi_version}" \
    summary="Kafka ${KAFKA_VERSION} image of the Strimzi Kafka Operator project." \
    description="Kafka image used by the Strimzi Kafka Operator to run Kafka ${KAFKA_VERSION} clusters."

# net-tools: netstat is used by the health check probes; hostname: used by scripts to find the node ID;
# findutils/tar/gzip/unzip: used by Kafka scripts and our Kafka Connect Build Dockerfiles.
# The security upgrade must run after the install in the same layer, otherwise the install reverts systemd-libs.
RUN dnf -y install --setopt=install_weak_deps=False --allowerasing \
            coreutils gettext net-tools unzip hostname findutils tar gzip \
    && dnf -y upgrade --releasever=latest curl-minimal libcurl-minimal systemd-libs krb5-libs \
    && dnf clean all && rm -rf /var/cache/dnf

# Hardening: Kafka needs no Python, dnf or gnupg
RUN rpm -qa 'python3*' 'rf-python3*' 'dnf*' 'yum*' 'libdnf*' 'libcomps*' 'libmodulemd*' 2>/dev/null \
        | xargs -r rpm -e --nodeps \
    ; (rpm -e --nodeps gnupg2-minimal 2>/dev/null || true); \
    rm -rf /var/cache/dnf /var/lib/dnf /etc/dnf /usr/lib/python3*

# Add kafka user with UID 1001
# The user is in the group 0 to have access to the mounted volumes and storage
RUN useradd -r -m -u 1001 -g 0 kafka

#####
# Add Kafka
#####
ENV KAFKA_HOME=/opt/kafka
ENV KAFKA_VERSION=${KAFKA_VERSION}
ENV STRIMZI_VERSION=${strimzi_version}

COPY $KAFKA_DIST_DIR $KAFKA_HOME
COPY ./scripts/ $KAFKA_HOME
RUN mkdir $KAFKA_HOME/plugins

#####
# Copy Kafka Exporter from the downloader stage and the helper scripts from local filesystem
#####
ENV KAFKA_EXPORTER_HOME=/opt/kafka-exporter
COPY --from=exporter-build /out/kafka_exporter $KAFKA_EXPORTER_HOME/kafka_exporter
COPY ./exporter-scripts $KAFKA_EXPORTER_HOME

#####
# Copy Prometheus JMX Exporter from the downloader stage
#####
ENV JMX_EXPORTER_HOME=/opt/prometheus-jmx-exporter
COPY --from=downloader $JMX_EXPORTER_HOME/jmx_prometheus_javaagent*.jar $JMX_EXPORTER_HOME/

#####
# Add Strimzi agents
#####
COPY ./tmp/kafka-agent-${STRIMZI_VERSION}.jar ${KAFKA_HOME}/libs/
COPY ./tmp/tracing-agent-${STRIMZI_VERSION}.jar ${KAFKA_HOME}/libs/

#####
# Add 3rd party libs
#####
COPY tmp/thirdparty-libs-${THIRD_PARTY_LIBS}/ ${KAFKA_HOME}/libs/

#####
# Add Cruise Control
#####
ENV CRUISE_CONTROL_HOME=/opt/cruise-control
RUN mkdir $CRUISE_CONTROL_HOME
COPY tmp/cc/ ${CRUISE_CONTROL_HOME}/libs/
COPY ./cruise-control-scripts $CRUISE_CONTROL_HOME

WORKDIR $KAFKA_HOME

USER 1001
