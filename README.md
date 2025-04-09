# Yocto Development in Docker (Ubuntu 22.04)


This guide walks through setting up a Yocto Project development environment inside a Docker container using Ubuntu 22.04. It avoids the common pitfalls of running `bitbake` as root and ensures compatibility with mounted host directories (e.g. `/media/...`).



## Required Packages

Yocto requires the following packages on a Ubuntu-based system:

```bash
sudo apt install build-essential chrpath cpio debianutils diffstat file gawk gcc git iputils-ping \
libacl1 liblz4-tool locales python3 python3-git python3-jinja2 python3-pexpect python3-pip \
python3-subunit socat texinfo unzip wget xz-utils zstd
```



## Dockerfile as root: Don't use it, it's just for archiving

This version does not work since Yocto is not working with user *root*

*Dockerfile_as_root*



## Dockerfile : Non-Root Yocto Dev with Host UID/GID Support

This version creates a `yocto` user and remaps its UID and GID to match the host's. This allows writing to mounted volumes such as `/media`.

*Dockerfile_not_root*

## Differences Between the Two Dockerfiles

| Feature                      | root Version         | UID/GID Remapped Version   |
|--||--|
| Runs as root                | Yes                | No (non-root user)       |
| Bitbake compatible          | No                 | Yes                      |
| Works with mounted volumes  | Permission issues  | Matches host UID/GID     |
| Suitable for development    | Limited/testing   | Recommended setup         |



## Build the Docker Image

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
cd [yocto folder]
source oe-init-build-env
```



## Fixing "Cannot write to build" Error

If you see:
```
Error: Cannot write to [yocto folder]
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



