# Using Kuma with cert-manager

## Prerequisites

1. You will need a working Kubernetes cluster with a CNI plugin installed. These instructions were tested with Kubernetes 1.22.2 and the Calico CNI plugin.
2. You will need the `kumactl` command-line utility installed on your system. Kuma 1.3.1 was used for these instructions (the latest release at the time this was written). Refer to [the Kuma docs](https://kuma.io/docs/1.3.1/) for installing `kumactl`.

## Install cert-manager

Follow the instructions [here](https://cert-manager.io/docs/installation/) to install cert-manager. These instructions were tested with cert-manager 1.5.4 (the latest release at the time this was written).

## Configure cert-manager

First, create the "kuma-system" namespace:

```yaml
---
apiVersion: v1
kind: Namespace
metadata:
  name: kuma-system
```

To get started creating the necessary TLS assets, start by generating a SelfSigned ClusterIssuer in cert-manager:

```yaml
---
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: selfsigned-issuer
spec:
  selfSigned: {}
```

Next, use the ClusterIssuer you just created to issue a CA certificate:

```yaml
---
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: kuma-root-ca
  namespace: kuma-system
spec:
  isCA: true
  commonName: kuma-root-ca
  secretName: kuma-root-ca-secret
  duration: 43800h # 5 years
  renewBefore: 720h # 30d
  privateKey:
    algorithm: RSA
    encoding: PKCS1
    size: 2048
  usages:
    - digital signature
    - key encipherment
    - cert sign
  issuerRef:
    name: selfsigned-issuer
    kind: ClusterIssuer
    group: cert-manager.io
```

Extract the CA certificate generated as you'll need it later (users not on macOS may need to use `base64 -d` instead of the uppercase `-D` shown below):

    kubectl -n kuma-system get secret kuma-root-ca-secret \
    -o jsonpath='{.data.tls\.crt}' | base64 -D > ca.crt
    kubectl -n kuma-system get secret kuma-root-ca-secret \
    -o jsonpath='{.data.tls\.crt}' > ca-bundle

Now specify the CA certificate as an Issuer in the "kuma-system" namespace:

```yaml
---
apiVersion: cert-manager.io/v1
kind: Issuer
metadata:
  name: kuma-ca-issuer
  namespace: kuma-system
spec:
  ca:
    secretName: kuma-root-ca-secret
```

At this point, you are now ready to issue the TLS certificate that will be consumed by Kuma for general TLS communications:

```yaml
---
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: raw-kuma-tls-general
  namespace: kuma-system
spec:
  commonName: kuma-tls-general-cert
  secretName: raw-kuma-tls-general
  duration: 8760h # 1 year
  renewBefore: 360h # 15d
  subject:
    organizations:
      - kuma
  isCA: false
  privateKey:
    algorithm: RSA
    encoding: PKCS1
    size: 2048
  usages:
    - digital signature
    - key encipherment
    - server auth
    - client auth
  dnsNames:
    - kuma-control-plane.kuma-system
    - kuma-control-plane.kuma-system.svc
  issuerRef:
    name: kuma-ca-issuer
    kind: Issuer
    group: cert-manager.io
```

Extract the certificate and key generated by cert-manager for use with `kumactl`:

    kubectl -n kuma-system get secret raw-kuma-tls-general \
    -o jsonpath='{.data.tls\.crt}' | base64 -D > tls.crt
    kubectl -n kuma-system get secret raw-kuma-tls-general \
    -o jsonpath='{.data.tls\.key}' | base64 -D > tls.key

Create a Kubernetes Secret containing the TLS certificate and key as well as the CA certificate:

    kubectl -n kuma-system create secret generic general-tls-certs \
    --from-file=tls.crt=/path/to/tls.crt \
    --from-file=tls.key=/path/to/tls.key \
    --from-file=ca.crt=/path/to/ca.crt

## Install Kuma

The final step is to generate the installation YAML via `kumactl` with this command:

    kumactl install control-plane --tls-general-secret=general-tls-certs \
    --tls-general-ca-bundle=$(cat /path/to/ca-bundle)

Save the output of this command to a file, or pipe it directly to `kubectl` to apply it to the intended Kubernetes cluster. If you do save it to a file, note that it cannot be re-used for other Kubernetes clusters as it still contains cryptographic material (a Base64-encoded copy of the CA certificate).

That's it---you've now installed Kuma with TLS assets generated by cert-manager.

## Files

Each of the YAML stanzas above is also found in a separate file for easy use with `kubectl apply -f`. Here's a quick explanation of each file:

* `kuma-system-ns.yaml` - For creating the "kuma-system" namespace
* `selfsigned-clusterissuer.yaml` - For creating the SelfSigned ClusterIssuer
* `kuma-root-ca.yaml` - For creating the root CA that will create the Kuma TLS certificate
* `kuma-ca-issuer.yaml` - For specifying the CA should be an Issuer
* `raw-kuma-tls-general.yaml` - For creating the TLS certificate and key that will be used by Kuma
