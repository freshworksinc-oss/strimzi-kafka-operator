# FIPS image build and vulnerability remediation runbook

Audience: an engineer or agent applying the same work on another branch of this repo
(reference branch: `release-0.45.x-fedramp-fixes`, Strimzi `0.45.2`, Kafka `3.9.2`).

Goal: build the Strimzi **operator**, **Kafka** and **central-utils** images on the RapidFort
FIPS Amazon Linux 2023 base, and drive scanner findings (RapidFort / `*.csv` reports) as low as possible.

Everything below is derived from changes already made on the reference branch. Re-verify each
step with the commands shown; versions will have moved on.

---

## 0. Ground rules

- Registry: `642367526092.dkr.ecr.us-east-1.amazonaws.com/central2.0/`
  - Operator: `k8s_operator:<tag>`
  - Kafka: `kafka_broker:<strimzi-version>-kafka-<kafka-version>-fips[.N]`
  - Utils: `central-utils:<tag>`
- Base images come from `396420358689.dkr.ecr.us-east-1.amazonaws.com/rfcurated`.
- **ECR tags are immutable.** Never reuse a tag. Bump a suffix every time (`-fips.5`, `-fips.6`, ...).
- Target cluster: `dev-central-eks-us-east-1-642367526092`. **Do not touch the staging context
  (`platform-central-staging-...`)** and do not change cluster resources unless explicitly asked.
- Node architecture is **arm64** (Graviton). An image built for the wrong arch fails with
  `exec format error`. Build with podman on an Apple Silicon host (native arm64) or pass `--arch`.
- Container tool: `podman` (`DOCKER_CMD=podman`). Build outputs are local only; **push from a shell
  that has valid AWS credentials** (the agent sandbox usually does not).
- Always `git diff` before and after. Do not commit scan outputs (`*.csv`, `.scan-tmp/`, `*.xlsx`).

---

## 1. Build architecture (how the images are produced)

```
 rfcurated:rfal-al23-fips-rfcurated-<TS>   (RapidFort FIPS AL2023, has rf-openssl)
            |
   docker-images/base/Dockerfile  ->  strimzi/base:latest        (+ Corretto 17, tar, shadow-utils,
            |                                                      curl/systemd/krb5 upgrade, tini)
            +--> docker-images/operator/Dockerfile       -> strimzi/operator   -> k8s_operator
            +--> docker-images/kafka-based/kafka/Dockerfile -> strimzi/kafka   -> kafka_broker
                    |-- Kafka tarball (docker-images/artifacts/binaries/kafka/archives)
                    |-- 3rd-party overlay jars  (artifacts/kafka-thirdparty-libs/3.9.x + cc)
                    |-- kafka_exporter (built from source in a golang stage)
                    `-- Strimzi agents (kafka-agent, mirror-maker-agent, tracing-agent)

 utils/Dockerfile (Go, independent) -> central-utils
```

Key mechanics:

- `make -C docker-images/<dir> docker_build` always tags `strimzi/<project>:latest` plus
  `strimzi/<project>:$(BUILD_TAG)`. Child Dockerfiles use `FROM strimzi/base:latest`,
  so **always rebuild base first**, then the children.
- Operator image contents come from Maven `-dist.zip` files in
  `docker-images/artifacts/binaries/` (produced by `mvn install`). Rebuild Java first.
- Kafka image contents come from: the Kafka tarball, `kafka-thirdparty-libs/<ver>` zip,
  `cc` zip (Cruise Control), and agent jars. The Makefile unpacks them into `tmp/`.

---

## 2. Prerequisites

```bash
brew install podman gnu-sed findutils grep coreutils bash yq jq   # macOS
# Java 17 + Maven
java -version   # 17
mvn -version
# AWS (for pull/push only)
aws ecr get-login-password --region us-east-1 \
  | podman login --username AWS --password-stdin 396420358689.dkr.ecr.us-east-1.amazonaws.com
