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

| Feature                      | Root Version           | UID/GID Remapped Version   |
|-----------------------------|------------------------|-----------------------------|
| Runs as root                | Yes                    | No (runs as non-root user)  |
| Bitbake compatible          | No                     | Yes                         |
| Works with mounted volumes  | Permission issues       | Matches host UID/GID        |
| Suitable for development    | Limited / Testing Only | Recommended setup           |


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

<br>
<br>

## Troubleshooting the Setup issues to Run QEMU with Yocto in Docker

### Step 1: Fix Yocto directory permissions (on host)

```bash
YOCTO_DIR="[path to repo]"

sudo chown -R $USER:$USER "$YOCTO_DIR"
```



### Step 2: Start Docker container with full access

```bash
docker run -it \
  --privileged \
  --network=host \
  --device /dev/kvm \
  --device /dev/net/tun \
  -v /dev:/dev \
  -v /lib/modules:/lib/modules \
  -v "$YOCTO_DIR":"$YOCTO_DIR" \
  ubuntu22_yocto_dev:latest
```



### 📦 Step 3: Install required packages inside Docker

```bash
apt update && apt install -y \
  iproute2 net-tools bridge-utils sudo \
  qemu-system-x86 xterm x11-xserver-utils
```



### Step 4: Inside Docker — check and fix TAP

```bash
# Optional safety if tap0 doesn't exist
[path to repo]/poky/scripts/runqemu-gen-tapdevs

# Assign IPv4 address to tap0
sudo ip addr add 192.168.7.1/24 dev tap0 || true
sudo ip link set tap0 up
```



### Step 5: Source the Yocto environment

```bash
cd [path to repo]/poky
source oe-init-build-env
```



### Step 6: Build your image (if not already built)

```bash
bitbake core-image-sato
```

Or another image like:

```bash
bitbake core-image-minimal
```



### Step 7: Launch QEMU with SSH or serial support

#### Option A: With graphical desktop + SSH

Make sure your image has `openssh` and root access. Then run:

```bash
runqemu qemux86-64
```

Login as:
```bash
ssh root@192.168.7.2
```

#### Option B: With serial (headless terminal access)

```bash
runqemu qemux86-64 serial
```

This gives you full terminal access directly to the guest, no SSH needed.


### Verification checklist

| Step                     | Check                       |
|--|--|
| `tap0` is UP             | `ip a | grep tap0`          |
| QEMU boots               | SDL or terminal opens       |
| Guest IP is reachable    | `ping 192.168.7.2`          |
| SSH works                | `ssh root@192.168.7.2`      |
| Serial works             | `runqemu ... serial`        |



## Notes

- **Don't run BitBake as root** — always use the non-root setup for real builds.
- Make sure `poky/` and any layers you mount are writable by the container user.
- You can also build Yocto in a container-local directory:
  ```bash
  source oe-init-build-env ~/my-build
  ```
