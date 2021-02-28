---
title: "How to enable Brotli compression on Ingress-Nginx"
date: 2021-02-28T17:05+0100
categories:
  - tech
tags:
  - Kubernetes
  - Container
  - Rancher
  - RKE
  - Ingress
---

Based on [this](https://geko.cloud/how-to-enable-brotli-compression-on-ingress-nginx/) awesome blog post:

> Brotli is a compression method developed by Google and released in 2015. Depending on the scenario, brotli is capable of achieving a compression rate improvement of between 20 and 30% over gzip, which is the ingress-nginx default compression method.

In this case, we'd like to enable brotli compression on a RKE1 Cluster for our Rancher managament cluster. But it should be more or less the same on other Ingress-Nginx.

First let's check if Brotli is already enabled or not:
```shell
$ curl -s -I -H 'Accept-Encoding: br' https://rancher.awesomecluster.com | grep content-encoding

$ curl -s -I -H 'Accept-Encoding: gzip' https://rancher.awesomecluster.com | grep content-encoding
content-encoding: gzip
```

Ok, we have gzip but no Brotli. Now, we prepare a yaml for patching. I name it `brotli.yaml`:
```yaml
data:
  brotli-level: "6"
  brotli-types: text/xml image/svg+xml application/x-font-ttf image/vnd.microsoft.icon
    application/x-font-opentype application/json font/eot application/vnd.ms-fontobject
    application/javascript font/otf application/xml application/xhtml+xml text/javascript  application/x-javascript
    text/plain application/x-font-truetype application/xml+rss image/x-icon font/opentype
    text/css image/x-win-bitmap
  enable-brotli: "true"
```

Now lets find the correct config map:
```shell
$ kubectl get cm -l app=ingress-nginx -A
NAMESPACE       NAME                  DATA   AGE
ingress-nginx   nginx-configuration   0      327d
```

It's currently empty ("0" Data = Ingress-Nginx is running in standard configuration). Let's patch:
```shell
$ kubectl patch -n ingress-nginx cm nginx-configuration --patch "$(cat brotli.yaml)"
configmap/nginx-configuration patched
```

Checking the result (output truncated):
```shell
$ kubectl get  -n ingress-nginx configmaps nginx-configuration -o yaml
apiVersion: v1
data:
  brotli-level: "6"
  brotli-types: text/xml image/svg+xml application/x-font-ttf image/vnd.microsoft.icon
    application/x-font-opentype application/json font/eot application/vnd.ms-fontobject
    application/javascript font/otf application/xml application/xhtml+xml text/javascript  application/x-javascript
    text/plain application/x-font-truetype application/xml+rss image/x-icon font/opentype
    text/css image/x-win-bitmap
  enable-brotli: "true"
kind: ConfigMap
metadata:
[...]
```

Wenn we change a config map, the pods won't notice it. So we need to delet/recreate them. If you do not want a short outage, delete them one after another and wait between the deletions until the newly created pod is up and running. In this case, we just delete them all:
```shell
$ for i in $(kubectl get pods -n ingress-nginx | awk '{ print $1}' | grep ingress); do kubectl delete pod -n ingress-nginx $i; done
pod "nginx-ingress-controller-gm649" deleted
pod "nginx-ingress-controller-n8pv8" deleted
pod "nginx-ingress-controller-pg8gc" deleted
```

We now have fancy new pods:
```shell
$ kubectl get pods -n ingress-nginx
NAME                                    READY   STATUS    RESTARTS   AGE
default-http-backend-67cf578fc4-h9ggr   1/1     Running   0          21h
nginx-ingress-controller-8r4lv          1/1     Running   0          5m11s
nginx-ingress-controller-nqwwp          1/1     Running   0          3m7s
nginx-ingress-controller-phvfz          1/1     Running   0          4m9s
```

And the final test:
```shell
$ curl -s -I -H 'Accept-Encoding: br' https://rancher.awesomecluster.com | grep content-encoding
content-encoding: br
```