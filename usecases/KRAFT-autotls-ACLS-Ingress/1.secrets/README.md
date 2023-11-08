# Secrets

[![en](https://img.shields.io/badge/lang-en-red.svg)](https://github.com/ogomezso/cfk-runbooks/blob/main/usecases/ZK-AutoTLS-RBAC-Ingress/1.secrets/README.md)
[![es](https://img.shields.io/badge/lang-es-yellow.svg)](https://github.com/ogomezso/cfk-runbooks/blob/main/usecases/ZK-AutoTLS-RBAC-Ingress/1.secrets/README.es.md)

On this section we gonna go through the different K8s secrets we will need to put in place for our use case deployment

## Way of working

- Use local files provided on `1.secrets` as base to generate the K8s secrets
- Base on that we are going to dynamically create the yaml resource file for the secret using `kubectl` commands:

```bash
kubectl create secret <specific secret config> --dry-run:client -o yaml > generated/<client-secret-name>.yaml
```

these files are going to be generated under the `generated` folder

- On file based secret creation is important to take care of the destination file name since operator can only search for specific ones. This name is the one your provide on the left side of `from-file=` field of the kubectl command

## TLS secrets

We will need two secrets for internal and external tls configuration:

### Internal

As we already saw on the `certificates` section autogenerated secret is managed by `Confluent Operator` itself.

So we will need to create a `tls` kind of secret with the exact name of `ca-pair-sslcerts`:

Generate the resource file:

```bash
kubectl create secret tls ca-pair-sslcerts --cert=../0.certs/generated/CAcert.pem --key=../0.certs/generated/CAkey.pem --dry-run=client -o yaml > generated/ca-pair-sslcerts.yaml
```

and finally apply it:

```bash
kubectl apply -f generated/ca-pair-sslcerts.yaml
```

### External

For external access we would need two kinds of secret depending on the Ingress Controller ssl passthrough capabilities:

One which will be use for the operator to create the appropriate certificate stores on component side:

```bash
kubectl create secret generic kafka-external-tls \
  --from-file=fullchain.pem=../0.certs/generated/externalKafka.pem \
  --from-file=privkey.pem=../0.certs/generated/externalKafka-key.pem \
  --from-file=cacerts.pem=../0.certs/generated/CAcert.pem \
  --namespace confluent --dry-run=client -o yaml > generated/kafka-external-tls.yaml
```

and another for Ingress configuration in case of need:

```bash
kubectl create secret tls services-external-tls \
  --cert=../0.certs/generated/externalServices.pem \
  --key=../0.certs/generated/externalServices-key.pem \
  --namespace confluent --dry-run=client -o yaml > generated/services-external-tls.yaml
```

## KRaft Controller

On this use case example `KRaft Controller` service is implemented with `autogenerated certs` for connections (note that Kafka Brokers are the only client that controller should have) and `plain` for authentication type.

For tls secret we just need to enable the `tls autogenerated` configuration and operator will use the already created `ca-pair-sslcerts`

For digest user we need to use a json file (provided as `controller-creds.json` on this folder) as source for user information:

```json
{
    "zk": "zk-secret"
}
```

and generate the resource file from it:

```bash
kubectl create secret generic controller-creds \
  --from-file=plain-users.json=controller-creds.json \
  --namespace confluent --dry-run=client -o yaml > generated/controller-creds.yaml
```

## Broker

On this example `Kafka Brokers` will use a variety of secrets depending on the use:

### Internal Communication

For **internal tls**  secret we just need to enable the `tls autogenerated` configuration and operator will use the already created `ca-pair-sslcerts`

On the **Internal Listener** (K8s internal clients) we gonna use a plain user that the broker gonna use to authenticate on the configuration. In this case we will use a different simple `JaaS config` file based on a plain text file `kafka-internal-creds.txt`:

```txt
username=kafka
password=kafka-secret
```

We gonna create a secret call `kafka-creds` to hold this secret

```bash
kubectl create secret generic kafka-creds \
 --from-file=plain-users.json=plain-users.json \
 --namespace confluent \
 --dry-run=client \
 --output yaml > generated/kafka-creds.yaml
```

### External Communication

As we gonna use **provided certificates** for external access we need to create the `kafka-external-tls` secret containing the `externalKafka` certificate create on the `certificates` section:

```bash
kubectl create secret generic kafka-external-tls \
  --from-file=fullchain.pem=../0.certs/generated/externalKafka.pem \
  --from-file=privkey.pem=../0.certs/generated/externalKafka-key.pem \
  --from-file=cacerts.pem=../0.certs/generated/CAcert.pem \
  --namespace confluent \
  --dry-run=client \
  -o yaml > generated/kafka-external-tls.yaml
```

we need to provide a  list of plain users that can access to our broker

from the plain-users.json file

```bash
kubectl create secret generic kafka-external-creds \
  --from-file=plain-users.json=plain-users.json \
  --namespace confluent --dry-run=client -o yaml > generated/kafka-external-creds.yaml
```

### Dependencies

As dependency each broker has:

#### Kraft Controller

we are going to use the already created secret `kafka-cred` credential for KRAFT controller

#### Kafka Rest Class

## Basic Credentials

We gonna create basic credentials for CP components (SR, Connect, C3) to authenticate to Kafka Brokers

```bash
kubectl create secret generic basic-creds \
  --from-file=plain.txt=plain.txt \
  --namespace confluent --dry-run=client -o yaml > generated/basic-creds.yaml
```

### Confluent Components external access

```bash
kubectl create secret tls services-external-tls \
  --cert=../0.certs/generated/externalServices.pem \
  --key=../0.certs/generated/externalServices-key.pem \
  --namespace confluent --dry-run=client -o yaml > generated/services-external-tls.yaml
```