aws ecr get-login-password --region us-east-1 \
  | podman login --username AWS --password-stdin 642367526092.dkr.ecr.us-east-1.amazonaws.com
```

macOS gotchas:
- BSD `sed -i` fails with `invalid command code`; use `perl -pi -e` for edits in scripts.
- `podman build` rejects `--platform` together with `--arch`. Use only `--arch`.

---

## 3. Triage procedure for a scan report

For each new CSV (`In use,Fix,Vulnerability ID,Package,InstalledVersion,FixedVersion,...,Location`):

1. Group by `Package` + `InstalledVersion`, count CVEs and severities, list `FixedVersion`.
2. Classify each package by owner:

| Location column | Owner | Typical action |
|---|---|---|
| `rpm`, package starts `rf-` | RapidFort | Newer `rfcurated` base tag, else vendor ticket |
| `rpm`, other (`curl`, `glibc`, `libcap`, `systemd-libs`, ...) | Amazon Linux | `dnf upgrade --releasever=latest <pkg>`; remove if unused; else justify |
| `/opt/...*.jar` | Java dependency | Bump version / overlay jar / exclude |
| `/opt/kafka-exporter/kafka_exporter` (Go stdlib) | Go toolchain | Rebuild from source with newer Go |

3. Check whether a fix actually exists **before** changing anything:
   - Maven: `curl -s https://repo1.maven.org/maven2/<group>/<artifact>/maven-metadata.xml | tail`
     or HEAD the `.pom` of the exact fixed version (HTTP 200 = published).
   - RPM: inside the built image, `dnf -q check-update --releasever=latest | grep <pkg>`.
   - Go: `curl -s https://proxy.golang.org/golang.org/toolchain/@v/list | grep linux-arm64`.
4. For every package, test that **removal is safe** with `rpm -e --test <pkg>` inside the image
   (shows which installed packages depend on it) before deleting it.
5. Compare against the previous report (diff by `(Vulnerability ID, Package, InstalledVersion)`) so
   you can report cleared vs new findings.

Helper used throughout (run from the repo root):

```bash
python3 - <<'PY'
import csv
from collections import defaultdict
def load(p):
    with open(p, newline='', encoding='utf-8-sig') as f: return list(csv.DictReader(f))
old, new = load('OLD.csv'), load('NEW.csv')
key = lambda r: (r['Vulnerability ID'], r['Package'], r['InstalledVersion'])
oi, ni = {key(r) for r in old}, {key(r) for r in new}
print('old', len(old), 'new', len(new), 'cleared', len(oi-ni), 'new-findings', len(ni-oi))
pk = defaultdict(lambda: [set(), set(), set()])
for r in new:
    k = (r['Package'], r['InstalledVersion'])
    pk[k][0].add(r['Vulnerability ID']); pk[k][1].add(r['Severity'])
    if r['FixedVersion'].strip(): pk[k][2].add(r['FixedVersion'][:40])
for k, v in sorted(pk.items(), key=lambda x: -len(x[1][0])):
    print(len(v[0]), k, sorted(v[1]), sorted(v[2])[:2] or 'no fix')
PY
```

---

## 4. Base image (`docker-images/base/Dockerfile`)

Changes made:

1. `FROM` the newest RapidFort FIPS tag. Used: `rfal-al23-fips-rfcurated-20261008T123703`
   (previous: `...20260916T220318`). **Always check for a newer tag first.** Compare package versions of
   candidate vs current tag:
   ```bash
   for i in <new-tag> <old-tag>; do podman run --rm --entrypoint sh $i -c \
     'rpm -q rf-openssl rf-expat rf-libxml2 rf-glib2 rf-rpm rf-pcre2 glibc curl-minimal systemd-libs'; done
   ```
   Adopt a new tag when it raises `rf-openssl`, `rf-expat`, `rf-libxml2`, `rf-glib2`.
