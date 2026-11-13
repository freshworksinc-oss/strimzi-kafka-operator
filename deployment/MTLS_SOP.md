# SOP: mTLS for client connections (listener `clientd`, port 9093)

Cluster `kafkadev`, namespace `strimzi-kraft-test`. Strimzi 1.3.1, Kafka 4.3.2.

## 0. Key facts

- `tls: true` on a listener only gives **server-side TLS** (encryption). The client still authenticates with the listener's `authentication` type.
- **mTLS** needs `tls: true` **and** `authentication.type: tls` on the listener. A listener has exactly one authentication type, so `clientd` cannot do SCRAM and mTLS at the same time. Keep SCRAM on 9092/9094/9095, or add a separate listener for mTLS.
- The Strimzi **Cluster CA** signs the broker certificates. The **Clients CA** signs client certificates. Both are created automatically.
- Under mTLS the Kafka principal is `CN=<KafkaUser name>`. `superUsers` and ACLs must use that form.
- FIPS image: use **PKCS12** stores (the Strimzi default). Avoid JKS.

## 1. Enable mTLS on the listener

`deployment/kafka/kafka-cluster.yaml`, listener `clientd`:

```yaml
      - name: clientd
        port: 9093
        type: internal
        tls: true
        authentication:
          type: tls          # was scram-sha-512
```

Apply it (this triggers a rolling restart of controllers and brokers):

```bash
kubectl apply -f deployment/kafka/kafka-cluster.yaml
kubectl -n strimzi-kraft-test get pods -w
kubectl -n strimzi-kraft-test get kafka kafkadev -o jsonpath='{.status.listeners[?(@.name=="clientd")]}'
```

Bootstrap address (in-cluster): `kafkadev-kafka-bootstrap.strimzi-kraft-test.svc:9093`.

## 2. Create an mTLS user and its ACLs

```yaml
apiVersion: kafka.strimzi.io/v1
kind: KafkaUser
metadata:
  name: app-mtls
  namespace: strimzi-kraft-test
  labels:
    strimzi.io/cluster: kafkadev
spec:
  authentication:
    type: tls
  authorization:
    type: simple
    acls:
      - resource: { type: topic, name: my-topic, patternType: literal }
        operations: [Read, Write, Describe]
      - resource: { type: group, name: my-group, patternType: literal }
        operations: [Read]
```

```bash
kubectl apply -f app-mtls.yaml
kubectl -n strimzi-kraft-test get kafkauser app-mtls      # READY must be True
kubectl -n strimzi-kraft-test get secret app-mtls         # created by the User Operator
```

The secret `app-mtls` holds `ca.crt`, `user.crt`, `user.key`, `user.p12` and `user.password`.

Option: for a user certificate issued by your own CA, use `authentication.type: tls-external`. Then configure the listener with `authentication.type: tls` and a custom Clients CA (`clientsCa.generateCertificateAuthority: false` plus a Secret holding your CA).

## 3. Client configuration (in-cluster pod)

### 3.1 Mount the secrets

```yaml
volumes:
  - name: cluster-ca
    secret: { secretName: kafkadev-cluster-ca-cert }
  - name: user-cert
    secret: { secretName: app-mtls }
volumeMounts:
  - { name: cluster-ca, mountPath: /certs/ca,   readOnly: true }
  - { name: user-cert,  mountPath: /certs/user, readOnly: true }
```

Cluster CA secret keys: `ca.crt`, `ca.p12`, `ca.password`.

### 3.2 Client properties

Read the passwords from the files, or expose them as environment variables from the Secrets (`ca.password` and `user.password`). Do not hardcode them.

```properties
bootstrap.servers=kafkadev-kafka-bootstrap.strimzi-kraft-test.svc:9093
security.protocol=SSL

ssl.truststore.type=PKCS12
ssl.truststore.location=/certs/ca/ca.p12
ssl.truststore.password=<contents of /certs/ca/ca.password>

ssl.keystore.type=PKCS12
ssl.keystore.location=/certs/user/user.p12
ssl.keystore.password=<contents of /certs/user/user.password>

ssl.endpoint.identification.algorithm=https
```

Test from a client pod (any pod with Kafka CLI, for example a broker pod):

```bash
kubectl -n strimzi-kraft-test exec -it kafkadev-broker-0 -- bash -c '
  P=$(cat /certs/user/user.password)   # or use values from your mounted secrets
  /opt/kafka/bin/kafka-topics.sh --bootstrap-server kafkadev-kafka-bootstrap:9093 \
    --command-config /tmp/client-mtls.properties --list'
```

