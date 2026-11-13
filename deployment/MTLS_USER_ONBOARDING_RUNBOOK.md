# Runbook: onboarding an application to Kafka with mTLS

> Names below are samples. Replace `my-kafka-cluster` (Kafka cluster name), `kafka` (Kafka namespace) and `clientd` (mTLS listener name) with the values for your environment.

Cluster `my-kafka-cluster`, namespace `kafka`, listener `clientd` (port 9093, `tls: true`, `authentication: tls`).
Bootstrap (in-cluster): `my-kafka-cluster-kafka-bootstrap.kafka.svc:9093`.

Two roles:
- **Dataverse team (Kafka cluster maintainers):** steps 1 to 4. Creates the user and ACLs and hands over the credentials.
- **Application team:** steps 5 to 8. Configures the application and verifies the connection.

Background: the Strimzi **Cluster CA** signs the broker certificates, so clients need it as their truststore. The **Clients CA** signs user certificates, which are the client's identity. Kafka sees the client as `User:CN=<KafkaUser name>`.

---

## Part A: Dataverse team

### 1. Collect the request

| Item | Example |
|---|---|
| App name and owning team | `workflow-service` |
| KafkaUser name (lowercase, DNS-safe) | `workflow-service` |
| Topics, with Read / Write / Describe | `workflow` (Read, Write, Describe), `workflow-dlq` (Write) |
| Consumer groups | `workflow-consumer` (Read) |
| Where the app runs | same cluster namespace `<ns>`, or another cluster |
| Java or non-Java client | Java (PKCS12) or librdkafka, Python, Go, Node (PEM) |

Give each app its own user. Never share one certificate between apps.

### 2. Create the KafkaUser with ACLs

```yaml
apiVersion: kafka.strimzi.io/v1
kind: KafkaUser
metadata:
  name: workflow-service
  namespace: kafka
  labels:
    strimzi.io/cluster: my-kafka-cluster
spec:
  authentication:
    type: tls
  authorization:
    type: simple
    acls:
      - resource: { type: topic, name: workflow, patternType: literal }
        operations: [Read, Write, Describe]
      - resource: { type: topic, name: workflow-dlq, patternType: literal }
        operations: [Write, Describe]
      - resource: { type: group, name: workflow-consumer, patternType: literal }
        operations: [Read]
```

```bash
kubectl apply -f workflow-service-user.yaml
kubectl -n kafka get kafkauser workflow-service          # READY must be True
kubectl -n kafka get secret workflow-service             # created by the User Operator
```

If the user stays not READY: `kubectl -n kafka logs deploy/my-kafka-cluster-entity-operator -c user-operator --tail=100`.

Use prefix ACLs (`patternType: prefix`) for topic families, for example `workflow-`.

### 3. Files the application needs

There are two secrets. All the needed files are inside them.

| Secret | Key | Needed by | Purpose |
|---|---|---|---|
| `my-kafka-cluster-cluster-ca-cert` | `ca.crt` | PEM clients | Trust the brokers (Cluster CA, public cert) |
| `my-kafka-cluster-cluster-ca-cert` | `ca.p12` + `ca.password` | Java (PKCS12) | Truststore and its password |
| `<user>` (for example `workflow-service`) | `user.crt` + `user.key` | PEM clients | Client certificate and private key |
| `<user>` | `user.p12` + `user.password` | Java (PKCS12) | Keystore and its password |
| `<user>` | `ca.crt` | optional | Clients CA (not needed for connecting) |

Each client needs only one set, either PKCS12 or PEM. Treat `user.key`, `user.p12` and the two passwords as secrets. Never put them in Git, chat or email.

### 4. Hand over the credentials

**Option A: same cluster, different namespace (recommended: copy the secrets into the app namespace)**
```bash
SRC=kafka; APP_NS=<app-namespace>; USER=workflow-service
for s in my-kafka-cluster-cluster-ca-cert $USER; do
  kubectl -n $SRC get secret $s -o json \
   | jq 'del(.metadata.namespace,.metadata.uid,.metadata.resourceVersion,.metadata.creationTimestamp,.metadata.ownerReferences,.metadata.labels,.metadata.annotations)' \
   | kubectl -n $APP_NS apply -f -
done
```
This is a point-in-time copy. User certificates renew and the CAs rotate, so automate the sync (External Secrets, Vault or similar) or re-run it on a schedule. The default validity is 365 days.

**Option B: another cluster or a team without access to this cluster**

