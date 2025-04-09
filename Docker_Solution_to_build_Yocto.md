# 🐳 Yocto Development in Docker (Ubuntu 22.04)

This guide walks through setting up a Yocto Project development environment inside a Docker container using Ubuntu 22.04. It avoids the common pitfalls of running `bitbake` as root and ensures compatibility with mounted host directories (e.g. `/media/...`).



## Required Packages

Yocto requires the following packages on a Ubuntu-based system:

```bash
sudo apt install build-essential chrpath cpio debianutils diffstat file gawk gcc git iputils-ping \
libacl1 liblz4-tool locales python3 python3-git python3-jinja2 python3-pexpect python3-pip \
python3-subunit socat texinfo unzip wget xz-utils zstd
```



## Dockerfile #1: Basic Yocto Setup (runs as root)

This version works fine for experimenting, **but will break when you run `bitbake`**, which refuses to run as root.

```Dockerfile
# Dockerfile (Basic)
FROM ubuntu:22.04

ENV DEBIAN_FRONTEND=noninteractive

RUN apt-get update && apt-get install -y \
    build-essential chrpath cpio debianutils diffstat file gawk gcc git iputils-ping \
    libacl1 liblz4-tool locales python3 python3-git python3-jinja2 python3-pexpect python3-pip \
    python3-subunit socat texinfo unzip wget xz-utils zstd \
    && apt-get clean && rm -rf /var/lib/apt/lists/*

RUN locale-gen en_US.UTF-8
ENV LANG=en_US.UTF-8
ENV LANGUAGE=en_US:en
ENV LC_ALL=en_US.UTF-8

WORKDIR /workdir
CMD ["/bin/bash"]
```



## Dockerfile #2: Non-Root Yocto Dev with Host UID/GID Support

This version creates a `yocto` user and remaps its UID and GID to match the host's. This allows writing to mounted volumes such as `/media`.

```Dockerfile
# Dockerfile_not_root
FROM ubuntu:22.04

ENV DEBIAN_FRONTEND=noninteractive

RUN apt-get update && apt-get install -y \
    build-essential chrpath cpio debianutils diffstat file gawk gcc git iputils-ping \
    libacl1 liblz4-tool locales python3 python3-git python3-jinja2 python3-pexpect python3-pip \
    python3-subunit socat texinfo unzip wget xz-utils zstd sudo \
    && apt-get clean && rm -rf /var/lib/apt/lists/*

RUN locale-gen en_US.UTF-8
ENV LANG=en_US.UTF-8
ENV LANGUAGE=en_US:en
ENV LC_ALL=en_US.UTF-8

# Create yocto user
RUN useradd -m yocto -s /bin/bash && \
    echo "yocto ALL=(ALL) NOPASSWD:ALL" >> /etc/sudoers

# Allow host UID/GID to be passed at build time
ARG HOST_UID=1000
ARG HOST_GID=1000

RUN groupmod -g ${HOST_GID} yocto && \
    usermod -u ${HOST_UID} -g ${HOST_GID} yocto

USER yocto
WORKDIR /home/yocto
CMD ["/bin/bash"]
```



## Differences Between the Two Dockerfiles

| Feature                      | Basic Version         | UID/GID Remapped Version   |
|--||--|
| Runs as root                | ✅ Yes                | ❌ No (non-root user)       |
| Bitbake compatible          | ❌ No                 | ✅ Yes                      |
| Works with mounted volumes  | ❌ Permission issues  | ✅ Matches host UID/GID     |
| Suitable for development    | ⚠️ Limited/testing   | ✅ Recommended setup         |



## 🏗Build the Docker Image

```bash
docker build -f Dockerfile_not_root -t ubuntu22_yocto_dev:latest \
  --build-arg HOST_UID=$(id -u) --build-arg HOST_GID=$(id -g) .
```



## Run the Container

Mount your project directory (e.g. Yocto sources under `/media/...`):

```bash
docker run -it \
  -v /media:/media \
  ubuntu22_yocto_dev:latest
```

Inside the container:
```bash
cd /media/mheydarc/MHH-Source/Yocto/poky
source oe-init-build-env
```



## Fixing "Cannot write to build" Error

If you see:
```
Error: Cannot write to /media/.../poky/build
```

It's because the `build/` directory is owned by `root`. You can fix it:

```bash
sudo chown -R yocto:yocto build
source oe-init-build-env
```

This allows BitBake to proceed without issue.



## Notes

- **Don't run BitBake as root** — always use the non-root setup for real builds.
- Make sure `poky/` and any layers you mount are writable by the container user.
- You can also build Yocto in a container-local directory:
  ```bash
  source oe-init-build-env ~/my-build
  ```