Quick test from your laptop without mounting anything:

```bash
NS=strimzi-kraft-test
kubectl -n $NS get secret kafkadev-cluster-ca-cert -o jsonpath='{.data.ca\.p12}'  | base64 -d > ca.p12
kubectl -n $NS get secret kafkadev-cluster-ca-cert -o jsonpath='{.data.ca\.password}' | base64 -d > ca.password
kubectl -n $NS get secret app-mtls -o jsonpath='{.data.user\.p12}' | base64 -d > user.p12
kubectl -n $NS get secret app-mtls -o jsonpath='{.data.user\.password}' | base64 -d > user.password
```

(`clientd` is an internal listener. From a laptop you need `kubectl port-forward` or an in-cluster client. Port-forward breaks hostname verification, so use in-cluster clients for real tests.)

### 3.3 Client in another namespace

Secrets are namespace-scoped, so copy them (re-copy after CA or certificate renewal):

```bash
kubectl -n strimzi-kraft-test get secret kafkadev-cluster-ca-cert -o yaml \
  | sed 's/namespace: strimzi-kraft-test/namespace: <app-ns>/' | kubectl apply -f -
kubectl -n strimzi-kraft-test get secret app-mtls -o yaml \
  | sed 's/namespace: strimzi-kraft-test/namespace: <app-ns>/' | kubectl apply -f -
```

Use the FQDN `kafkadev-kafka-bootstrap.strimzi-kraft-test.svc:9093`. Strimzi can also sync secrets with the `Kafka`/`KafkaUser` `template.secret` approach, or an external secret operator can do it.

## 4. Verification

```bash
# Handshake and certificate chain from inside the cluster
kubectl -n strimzi-kraft-test exec kafkadev-broker-0 -- \
  openssl s_client -connect kafkadev-kafka-bootstrap:9093 \
  -cert /certs/user/user.crt -key /certs/user/user.key -CAfile /certs/ca/ca.crt </dev/null | head -30
```

Expected results:

| Test | Result |
|---|---|
| Client with valid keystore and ACLs | works |
| Client without a keystore | SSL handshake failure (client auth required) |
| Client with a valid certificate but no ACL | `TopicAuthorizationException` or `GroupAuthorizationException` |

Broker-side check: `kubectl -n strimzi-kraft-test logs kafkadev-broker-0 | grep -i -E "ssl|handshake|authoriz"`.

## 5. Certificate renewal and rotation

- Client certificate validity is 365 days by default. The User Operator renews it automatically in the renewal window and updates the secret. Clients must reload the keystore (restart the pods or use a reloading library).
- Cluster CA and Clients CA are renewed by the Cluster Operator. During CA renewal the old and new CA are both trusted, and brokers roll. Make sure clients re-read `ca.p12` after renewal.
- Force a user certificate renewal: `kubectl -n strimzi-kraft-test annotate secret app-mtls strimzi.io/force-renew=true`.
- Check expiry: `kubectl -n strimzi-kraft-test get secret app-mtls -o jsonpath='{.data.user\.crt}' | base64 -d | openssl x509 -noout -enddate`.

## 6. Troubleshooting

| Symptom | Likely cause and fix |
|---|---|
| `SSLHandshakeException: PKIX path building failed` | Truststore missing the cluster CA, or stale after CA renewal. Re-mount `kafkadev-cluster-ca-cert`. |
| `No subject alternative names matching` | Client used a hostname not covered by the broker certificate. Use the `kafkadev-kafka-bootstrap` service FQDN or the broker pod DNS names. |
| `SSL handshake failed` / `bad_certificate` | Keystore missing, or signed by a CA other than the Clients CA. |
| `Authentication failed` with SCRAM config | The client still uses `SASL_SSL`/SCRAM against an mTLS listener. Use `security.protocol=SSL`. |
| `AuthorizationException` | Add ACLs on the KafkaUser. The principal is `User:CN=app-mtls`, not `app-mtls`. |
| KafkaUser not READY | Check `kubectl -n strimzi-kraft-test logs deploy/kafkadev-entity-operator -c user-operator`. |

## 7. Rollback

Set `clientd` authentication back to `scram-sha-512` and re-apply `kafka-cluster.yaml`. Existing SCRAM users (`admin`, and so on) are unaffected.
