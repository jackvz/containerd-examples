# [containerd](https://containerd.io/) Examples with [Visual Studio Code](https://code.visualstudio.com/)

Platform-agnostic System Container Orchestration with the [Open Container Initiative (OCI)](https://opencontainers.org/). Like [Docker](https://www.docker.com/), but good because it's open and runs everywhere.

## Requirements

[Get and install Visual Studio Code](https://code.visualstudio.com/download) (or [Code OSS](https://github.com/microsoft/vscode)), available for Windows, macOS, Linux and FreeBSD, and the [VSCode EditorConfig extension](https://marketplace.visualstudio.com/items?itemName=EditorConfig.EditorConfig).

- Get [containerd](https://github.com/containerd/containerd) and install [nerdctl](https://github.com/containerd/nerdctl), available for Windows, macOS, Linux and FreeBSD.

[Here is a `containerd` and `nerdctl` setup guide specifically for FreeBSD 13.](https://github.com/jackvz/containerd-examples#freebsd-13-containerd-and-nerdctl-setup-guide)

## The Examples

<!---
- [MkDocs Example](./mkdocs/readme.md)
--->
- [Node.js Example](./node/readme.md)
- [Python Example](./python/readme.md)
<!---
- [GitLab Example](./gitlab/readme.md)
- [VSCode Example](./vscode/readme.md)
- [Sentry Example](./sentry/readme.md)
- [NestJS Example](./nest/readme.md)
- [HashiCorp Examples](./hashicorp/readme.md)
--->

## FreeBSD 13 `containerd` and `nerdctl` Setup Guide

### Overview

Based on the article [Running FreeBSD jails with containerd 1.5](https://samuel.karp.dev/blog/2021/05/running-freebsd-jails-with-containerd-1-5/).

### Build and Install

Create a `Projects` folder, install [Go](https://go.dev/), and build and install [containerd](https://github.com/containerd/containerd), [runj](https://github.com/samuelkarp/runj), [umoci](https://github.com/opencontainers/umoci) and [nerdctl](https://github.com/containerd/nerdctl.git):

```sh
mkdir ~/Projects && cd ~/Projects

sudo pkg install -y go

git clone https://github.com/jackvz/containerd
cd containerd
git checkout v1.5.0
go build ./cmd/containerd
go build ./cmd/ctr
sudo install -o 0 -g 0 containerd ctr /usr/local/bin
cd ..

git clone https://github.com/jackvz/runj
cd runj
make && sudo make install
cd ..

git clone https://github.com/jackvz/umoci
cd umoci
go build ./cmd/umoci
sudo install -o 0 -g 0 umoci /usr/local/bin
cd ..

git clone https://github.com/jackvz/nerdctl
cd nerdctl
go build ./cmd/nerdctl
sudo install -o 0 -g 0 nerdctl /usr/local/bin
cd ..
```

Note: [BuildKit](https://github.com/moby/buildkit), the Dockerfile-agnostic builder toolkit, is not yet ported to FreeBSD, but [`umoci`](https://github.com/opencontainers/umoci) can be used to modify and repack container images.

### Configure

Set up [ZFS](https://docs.freebsd.org/en/books/handbook/zfs/) on a disk partition, and configure [containerd](https://github.com/containerd/containerd) and [nerdctl](https://github.com/containerd/nerdctl.git):

```sh
sudo zpool create -f zroot /dev/[disk-partition-eg-da0s3]
sudo zfs create -o mountpoint=/var/lib/containerd/io.containerd.snapshotter.v1.zfs zroot/containerd
sudo df
sudo ls -al /var/lib/containerd/

sudo cp /etc/containerd/config.toml /etc/containerd/config.toml.bak
sudo sh -c 'echo "" > /etc/containerd/config.toml'
sudo sh -c 'echo "version = 2" >> /etc/containerd/config.toml'
sudo sh -c 'echo "[plugins]" >> /etc/containerd/config.toml'
sudo sh -c 'echo "[plugins.\"io.containerd.snapshotter.v1.zfs\"]" >> /etc/containerd/config.toml'
sudo sh -c 'echo "root_path = \"/var/lib/containerd/io.containerd.snapshotter.v1.zfs\"" >> /etc/containerd/config.toml'
sudo cat /etc/containerd/config.toml

sudo mkdir -p /etc/nerdctl
sudo sh -c 'echo "" > /etc/nerdctl/nerdctl.toml'
sudo sh -c 'echo "debug = true" >> /etc/nerdctl/nerdctl.toml'
sudo sh -c 'echo "debug_full = true" >> /etc/nerdctl/nerdctl.toml'
sudo sh -c 'echo "address = \"unix:////run/containerd/containerd.sock\"" >> /etc/nerdctl/nerdctl.toml'
sudo sh -c 'echo "#namespace = \"k8s.io\"" >> /etc/nerdctl/nerdctl.toml'
sudo sh -c 'echo "snapshotter = \"zfs\"" >> /etc/nerdctl/nerdctl.toml'
sudo sh -c 'echo "#cgroup_manager = \"cgroupfs\"" >> /etc/nerdctl/nerdctl.toml'
sudo sh -c 'echo "#hosts_dir = [\"/etc/containerd/certs.d\", \"/etc/docker/certs.d\"]" >> /etc/nerdctl/nerdctl.toml'
sudo cat /etc/nerdctl/nerdctl.toml
```

### Run

For now, run `containerd` in the background. In production, run it as a daemon. @todo: [Practical rc.d scripting in BSD](https://docs.freebsd.org/en/articles/rc-scripting/index.html)

```sh
sudo containerd > containerd.log 2>&1 &
export CONTAINERD_PID=$!
```

When done, remember to stop the `containerd` process:

```sh
kill $CONTAINERD_PID
export CONTAINERD_PID=
```

### Container Orchestration

Pull images from any OCI or Docker-compatible container registry.

```sh
sudo ctr image pull --snapshotter zfs public.ecr.aws/samuelkarp/freebsd:12.1-RELEASE
```

[Run a container instance with ctr](https://github.com/containerd/containerd/blob/main/docs/getting-started.md#interacting-with-containerd-via-cli):

```sh
sudo ctr run \
  --snapshotter zfs \
  --runtime wtf.sbk.runj.v1 \
  --rm \
  public.ecr.aws/samuelkarp/freebsd:12.1-RELEASE \
  my-container-id \
  sh -c 'echo "Hello from the container!"'
```

[Run a container instance with nerdctl](https://github.com/containerd/containerd/blob/main/docs/getting-started.md#interacting-with-containerd-via-cli):

```sh
sudo nerdctl run --net none -it \
  --snapshotter zfs \
  --runtime wtf.sbk.runj.v1 \
  --rm \
  public.ecr.aws/samuelkarp/freebsd:12.1-RELEASE \
  sh -c 'echo "Hello from the orchestrated container!"'
```

List images and containers:

```
nerdctl images
nerdctl ps
```

Push a container image to [DockerHub](https://hub.docker.com/):

```sh
nerdctl login registry-1.docker.io
nerdctl tag public.ecr.aws/samuelkarp/freebsd:12.1-RELEASE jackvanzyl1/freebsd:12.1-RELEASE
nerdctl image push jackvanzyl1/freebsd:12.1-RELEASE
```