2. Keep `dnf -y update && dnf -y install java-17-amazon-corretto-headless shadow-utils tar gzip findutils`.
3. Add a second layer pulling security fixes from the latest AL2023 release repo (the pinned repo lags):
   ```dockerfile
   RUN dnf -y upgrade --releasever=latest \
           curl-minimal libcurl-minimal \
           systemd-libs krb5-libs \
           java-${JAVA_VERSION}-amazon-corretto-headless \
       && dnf clean all && rm -rf /var/cache/dnf \
       && curl --version | head -1
   ```
   This cleared all curl/libcurl CRITICAL and HIGH findings. If it fails on conflicts with
   RapidFort packages, fall back to removing curl in leaf images.
4. Tini install block is arch-aware via `TARGETOS/TARGETARCH`; leave as is.

Build and tag:

```bash
make -C docker-images/base DOCKER_CMD=podman docker_build BUILD_TAG=0.45.2-fips.<N>
```

---

## 5. Operator image

### 5.1 Java dependency updates (root `pom.xml`)

Properties changed (reference values; re-check for newer patch releases):

| Property | Value |
|---|---|
| `fasterxml.jackson-*.version` (core, databind, dataformat, annotations, datatype, jaxrs) | `2.18.11` |
| `vertx.version`, `vertx-junit5.version` | `4.5.30` |
| `zookeeper.version` | `3.8.6` |
| `log4j.version` | `2.25.5` |
| `jetty.version` | `9.4.58.v20250814` (9.4.60+ not yet on Maven Central; bump when published) |
| `netty.version` | `4.1.138.Final` |
| `micrometer.version` | `1.15.12` |
| `commons-io.version` (new) | `2.18.0` |
| `lz4.version` (new) | `1.11.4` (`at.yawk.lz4:lz4-java` replaces `org.lz4`) |

Added `dependencyManagement` overrides for `commons-io` and `at.yawk.lz4:lz4-java`
(transitive CVE remediation).

### 5.2 Required code changes for Micrometer 1.15 + Vert.x 4.5 (**do not skip**)

Upgrading Micrometer past 1.13 renames `io.micrometer.prometheus` to `io.micrometer.prometheusmetrics`
and changes `scrape(Writer)`. Symptom if skipped: cluster operator crash
`NoClassDefFoundError: io/micrometer/prometheus/PrometheusMeterRegistry`.

- Imports changed to `io.micrometer.prometheusmetrics.*` in:
  `operator-common/.../HealthCheckAndMetricsServer.java` (+ `HealthCheckAndMetricsServerTest.java`),
  `topic-operator/.../TopicOperatorMain.java`, `user-operator/.../Main.java`.
- `HealthCheckAndMetricsServer`: `prometheusMeterRegistry.scrape(response.getOutputStream())`
  (was `getWriter()`).
- `cluster-operator/.../Main.java`: Vert.x 4.5 still references the old class when
  `VertxPrometheusOptions` is enabled, even with an injected registry. Fix applied:
  ```java
  PrometheusMeterRegistry prometheusMeterRegistry = new PrometheusMeterRegistry(PrometheusConfig.DEFAULT);
  VertxOptions options = new VertxOptions().setMetricsOptions(
      new MicrometerMetricsOptions().setJvmMetricsEnabled(true).setEnabled(true));   // NO VertxPrometheusOptions
  Vertx vertx = Vertx.builder().with(options)
      .withMetrics(new MicrometerMetricsFactory(prometheusMeterRegistry)).build();
  MetricsProvider metricsProvider = new MicrometerMetricsProvider(prometheusMeterRegistry);
  ```
  Metrics remain on `:8080/metrics` (served by `startHealthServer`). Drop the
  `BackendRegistries.getDefaultNow()` usage.

### 5.3 Build the Java artifacts

```bash
mvn clean install -DskipTests -pl cluster-operator,topic-operator,user-operator,kafka-init -am -q
ls docker-images/artifacts/binaries/*-<version>-dist.zip      # 4 zips must be fresh
```

(Run `mvn checkstyle:check` and the touched unit tests before delivering.)