Export the files and deliver them through the approved secret channel (Vault, AWS Secrets Manager or similar). Never use email, Slack or Git.
```bash
NS=kafka; USER=workflow-service
mkdir -p out && cd out
for k in ca.crt ca.p12 ca.password;   do kubectl -n $NS get secret my-kafka-cluster-cluster-ca-cert -o jsonpath="{.data.${k//./\\.}}" | base64 -d > $k; done
for k in user.crt user.key user.p12 user.password; do kubectl -n $NS get secret $USER -o jsonpath="{.data.${k//./\\.}}" | base64 -d > $k; done
openssl pkcs12 -in user.p12 -passin file:user.password -noout && echo "user.p12 password OK"
```
Delete the local copies once they are stored.

Tell the app team:
- the bootstrap address, and which one to use (`my-kafka-cluster-kafka-bootstrap...:9093`, in-cluster only)
- the topics, groups and operations they have
- which certificate set (PKCS12 or PEM) to use
- the renewal behaviour (see step 9)

For apps in another cluster, 9093 is not reachable because it is an internal listener. An external listener (for example an internal NLB) is needed first. See `MTLS_SOP.md`.

---

## Part B: Application team

### 5. Mount the secrets in the application pod

```yaml
spec:
  containers:
    - name: app
      volumeMounts:
        - { name: kafka-ca,   mountPath: /etc/kafka/certs/ca,   readOnly: true }
        - { name: kafka-user, mountPath: /etc/kafka/certs/user, readOnly: true }
      env:
        - name: KAFKA_BOOTSTRAP
          value: my-kafka-cluster-kafka-bootstrap.kafka.svc:9093
        - name: KAFKA_TRUSTSTORE_PASSWORD
          valueFrom: { secretKeyRef: { name: my-kafka-cluster-cluster-ca-cert, key: ca.password } }
        - name: KAFKA_KEYSTORE_PASSWORD
          valueFrom: { secretKeyRef: { name: workflow-service, key: user.password } }
  volumes:
    - name: kafka-ca
      secret: { secretName: my-kafka-cluster-cluster-ca-cert }
    - name: kafka-user
      secret: { secretName: workflow-service }
```
Secret volume updates reach the pod automatically after a renewal, but most clients read the files only at startup. See step 9.

### 6. Configure the client

**Java (Spring Boot, Kafka clients) with PKCS12**
```properties
bootstrap.servers=${KAFKA_BOOTSTRAP}
security.protocol=SSL
ssl.truststore.type=PKCS12
ssl.truststore.location=/etc/kafka/certs/ca/ca.p12
ssl.truststore.password=${KAFKA_TRUSTSTORE_PASSWORD}
ssl.keystore.type=PKCS12
ssl.keystore.location=/etc/kafka/certs/user/user.p12
ssl.keystore.password=${KAFKA_KEYSTORE_PASSWORD}
ssl.endpoint.identification.algorithm=https
```
Spring Boot uses the same keys with the `spring.kafka.properties.` prefix. For example, `spring.kafka.properties.security.protocol=SSL` and `spring.kafka.ssl.trust-store-location=file:/etc/kafka/certs/ca/ca.p12`.

**Java with PEM (no passwords)**
```properties
security.protocol=SSL
ssl.truststore.type=PEM
ssl.truststore.location=/etc/kafka/certs/ca/ca.crt
ssl.keystore.type=PEM
ssl.keystore.certificate.chain=/etc/kafka/certs/user/user.crt
ssl.keystore.key=/etc/kafka/certs/user/user.key
```

**librdkafka (confluent-kafka-python, node-rdkafka, Go confluent client)**
```properties
bootstrap.servers=my-kafka-cluster-kafka-bootstrap.kafka.svc:9093
security.protocol=ssl
ssl.ca.location=/etc/kafka/certs/ca/ca.crt
ssl.certificate.location=/etc/kafka/certs/user/user.crt
ssl.key.location=/etc/kafka/certs/user/user.key
```

**kafka-python**
```python
KafkaProducer(bootstrap_servers=BOOTSTRAP, security_protocol="SSL",
              ssl_cafile="/etc/kafka/certs/ca/ca.crt",
              ssl_certfile="/etc/kafka/certs/user/user.crt",
              ssl_keyfile="/etc/kafka/certs/user/user.key")
```

**Go (sarama)**
```go
cert, _ := tls.LoadX509KeyPair("/etc/kafka/certs/user/user.crt", "/etc/kafka/certs/user/user.key")
pool := x509.NewCertPool(); ca, _ := os.ReadFile("/etc/kafka/certs/ca/ca.crt"); pool.AppendCertsFromPEM(ca)
cfg.Net.TLS.Enable = true
cfg.Net.TLS.Config = &tls.Config{Certificates: []tls.Certificate{cert}, RootCAs: pool}
```

