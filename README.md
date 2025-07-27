# OpenZiti Kubernetes

## Requirements

* Docker
* Kubernetes (tested on Docker Desktop Kubernetes using Kubeadm)
* Helm

## Installation

Add OpenZiti and Jetstack Helm repositories

```
helm repo add openziti https://docs.openziti.io/helm-charts/
helm repo add jetstack https://charts.jetstack.io
helm repo update
```

```
helm install ingress-nginx ingress-nginx/ingress-nginx --namespace ingress-nginx --create-namespace
```

```
kubectl create namespace ziti
```

```
helm upgrade --install cert-manager jetstack/cert-manager \
    --namespace cert-manager --create-namespace \
    --set crds.enabled=true
helm upgrade --install trust-manager jetstack/trust-manager \
    --namespace cert-manager \
    --set crds.keep=false \
    --set app.trust.namespace=ziti
```

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
export ZITI_ADMIN_PASSWORD=fwMboaN7AhwiQf1x85plQUDmcRAt3Rum
```

```
ziti edge login -u admin -p $ZITI_ADMIN_PASSWORD
```

Enter controller host[:port] (default localhost:1280): ctrl1-127-0-0-1.nip.io:443
Untrusted certificate authority retrieved from server
Verified that server supplied certificates are trusted by server
Server supplied 4 certificates
Trust server provided certificate authority [Y/N]: y
Server certificate chain written to /Users/moscac/.ziti/certs/ctrl1-127-0-0-1.nip.io
Token: bc14d4c3-9902-4c41-bc8b-e338eeff1448
Saving identity 'default' to /Users/moscac/.ziti/ziti-cli.json

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
