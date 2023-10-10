# NGINX Kubernetes Ingress Controller

[![en](https://img.shields.io/badge/lang-en-red.svg)](https://github.com/ogomezso/cfk-runbooks/blob/main/ingress/nginx/README.md)
[![es](https://img.shields.io/badge/lang-es-yellow.svg)](https://github.com/ogomezso/cfk-runbooks/blob/main/ingress/nginx/README.es.md)

We will rely on the [Kubernetes Ingress NGINX distribution](https://kubernetes.github.io/ingress-nginx/) that we gonna instal by their public Helm Chart:

Add the repo to helm:

```bash
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update
```

Install the chart:

```bash
helm upgrade  --install ingress-nginx ingress-nginx/ingress-nginx \
  --set controller.extraArgs.enable-ssl-passthrough="true" \
  --namespace confluent
```

> NOTES:
> Our `static host based solution` rely on the `ingress ssl passthrough` capabilities of nginx so set this feature to `true` as the code example shows is specially imporant
>
> Install this specific chart distribution on at least 1.9.1 version