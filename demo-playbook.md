## Install Kind cluster
kind create cluster --name linkerd-demo

## Install Linkerd CLI
curl -sL https://run.linkerd.io/install | sh

Add the linkerd CLI to your path with:

  export PATH=$PATH:/home/infracloud/.linkerd2/bin

## Pre-Check Linkerd setup
```
linkerd check --pre
```

## Install Argo CLI
curl -sSL -o argocd-linux-amd64 https://github.com/argoproj/argo-cd/releases/latest/download/argocd-linux-amd64
sudo install -m 555 argocd-linux-amd64 /usr/local/bin/argocd
rm argocd-linux-amd64

## Deploy Argo
```
kubectl create ns argocd

kubectl -n argocd apply -f \
  https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```
## Access Argo UI
```
kubectl -n argocd port-forward svc/argocd-server 8080:443  \
  > /dev/null 2>&1 &
```
The Argo CD dashboard is now accessible at https://localhost:8080

password=`kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d`

## Authenticate the Argo CD CLI
```
argocd login 127.0.0.1:8080 \
  --username=admin \
  --password="${password}" \
  --insecure
```
