# Autocert

**Autocert** is a kubernetes add-on that automatically injects TLS/HTTPS certificates into your containers.

<p align="center"><img src="demo.gif" style="max-width: 600px" alt="Animated terminal showing autocert in practice"></p>

To request a certificate simply annotate your pods with a name. Certificates are issued by a private **internal certificate authority** that runs on your cluster and are mounted at `/var/run/autocert.step.sm` along with a corresponding private key and root certificate.

TLS (e.g., HTTPS) is the most widely deployed cryptographic protocol in the world. Mutual TLS (mTLS) provides end-to-end security for service-to-service communication and can **replace complex VPNs** to secure communication into, out of, and between kubernetes clusters. But **to use mTLS you need certificates issued by your own certificate authority (CA)**.

Building and operating a CA, issuing certificates, and making sure they're renewed before they expire is tricky. Autocert does all of this for you.

## Key Features

 * A complete public key infrastructure that you control for your kubernetes clusters
 * Certificate authority that's easy to initialize and install
 * Automatic injection of certificates and keys in annotated containers
 * Enable on a per-namespace basis
 * Namespaced installation to restrict access to privileged CA and provisioner containers
 * Ability to run subordinate to an existing public key infrastructure
 * Supports federatation with other roots
 * Short-lived certificates
 * Automatic renewal
 * Uses your own certificate authority -- you control who or what gets a certificate

## Getting Started

These instructions will get `autocert` installed quickly on an existing kubernetes cluster.

### Prerequisites

Make sure you've [`installed step`](https://github.com/smallstep/cli#installing) version `0.8.3` or later:

```bash
$ step version
Smallstep CLI/0.8.3 (darwin/amd64)
Release Date: 2019-01-16 01:46 UTC
```

You'll also need `kubectl` and a kubernetes cluster running version `1.9` or later with [webhook admission controllers](https://kubernetes.io/docs/reference/access-authn-authz/extensible-admission-controllers/#admission-webhooks) enabled:

```bash
$ kubectl version --short
Client Version: v1.13.1
Server Version: v1.10.11
$ kubectl api-versions | grep "admissionregistration.k8s.io/v1beta1"
admissionregistration.k8s.io/v1beta1
```

We'll be creating a new kubernetes namespace and setting up some RBAC rules during installation. You'll need appropriate permissions in your cluster (e.g., you may need to be cluster-admin).

```bash
TODO: Check whether you have cluster permissions..? GKE instructions here if you don't have them.
```

In order to grant these permissions you may need to give yourself cluster-admin rights in your cluster. GKE, in particular, does not give the cluster owner these rights by default. You can give yourself cluster-admin rights by running:

```bash
kubectl create clusterrolebinding cluster-admin-binding \
    --clusterrole cluster-admin \
    --user $(gcloud config get-value account)
```

### Install

To install `step certificates` and `autocert` in one step run:

```bash
$ kubectl run autocert-init -it --rm --image smallstep/autocert-init --restart Never
```

You may need to adjust the RBAC policies to run `autocert-init`:

```bash
$ kubectl create clusterrolebinding autocert-init-binding --clusterrole cluster-admin --user "system:serviceaccount:default:default"
```

Once `autocert-init` is complete you can delete this binding:

```bash
$ kubectl delete clusterrolebinding autocert-init-binding
```

You can also [install manually](INSTALL.md).

### Enable autocert

To enable `autocert` for a namespace the `autocert.step.sm=enabled` label (the `autocert` webhook will not affect namespaces for which it is not enabled). To enable `autocert` for the default namespace run:

```bash
$ kubectl label namespace default autocert.step.sm=enabled
```

To check which namespaces have `autocert` enabled run:

```bash
$ kubectl get namespace -L autocert.step.sm
NAME          STATUS   AGE   AUTOCERT.STEP.SM
default       Active   59m   enabled
...
```

### Annotate pods

In addition to enabling `autocert` for a namespace, pods must be annotated with their name for certificates to be injected. The annotated name will appear as the common name and SAN in the issued certificate.

To trigger certificate injection pods must be annotated at creation time. You can do this in your deployment YAMLs:

```bash
$ cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata: {name: sleep}
spec:
  replicas: 1
  selector: {matchLabels: {app: sleep}}
  template:
    metadata:
      annotations:
        autocert.step.sm/name: sleep.default.svc.cluster.local
      labels: {app: sleep}
    spec:
      containers:
      - name: sleep
        image: alpine
        command: ["/bin/sleep", "86400"]
        imagePullPolicy: IfNotPresent
EOF
```

The `autocert` admission webhook will intercept this pod creation request and inject an init container and sidecar to manage certificate issuance and renewal, respectively.

```bash
$ kubectl get pods -l app=sleep \
    -o=custom-columns=NAME:.metadata.name,CONTAINERS:.spec.containers[*].name,INIT_CONTAINERS:.spec.initContainers[*].name
NAME                    CONTAINERS               INIT_CONTAINERS
sleep-f996bd578-tzwvm   sleep,autocert-renewer   autocert-bootstrapper
```