### 5.4 Operator Dockerfile and Makefile changes

`docker-images/operator/Makefile`: wipe `tmp/` on every image build
(`docker_build: clean ...`, `.tmp-prepared` target). Without this, `unzip -o` leaves **stale jars from
previous builds in the image**, which scanners flag. (Hit once; verify with
`podman run --rm --entrypoint ls <img> /opt/strimzi/lib | grep -i <lib>`.)

`docker-images/operator/Dockerfile`, hardening layer (the operator needs only the JRE and bash):

```dockerfile
RUN rpm -qa 'python3*' 'rf-python3*' 'dnf*' 'yum*' 'libdnf*' 'libcomps*' 2>/dev/null \
        | xargs -r rpm -e --nodeps \
    && rpm -e --nodeps curl-minimal libcurl-minimal gnupg2-minimal 2>/dev/null || true \
    && rm -rf /var/cache/dnf /var/lib/dnf /etc/dnf /usr/lib/python3* \
    && java -version && id strimzi
```

**Do not remove** `sqlite-libs` (breaks `rpm`), `libcap`, `libblkid`, `libattr` (needed by
`coreutils`, `shadow-utils`, `libmount`, `rf-rpm-libs`).

### 5.5 Build and tag

```bash
make -C docker-images/base     DOCKER_CMD=podman docker_build BUILD_TAG=0.45.2-fips.<N>
make -C docker-images/operator DOCKER_CMD=podman docker_build BUILD_TAG=0.45.2-fips.<N>
podman tag localhost/strimzi/operator:0.45.2-fips.<N> \
  642367526092.dkr.ecr.us-east-1.amazonaws.com/central2.0/k8s_operator:0.45.2-fips.<N>
```

Smoke test:

```bash
podman run --rm --entrypoint sh strimzi/operator:0.45.2-fips.<N> -c \
  'java -version 2>&1|head -1; rpm -q rf-openssl systemd-libs; rpm -q curl-minimal gnupg2-minimal; uname -m'
# expect: Corretto 17, new rf-openssl, curl/gnupg "not installed", aarch64
```

---

## 6. Kafka image (`docker-images/kafka-based/kafka/`)

### 6.1 Third-party overlay poms

Files: `docker-images/artifacts/kafka-thirdparty-libs/3.9.x/pom.xml`, `3.8.x/pom.xml`, `cc/pom.xml`
(Cruise Control), and `3.9.x/ignorelist` (snakeyaml version).

The Kafka tarball ships old jars; we **delete them in the Makefile and supply patched ones via the
third-party zip**.

3.x poms: properties `jackson.version`, `netty.version`, `zookeeper.version`, `lz4.version`,
`plexus-utils.version`; `netty-bom` and `jackson-bom` imported in `dependencyManagement`;
explicit dependencies for jackson-core/databind/annotations, netty (handler, codec, common, buffer,
transport, resolver, epoll classes + native for linux-x86_64 and linux-aarch_64, unix-common),
`zookeeper` (exclude all transitives) + `zookeeper-jute`, `at.yawk.lz4:lz4-java`, `plexus-utils`,
`org.jline:jline:3.30.17` (replaces jline 3.25.1), `org.xerial.snappy:snappy-java:1.1.10.11`.

Reference versions: jackson `2.18.11`, netty `4.1.138.Final`, zookeeper `3.8.6`, lz4 `1.11.4`.

`cc/pom.xml`: `dependencyManagement` imports `netty-bom`, `jackson-bom`, `jetty-bom`, `log4j-bom`,
`vertx-stack-depchain`; pins `commons-io 2.18.0`, `commons-beanutils 1.11.0`, `commons-lang3 3.18.0`,
`jose4j 0.9.6`, `at.yawk.lz4:lz4-java`, `kafka-clients 4.0.2`, `snappy-java 1.1.10.11`,
`org.json:json 20260814`; excludes `org.lz4:lz4-java` from cruise-control and adds the yawk fork.
Reference: jackson `2.18.11`, netty `4.1.138.Final`, vertx `4.5.30`, jetty `9.4.58.v20250814`,
log4j `2.25.5`, lz4 `1.11.4`.

