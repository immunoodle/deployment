# Deployment

TODO: Short blurb on what Immunoodle is.

This repo provides instructions and manifests for deploying Immunoodle in your choice of kubernetes-based orchestration platform.

If you don't have a container orchestration platform, you can use [k3s](https://rancher.com/docs/k3s/latest/en/). Instructions are provided below.

Deploy the resources in the order listed below.

## Hardware Requirements

To run all the compoenents of immunoodle, you'll need at least 2 CPU cores and 8GB of RAM.  These instructions have been tested on Rocky8, Rocky8, and Ubuntu 24.04.3.

### Configuration for your environment

On the system where you'll be installing immunoodle, clone this git repository and make the changes below to configure immunoodle for your environment.

```shell
git clone https://github.com/immunoodle/deployment.git
cd deployment

# Replace PUT_YOUR_HOSTNAME_HERE with the hostname that users will use to access the immunoodle service, then run the command
sed -i "s/IMMUNOODLE_HOSTNAME/PUT_YOUR_HOSTNAME_HERE/g" k8s-manifests/*

# Replace PUT_YOUR_IP_ADDRESS_HERE with the IP address used to access this host, then run the command
sed -i "s/IMMUNOODLE_IP_ADDRESS/PUT_YOUR_IP_ADDRESS_HERE/g" k8s-manifests/*

# Replace PUT_YOUR_POSTGRES_PASSWORD_HERE with a strong password for the `postgres` user in PostgreSQL, then run the command
sed -i "s/IMMUNOODLE_POSTGRES_PASSWORD/PUT_YOUR_POSTGRES_PASSWORD_HERE/g" k8s-manifests/*

# Replace PUT_YOUR_REDIS_PASSWORD_HERE with a strong password used for accessing REDIS, then run the command
sed -i "s/IMMUNOODLE_REDIS_AUTH/PUT_YOUR_REDIS_PASSWORD_HERE/g" k8s-manifests/*

# Replace PUT_YOUR_MINIO_ROOT_PASSWORD_HERE with a strong password used for accessing Minio (Local S3 Object Storage), then run the command
sed -i "s/IMMUNOODLE_MINIO_ROOT_PASSWORD/PUT_YOUR_MINIO_ROOT_PASSWORD_HERE/g" k8s-manifests/*

# Run the following two commands to generate a random string which will be used as part of the authentication service
IMMUNOODLE_OAUTH_CLIENT_ID=$(openssl rand -hex 32)
sed -i "s/IMMUNOODLE_OAUTH_CLIENT_ID/$IMMUNOODLE_OAUTH_CLIENT_ID/g" k8s-manifests/*

# Run the following two commands to generate a random string which will be used as part of the authentication service
IMMUNOODLE_OAUTH_SECRET=$(openssl rand -hex 32)
sed -i "s/IMMUNOODLE_OAUTH_SECRET/$IMMUNOODLE_OAUTH_SECRET/g" k8s-manifests/*

# Run the following two commands to generate a random string which will be used as part of the authentication service
IMMUNOODLE_OAUTH_COOKIE_SECRET=$(openssl rand -hex 16)
sed -i "s/IMMUNOODLE_OAUTH_COOKIE_SECRET/$IMMUNOODLE_OAUTH_COOKIE_SECRET/g" k8s-manifests/*
```

## Install k3s (Only requireed if you don't already have a Kubernetes cluster)

If you don't have Kubernetes already installed, you can follow these instructions for deploying K3s (a lightweight Kubernetes environment).

[K3s Install](K3S.md)

## Create immunoodle namespace

These instructions expect all components of immunoodle to be installed in the immunoodle namespace.  If you haven't already created the `immunoodle` namespace, please do it now.

```shell
sudo kubectl create ns immunoodle
```

## Traefik

Traefik is a Ingress Controller that provides access to the various Immunoodle Components

```shell
sudo kubectl -n immunoodle apply -f k8s-manifests/traefik.yml
```

## Dex

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


