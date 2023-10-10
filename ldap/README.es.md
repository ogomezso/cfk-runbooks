# Despliegue de OpenLDAP

[![en](https://img.shields.io/badge/lang-en-red.svg)](https://github.com/ogomezso/cfk-runbooks/blob/main/ldap/README.md)
[![es](https://img.shields.io/badge/lang-es-yellow.svg)](https://github.com/ogomezso/cfk-runbooks/blob/main/ldap/README.es.md)

## Creación de un Namespace para el servicio

```bash
kubectl create namespace ldap
```

## Despliegue del `helm chart``

Este chart esta configurado usando el fichero `values.yaml`

```bash
helm upgrade --install \
  open-ldap openldap \
  -f openldap/ldaps-rbac.yaml \
  --namespace ldap
```

## Test del sercio

Usaremos `ldapsearch` desde dentro del contenedor de OpenLdap

```bash
kubectl exec -it ldap-0 -n ldap -- bash

ldapsearch -LLL -x -H ldap://ldap.ldap.svc.cluster.local:389 -b 'dc=confluent,dc=acme,dc=com' -D "cn=mds,dc=confluent,dc=acme,dc=com" -w 'Developer!'

```

## Cambio de contenido en LDAP

Esta distribución no incluye ninguna interfaz gráfica de administración (i.e. [phpLDAPAdmin](http://phpldapadmin.sourceforge.net/wiki/index.php/Main_Page) ) la administración de usuarios se realiza mediante la modificación del fichero [values.yaml](openldap/values.yaml) permitiendo la creación/modificación de usuarios/grupos así como del baseDN.

Para la correcta aplicación de los cambios necesitaremos deinstalar el chart (como se se describe en la siguiente sección) y volver a instalarlo

## Desinstanlación del servicio

```bash
helm uninstall open-ldap -n ldap
kubectl delete pvc ldap-data-ldap-0 -n ldap
kubectl delete pvc ldap-config-ldap-0 -n ldap
```
