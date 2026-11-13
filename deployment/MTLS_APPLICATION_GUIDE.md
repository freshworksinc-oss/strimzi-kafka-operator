# Connecting your application to Kafka with mTLS

This guide shows how to configure your application to connect securely to kafka cluster

> Names below are samples. Use the values the Dataverse team gives you for the bootstrap address, secret names and topics.

## 1. How it works

- The connection is encrypted with TLS, and **both sides present a certificate** (mutual TLS).
- Your application **trusts the Kafka brokers** using the cluster CA certificate (the truststore).
- Your application **proves its identity** with a client certificate issued for your application (the keystore).
- Kafka identifies your application by the certificate name, `CN=<your-user>`. Access to topics and consumer groups is controlled by ACLs that the Dataverse team sets up for that user.
- There is no username or password login. Do not configure SASL.

## 2. What you get from the Dataverse team

| Item | Example |
|---|---|
| Bootstrap address | `my-kafka-cluster-kafka-bootstrap.kafka.svc:9093` |
| Kafka user name | `workflow-service` |
| Topics and permissions | `workflow` (read, write), `workflow-dlq` (write) |
| Consumer group(s) | `workflow-consumer` |
| Two Kubernetes secrets in your namespace | `my-kafka-cluster-cluster-ca-cert` (CA) and `workflow-service` (your client certificate) |

If your application runs outside the Kafka cluster's Kubernetes cluster, the Dataverse team will send the files through the approved FWSS. 

Files inside the two secrets (you need only one format):

| Format | CA (trust the brokers) | Client identity |
|---|---|---|
| **PKCS12** (Java) | `ca.p12` + `ca.password` | `user.p12` + `user.password` |
| **PEM** (Python, Go, Node, librdkafka, also Java) | `ca.crt` | `user.crt` + `user.key` |

Treat the private key (`user.key`, `user.p12`) and the passwords as secrets. Never commit them to Git, paste them in chat, or bake them into images.

## 3. Mount the secrets in your pod

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
        # Only needed for PKCS12 (Java)
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

The files then appear under `/etc/kafka/certs/ca/` and `/etc/kafka/certs/user/`.

## 4. Configure your client

Always use the full bootstrap name you were given. The broker certificate is issued for that name, so a different hostname or an IP address fails hostname verification.

### Java / Spring Boot (PKCS12)

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

Spring Boot:
```yaml
spring:
  kafka:
    bootstrap-servers: ${KAFKA_BOOTSTRAP}
    security:
      protocol: SSL
    ssl:
      trust-store-type: PKCS12
      trust-store-location: file:/etc/kafka/certs/ca/ca.p12
      trust-store-password: ${KAFKA_TRUSTSTORE_PASSWORD}
      key-store-type: PKCS12
      key-store-location: file:/etc/kafka/certs/user/user.p12
      key-store-password: ${KAFKA_KEYSTORE_PASSWORD}
    consumer:
      group-id: workflow-consumer
```

### Java (PEM, no passwords, Kafka clients 3.3+)

```properties
security.protocol=SSL
ssl.truststore.type=PEM
ssl.truststore.location=/etc/kafka/certs/ca/ca.crt
ssl.keystore.type=PEM
ssl.keystore.certificate.chain=/etc/kafka/certs/user/user.crt
ssl.keystore.key=/etc/kafka/certs/user/user.key
```

### Python (confluent-kafka / librdkafka based clients, including node-rdkafka)

```python
conf = {
    "bootstrap.servers": os.environ["KAFKA_BOOTSTRAP"],
    "security.protocol": "ssl",
    "ssl.ca.location": "/etc/kafka/certs/ca/ca.crt",
    "ssl.certificate.location": "/etc/kafka/certs/user/user.crt",
    "ssl.key.location": "/etc/kafka/certs/user/user.key",
}
```

### Python (kafka-python)