### 6.2 Makefile (`docker-images/kafka-based/kafka/Makefile`)

- Wipe `tmp/thirdparty-libs-*`, `tmp/cc`, `tmp/<kafka-version>` before unpacking (same stale-jar reason).
- Delete CC `lz4-java-*.jar`/`org.lz4*.jar`.
- After untarring Kafka, delete bundled jars that the overlays replace:
  `netty-*`, `jackson-core/databind/annotations-*`, `zookeeper-*`, `lz4-java-*`, `jline-*`,
  `snappy-java-*`, `plexus-utils-*`.
  **Every jar you add as an overlay for a Kafka-bundled library needs a matching `-name` line here**,
  otherwise two versions end up on the classpath and the scanner still reports the old one.

### 6.3 Regenerate the overlay zips (needed after any pom change)

```bash
cd docker-images/artifacts
for lib in 3.9.x cc; do
  rm -rf kafka-thirdparty-libs/$lib/target
  mvn -q dependency:copy-dependencies -f kafka-thirdparty-libs/$lib/pom.xml
  mkdir -p binaries/kafka-thirdparty-libs
  rm -f binaries/kafka-thirdparty-libs/$lib.zip
  zip -qj binaries/kafka-thirdparty-libs/$lib.zip kafka-thirdparty-libs/$lib/target/dependency/*
done
ls kafka-thirdparty-libs/cc/target/dependency   | grep -E 'jackson-(core|databind)|netty-codec-http2|vertx-core|snappy|^json|lz4'
ls kafka-thirdparty-libs/3.9.x/target/dependency | grep -E 'jackson|jline|lz4|snappy|netty-codec-4'
```

(`docker-images/artifacts/build.sh` does this for every Kafka version but also downloads all
tarballs; the loop above is the fast path when only 3.9.2 is needed. The Kafka tarball must already
be present under `docker-images/artifacts/binaries/kafka/archives/`.)

### 6.4 Dockerfile changes

1. AL2023 package install (full `coreutils` gives the tool parity the scripts expect) followed by
   security upgrade. **The `dnf install` layer reverts `systemd-libs`, so the upgrade must run in the
   same layer after it:**
   ```dockerfile
   RUN dnf -y install --setopt=install_weak_deps=False --allowerasing \
           coreutils gettext net-tools unzip hostname findutils tar \
       && dnf -y upgrade --releasever=latest curl-minimal libcurl-minimal systemd-libs krb5-libs \
       && dnf clean all && rm -rf /var/cache/dnf
   ```
2. Hardening layer (Kafka has no Python/dnf/gnupg need):
   ```dockerfile
   RUN rpm -qa 'python3*' 'rf-python3*' 'dnf*' 'yum*' 'libdnf*' 'libcomps*' 2>/dev/null \
           | xargs -r rpm -e --nodeps \
       && (rpm -e --nodeps gnupg2-minimal 2>/dev/null || true) \
       && rm -rf /var/cache/dnf /var/lib/dnf /etc/dnf /usr/lib/python3*
   ```
   Keep `net-tools` (`netstat` is used by `kafka_liveness.sh` and `kafka_readiness_kraft.sh`).
3. **No `nmap-ncat`**: it pulls `libpcap` (5 HIGH findings, no fixed RPM published). Only
   `scripts/zookeeper_healthcheck.sh` used `nc`; it was rewritten with bash `/dev/tcp`:
   ```bash
   OK=$( { exec 3<>/dev/tcp/127.0.0.1/12181 && echo ruok >&3 && cat <&3; } 2>/dev/null || true )
   [ "$OK" == "imok" ] && exit 0 || exit 1
   ```
   Before removing any tool, `grep -rnE '\b(nc|netstat|rpm|python3?)\b' docker-images/kafka-based/kafka/{scripts,cruise-control-scripts,exporter-scripts}`.
