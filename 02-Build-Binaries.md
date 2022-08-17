# Build Binaries from Source Code

Since ARMv8 is not officially supported by Kubernetes, the binaries need to be built from the source in order to get them properly to work. They are all written in Go which facilitates the process of cross-compiling. However, I found it even easier using qemu to to run ARM native golang docker images to compile. 

It's not required, but if you want to follow that path, then install `qemu-system-arm`, `qemu-user-static` and `binfmt-support`.

Use docker to build the binaries, although it’s not necessary, it removes the clutter from my local environment, as I don’t normally use Go.

* [arm64v8/golang](https://hub.docker.com/r/arm64v8/golang) for armv8 - Pi 4
* [arm64v8/golang](https://hub.docker.com/r/arm64v8/golang) for armv8 - Pi Zero

## Build Kubernetes Binaries for Master Node

```shell
git clone https://github.com/kubernetes/kubernetes.git
cd kubernetes

docker run --rm -it -v "$PWD":/usr/src/myapp -w /usr/src/myapp arm64v8/golang:1.19.0-bullseye bash

root@f39f79b0536b:/usr/src/myapp# export GOOS="linux"
root@f39f79b0536b:/usr/src/myapp# export GOARCH="arm"
root@f39f79b0536b:/usr/src/myapp# export CGO_ENABLED=0
root@f39f79b0536b:/usr/src/myapp# export GOARM=6
root@f39f79b0536b:/usr/src/myapp# export GOFLAGS=""
root@f39f79b0536b:/usr/src/myapp# export LDFLAGS='-d -s -w -linkmode=external -extldflags="-static" -static -static-libgcc'

root@f39f79b0536b:/usr/src/myapp# apt update
root@f39f79b0536b:/usr/src/myapp# apt install rsync

root@f39f79b0536b:/usr/src/myapp# time make WHAT=cmd/kube-scheduler
real	8m12.605s
user	16m58.260s
sys	2m11.420s

root@f39f79b0536b:/usr/src/myapp# time make WHAT=cmd/kube-controller-manager
real	5m59.615s
user	15m26.843s
sys	1m53.155s

root@f39f79b0536b:/usr/src/myapp# time make WHAT=cmd/kube-apiserver         
real	1m19.790s
user	2m9.285s
sys	0m22.332s


root@f39f79b0536b:/usr/src/myapp# time make WHAT=cmd/kubectl
real	1m4.805s
user	2m26.357s
sys	0m27.512s
```

This process can take a while to complete, be patient. Once it's finished, copy the binaries to master to then transfer to worker nodes.

```shell
scp _output/local/bin/linux/arm64/kube* pi@rpi-k8s-master.hide.lukasmaly.net:~/bin
```

> **Note** - kubectl can be reused for worker nodes as well, it worked for me.

## Build Kubernetes Binaries for Worker Nodes

#Before building kubelet, I found numerous issues with a missing cgroup (cpuset) in the raspberry pi zero. I’m not entirely sure why this is a requirement and I removed it from the code. I published my findings in the [raspberry pi forum](https://www.raspberrypi.org/forums/viewtopic.php?f=66&t=219644#p1348691). 

#To avoid having those issues I removed the validation as follows: 

#```
#File: pkg/kubelet/cm/container_manager_linux.go
#- expectedCgroups := sets.NewString("cpu", "cpuacct", "cpuset", "memory")
#+ expectedCgroups := sets.NewString("cpu", "cpuacct", "memory")
#```

```shell
docker run --rm -it -v "$PWD":/usr/src/myapp -w /usr/src/myapp arm64v8/golang:1.19.0-bullseye bash

root@e3b475b2f53e:/usr/src/myapp# export GOOS="linux"
root@e3b475b2f53e:/usr/src/myapp# export GOARCH="arm"
root@e3b475b2f53e:/usr/src/myapp# export CGO_ENABLED=0
root@e3b475b2f53e:/usr/src/myapp# export GOARM=6
root@e3b475b2f53e:/usr/src/myapp# export GOFLAGS=""
root@e3b475b2f53e:/usr/src/myapp# export LDFLAGS='-d -s -w -linkmode=external -extldflags="-static" -static -static-libgcc'

root@e3b475b2f53e:/usr/src/myapp# apt update
root@e3b475b2f53e:/usr/src/myapp# apt install rsync

root@e3b475b2f53e:/usr/src/myapp# time make WHAT=cmd/kubelet
real	1m50.322s
user	3m22.416s
sys	0m35.184s

root@e3b475b2f53e:/usr/src/myapp# time make WHAT=cmd/kube-proxy
real	0m26.961s
user	0m29.413s
sys	0m10.101s
```

Copy the binaries to master to then transfer to worker nodes

```shell
scp _output/local/bin/linux/arm64/kubelet pi@rpi-k8s-master.hide.lukasmaly.net:~/bin
scp _output/local/bin/linux/arm64/kube-proxy pi@rpi-k8s-master.hide.lukasmaly.net:~/bin
```

## Build Container Networking Plugins

We are not going to use all the networking plugins, but to make our life easier, build them all and distribute them to the nodes anyway. 

```shell
git clone https://github.com/containernetworking/plugins.git 
cd plugins

sed -i 's/\$GO\ build -o \"\${PWD}\/bin\/\$plugin\" \"\$\@\"/\$GO\ build -o \"\$\{PWD\}\/bin\/\$plugin\" \"\$\@\"\ \-ldflags\=\"\-d\ \-s\ \-w\"/' build_linux.sh

docker run --rm -it -v "$PWD":/usr/src/myapp -w /usr/src/myapp arm64v8/golang:1.19.0-bullseye bash

root@042cbfe7a200:/usr/src/myapp# ./build_linux.sh

scp bin/* pi@rpi-k8s-master.hide.lukasmaly.net:~/plugins/
```

## Build Runc

`runc` is the actual low level container runtime to launches the child processes.

```shell
git clone https://github.com/opencontainers/runc.git
cd runc

docker run --rm -it -v "$PWD":/usr/src/myapp -w /usr/src/myapp arm64v8/golang:1.19.0-bullseye bash

root@cda70ed7dfb3:/usr/src/myapp# apt update
root@cda70ed7dfb3:/usr/src/myapp# apt install libseccomp-dev
root@cda70ed7dfb3:/usr/src/myapp# make static
root@cda70ed7dfb3:/usr/src/myapp# ./runc --version
runc version 1.1.0+dev
commit: v1.1.0-262-g480e1298
spec: 1.0.2-dev
go: go1.19
libseccomp: 2.5.1
root@cda70ed7dfb3:/usr/src/myapp# ./contrib/cmd/recvtty/recvtty --version
?

scp ./runc ./contrib/cmd/recvtty/recvtty.go pi@rpi-k8s-master.hide.lukasmaly.net:~/bin/
```

## Build Containerd

`containerd` is the daemon that controls containers at runtime.

```shell
git clone https://github.com/containerd/containerd.git
cd containerd

docker run --rm -it -v "$PWD":/go/src/github.com/containerd/containerd -w /go/src/github.com/containerd/containerd arm64v8/golang:1.19.0-bullseye bash

root@809ff752ed2d:/usr/src/myapp# apt update
root@809ff752ed2d:/usr/src/myapp# apt install btrfs-progs libbtrfs-dev
root@809ff752ed2d:/usr/src/myapp# apt install protobuf-compiler
root@809ff752ed2d:/usr/src/myapp# make

scp bin/* pi@rpi-k8s-master.hide.lukasmaly.net:~/bin/
```

## Build CRI-tools

`cri-tools` provides a communication platform with the current `CRI compliant` container runtime.

```shell
git clone https://github.com/kubernetes-sigs/cri-tools.git
cd cri-tools

docker run --rm -it -v "$PWD":/usr/src/myapp -w /usr/src/myapp arm64v8/golang:1.19.0-bullseye bash

root@da0beb1cf478:/usr/src/myapp# make
root@da0beb1cf478:/usr/src/myapp# ./build/bin/crictl --version
crictl version 1.24.2-35-g77d02a03

scp build/bin/* pi@rpi-k8s-master.hide.lukasmaly.net:~/bin/
```

## Build Etcd 

`etcd` is the main database that maintains the cluster state. 

### Compile the binaries

```shell
git clone https://github.com/etcd-io/etcd.git
cd etcd

docker run --rm -it -v "$PWD":/usr/src/myapp -w /usr/src/myapp arm64v8/golang:1.19.0-bullseye bash

root@3a6d3a16f556:/usr/src/myapp# make
root@3a6d3a16f556:/usr/src/myapp# ls -l bin/
total 42624
-rwxr-xr-x 1 root root 24999558 Nov 26 11:22 etcd
-rwxr-xr-x 1 root root 18641042 Nov 26 11:24 etcdctl
```

### Transfer the binaries to the master

```shell
scp ./bin/* pi@rpi-k8s-master.hide.lukasmaly.net:~/bin
```

### Test the binaries in the master node

The environment variable ETCD_UNSUPPORTED_ARCH=arm needs setting, otherwise we get an explicit error message. Inside master node $HOME directory.

```shell
$ ETCD_UNSUPPORTED_ARCH=arm bin/etcd --version
running etcd on unsupported architecture "arm" since ETCD_UNSUPPORTED_ARCH is set
etcd Version: 3.6.0-alpha.0
Git SHA: ff6b85da8
Go Version: go1.19
Go OS/Arch: linux/arm64

$ bin/etcdctl version
etcdctl version: 3.6.0-alpha.0
API version: 3.6
```

## Build Pause

This step is more for the sake of completeness of building everything from the source, in case you want to create your own version of the `pause` container which is a fundamental pillar of the pod structure.

```shell
cd kubernetes

docker run --rm -it -v "$PWD":/usr/src/myapp -w /usr/src/myapp arm64v8/golang:1.19.0-bullseye bash

root@637ae3b798bc:/usr/src/myapp# cd build/pause/linux
root@637ae3b798bc:/usr/src/myapp/build/pause/linux# gcc -Os -Wall -Werror -static -s -g -fPIE -fPIC -o pause pause.c
```

```shell
scp pause pi@rpi-k8s-master.hide.lukasmaly.net:~/bin
```