```python
KafkaProducer(bootstrap_servers=os.environ["KAFKA_BOOTSTRAP"], security_protocol="SSL",
              ssl_cafile="/etc/kafka/certs/ca/ca.crt",
              ssl_certfile="/etc/kafka/certs/user/user.crt",
              ssl_keyfile="/etc/kafka/certs/user/user.key")
```

### Go (sarama)

```go
cert, _ := tls.LoadX509KeyPair("/etc/kafka/certs/user/user.crt", "/etc/kafka/certs/user/user.key")
pool := x509.NewCertPool()
ca, _ := os.ReadFile("/etc/kafka/certs/ca/ca.crt")
pool.AppendCertsFromPEM(ca)
cfg.Net.TLS.Enable = true
cfg.Net.TLS.Config = &tls.Config{Certificates: []tls.Certificate{cert}, RootCAs: pool}
```

## 5. Test the connection

From the application pod, or a debug pod with the same secret mounts and the Kafka CLI:

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
kafka-topics.sh           --bootstrap-server $B --command-config  /tmp/client.properties --list
kafka-console-producer.sh --bootstrap-server $B --producer.config /tmp/client.properties --topic workflow
kafka-console-consumer.sh --bootstrap-server $B --consumer.config /tmp/client.properties --topic workflow --group workflow-consumer --from-beginning
```

How to read the result:

| Result | Meaning |
|---|---|
| Works | TLS and your ACLs are correct. |
| `TopicAuthorizationException` / `GroupAuthorizationException` | The connection and certificate are fine. You lack the permission. Ask the Dataverse team to add it. |
| SSL handshake error | Certificate or truststore problem. See the table below. |

## 6. Troubleshooting

| Symptom | Cause and fix |
|---|---|
| Bootstrap server cannot be resolved | Wrong address, or you are in another namespace or cluster. Use the exact bootstrap name from the Dataverse team. |
| `keystore password was incorrect` | Wrong password. The keystore password is `user.password` and the truststore password is `ca.password`. Do not mix them up, and do not paste a placeholder or trailing spaces. |
| `PKIX path building failed` | The truststore doesn't contain the cluster CA, or it is outdated after a CA rotation. Get the current CA secret. |
| `No subject alternative names matching` | The hostname you used isn't in the broker certificate. Use the exact bootstrap name you were given. |
| `bad_certificate` / `certificate_required` / handshake failure | The client certificate is missing, wrong, or expired. Check the keystore path and expiry (see below). |
| Authorization exceptions | Missing ACLs. Send the Dataverse team the topic, group and operations you need. |
| Timeouts with no error | Network policy or firewall blocks port 9093 from your pod. |

Check certificate expiry:
```bash
openssl x509 -in /etc/kafka/certs/user/user.crt -noout -enddate
```

## 7. Certificate renewal

- Your client certificate is renewed automatically by the cluster (valid for about a year). The secret in the Kafka cluster's namespace updates when it renews.
- Most clients read the certificate files only at startup. **Restart your pods after a renewal** (or use a tool such as Reloader to restart when the secret changes), or the app will fail once the old certificate expires.
- If the Dataverse team copied the secret into your namespace, they must refresh the copy after a renewal. Agree on who does this and when. Automatic syncing is best.
- The cluster CA is rotated rarely. The Dataverse team will notify you ahead of time. Refresh your truststore and restart when they do.

## 8. Security rules

- One Kafka user per application. Never share certificates between apps or environments.
- Do not log, commit, email or chat the private key or passwords.
- Mount secrets read-only. Pull passwords from Kubernetes secrets, not from config files in your image.
- Report a suspected key leak to the Dataverse team right away. They revoke it by deleting the user, and a new certificate is issued.

## 9. Requesting access or changes

Send the Dataverse team:
- application name, owning team and environment
- the Kafka user name you want
- topics with operations (read, write, describe) and consumer groups
- where the application runs (namespace and cluster)
- client language and library (Java, Python, Go, ...)