4. **`rev` shim**: AL2023 `coreutils` and `coreutils-single` do not ship `/usr/bin/rev`, but
   `scripts/zookeeper_run.sh` uses `hostname | rev | cut ... | rev`. Keep upstream `zookeeper_run.sh`
   unchanged and install the compatibility script
   `docker-images/kafka-based/kafka/rev-al2023-compat.sh` (fold + tac) as `/usr/local/bin/rev`:
   ```dockerfile
   COPY rev-al2023-compat.sh /usr/local/bin/rev
   RUN chmod 755 /usr/local/bin/rev
   ```
   Symptom if missing: `rev: command not found` and ZooKeeper pods crash looping.
5. **kafka_exporter built from source** to get a patched Go stdlib (the upstream 1.10.0 release binary
   embeds Go 1.27.1: `net/http`, `crypto/tls`, `mime/multipart` CVEs fixed in 1.27.2). Multi-stage
   at the top of the Dockerfile:
   ```dockerfile
   ARG RF_GOLANG_IMAGE=396420358689.dkr.ecr.us-east-1.amazonaws.com/rfcurated:golang-alpine3.21-rfcurated-20261001
   FROM ${RF_GOLANG_IMAGE} AS exporter-build
   USER root
   ARG KAFKA_EXPORTER_VERSION=1.10.0
   ARG KAFKA_EXPORTER_SRC_SHA256=b76e9882a07d5ae23d32b3e801c3b7bc88fec4444d122bb052c009c8583cb237
   ARG GO_TOOLCHAIN=go1.27.2
   RUN set -ex; \
       wget -q -O /tmp/ke.tgz https://github.com/danielqsj/kafka_exporter/archive/refs/tags/v${KAFKA_EXPORTER_VERSION}.tar.gz; \
       echo "${KAFKA_EXPORTER_SRC_SHA256}  /tmp/ke.tgz" | sha256sum -c -; \
       mkdir /src && tar xzf /tmp/ke.tgz -C /src --strip-components=1; cd /src; \
       GOTOOLCHAIN=${GO_TOOLCHAIN} CGO_ENABLED=0 go build -trimpath -ldflags="-s -w" -o /out/kafka_exporter .; \
       cp LICENSE /out/LICENSE
   FROM strimzi/base:latest
   ...
   COPY --from=exporter-build /out/ $KAFKA_EXPORTER_HOME/
   ```
   When bumping the exporter version, recompute the source SHA256:
   `curl -sL <tarball-url> | shasum -a 256`. The builder stage runs natively on the target arch, so
   one Dockerfile serves amd64 and arm64.

### 6.5 Build and tag (only Kafka 3.9.2)

```bash
cd docker-images/kafka-based/kafka
make clean
make DOCKER_CMD=podman \
  DOCKER_BUILD_ARGS="--build-arg KAFKA_VERSION=3.9.2 --build-arg KAFKA_DIST_DIR=./tmp/3.9.2 --build-arg THIRD_PARTY_LIBS=3.9.x" \
  BUILD_TAG=build-kafka-3.9.2 KAFKA_VERSION=3.9.2 THIRD_PARTY_LIBS=3.9.x docker_build

podman tag strimzi/kafka:build-kafka-3.9.2 \
  642367526092.dkr.ecr.us-east-1.amazonaws.com/central2.0/kafka_broker:0.45.2-kafka-3.9.2-fips.<N>
```

(The Makefile prints `warning: overriding commands for target docker_build`; this is harmless and
upstream. The final step runs `find-classpath-collision.sh` and fails the build if jars collide.)

Smoke test:

