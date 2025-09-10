# Immunoodle Container Images in an Air Gap Environment

## Download/Save Images

The first step to making Immunoodle available in an air gap environment is to download the images from a computer connected to the internet with `podman` or `docker` installed.

Here are the example commands to download the images.

> NOTE: You probably will need to update the image version numbers
 
```shell
docker pull redis:8
docker pull quay.io/minio/minio:RELEASE.2025-07-23T15-54-02Z
docker pull dexidp/dex:v2.41.1-alpine
docker pull postgres:17.2
docker pull traefik:v3.2.2
docker pull traefik/whoami:latest
docker pull quay.io/oauth2-proxy/oauth2-proxy:v7.8.1
docker pull ghcr.io/immunoodle/signup:main
```

With the images downloaded, save them as tarballs.

```shell
docker save -o redis.tar redis:8
docker save -o minio.tar quay.io/minio/minio:RELEASE.2025-07-23T15-54-02Z
docker save -o dex.tar dexidp/dex:v2.41.1-alpine
docker save -o postgres.tar postgres:17.2
docker save -o traefik.tar  traefik:v3.2.2
docker save -o whoami.tar traefik/whoami:latest
docker save -o oauth2-proxy.tar quay.io/oauth2-proxy/oauth2-proxy:v7.8.1
docker save -o signup.tar ghcr.io/immunoodle/signup:main
```

Transfer the tarballs to a USB drive.

## Making Images Available

### K3s

In K3s, you can copy the tarballs to the `/var/lib/rancher/k3s/agent/images/` directory.  Within a few minutes, the images will be available for use.