Rules:
- Use the full service name (`my-kafka-cluster-kafka-bootstrap.kafka.svc`) so hostname verification matches the certificate.
- Do not set any SASL settings for this listener. mTLS needs `security.protocol=SSL` only.
- Keep the truststore password (`ca.password`) and the keystore password (`user.password`) separate.

### 7. Verify from the application pod (or a debug pod with the same mounts)

```bash
cat > /tmp/client.properties <<EOF
security.protocol=SSL
ssl.truststore.type=PKCS12
ssl.truststore.location=/etc/kafka/certs/ca/ca.p12
ssl.truststore.password=$(cat /etc/kafka/certs/ca/ca.password)
ssl.keystore.type=PKCS12
ssl.keystore.location=/etc/kafka/certs/user/user.p12
ssl.keystore.password=$(cat /etc/kafka/certs/user/user.password)
ssl.endpoint.identification.algorithm=https
EOF
B=my-kafka-cluster-kafka-bootstrap.kafka.svc:9093
kafka-topics.sh --bootstrap-server $B --command-config /tmp/client.properties --list
kafka-console-producer.sh --bootstrap-server $B --producer.config /tmp/client.properties --topic workflow
kafka-console-consumer.sh --bootstrap-server $B --consumer.config /tmp/client.properties --topic workflow --group workflow-consumer --from-beginning
```
`ca.password` and `user.password` files exist only if the secret mounts include those keys, which they do in step 5.

### 8. Troubleshooting

| Symptom | Cause and fix |
|---|---|
| `DNS resolution failed` / unresolvable bootstrap | Wrong service name. Use `my-kafka-cluster-kafka-bootstrap` (no `-clientd-`). Cross-namespace needs the `.svc` FQDN. |
| `keystore password was incorrect` | Wrong password or a mismatched copy. Use `user.password` for the keystore and `ca.password` for the truststore, taken from the same secret version as the `.p12`. |
| `PKIX path building failed` | Truststore is missing the Cluster CA, or is stale after a CA rotation. Refresh `my-kafka-cluster-cluster-ca-cert`. |
| `No subject alternative names matching` | Hostname not covered by the broker certificate. Use `my-kafka-cluster-kafka-bootstrap.kafka.svc`. |
| `SSL handshake failed` / `bad_certificate` / `certificate_required` | No keystore, wrong keystore, or the user certificate is not from the Clients CA. |
| `TopicAuthorizationException` / `GroupAuthorizationException` | mTLS works. ACLs are missing. The principal is `User:CN=<user>`. Ask the Dataverse team to add ACLs. |
| Timeout, no error | Network policy or security group blocks 9093. |

### 9. Renewal and rotation

- **User certificate:** renewed automatically by the User Operator (the Secret changes, the password may change too). The app must reload the keystore, so restart the pods or use a client that reloads certificates. A restart after a secret change is the simplest approach. Tools such as Reloader can do it automatically.
- **Cluster CA and Clients CA:** renewed by the Cluster Operator. Old and new CAs are both trusted during the transition, but clients must refresh `ca.crt` / `ca.p12` before the old CA is removed.
- Check expiry: `kubectl -n kafka get secret workflow-service -o jsonpath='{.data.user\.crt}' | base64 -d | openssl x509 -noout -enddate`
- Force a renewal: `kubectl -n kafka annotate secret workflow-service strimzi.io/force-renew=true`.
- If secrets were copied to other namespaces or clusters, refresh the copies after every renewal.

### 10. Offboarding

```bash
kubectl -n kafka delete kafkauser workflow-service       # removes the user, its ACLs and the secret
kubectl -n <app-namespace> delete secret workflow-service my-kafka-cluster-cluster-ca-cert   # remove the copies
```

---

## Checklist

Dataverse team:
- [ ] Request details collected (topics, groups, operations, location, client type)
- [ ] KafkaUser applied and READY
- [ ] Secrets copied or exported through the approved channel
- [ ] Bootstrap address and ACL summary sent to the app team

Application team:
- [ ] Secrets mounted at `/etc/kafka/certs/...`, passwords from Secret env vars
- [ ] `security.protocol=SSL` with the right truststore and keystore
- [ ] CLI test passes (list, produce, consume)
- [ ] Restart or reload procedure for renewals agreed