```bash
podman run --rm --entrypoint bash strimzi/kafka:build-kafka-3.9.2 -c '
  rpm -q curl-minimal systemd-libs rf-openssl; rpm -q python3 nmap-ncat libpcap gnupg2-minimal
  ls /opt/kafka/libs | grep -E "jackson-databind|jline|lz4|snappy|zookeeper-3|netty-codec-4"
  ls /opt/cruise-control/libs | grep -E "jackson-databind|netty-codec-http2|vertx-core|snappy|^json"
  /opt/kafka/bin/kafka-topics.sh --version | tail -1
  /opt/kafka-exporter/kafka_exporter --version 2>&1 | head -1
  echo abc | rev; command -v netstat; java -version 2>&1 | head -1'
```

Expected: python/ncat/libpcap/gnupg "not installed"; exactly one version of each overlaid jar;
`3.9.2`; `cba`.

Recommended: test the new broker image on a **non-production** cluster first (jackson, netty, jline,
lz4, snappy, zookeeper jars changed; run a full broker or ZooKeeper start and the readiness/liveness probes).

---

## 7. central-utils (`utils/`, Go)

Findings were all Go stdlib (old builder 1.24.0) plus `golang.org/x/net` and `x/text`.

- `utils/go.mod`: `go 1.25.0`, `golang.org/x/net v0.56.0`, `golang.org/x/text v0.39.0`
  (+ related `x/sys`, `x/term`).
- `utils/Dockerfile`: builder `rfcurated:golang-alpine3.21-rfcurated-20261001` (Go 1.27.x), `USER root`
  for the build, `scratch` runtime with CA bundle copied from the builder.
- **Architecture bug fixed:** the Dockerfile used `GOARCH=${TARGETARCH:-amd64}`; podman does not set
  `TARGETARCH`, so arm64 builds silently produced **amd64 binaries** (`exec format error` on Graviton).
  Current form compiles for the builder's native arch: `RUN CGO_ENABLED=0 go build -o /tmp/central-utils -v`.
  Build with `podman build --arch arm64 ...` (and `--arch amd64` for x86).
- `utils/Makefile`: bump `BINARY_VERSION` for every release (immutable tags). `make build-podman`
  builds both archs and a manifest, `make image-release` pushes the manifest.
- Verify: `podman image inspect <img> --format '{{.Architecture}}'` and
  `podman run --rm --arch arm64 <img> /central-utils` (expect only a "namespace" error outside a cluster).

---

## 8. Push and deploy (human step, needs AWS credentials)

```bash
aws ecr get-login-password --region us-east-1 | podman login --username AWS --password-stdin 642367526092.dkr.ecr.us-east-1.amazonaws.com
podman push 642367526092.dkr.ecr.us-east-1.amazonaws.com/central2.0/k8s_operator:0.45.2-fips.<N>
podman push 642367526092.dkr.ecr.us-east-1.amazonaws.com/central2.0/kafka_broker:0.45.2-kafka-3.9.2-fips.<N>
```

Then roll the deployments (only when asked). Check the node arch first:
`kubectl get nodes -L kubernetes.io/arch`. For mixed clusters, push a multi-arch manifest.

---

## 9. Results on the reference branch (for expectations)

| Image | Start | After this work | Notes |
|---|---|---|---|
| Operator (`k8s_operator`) | 94 findings, 7 CRITICAL | 32 findings, 0 CRITICAL, 11 HIGH (`-fips.6`) | |
| Kafka (`kafka_broker`) | 183 findings, 7 CRITICAL, 65 HIGH | 41 findings, 0 CRITICAL, 13 HIGH (`-kafka-3.9.2-fips.3`) | |

Order of impact: (1) new rfcurated base, (2) curl/systemd/krb5 upgrade + removing curl/Python/dnf/gnupg,
(3) Java overlay bumps, (4) removing `nmap-ncat`/`libpcap`, (5) source-built exporter, (6) snappy/jline overlay.

---

## 10. Known remaining findings (no repo-side fix as of the reference scans)

