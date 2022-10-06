# Deploying infrastructure to the cluster

## Setting up kubectl, helm and kuploy locally

Install the tools kubectl and helm on your computer. I also use my own tool [kuploy](https://github.com/micw/kuploy)
which allows to have all deployment settings in git (including encrypted passwords).

```
pip install kuploy>=0.2
```

Now the kubeconfig file can be copied from the server (`/data/k3s/kubeconfig`) to ~/.kube/config.mail.wyraz.net

You need to change `server: https://127.0.0.1:6443` to the actual IP or name of the server. Also change the name of the context from `default` to a meaningful name (`mail.wyraz.net` in my case). Do the same change to the `current-context`.

Test the connection:

```
kubectl --kubeconfig ~/.kube/config.mail.wyraz.net cluster-info
```

## Initialize the master password

Kuploy uses an AES key stored as kubernetes secret to encrypt passwords in yamls. This way, the yamls can safely be commited to git. To access the passwords, access to the kubernetes cluster is required.

```
kuploy secrets --context mail.wyraz.net --init-master-key
```

Backup the generated master key on a secure place.

## Deploy infrastructure

Infrastructure deployments are in [kuploy/infrastructure.yaml](kuploy/infrastructure.yaml).

### prometheus-crds

This installs a kube-prometheus-stack where _everyhting_ is disabled. The chart will the only install prometheus CRDs, so that ServiceMonitor objects can be used.

```
kuploy infrastructure.yaml prometheus-crds
```

### victoria-metrics-k8s-stack

This is a prometheus compatible monitoring stack to store cluster metrics.

```
kuploy infrastructure.yaml victoria-metrics-k8s-stack
```

### nginx

Nginx is the ingress controller used for http and https

```
kuploy infrastructure.yaml nginx-ingress
# Test with
curl mail.wyraz.net
# this should return "default backend - 404"
```

### cert-manager

Cert-Manager is the tool used to create and renew Let's Encrypt certificates. Another small helper chart is deployed which installs a
default certificate issuer fpor Let's Encrypt.

```
kuploy infrastructure.yaml cert-manager cert-manager-issuer
```


### kubernetes dashboard

The dashboard will be available under https://mail.wyraz.net/kubernetes/ and will allow to manage the kubernetes cluster via web.

An admin accont needs to be created manually to get a token for dashboard access.

```
kuploy infrastructure.yaml kubernetes dashboard
kubectl --kubeconfig ~/.kube/config.mail.wyraz.net apply -f ../resources/admin-account.yaml
kubectl --kubeconfig ~/.kube/config.mail.wyraz.net --namespace infrastructure create token admin-user
```

You can use the token to log-in into the dashboard.
