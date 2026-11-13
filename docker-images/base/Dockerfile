ARG RF_BASE_IMAGE=396420358689.dkr.ecr.us-east-1.amazonaws.com/rfcurated:rfal-al23-fips-rfcurated-20261110

#####
# Build the helper container image for downloading prerequisites
#####
FROM registry.access.redhat.com/ubi9/ubi-minimal:latest AS downloader

ARG TARGETOS
ARG TARGETARCH

#####
# Get Tini
#####
ENV TINI_VERSION=v0.19.0
ENV TINI_SHA256_AMD64=93dcc18adc78c65a028a84799ecf8ad40c936fdfc5f2a57b1acda5a8117fa82c
ENV TINI_SHA256_ARM64=07952557df20bfd2a95f9bef198b445e006171969499a1d361bd9e6f8e5e0e81
ENV TINI_SHA256_PPC64LE=3f658420974768e40810001a038c29d003728c5fe86da211cff5059e48cfdfde
ENV TINI_SHA256_S390X=931b70a182af879ca249ae9de87ef68423121b38d235c78997fafc680ceab32d

RUN set -ex; \
    if [[ "${TARGETOS}/${TARGETARCH}" = "linux/ppc64le" ]]; then \
        curl -s -L https://github.com/krallin/tini/releases/download/${TINI_VERSION}/tini-ppc64le -o /usr/bin/tini; \
        echo "${TINI_SHA256_PPC64LE} */usr/bin/tini" | sha256sum -c; \
        chmod +x /usr/bin/tini; \
    elif [[ "${TARGETOS}/${TARGETARCH}" = "linux/arm64" ]]; then \
        curl -s -L https://github.com/krallin/tini/releases/download/${TINI_VERSION}/tini-arm64 -o /usr/bin/tini; \
        echo "${TINI_SHA256_ARM64} */usr/bin/tini" | sha256sum -c; \
        chmod +x /usr/bin/tini; \
    elif [[ "${TARGETOS}/${TARGETARCH}" = "linux/s390x" ]]; then \
        curl -s -L https://github.com/krallin/tini/releases/download/${TINI_VERSION}/tini-s390x -o /usr/bin/tini; \
        echo "${TINI_SHA256_S390X} */usr/bin/tini" | sha256sum -c; \
        chmod +x /usr/bin/tini; \
    else \
        curl -s -L https://github.com/krallin/tini/releases/download/${TINI_VERSION}/tini -o /usr/bin/tini; \
        echo "${TINI_SHA256_AMD64} */usr/bin/tini" | sha256sum -c; \
        chmod +x /usr/bin/tini; \
    fi

#####
# Build the main container image
#####
FROM ${RF_BASE_IMAGE}

LABEL org.opencontainers.image.source='https://github.com/strimzi/strimzi-kafka-operator'

ARG JAVA_VERSION=21
ARG TARGETOS
ARG TARGETARCH
ARG strimzi_version

LABEL name='base' \
    vendor='Strimzi' \
    version="${strimzi_version}" \
    release="${strimzi_version}" \
    summary='Base image of the Strimzi Kafka Operator project.' \
    description='Strimzi Kafka Operator base image, containing common utilities across all component images.'

USER root

# RapidFort FIPS Amazon Linux 2023 base (ships rf-openssl), plus Java and the tools needed by our scripts
RUN dnf -y update \
    && dnf -y install --setopt=install_weak_deps=False \
            java-${JAVA_VERSION}-amazon-corretto-headless \
            shadow-utils tar gzip findutils \
    && dnf clean all && rm -rf /var/cache/dnf

# Pull the latest security fixes from the newest AL2023 release repo (the pinned repo lags)
RUN dnf -y upgrade --releasever=latest \
            curl-minimal libcurl-minimal \
            systemd-libs krb5-libs ca-certificates \
            system-release amazon-linux-repo-cdn \
            java-${JAVA_VERSION}-amazon-corretto-headless \
    && dnf clean all && rm -rf /var/cache/dnf \
    && curl --version | head -1 \
    # The newer ca-certificates package ships /etc/ssl/openssl.cnf -> /etc/pki/tls/openssl.cnf, while the RapidFort
    # openssl keeps /etc/pki/tls/openssl.cnf -> /etc/ssl/openssl.cnf. That is a symlink loop which breaks the openssl
    # CLI (used by the User Operator and Cluster Operator for certificate generation). Restore the RapidFort layout.
    && ln -sfn openssl.cnf.non-strict-140-3 /etc/ssl/openssl.cnf \
    && openssl req -new -batch -nodes -newkey rsa:2048 -keyout /dev/null -subj /CN=t >/dev/null

ENV JAVA_HOME=/usr/lib/jvm/java-${JAVA_VERSION}-amazon-corretto

#####
# Copy Tini from the downloader stage
#####
COPY --from=downloader /usr/bin/tini /usr/bin/tini