Certificates are mounted in containers at `/var/run/autocert.step.sm`. We can inspect this directory to make sure everything worked correctly:

```bash
$ kubectl exec -it sleep-f996bd578-nch7c -c sleep -- ls -lias /var/run/autocert.step.sm
total 20
1593393      4 drwxrwxrwx    2 root     root          4096 Jan 17 21:27 .
1339651      4 drwxr-xr-x    1 root     root          4096 Jan 17 21:27 ..
1593451      4 -rw-------    1 root     root           574 Jan 17 21:27 root.crt
1593442      4 -rw-r--r--    1 root     root          1352 Jan 17 21:41 site.crt
1593443      4 -rw-r--r--    1 root     root           227 Jan 17 21:27 site.key
```

The `autocert-renewer` sidecar also installs the `step` CLI tool, which we can use to inspect the issued certificate:

```bash
$ kubectl exec -it sleep-f996bd578-nch7c -c autocert-renewer -- step certificate inspect /var/run/autocert.step.sm/site.crt
Certificate:
    Data:
        Version: 3 (0x2)
        Serial Number: 46935033335539540860078000614852612373 (0x234f5bce23705f015a8377ab1cfd5115)
    Signature Algorithm: ECDSA-SHA256
        Issuer: CN=Autocert Intermediate CA
        Validity
            Not Before: Jan 17 21:41:04 2019 UTC
            Not After : Jan 17 21:46:14 2019 UTC
        Subject: CN=sleep.default.svc.cluster.local
        Subject Public Key Info:
            Public Key Algorithm: ECDSA
                Public-Key: (256 bit)
                X:
                    31:aa:a1:7f:c8:b4:c6:da:90:fc:b8:3a:e9:cc:48:
                    f9:89:b9:5d:d7:a4:63:80:76:9f:21:6d:e5:88:4c:
                    a8:e4
                Y:
                    ed:21:38:57:cd:3f:32:71:6f:ca:81:34:b0:4a:bd:
                    a3:c4:8d:d1:87:bc:2c:4c:42:79:e5:35:49:38:3f:
                    b7:c8
                Curve: P-256
        X509v3 extensions:
            X509v3 Key Usage: critical
                Digital Signature, Key Encipherment
            X509v3 Extended Key Usage:
                TLS Web Server Authentication, TLS Web Client Authentication
            X509v3 Subject Key Identifier:
                43:0E:0A:50:30:A5:5B:AF:22:AC:28:49:26:53:2A:B4:D4:20:E0:E0
            X509v3 Authority Key Identifier:
                keyid:61:45:1E:E4:95:4C:0A:6B:37:4C:43:41:FD:54:2E:8E:5E:A2:24:EF
            X509v3 Subject Alternative Name:
                DNS:sleep.default.svc.cluster.local

    Signature Algorithm: ECDSA-SHA256
         30:44:02:20:0c:c5:ab:0d:22:17:a2:04:9f:ff:5f:b1:c0:a5:
         8b:94:88:e0:40:66:e1:19:e9:34:2f:67:74:12:4f:bb:51:8b:
         02:20:01:7e:0d:44:ce:b2:92:41:d5:78:0d:02:5a:68:05:7c:
         c2:a9:81:28:71:5c:95:6d:56:51:49:e0:37:b7:09:87
```

### Test your installation

To test your installation you can install the `hello-mtls` demo app.

* Install app, which uses mTLS and responds "hello, `identity`"
* Do a `kubectl run` of `step-cli` then get a certificate using `step` and `curl hello-mtls` from within the cluster
* Port forward from localhost to get a certificate then `curl` with `--resolve`

### Further reading

* Link to ExternalDNS example
* Link to multiple cluster with Service type ExternalDNS so they can communicate

### Uninstall

* Delete the `sleep` deployment (if you created it)
* Remove labels (show how to find labelled namespaces)
* Remove annotations (show how to find any annotated pods)
* Remove secrets (show how to find labelled secrets)
* Delete `step` namespace

### Questions

#### How is this different than [`cert-manager`](https://github.com/jetstack/cert-manager)

#### Doesn't kubernetes already ship with a certificate authority?

Yes, but it's designed for use by the kubernetes control plane rather than by your data plane services. You could use the kubernetes CA to issue certificates for data plane communication, but it's probably not a good idea.

#### Why not use kubernetes CSR resources for this?

It's harder and less secure.

#### What are `autocert` certificates good for?

Autocert certificates let you secure your data plane (service-to-service) communication using mutual TLS (mTLS). Services and proxies can limit access to clients that also have a certificate issued by your certificate authority (CA). Servers can identify which client is connecting improving visibility and enabling granular access control.

Once certificates are issued you can use mTLS to secure communication in to, out of, and between kubernetes clusters. Services can use mTLS to only allow connections from clients that have their own certificate issued from your CA.

It's like your own Let's Encrypt, but you control who gets a certificate.