| Finding | Why it remains | What to watch |
|---|---|---|
| `rf-rpm` (5 HIGH), `rf-expat` (1 HIGH), `rf-glib2` | RapidFort has not shipped fixes | New `rfal-al23-fips-rfcurated-*` tag, RF ticket |
| `libcap` (7, 3 HIGH) | No Amazon fix; required by `coreutils`, `systemd-libs`, `rf-rpm-libs` | AL2023 updates |
| Jetty 9.4.5x (`jetty-http`, `jetty-server`) | Fixes need 9.4.60+/9.4.64, not on Maven Central (max `9.4.58.v20250814`) | `maven-metadata.xml` for `org.eclipse.jetty:jetty-server`; then bump `jetty.version` in root `pom.xml` and `cc/pom.xml` (Kafka tarball also bundles jetty 9.4.57) |
| Corretto 17.0.20 (3), HdrHistogram 2.2.2 (3), `commons-lang` 2.6 | No fixed versions published | Scanner DB refresh; justification |
| `sqlite-libs`, `libblkid`, `libattr`, `libgcc`, `systemd`, `diffutils`, `net-tools` | No fix, and dependents (`rpm`, `libmount`, `coreutils`) need them | AL2023 updates |
| `opentelemetry-api` 1.32.0 (Kafka) | Fix is 1.62.0, but the SDK/instrumentation are pinned at 1.34.x; untested | Test a tracing-enabled broker before overlaying |

Draft a written justification for each remaining item (exposure: operator and brokers never parse
untrusted XML via expat, never run `rpm`, Jetty only serves cluster-internal health/metrics ports).

---

## 11. Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| `exec format error` on a node | Wrong-arch binary/image | Rebuild with `podman build --arch <arch>`; do not default `GOARCH` to `amd64` |
| `NoClassDefFoundError: io/micrometer/prometheus/PrometheusMeterRegistry` | Micrometer 1.15 with Vert.x 4.5 `VertxPrometheusOptions` | Section 5.2 |
| `rev: command not found` (ZooKeeper) | AL2023 coreutils has no `rev` | `rev-al2023-compat.sh` shim, section 6.4 |
| `rpm: error while loading shared libraries: libsqlite3.so.0` | `sqlite-libs` removed | Never remove `sqlite-libs` |
| `dnf upgrade` conflicts with RapidFort packages | RF-owned package in the set | Upgrade only the listed packages; remove curl in leaf images instead |
| Scanner still shows an old jar | Stale file in `tmp/` or old jar kept from the Kafka tarball | Wipe `tmp/`; add `-name` to the Makefile delete list; re-zip overlays |
| `systemd-libs` back to an old version | Later `dnf install` layer reverted it | Run `dnf upgrade systemd-libs` after the install in the same layer |
| ECR push `tag already exists` | Immutable tag | Bump suffix (`-fips.<N+1>`) |
| ECR login: `UnrecognizedClientException` / `Password:` prompt | AWS credentials expired or sandbox | Refresh credentials and push from a normal shell |
| `unzip -o` leaves old jars | `-o` overwrites only same-named files | Makefile already wipes `tmp/`; keep that |
| BSD `sed -i` errors | macOS sed | `perl -pi -e` or `gsed` |

---

## 12. Checklist for the next update cycle

1. Pull the newest `rfcurated` FIPS tag; diff its package versions against the current one (section 4).
2. Receive the new CSV; run the triage helper (section 3); decide owner per package.
3. Check Maven Central / Go proxy / `dnf check-update` for each fix before editing.
4. Apply poms (root, `kafka-thirdparty-libs/*`), Makefile delete lists, Dockerfile changes.
5. `mvn clean install -DskipTests -pl cluster-operator,topic-operator,user-operator,kafka-init -am`
   plus checkstyle/unit tests; regenerate overlay zips.
6. Build base, then operator, then Kafka; tag with a **new** suffix.
7. Run the smoke tests (sections 5.5, 6.5); test the Kafka image on a non-prod cluster.
8. Push from an authenticated shell, deploy only when asked, rescan, compare CSVs, update section 9/10.
