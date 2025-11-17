# Deployment

This repo provides the instructions and Kubernetes manifests to deploy Immunoodle in your choice of container orchestration platform.

If you don't have a container orchestration platform, you can use [k3s](https://rancher.com/docs/k3s/latest/en/) to deploy . Instructions are provided further below in this README. 

Deploy the resources in the order listed in the Table Of Contents, which is broken into two parts - Infrastructure and Application.


Start with Infrastructure below and then continue onto [Application](https://github.com/immunoodle/deployment#Application)

## Table of Contents - Infrastructure

[k3s](https://github.com/immunoodle/deployment#k3s) **Only needed if you don't have a Kubernetes environment for deployment**

[postgresql](https://github.com/immunoodle/deployment#Postgresql)

[dex](https://github.com/immunoodle/deployment#Dex)

[redis](https://github.com/immunoodle/deployment#Redis)

[traefik](https://github.com/immunoodle/deployment#Traefik)

[whoami](https://github.com/immunoodle/deployment#Whoami)

### Setup

On the system where you'll be installing immunoodle, clone the deployment repo:

```shell
git clone https://github.com/immunoodle/deployment.git
cd deployment

# Replace PUT_YOUR_HOSTNAME_HERE with your extenal facing hostname
sed -i "s/IMMUNOODLE_HOSTNAME/PUT_YOUR_HOSTNAME_HERE/g" k8s-manifests/*
sed -i "s/IMMUNOODLE_IP_ADDRESS/PUT_YOUR_IP_ADDRESS_HERE/g" k8s-manifests/*
sed -i "s/IMMUNOODLE_POSTGRES_PASSWORD/PUT_YOUR_POSTGRES_PASSWORD_HERE/g" k8s-manifests/*
sed -i "s/IMMUNOODLE_REDIS_AUTH/PUT_YOUR_REDIS_PASSWORD_HERE/g" k8s-manifests/*
IMMUNOODLE_OAUTH_CLIENT_ID=$(openssl rand -hex 32)
IMMUNOODLE_OAUTH_SECRET=$(openssl rand -hex 32)
IMMUNOODLE_OAUTH_COOKIE_SECRET=$(openssl rand -hex 16)
sed -i "s/IMMUNOODLE_OAUTH_CLIENT_ID/$IMMUNOODLE_OAUTH_CLIENT_ID/g" k8s-manifests/*
sed -i "s/IMMUNOODLE_OAUTH_SECRET/$IMMUNOODLE_OAUTH_SECRET/g" k8s-manifests/*
sed -i "s/IMMUNOODLE_OAUTH_COOKIE_SECRET/$IMMUNOODLE_OAUTH_COOKIE_SECRET/g" k8s-manifests/*
```

### k3s

Requirements:

https://docs.k3s.io/installation/requirements

Pass custom arguments to k3s install (We use a custom traefik in the immunoodle namespace and this has been tested on k3s v 1.3.35)

```shell
curl -sfL https://get.k3s.io | INSTALL_K3S_VERSION=v1.33.5+k3s1 sh -s - --disable=traefik
sudo ln -s /usr/local/bin/kubectl /bin/kubectl
sudo kubectl get node -o wide
```

#### CoreDNS

```shell
sudo kubectl apply -f k8s-manifests/coredns.yml
sudo kubectl -n kube-system rollout restart deploy coredns
```

#### Create immunoodle namespace

These instructions expect all components of immunoodle to be installed in the immunoodle namespace.

```shell
sudo kubectl create ns immunoodle
```

#### Install cert-manager

```shell
sudo kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.19.1/cert-manager.yaml
```

```shell
cat <<EOF | sudo kubectl apply -f -
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

```shell
sudo kubectl describe ClusterIssuer -n cert-manager
```

Create intermediate CA

```shell
cat <<EOF | sudo kubectl apply -f -
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

```shell
sudo kubectl get secret root-ca-secret --namespace=cert-manager -o yaml \
  | sed 's/namespace: cert-manager/namespace: immunoodle/' \
  | sudo kubectl create -f -
```

#### Export self-signed Root CA for import to browsers

```shell
sudo kubectl -n cert-manager get secret root-ca-secret -o jsonpath='{.data.tls\.crt}' | base64 -d > cert.crt
```

### Traefik

Traefik is a Ingress Controller that provides access to the various Immunoodle Components

```shell
sudo kubectl -n immunoodle apply -f k8s-manifests/traefik.yml
```

### Dex

Dex is used for auth for the Immunoodle Components. First we will spin up the deployment, spin it down to copy template files into place

```shell
sudo kubectl -n immunoodle apply -f k8s-manifests/dex.yml
# TODO: Improve this process
sudo kubectl -n immunoodle scale deploy dex --replicas=0
# This will get you into a shell with access to the pvc
sudo kubectl -n immunoodle run -it --rm debug --image=busybox --restart=Never   --overrides='{"spec":{"containers":[{"name":"debug","image":"busybox","stdin":true,"tty":true,"volumeMounts":[{"name":"storage","mountPath":"/var/dex"}]}],"volumes":[{"name":"storage","persistentVolumeClaim":{"claimName":"dex"}}]}}'
# run this in another terminal window/tab
sudo kubectl -n immunoodle cp templates/web.zip debug:/var/dex/
# in the other terminal window/tab
cd /var/dex && unzip /var/dex/web.zip && exit
# scale it back up
sudo kubectl -n immunoodle scale deploy dex --replicas=1
```

### Whoami

Whoami lets us test the components installed thus far

```shell
sudo kubectl -n immunoodle apply -f k8s-manifests/whoami.yml
```

*Use the whoami application to confirm the basic components such as traefik and cert-manager work thus far. You'll need to use Signup to create the first user account*

### PostgreSQL

PostgresQL is used for backing database for Dex for Auth and for the various Immunoodle Components.

```shell
sudo kubectl -n immunoodle apply -f k8s-manifests/postgresql.yml
```

### Redis

Redis is the key-value database for Immunoodle

```shell
sudo kubectl -n immunoodle apply -f k8s-manifests/redis.yml
```

### Minio

Minio provides storage for Immunoodle

```shell
sudo kubectl -n immunoodle apply -f k8s-manifests/minio.yml
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
sudo kubectl -n immunoodle apply -f k8s-manifests/signup.yml
```

### Worker

Worker handles task management for data processing in the Immunoodle application stack

```
sudo kubectl -n immunoodle apply -f k8s-manifests/worker.yml
```

### API

API provides API endpoints for data processing in the Immunoodle application stack

```
sudo kubectl -n immunoodle apply -f k8s-manifests/api.yml
```

### DataPortal

Data Portal Database

```
sudo kubectl -n immunoodle exec -it deploy/postgresql -- psql -U postgres postgres < db-dumps/dataportal.sql
```

```
sudo kubectl -n immunoodle apply -f k8s-manifests/data-portal.yml
```


### I-SPI

I-SPI is an interactive R Shiny application for processing, analyzing, and visualizing Luminex bead-based immunoassay data. It provides a unified platform for managing serology experiments with robust features for data import, quality control, curve fitting, and results visualization. I-SPI depends on the rest of the Infrastructure and Application stacks being deployed first

Set-up the database for the application first. Find the name of the postgresql pod in the immunoodle namespace:

```
sudo kubectl -n immunoodle exec -it deploy/postgresql -- psql -U postgres -c "CREATE DATABASE immunoodle;"
sudo kubectl -n immunoodle exec -it deploy/postgresql -- psql -U postgres immunoodle < db-dumps/i-spi-db.sql
```

```
sudo kubectl -n immunoodle apply -f k8s-manifests/ispi.yml
```


