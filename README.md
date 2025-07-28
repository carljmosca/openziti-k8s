# OpenZiti Kubernetes

## Requirements

* Docker
* Kubernetes (tested on Docker Desktop Kubernetes using Kubeadm)
* Helm
* Ingress-Nginx

## Installation

Add OpenZiti and Jetstack Helm repositories

```
helm repo add openziti https://docs.openziti.io/helm-charts/
helm repo add jetstack https://charts.jetstack.io
helm repo update
```

Install Ingress-Nginx (skip if already installed)

```
helm install ingress-nginx ingress-nginx/ingress-nginx --namespace ingress-nginx --create-namespace
```

Create the ziti namespace

```
kubectl create namespace ziti
```

Install Jetstack cert-manager and trust-manager

```
helm upgrade --install cert-manager jetstack/cert-manager \
    --namespace cert-manager --create-namespace \
    --set crds.enabled=true
helm upgrade --install trust-manager jetstack/trust-manager \
    --namespace cert-manager \
    --set crds.keep=false \
    --set app.trust.namespace=ziti
```

Patch CoreDNS to include entries for ctrl1.locahost and router1.localhost

View current configmap

```
kubectl get configmap coredns -n kube-system -o yaml
```

Get address of hot

```
export HOST_IP=$(ifconfig | grep "inet " | grep -v 127.0.0.1 | head -1 | awk '{print $2}')
```

Patch CoreDNS configmap

```
kubectl patch configmap coredns -n kube-system --type merge -p="
{
  \"data\": {
    \"Corefile\": \".:53 {\n    errors\n    health {\n       lameduck 5s\n    }\n    ready\n    hosts {\n        ${HOST_IP} ctrl1.localhost\n        ${HOST_IP} router1.localhost\n        fallthrough\n    }\n    kubernetes cluster.local in-addr.arpa ip6.arpa {\n       pods insecure\n       fallthrough in-addr.arpa ip6.arpa\n       ttl 30\n    }\n    prometheus :9153\n    forward . /etc/resolv.conf {\n       max_concurrent 1000\n    }\n    cache 30 {\n       disable success cluster.local\n       disable denial cluster.local\n    }\n    loop\n    reload\n    loadbalance\n}\"
  }
}"
```


```
kubectl rollout restart deployment coredns -n kube-system
```

```
kubectl get pods -n kube-system -l k8s-app=kube-dns
```

```
kubectl run dns-test --image=busybox --rm -it --restart=Never -- nslookup ctrl1.localhost
```

Install OpenZiti controller

```
helm pull openziti/ziti-controller
tar -xvf ziti-controller-*.tgz
rm ziti-controller-*.tgz
CM_NAMESPACE=cert-manager \
CM_RELEASE_NAME=cert-manager \
TM_NAMESPACE=cert-manager \
TM_RELEASE_NAME=trust-manager \
ZITI_NAMESPACE=ziti \
chmod +x ./ziti-controller/files/chown-cert-manager.bash \
./ziti-controller/files/chown-cert-manager.bash
```

```
kubectl patch deployment "ingress-nginx-controller" \
    --namespace ingress-nginx \
    --type json \
    --patch '[{"op": "add",
        "path": "/spec/template/spec/containers/0/args/-",
        "value":"--enable-ssl-passthrough"
    }]'
```

```
helm upgrade ziti-controller openziti/ziti-controller \
    --install \
    --namespace ziti \
    --values ziti-controller-values.yaml
```

```
export ZITI_ADMIN_PASSWORD=mjLcFk9pEVvGxYiiBhW04jW0pOEqUfvS
```

```
ziti edge login -u admin -p $ZITI_ADMIN_PASSWORD
```

Enter controller host[:port] (default localhost:1280): ctrl1.localhost


```
ziti edge create edge-router "router1" \
  --tunneler-enabled --jwt-output-file router1.jwt
```

```
helm upgrade --install \
  "ziti-router-1" \
  openziti/ziti-router \
    --set-file enrollmentJwt=router1.jwt \
    --values router-values.yaml
```
