## Prerequisites

- Kubernetes cluster. To quickly launch a Kubernetes cluster you can use [KIND](https://kind.sigs.k8s.io/docs/user/quick-start/#installation)
- [linkerd-cli(2.13.3)](https://github.com/linkerd/linkerd2/releases/tag/stable-2.13.3)
- [kubeseal](https://github.com/bitnami-labs/sealed-secrets/releases)
- [step-cli](https://smallstep.com/docs/step-cli/installation/)
- [argocd-cli](https://argo-cd.readthedocs.io/en/stable/cli_installation/)


## Install step-cli to create certificates
```
wget https://dl.smallstep.com/gh-release/cli/docs-cli-install/v0.24.4/step-cli_0.24.4_amd64.deb
sudo dpkg -i step-cli_0.24.4_amd64.deb
```
## Install kubeseal

https://github.com/bitnami-labs/sealed-secrets/releases

## Install Kind cluster
```
kind create cluster --name linkerd-demo
```

## Install Linkerd CLI
```
curl -sL https://run.linkerd.io/install | sh
```
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

## Configure project access and permissions
```
kubectl apply -f gitops/project.yaml
```
## Deploy the applications
```
kubectl apply -f gitops/main.yaml
```
```
argocd app sync main
```

## Deploy cert-manager
```
argocd app sync cert-manager
```
Confirm that cert-manager is running:
```
for deploy in "cert-manager" "cert-manager-cainjector" "cert-manager-webhook"; \
  do kubectl -n cert-manager rollout status deploy/${deploy}; \
done
```

## Deploy Sealed Secrets
```
argocd app sync sealed-secrets
```

Confirm that sealed-secrets is running:
```
kubectl -n kube-system rollout status deploy/sealed-secrets
```

## Create mTLS trust anchor
```
step certificate create root.linkerd.cluster.local sample-trust.crt sample-trust.key \
  --profile root-ca \
  --no-password \
  --not-after 43800h \
  --insecure
```
Confirm the details (encryption algorithm, expiry date, SAN etc.) of the new trust anchor:
```
step certificate inspect sample-trust.crt
```

Now create the SealedSecret resource to store the encrypted trust anchor:
```
kubectl create ns linkerd
kubectl -n linkerd create secret tls linkerd-trust-anchor \
  --cert sample-trust.crt \
  --key sample-trust.key \
  --dry-run=client -oyaml | \
kubeseal --controller-name=sealed-secrets -oyaml - | \
kubectl patch -f - \
  -p '{"spec": {"template": {"type":"kubernetes.io/tls", "metadata": {"labels": {"linkerd.io/control-plane-component":"identity", "linkerd.io/control-plane-ns":"linkerd"}, "annotations": {"linkerd.io/created-by":"linkerd/cli stable-2.12.0"}}}}}' \
  --dry-run=client \
  --type=merge \
  --local -oyaml > gitops/resources/linkerd/trust-anchor.yaml
```


Commit and push the new trust anchor secret to Github
```
git add gitops/resources/linkerd/trust-anchor.yaml
git commit -m "update encrypted trust anchor"
git push
```

## Deploy linkerd-bootstrap

Synchronize the linkerd-bootstrap application
```
argocd app sync linkerd-bootstrap
```
