# Certificates

[![en](https://img.shields.io/badge/lang-en-red.svg)](https://github.com/ogomezso/cfk-runbooks/blob/main/usecases/IntAutoTLS-ExtTLS-RBAC-Ingress/0.certs/README.md)
[![es](https://img.shields.io/badge/lang-es-yellow.svg)](https://github.com/ogomezso/cfk-runbooks/blob/main/ussescases/IntAutoTLS-ExtTLS-RBAC-Ingress/0.certs/README.es.md)

This folders provide an opinionated way to create self-signed certificates that can be used for the exercises that require them.

## Requirements

- Openssl
- [cfssl](https://cfssl.org/) (CloudFlare's SSL toolkit)

## Prepare INTERNAL Certificates

### Self-signing Internal CA Cert and Key

Pre-create a Authorities for the K8s Cluster (Internal).

```bash
openssl genrsa -out generated/InternalCAkey.pem 2048
```

```bash
openssl req -x509 -new -nodes \
  -key generated/InternalCAkey.pem \
  -days 365 \
  -out generated/InternalCAcert.pem \
  -subj "/C=ES/ST=VLC/L=VLC/O=Demo/OU=GCP/CN=InternalCA"
```

### Internal (to K8s) certificates

We will rely on [CFK Certificate autogeneration](https://docs.confluent.io/operator/current/co-network-encryption.html#auto-generated-tls-certificates). To make this work we need to provide the signing CA Cert and Key is a secret for CFK,as we show on  [Secrets](https://github.com/ogomezso/cfk-runbooks/blob/main/usecases/IntAutoTLS-ExtTLS-RBAC-Ingress/secrets/README.md) section.

## Prepare EXTERNAL Certificates

These certificates are used for external K8s client access.

### Self-signing External CA Cert and Key

Generate one CA certificate and key to sign "external" to K8s cluster certificates.

```bash
$ openssl genrsa -out generated/ExternalCAkey.pem 2048

$ openssl req -x509 -new -nodes \
  -key generated/ExternalCAkey.pem \
  -days 365 \
  -out generated/ExternalCAcert.pem \
  -subj "/C=ES/ST=VLC/L=VLC/O=Demo/OU=GCP/CN=ExternalCA"
```

### Create CFSSL Profiles

A simple file with the usages for a certificate profile `cert-req-conf.json`

```json
{
    "signing": {
        "default": {
            "expiry": "1440h"
        },
        "profiles": {
            "server": {
                "expiry": "1440h",
                "usages": [
                    "signing",
                    "key encipherment",
                    "server auth",
                    "client auth"
                ]
            },
            "client": {
                "expiry": "1440h",
                "usages": [
                    "signing",
                    "key encipherment",
                    "client auth"
                ]
            }
        }
    }
}
```

### Create Server Certificate Request

We gonna create a specfic certificate for `Kafka` and another for the rest of the components

CN and [SANs](https://docs.confluent.io/operator/current/co-network-encryption.html#define-san)s definition for the certs.

On this example we gonna use  [nip.io](https://nip.io/) for DNS resolution.

For configuring properly these SANS we will need to know the `ingress-controller` public IP:

```bash
# fetch the IP of the Ingress controller...
INGRESS_IP=$(kubectl get svc ingress-nginx-controller -o jsonpath='{.status.loadBalancer.ingress[*].ip}')
```

Example `external-kafka-cert-req.json`

```json
{
  "CN": "external.kafka.confluent",
  "hosts": [
    "bootstrap.$INGRESS_IP.nip.io",
    "broker-0.$INGRESS_IP.nip.io",
    "broker-1.$INGRESS_IP.nip.io",
    "broker-2.$INGRESS_IP.nip.io",
    "*.$INGRESS_IP.nip.io"
  ],
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "Universe",
      "ST": "Pangea",
      "L": "Earth"
    }
  ]
}
```

As we will see on the  [Broker]() section its important to map not only the bootstrap but each one of the broker.

Example `external-services-cert-req.json`

```json
{
  "CN": "external.services.confluent",
  "hosts": [
    "schemaregistry.$INGRESS_IP.nip.io",
    "*.schemaregistry.$INGRESS_IP.nip.io",
    "connect.$INGRESS_IP.nip.io",
    "*.connect.$INGRESS_IP.nip.io",
    "ksqldb.$INGRESS_IP.nip.io",
    "*.ksqldb.$INGRESS_IP.nip.io",
    "controlcenter.$INGRESS_IP.nip.io"
  ],
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "Universe",
      "ST": "Pangea",
      "L": "Earth"
    }
  ]
}
```

>Notes:
>The SAN configuraded on certificates must match with the Ingress and external access configuration on the components
>
>Some Ingress Controller (such NGINX) can only work with explicit SAN but not with wildcards

### Generate the EXTERNAL Certificate and Key for Kafka and Services

```bash
$ cfssl gencert -ca=generated/ExternalCAcert.pem \
    -ca-key=generated/ExternalCAkey.pem \
    -config=cert-req-conf.json \
    -profile=server external-kafka-cert-req.json \
    | cfssljson -bare generated/externalKafka

$ cfssl gencert -ca=generated/ExternalCAcert.pem \
    -ca-key=generated/ExternalCAkey.pem \
    -config=cert-req-conf.json \
    -profile=server external-services-cert-req.json \
    | cfssljson -bare generated/externalServices
```
