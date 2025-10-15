## Introduction

This is not part of the original [blog](https://developers.redhat.com/blog/2025/10/06/using-argo-cd-agent-openshift-gitops) but I was curious about how to use cert-manager instead of argocd-agentctl to manage the PKI requirements
for the Argo Agent.

Note instructions here have not been well tested, more POC level at this point.

## Prerequisites

* openssl is installed on laptop
* cert-manager has been installed on your cluster

## Installing the Principal

### Provision the operator and namespaces from the blog:

Operator:

```
oc apply -k operator/base
```

Namespaces:

```
oc apply -k control-plane/namespaces/base
```

### Create CA

Create private key with openssl

```
openssl genrsa -out ca.key 4096
```

Create root certificate using the generated key:

```
openssl req -new -x509 -sha256 -days 3650 -key ca.key -out ca.crt
```

Create a CA secret in Kubernetes:

```
kubectl create secret tls argocd-agent-ca --cert=ca.crt --key=ca.key -n argocd
```

### Create cert-manager Issuer for CA

Create cert-manager issuer for the CA we generated previously:

```
apiVersion: cert-manager.io/v1
kind: Issuer
metadata:
  name: argocd-agent-ca
  namespace: argocd
spec:
  ca:
    secretName: argocd-agent-ca
```

### Generate TLS certs for principal

Use cert-manager to generate the tls certificates for the Principal starting with `argocd-agent-principal-tls`. Make
sure you update the `dnsNames` to reflect the URL used to expose the Agent on your system:

```
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: argocd-agent-principal-tls
  namespace: argocd
spec:
  secretName: argocd-agent-principal-tls
  issuerRef:
    name: argocd-agent-ca
    kind: Issuer
  dnsNames:
  - argocd-agent-principal-argocd.apps.cluster-csxcn.dynamic.redhatworkshops.io
```

Next generate the certificate for the resource proxy, note the `dnsNames` does not
need to be changed here since it is an internal service unless you are using a different
namespace then `argocd`:

```
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: argocd-agent-resource-proxy-tls
  namespace: argocd
spec:
  secretName: argocd-agent-resource-proxy-tls
  issuerRef:
    name: argocd-agent-ca
    kind: Issuer
  dnsNames:
  - argocd-agent-resource-proxy.argocd.svc.cluster.local
```

Confirm the certificates were generated, `READY` should be `True` for both certs as
per this example:

```
$ oc get certificate -n argocd
NAME                              READY   SECRET                            AGE
argocd-agent-principal-tls        True    argocd-agent-principal-tls        4m8s
argocd-agent-resource-proxy-tls   True    argocd-agent-resource-proxy-tls   64s
```

###
