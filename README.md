## About The Project

This project contains an up-to-date K3S image to run NVIDIA workloads on Kubernetes following the [K3d official documentation](https://github.com/rancher/k3d/tree/main/docs/usage/guides/cuda). 

The native K3S image is based on Alpine but the NVIDIA container runtime is not supported on Alpine. Therefore, we need to build a custom image based on Ubuntu that supports it.


## Prerequisites

First, you need to install the [NVIDIA Container Toolkit](https://docs.nvidia.com/datacenter/cloud-native/container-toolkit/install-guide.html) on your machine.

Also make sure that you have `zstd` installed, since the last versions of K3S use it to generate a embeedded tarball.

## Build the K3S image

Run the `build.sh` script, which takes the K3S git tag as argument, by default it uses: `release-1.21`. 

The script performs the following steps:

* pulls K3S
* builds K3S
* build the custom K3S Docker image

The resulting image is tagged as k3s-gpu:&lt;version tag&gt;. The version tag is the git tag but the '+' sign is replaced with a '-'.

[build.sh](build.sh):

```bash
#!/bin/bash
set -e
cd $(dirname $0)

K3S_TAG="${1:-release-1.21}"
IMAGE_TAG="${K3S_TAG/+/-}"

if [ -d k3s ]; then
    rm -rf k3s
fi
git clone --depth 1 https://github.com/rancher/k3s.git -b $K3S_TAG
cd k3s
mkdir -p build/data && ./scripts/download && go generate
SKIP_VALIDATE=true make
cd ..
unzstd k3s/build/out/data.tar.zst
docker build -t k3s-gpu:$IMAGE_TAG .
```

## Run and test the custom image with Docker

You can run a container based on the new image with Docker:

```bash
docker run --name k3s-gpu -d --privileged --gpus all k3s-gpu:release-1.21
```

Deploy a [test pod](cuda-vector-add.yaml):

```bash
docker cp cuda-vector-add.yaml k3s-gpu:/cuda-vector-add.yaml
docker exec k3s-gpu kubectl apply -f /cuda-vector-add.yaml
docker exec k3s-gpu kubectl logs cuda-vector-add
```

## Run and test the custom image with k3d

Tou can use the image with k3d:

```bash
k3d cluster create --no-lb --image k3s-gpu:release-1.21 --gpus all
```

Deploy a [test pod](cuda-vector-add.yaml):

```bash
kubectl apply -f cuda-vector-add.yaml
kubectl logs cuda-vector-add
```

## Acknowledgements

Most of the work in the repo is based on the official K3d documentation and the following articles:

* [Running CUDA workloads on K3S](https://k3d.io/usage/guides/cuda/)
* [Add NVIDIA GPU support to k3s with containerd](https://dev.to/mweibel/add-nvidia-gpu-support-to-k3s-with-containerd-4j17)
* [microk8s](https://github.com/ubuntu/microk8s)
* [K3S](https://github.com/rancher/k3s)