## Vosk API - Docker/GPU

[Vosk](https://github.com/alphacep/vosk-api) docker images with GPU for Jetson Nano / Xavier NX boards and PCs with NVIDIA cards.

### Usage

Pull an existing [image](https://hub.docker.com/r/sskorol/vosk-api) with a required tag.

```shell
docker pull sskorol/vosk-api:$TAG
```

Use it as a baseline in your app's Dockerfile:

```shell
FROM sskorol/vosk-api:$TAG
```

### Build prerequisites

- You have to enable [nvidia runtime](https://github.com/dusty-nv/jetson-containers#docker-default-runtime) before building the images.
- In the case of Jetson boards, your JetPack should match at least 32.5 version (0.3.32 images were built against 32.6.1).
- For PCs make sure you met the following [prerequisites](https://medium.com/geekculture/installing-cudnn-and-cuda-toolkit-on-ubuntu-20-04-for-machine-learning-tasks-f41985fcf9b2).

### Building

Clone sources and check a build file help:

```shell
git clone https://github.com/sskorol/vosk-api-gpu.git
cd vosk-api-gpu
```

#### Jetson boards

Run a build script with the required args depending on your platform, e.g.:

```shell
cd jetson && ./build.sh -m nano -i ml -t 0.3.37
```

You can check the available NVIDIA base image tags [here](https://ngc.nvidia.com/catalog/containers/nvidia:l4t-base) and [here](https://ngc.nvidia.com/catalog/containers/nvidia:l4t-ml). 

#### PCs

To build images for PC, use the following script:

```shell
cd pc && ./build.sh -c 11.3.1-devel-ubuntu20.04 -t 0.3.37
```

Here, you have to provide a base cuda image tag and the output container's tag. You can read more by running the script with `-h` flag.

This script will build 2 images: base and a sample Vosk server.

#### PC Windows 11 + WSL2
1. Followed the instructions at https://docs.docker.com/desktop/windows/wsl/ to install Docker Desktop and make sure the WSL 2 backend is enabled.

2. Use the following command line test to confirm that the WSL2 GPU environment is working.
```shell
docker run --rm -it --gpus=all nvcr.io/nvidia/k8s/cuda-sample:nbody nbody -gpu -benchmark
```
3. Prepare an English voice wav in ./vosk-api-gpu/weather.wav  
Prepare the english model in ./vosk-api-gpu/model
```shell
./test.sh 0.3.37-pc
```

#### Apple M1

To build images (w/o GPU) for Apple M1, use the following script:

```shell
cd m1 && ./build.sh -t 0.3.37
```

To build Kaldi and Vosk API locally (w/o Docker), use the following script (thanks to @aivoicesystems):

```shell
cd m1 && ./build-local.sh
```

Note that there's a required software check when you start this script. If you see missing requirements, chances are you'll need to install the following packages:

```shell
brew install autoconf cmake automake libtool
```

#### GCP

To test images on GCP with NVIDIA Tesla T4, use the following steps:

- Install [terraform](https://learn.hashicorp.com/tutorials/terraform/install-cli)
- Create a new project on GCP
- Install and init [gcloud-cli](https://cloud.google.com/sdk/docs/install-sdk)
- Deploy a new Compute Engine instance with the following commands:

```shell
cd gcp && terraform init && terraform apply
```

Note that you'll be prompted to type your GCP project name. When a new instance is deployed, you can now ssh to it:

```shell
gcloud compute ssh --project $PROJECT_NAME --zone us-central1-a gpu
```

Clone this repo and build Vosk images on a relatively powerful machine:

```shell
git clone https://github.com/sskorol/vosk-api-gpu && cd vosk-api-gpu/gcp && ./build.sh
```

Note that some variables are hardcoded at the moment. Feel free to change them if you want.

### Running

The following script will start docker-compose, install requirements and run a simple test:

```shell
./test.sh $TAG
```

- Pass a newly built image tag as an argument.
- You have to download and extract a required [model](https://alphacephei.com/vosk/models) into `./model` folder first, unless you use a GCP instance.
- Use your own recording to test it against any other language than RU.

### Important notes

- Jetson Nano won't work with the latest large model due to high memory requirements (at least 8Gb RAM).
- Jetson Xavier **will** work with the latest large model if you remove `rnnlm` folder from `model`.
- Make sure you have at least Docker (20.10.6) and Compose (1.29.1) versions.
- Your host's CUDA version must match the container's as they share the same runtime. Jetson images were built with CUDA 10.1. As per the desktop version: CUDA 11.3.1 was used.
- If you plan to use `rnnlm`, make sure you allocated at least 12Gb of RAM to your Docker instance (16Gb is optimal).
- In case of GCP usage, there's a know issue with K80 instance. Seems like it has an outdated architecture. So it's recommended to take at least NVIDIA T4.
- Not all the models are adopted for GPU-usage, e.g. in RU model, you have to manually patch configs to make it work (it's done automatically for GCP instance):
  - remove `min-active` flag from `model/conf/model.conf`
  - copy/paste `ivector.conf` from big EN model
