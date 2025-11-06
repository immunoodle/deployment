# Deployment

This repo provides the instructions and Kubernetes manifests to deploy Immunoodle in your choice of container orchestration platform.

If you don't have a container orchestration platform, you can use [k3s](https://rancher.com/docs/k3s/latest/en/) to deploy . Instructions are provided further below in this README. 

Deploy the resources in the order listed in the Table Of Contents, which is broken into two parts - Infrastructure and Application.

**Replace IMMUNOODLE_HOSTNAME and IMMUNOODLE_PASSWORD in manifests**

Start with Infrastructure below and then continue onto [Application](https://github.com/immunoodle/deployment#Application)

## Table of Contents - Infrastructure

[k3s](https://github.com/immunoodle/deployment#k3s) **Only needed if you don't have a Kubernetes environment for deployment**

[postgresql](https://github.com/immunoodle/deployment#Postgresql)

[dex](https://github.com/immunoodle/deployment#Dex)

[redis](https://github.com/immunoodle/deployment#Redis)

[traefik](https://github.com/immunoodle/deployment#Traefik)

[whoami](https://github.com/immunoodle/deployment#Whoami)

### k3s

Requirements:

https://docs.k3s.io/installation/requirements

Note: firewall requirements if running one

```
firewall-cmd --permanent --add-port=6443/tcp
firewall-cmd --permanent --zone=trusted --add-source=10.42.0.0/16 
firewall-cmd --permanent --zone=trusted --add-source=10.43.0.0/16 
firewall-cmd --reload
```

Pass custom arguments to k3s install (We use a custom traefik in the immunoodle namespace and this has been tested on k3s v 1.3.35)

```
curl -sfL https://get.k3s.io | INSTALL_K3S_VERSION=v1.33.5+k3s1 sh -s - --disable=traefik
sudo kubectl get nodes
```

#### Create immunoodle namespace

```bash
kubectl create ns immunoodle
```

#### Install cert-manager

```
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.19.1/cert-manager.yaml
```

Import your own certificate if you have your own:

Place the PEM encoded certificate you received from your CA in a file named `tls.crt`. Append the (entire) intermediate chain to that file, so at the top you have your certificate and below, in order, the certificate chain.

Place your PEM encoded private key in a file named `tls.key`. 

```
 kubectl create secret generic
 cert-official --from-file=tls.crt=tls.crt --from-file=tls.key=tls.key -n immunoodle
```
Otherwise - create your own root CA:

```
cat <<EOF | kubectl apply -f -
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: root-ca-issuer-selfsigned
spec:
  selfSigned: {}
---
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: root-ca
  namespace: cert-manager
spec:
  isCA: true
  commonName: root-ca
  secretName: root-ca-secret
  duration: 87600h # 10y
  renewBefore: 78840h # 9y
  privateKey:
    algorithm: ECDSA
    size: 256
    rotationPolicy: Always
  issuerRef:
    name: root-ca-issuer-selfsigned
    kind: ClusterIssuer
    group: cert-manager.io
---
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: root-ca-issuer
spec:
  ca:
    secretName: root-ca-secret
EOF
```

Confirm CA is ready

```
kubectl describe ClusterIssuer -n cert-manager
```

Create intermediate CA

```
cat <<EOF | kubectl apply -f -
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: intermediate-ca1
  namespace: cert-manager
spec:
  isCA: true
  commonName: intermediate-ca1
  secretName: intermediate-ca1-secret
  duration: 43800h # 5y
  renewBefore: 35040h # 4y
  privateKey:
    algorithm: ECDSA
    size: 256
    rotationPolicy: Always
  issuerRef:
    name: root-ca-issuer
    kind: ClusterIssuer
    group: cert-manager.io
---
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: intermediate-ca1-issuer
spec:
  ca:
    secretName: intermediate-ca1-secret
EOF
```

Copy root-ca-secret to immunoodle namespace

```
 kubectl get secret root-ca-secret --namespace=cert-manager -o yaml \
  | sed 's/namespace: cert-manager/namespace: immunoodle/' \
  | kubectl create -f -
```

#### Export self-signed Root CA for import to browsers

```
kubectl -n cert-manager get secret root-ca-secret -o jsonpath='{.data.tls\.crt}' | base64 -d > cert.crt
```

```
kubectl -n cert-manager get secret root-ca-secret -o jsonpath='{.data.tls\.key}' | base64 -d > cert.key
```

Create a cert that can be imported into keychain or cert manager 

```
openssl pkcs12 -export -out immunoodle.pfx -inkey cert.key -in cert.crt

```

### Traefik

Traefik is a Ingress Controller that provides access to the various Immunoodle Components

```
kubectl -n immunoodle apply -f k8s-manifests/traefik.yml
```

### Dex

Dex is used for auth for the Immunoodle Components. First we will spin up the deployment, spin it down to copy template files into place

```
Create a unique id for each static client in k8s-manifests/dex.yml
kubectl -n immunoodle apply -f k8s-manifests/dex.yml
kubectl -n immunoodle scale deploy dex --replicas=0
# This will get you into a shell with access to the pvc
kubectl -n immunoodle run -it --rm debug --image=busybox --restart=Never   --overrides='{"spec":{"containers":[{"name":"debug","image":"busybox","stdin":true,"tty":true,"volumeMounts":[{"name":"storage","mountPath":"/var/dex"}]}],"volumes":[{"name":"storage","persistentVolumeClaim":{"claimName":"dex"}}]}}'
# run this in another terminal window/tab
kubectl -n immunoodle cp templates/web.zip debug:/var/dex/
# in the other terminal window/tab
cd /var/dex && unzip /var/dex/web.zip && exit
# scale it back up
kubectl -n immunoodle scale deploy dex --replicas=1
```

### Whoami

Whoami let's us test the components installed thus far

```
kubectl -n immunoodle apply -f k8s-manifests/whoami.yml
```

*Use the whoami application to confirm the basic components such as traefik and cert-manager work thus far.*

### PostgreSQL 

PostgresQL is used for backing database for Dex for Auth and for the various Immunoodle Components.

```
kubectl -n immunoodle apply -f k8s-manifests/postgresql.yml
```

### Redis

Redis is the key-value database for Immunoodle

```
kubectl -n immunoodle apply -f k8s-manifests/redis.yml
```

### Minio

Minio provides storage for Immunoodle

```
kubectl -n immunoodle apply -f k8s-manifests/minio.yml
```


Once the Immunoodle Infrastructure has been deployed and tested, move onto Imunoodle application deployment.

## Application 

## Table of Contents - Immunodle Application

[SignUp](https://github.com/immunoodle/deployment#Signup)

[Worker](https://github.com/immunoodle/deployment#Worker)

[API](https://github.com/immunoodle/deployment#API)

[Data Portal](https://github.com/immunoodle/deployment#DataPortal)

[I-SPI](https://github.com/immunoodle/deployment#I-SPI)

### Signup

Signup creates and manages local users for the Immunoodle application stack as integrates with your choice of Oauth2 provider.

```
kubectl -n immunoodle apply -f k8s-manifests/signup.yml
```

### Worker

Worker handles task management for data processing in the Immunoodle application stack

```
kubectl -n immunoodle apply -f k8s-manifests/worker.yml
```

### API

API provides API endpoints for data processing in the Immunoodle application stack

```
kubectl -n immunoodle apply -f k8s-manifests/api.yml
```

### DataPortal

Data Portal Database

```
kubectl -n immunoodle exec -it deploy/postgresql -- psql -U postgres postgres < db-dumps/dataportal.sql
```

```
kubectl -n immunoodle apply -f k8s-manifests/data-portal.yml
```


### I-SPI

I-SPI is an interactive R Shiny application for processing, analyzing, and visualizing Luminex bead-based immunoassay data. It provides a unified platform for managing serology experiments with robust features for data import, quality control, curve fitting, and results visualization. I-SPI depends on the rest of the Infrastructure and Application stacks being deployed first

Set-up the database for the application first. Find the name of the postgresql pod in the immunoodle namespace:

```
kubectl -n immunoodle exec -it deploy/postgresql -- psql -U postgres -c "CREATE DATABASE immunoodle;"
kubectl -n immunoodle exec -it deploy/postgresql -- psql -U postgres immunoodle < db-dumps/i-spi-db.sql
```

```
kubectl -n immunoodle apply -f k8s-manifests/ispi.yml
```


