# Certificates

[![en](https://img.shields.io/badge/lang-en-red.svg)](https://github.com/ogomezso/cfk-runbooks/blob/main/usecases/IntAutoTLS-ExtTLS-RBAC-Ingress/0.certs/README.md)
[![es](https://img.shields.io/badge/lang-es-yellow.svg)](https://github.com/ogomezso/cfk-runbooks/blob/main/ussescases/IntAutoTLS-ExtTLS-RBAC-Ingress/0.certs/README.es.md)

Está sección describe _nuestra forma_ de generar los certificados requeridos para este caso de uso  

## Requirements

- Openssl
- [cfssl](https://cfssl.org/) (CloudFlare's SSL toolkit)

## Preparación de los certificados INTERNOS

### CA Cert and Key Internos autofirmados

Pre-creación de la Autoridad certificadora para el Cluster K8s (interna)

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

### Certificados Internos (para K8s)

Estos certificados seran creados automaticamente por el operador usando:  [CFK Certificate autogeneration](https://docs.confluent.io/operator/current/co-network-encryption.html#auto-generated-tls-certificates). 

Para hacerlo funcionar deberemos crear el secreto apropiado tal como se muestra en la sección ([Secrets](https://github.com/ogomezso/cfk-runbooks/blob/main/usecases/IntAutoTLS-ExtTLS-RBAC-Ingress/secrets/README.es.md))

It is advisable to create a Secret RD to manage this secret

```bash
kubectl create secret tls ca-pair-sslcerts \
--cert=generated/InternalCAcert.pem \
--key=generated/InternalCAkey.pem \
-n confluent --dry-run=client -output yaml > ca-pair-sslcerts.yaml

kubectl apply -f ca-pair-sslcerts.yaml
```

## Preparación de los Certificados EXTERNOS

Estos certificados se usarán para la conexión de clientes externos al cluster K8s.

### CA Cert y Key externas, autofirmadas

Generacion de un CA Certificate para los clientes "externos" al cluster K8s.

```bash
openssl genrsa -out generated/ExternalCAkey.pem 2048
```

```bash
$ openssl req -x509 -new -nodes \
  -key generated/ExternalCAkey.pem \
  -days 365 \
  -out generated/ExternalCAcert.pem \
  -subj "/C=ES/ST=VLC/L=VLC/O=Demo/OU=GCP/CN=ExternalCA"
```

### Creación de los perfiles CFSSL

Fichero simplificado que contiene la configuración para CFSSL: `cert-req-conf.json`

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

### Creación de los Server Certificate Request

Crearemos un certificado para `Kafka` y otro para el resto de componentes

Definición del CN and [SANs](https://docs.confluent.io/operator/current/co-network-encryption.html#define-san)s para los certificados.

Para este ejemplo usaremos [nip.io](https://nip.io/) para la resolución DNS.

Para ellos necesitarás conocer la ip external del servicio del `ingress-controller`:

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

Como veremos en la seccion [Broker]() es importante mapear no solo el bootstrap sino cada uno de los broker.

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
>Los SAN configurados deben coincidir con la configuración del Ingress y external access de los distintos componentes
>
>Algunas soluciones de Ingress Controller (como NGINX) solo funcionan con SAN específicas y no wildcards


### Generación del certificado y key EXTERNO para Kafka y demás Servicios

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

### Generate Secrets to hold the certificates

#### Secret for Kafka external access

```bash
kubectl create secret generic kafka-external-tls \
  --from-file=fullchain.pem=generated/externalKafka.pem \
  --from-file=privkey.pem=generated/externalKafka-key.pem \
  --from-file=cacerts.pem=generated/ExternalCAcert.pem \
  --namespace confluent
```

Note the time of secret difference since this is going to be used by NGINX Ingress Controller

```bash
kubectl create secret tls services-external-tls \
  --cert=generated/externalServices.pem \
  --key=generated/externalServices-key.pem \
  --namespace confluent
```
