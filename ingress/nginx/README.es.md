# NGINX Kubernetes Ingress Controller

[![en](https://img.shields.io/badge/lang-en-red.svg)](https://github.com/ogomezso/cfk-runbooks/blob/main/ingress/nginx/README.md)
[![es](https://img.shields.io/badge/lang-es-yellow.svg)](https://github.com/ogomezso/cfk-runbooks/blob/main/ingress/nginx/README.es.md)

Instalaremos la distribución [Kubernetes Ingress NGINX distribution](https://kubernetes.github.io/ingress-nginx/) a través de su Helm Chart:

Añadir el repo a Helm:

```bash
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update
```

Instalación del chart:

```bash
helm upgrade  --install ingress-nginx ingress-nginx/ingress-nginx \
  --set controller.extraArgs.enable-ssl-passthrough="true" \
  --namespace confluent
```

> NOTES:
> Nuestra `static host based solution` se base en la capacidad de Nginx para el `ingress ssl passthrough` por lo que setear este _feature_ a `true` como muestra el ejemplo de código es especialmente importante 
>
> Es necesario instalar esta distribución es específico en al menos versión 1.9.1.
