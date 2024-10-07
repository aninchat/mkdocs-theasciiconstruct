---
date:
  created: 2022-01-24
  updated: 2022-01-24
authors:
    - aninchat
blog_toc: true
categories:
  - juniper
  - junos
  - containerlab
tags:
  - juniper
  - junos
  - containerlab
---
# Juniper vQFX and Containerlab
In this post, we look at how Containerlab can be used to quickly spin up vQFX topologies for network validation and testing. We'll walk through the entire process - how to build docker images from vQFX images, what happens behind the scenes when bringing these containers up and how to build/verify your topology.

## What is Containerlab?

Containerlab is an open source, network validation/testing platform that allows you to easily spin up network labs and manage end to end lab life cycle. It supports several network operating systems, across many vendors. As of this writing, it supports Nokia SR Linux, Juniper cRPD, Cumulus VX, Arista cEOS. Outside of these natively containerized operating systems, it also supports VM based devcies like Junipers vQFX and vMX, Nexus 9000v, IOS XRv, Arista vEOS and so on.

More information can be found on their homepage - https://containerlab.srlinux.dev/

<!-- more -->

## Building the vQFX docker container
### Downloading vrnetlab

To build VM based containers, you need to download a fork of vrnetlab that was modified for this - https://github.com/hellt/vrnetlab. It essentially packages the VM inside a container. Simply gitclone the repo to your working directory (clear instructions are provided [here](https://containerlab.srlinux.dev/manual/vrnetlab/#building-vrnetlab-images)).

```
root@aninchat-ubuntu:~# git clone https://github.com/hellt/vrnetlab .
Cloning into '.'...
remote: Enumerating objects: 3442, done.
remote: Counting objects: 100% (312/312), done.
remote: Compressing objects: 100% (205/205), done.
remote: Total 3442 (delta 158), reused 241 (delta 107), pack-reused 3130
Receiving objects: 100% (3442/3442), 1.81 MiB | 2.15 MiB/s, done.
Resolving deltas: 100% (2090/2090), done.
```

Within this folder, you have various vendor VM options and instructions on how to build containers for these, along with relevant code files to do this.

```
root@aninchat-ubuntu:~/vrnetlab# ls -l
total 128
-rw-r--r-- 1 root root   94 Feb  5 15:41 CODE_OF_CONDUCT.md
-rw-r--r-- 1 root root  706 Feb  5 15:41 CONTRIBUTING.md
-rw-r--r-- 1 root root 1109 Feb  5 15:41 LICENSE
-rw-r--r-- 1 root root  324 Feb  5 15:41 Makefile
-rw-r--r-- 1 root root 3973 Feb  5 15:41 README.md
drwxr-xr-x 2 root root 4096 Feb  5 15:41 ci-builder-image
drwxr-xr-x 2 root root 4096 Feb  5 15:41 common
drwxr-xr-x 3 root root 4096 Feb  5 15:41 config-engine-lite
drwxr-xr-x 3 root root 4096 Feb  5 15:41 csr
drwxr-xr-x 3 root root 4096 Feb  5 15:41 ftosv
-rwxr-xr-x 1 root root 5210 Feb  5 15:41 git-lfs-repo.sh
-rw-r--r-- 1 root root 3158 Feb  5 15:41 makefile-install.include
-rw-r--r-- 1 root root  370 Feb  5 15:41 makefile-sanity.include
-rw-r--r-- 1 root root 1898 Feb  5 15:41 makefile.include
drwxr-xr-x 3 root root 4096 Feb  5 15:41 n9kv
drwxr-xr-x 3 root root 4096 Feb  5 15:41 nxos
drwxr-xr-x 3 root root 4096 Feb  5 15:41 openwrt
drwxr-xr-x 3 root root 4096 Feb  5 15:41 pan
drwxr-xr-x 3 root root 4096 Feb  5 15:41 routeros
drwxr-xr-x 3 root root 4096 Feb  5 15:41 sros
drwxr-xr-x 2 root root 4096 Feb  5 15:41 topology-machine
drwxr-xr-x 3 root root 4096 Feb  5 15:41 veos
drwxr-xr-x 3 root root 4096 Feb  5 15:41 vmx
drwxr-xr-x 3 root root 4096 Feb  5 15:47 vqfx
drwxr-xr-x 3 root root 4096 Feb  5 15:41 vr-bgp
drwxr-xr-x 2 root root 4096 Feb  5 15:41 vr-xcon
-rw-r--r-- 1 root root 1135 Feb  5 15:41 vrnetlab.sh
drwxr-xr-x 3 root root 4096 Feb  5 15:41 vrp
drwxr-xr-x 3 root root 4096 Feb  5 15:41 vsr1000
drwxr-xr-x 3 root root 4096 Feb  5 15:41 xrv
drwxr-xr-x 3 root root 4096 Feb  5 15:41 xrv9k
```

### Inspecting the Makefile

For the purposes of this post, we'll be focused on the vQFX, so let's look what's inside there.

```
root@aninchat-ubuntu:~/vrnetlab/vqfx# ls -l
total 12
-rw-r--r-- 1 root root 2045 Feb  6 07:29 Makefile
-rw-r--r-- 1 root root  161 Feb  6 07:29 README.md
drwxr-xr-x 2 root root 4096 Feb  6 07:29 docker
```

There's a Makefile, a readme, and a 'docker' folder that contains the Dockerfile and a launch python file.

```
root@aninchat-ubuntu:~/vrnetlab/vqfx/docker# ls -l
total 16
-rw-r--r-- 1 root root  638 Feb  6 07:29 Dockerfile
-rwxr-xr-x 1 root root 8400 Feb  6 07:29 launch.py
```

Before we do anything, let's confirm that no docker images are created for any VM. The only thing present is the base Ubuntu 20.04 image.

```
root@aninchat-ubuntu:~/clabs/juniper# docker images
REPOSITORY   TAG       IMAGE ID       CREATED      SIZE
ubuntu       20.04     54c9d81cbb44   3 days ago   72.8MB
```

The Makefile for vQFX contains the following:

```
root@aninchat-ubuntu:~/vrnetlab/vqfx# cat Makefile
VENDOR=Juniper
NAME=vQFX
IMAGE_FORMAT=qcow2
IMAGE_GLOB=*.qcow2

# New vqfx are named: vqfx-19.4R1.10-re-qemu.qcow2
VERSION=$(shell echo $(IMAGE) |  sed -e 's/^vqfx-//'|sed -e 's/-re-qemu.qcow2//' )
# Old QFX are named: vqfx10k-re-15.1X53-D60.vmdk, and vqfx10k-re-18.4R1.8.vmdk and vqfx10k-pfe-18.4R1.8.vmdk so we need that version number extracted to convert them.
VMDK_VERSION:=$(shell ls *-re-*.vmdk | sed -re 's/vqfx10k-re-([^;]*)\.vmdk.*$$/\1/')

PFE_BASE_VERSION=$(shell echo $VERSION | sed -e s'/.*//')
PFE_IMAGE=$(shell ls vqfx-$(PFE_BASE_VERSION)*-pfe-qemu.qcow*)

-include ../makefile-sanity.include
-include ../makefile.include

format-legacy-images:
	@if ls *.vmdk; then echo "VMDKs exist, converting them to qcow format"; qemu-img convert -f vmdk -O qcow2 *-re-*.vmdk vqfx-$(VMDK_VERSION)-re-qemu.qcow2 && qemu-img convert -f vmdk -O qcow *-pfe-*.vmdk vqfx-$(VMDK_VERSION)-pfe-qemu.qcow; echo "VMDKs have been converted"; fi

# TODO: we should make sure we only copy one PFE image (the latest?), in case there are many
docker-pre-build:

	echo "pfe     $(PFE_IMAGE)"
	cp vqfx*-pfe*.qcow* docker/
	echo "image   $(IMAGE)"
	echo "version $(VERSION)"

# mostly copied from makefile.include, i wish this was easier to change
docker-build-common: docker-clean-build docker-pre-build
	@if [ -z "$$IMAGE" ]; then echo "ERROR: No IMAGE specified"; exit 1; fi
	@if [ "$(IMAGE)" = "$(VERSION)" ]; then echo "ERROR: Incorrect version string ($(IMAGE)). The regexp for extracting version information is likely incorrect, check the regexp in the Makefile or open an issue at https://github.com/plajjan/vrnetlab/issues/new including the image file name you are using."; exit 1; fi
	@echo "Building docker image using $(IMAGE) as $(REGISTRY)vr-$(VR_NAME):$(VERSION)"
	cp ../common/* docker/
	$(MAKE) IMAGE=$$IMAGE docker-build-image-copy
	(cd docker; docker build --build-arg http_proxy=$(http_proxy) --build-arg https_proxy=$(https_proxy) --build-arg RE_IMAGE=$(IMAGE) --build-arg PFE_IMAGE=$(PFE_IMAGE) -t $(REGISTRY)vr-$(VR_NAME):$(VERSION) .)
```

The Makefile is essentially looking for your vQFX images (there needs to be two - one PFE and one RE image). This version of Containerlab also has support for qcow/qcow2 based file systems (some older versions only supported vmdk files). As part of the Makefile, we're also copying over files from vrnetlab/common/ to the 'Docker' folder under vQFX. These files are the healthcheck and vrnetlab python files:

```
root@aninchat-ubuntu:~/vrnetlab/vqfx# ls -l ../common/
total 32
-rwxr-xr-x 1 root root   339 Feb  5 15:41 healthcheck.py
-rw-r--r-- 1 root root 24925 Feb  5 15:41 vrnetlab.py
```

Then finally, the Makefile calls docker build to build the image itself. 

### Start the vQFX docker image build

First, copy over the RE and PFE images to the vqfx folder under vrnetlab:

```
root@aninchat-ubuntu:~/vrnetlab/vqfx# ls -l
total 1965464
-rw-r--r-- 1 root root      2045 Feb  3 06:27 Makefile
-rw-r--r-- 1 root root       161 Feb  3 06:27 README.md
drwxr-xr-x 2 root root      4096 Feb  3 16:34 docker
-rw-r--r-- 1 root root 762839040 Feb  3 08:06 vqfx-20.2R1-2019010209-pfe-qemu.qcow
-rw-r--r-- 1 root root 675020800 Feb  3 08:08 vqfx-20.2R1.10-re-qemu.qcow2
```

vQFX images (upto a certain version) can be downloaded for free from Juniper's software downloads page. Once you have both images in there, you can run 'make' to trigger the docker image build process. 'make' may not be installed by default, so if it is not, simply run 'apt-get install make' to download it. A snippet of what it does below:

```
root@aninchat-ubuntu:~/vrnetlab/vqfx# make
ls: cannot access '*-re-*.vmdk': No such file or directory
Makefile:23: warning: overriding recipe for target 'docker-pre-build'
../makefile.include:18: warning: ignoring old recipe for target 'docker-pre-build'
Makefile:30: warning: overriding recipe for target 'docker-build-common'
../makefile.include:24: warning: ignoring old recipe for target 'docker-build-common'
for IMAGE in vqfx-20.2R1.10-re-qemu.qcow2 vqfx-21.4R1.12-re-qemu.qcow2; do \
	echo "Making $IMAGE"; \
	make IMAGE=$IMAGE docker-build; \
done
Making vqfx-20.2R1.10-re-qemu.qcow2
make[1]: Entering directory '/root/vrnetlab/vqfx'
ls: cannot access '*-re-*.vmdk': No such file or directory
Makefile:23: warning: overriding recipe for target 'docker-pre-build'
../makefile.include:18: warning: ignoring old recipe for target 'docker-pre-build'
Makefile:30: warning: overriding recipe for target 'docker-build-common'
../makefile.include:24: warning: ignoring old recipe for target 'docker-build-common'
rm -f docker/*.qcow2* docker/*.tgz* docker/*.vmdk* docker/*.iso
echo "pfe     vqfx-20.2R1-2019010209-pfe-qemu.qcow"
pfe     vqfx-20.2R1-2019010209-pfe-qemu.qcow
cp vqfx*-pfe*.qcow* docker/

*snip*

Step 5/16 : ARG RE_IMAGE
 ---> Running in ff86eca5251c
Removing intermediate container ff86eca5251c
 ---> 5e47c7b532d7
Step 6/16 : ARG PFE_IMAGE
 ---> Running in 9507880e1206
Removing intermediate container 9507880e1206
 ---> 7fc77a85c443
Step 7/16 : COPY $RE_IMAGE /
 ---> aca4ac1a8b0a
Step 8/16 : COPY $PFE_IMAGE /
 ---> 7797dfaf0bc4
Step 9/16 : RUN echo $RE_IMAGE > /re_image
 ---> Running in fcf1a20d1000
Removing intermediate container fcf1a20d1000
 ---> 7b64b2529b14
Step 10/16 : RUN echo $PFE_IMAGE > /pfe_image
 ---> Running in ae939bea46fc
Removing intermediate container ae939bea46fc
 ---> 03e28ae683ca
Step 11/16 : COPY healthcheck.py /
 ---> 4305e6c8e60e
Step 12/16 : COPY vrnetlab.py /
 ---> a0ada6fb5780
Step 13/16 : COPY launch.py /
 ---> 5dd5b9427cb8
Step 14/16 : EXPOSE 22 161/udp 830 5000 10000-10099
 ---> Running in f19a64ee2141
Removing intermediate container f19a64ee2141
 ---> 2d9f70303f5b
Step 15/16 : HEALTHCHECK CMD ["/healthcheck.py"]
 ---> Running in 280b13b45453
Removing intermediate container 280b13b45453
 ---> 71e3c515a0d9
Step 16/16 : ENTRYPOINT ["/launch.py"]
 ---> Running in 6e6c979ccab4
Removing intermediate container 6e6c979ccab4
 ---> 3e9c3e64af46
Successfully built 3e9c3e64af46
Successfully tagged vrnetlab/vr-vqfx:20.2R1.10
make[1]: Leaving directory '/root/vrnetlab/vqfx'
```

At the end of this, the 'docker' folder inside /vrnetlab/vqfx should have been populated with several python scripts and your RE/PFE images:

```
root@aninchat-ubuntu:~/vrnetlab/vqfx/docker# ls -l
total 1306296
-rw-r--r-- 1 root root       638 Feb  3 06:27 Dockerfile
-rwxr-xr-x 1 root root       339 Feb  5 03:14 healthcheck.py
-rwxr-xr-x 1 root root      8395 Feb  3 16:34 launch.py
-rw-r--r-- 1 root root 762839040 Feb  5 03:14 vqfx-20.2R1-2019010209-pfe-qemu.qcow
-rw-r--r-- 1 root root 574750720 Feb  5 03:14 vqfx-21.4R1.12-re-qemu.qcow2
-rw-r--r-- 1 root root     24925 Feb  5 03:14 vrnetlab.py
```

A new docker image should also be available for vQFX. This is under the vrnetlab/vr-vqfx repo and is tagged appropriately with the vQFX version.

```
root@aninchat-ubuntu:~/vrnetlab/vqfx# docker images
REPOSITORY         TAG         IMAGE ID       CREATED         SIZE
vrnetlab/vr-vqfx   20.2R1.10   3e9c3e64af46   4 minutes ago   1.84GB
ubuntu             20.04       54c9d81cbb44   3 days ago      72.8MB
```

!!! note
        Make sure to stay on the newer versions of containerlab because some of the older versions do not support building a docker image from qcow files (the makefile does not have code for this).

### What's happening behind the scenes

So, let's break down what's actually happening. Like a Makefile, there is a Dockerfile that contains instructions on how to build the docker image itself (for vQFX, in this case). When Makefile calls 'docker build', it looks for a Dockerfile in that directory.

The Dockerfile is available inside the 'docker' folder. This is what it contains for vQFX:

```
root@aninchat-ubuntu:~/clabs/juniper# cat ~/vrnetlab/vqfx/docker/Dockerfile
FROM ubuntu:20.04
MAINTAINER Kristian Larsson <kristian@spritelink.net>

ENV DEBIAN_FRONTEND=noninteractive

RUN apt-get update -qy \
 && apt-get upgrade -qy \
 && apt-get install -y \
    bridge-utils \
    iproute2 \
    python3-ipy \
    socat \
    qemu-kvm \
    procps \
    tcpdump \
    openvswitch-switch \
 && rm -rf /var/lib/apt/lists/*

ARG RE_IMAGE
ARG PFE_IMAGE

COPY $RE_IMAGE /
COPY $PFE_IMAGE /

RUN echo $RE_IMAGE > /re_image
RUN echo $PFE_IMAGE > /pfe_image

COPY healthcheck.py /
COPY vrnetlab.py /
COPY launch.py /

EXPOSE 22 161/udp 830 5000 10000-10099
HEALTHCHECK CMD ["/healthcheck.py"]
ENTRYPOINT ["/launch.py"]
```

Outside of updating and installing several packages, it is copying the PFE and RE images as well as exposing several ports. The entrypoint ('ENTRYPOINT ["/launch.py"]') tells Docker what to run once the container starts. So, in this case, it is telling it to run the launch.py Python script. This script does numerous things, along with bootstrapping configuration for vQFX (we'll look at the logs further down when we actually start our lab).

The Dockerfile also has a custom healthcheck command that points to the healthcheck.py file. With a custom healthcheck, a container can be starting, healthy or unhealthy. The health is determined with 0 or 1. If it is 0, the container is healthy and if it is 1, the container is unhealthy. 

```
import sys

try:
    health_file = open("/health", "r")
    health = health_file.read()
    health_file.close()
except FileNotFoundError:
    print("health status file not found")
    sys.exit(2)

exit_status, message = health.strip().split(" ", 1)

if message != '':
    print(message)

sys.exit(int(exit_status))
```

The actual health check is performed in the vrnetlab.py file - it essentially checks the health of the qemu (if the VM is running successfull or not) and writes into the /health folder (it will write the exit status as 0 or 1), which the healthcheck.py file periodically checks to determine container status.

## Building and deploying a Containerlab lab with vQFX
### Describing the topology for Containerlab

Containerlab needs a description of what the topology looks and this is very easily done using a topology file, written in yaml. An example of a simple, two leaf two spine topology:

```
root@aninchat-ubuntu:~/clabs/juniper# cat 2L2S_vqfx.yml
name: 2L2S_vqfx
topology:
  nodes:
    spine1:
            kind: vr-vqfx
            image: vrnetlab/vr-vqfx:20.2R1.10
    spine2:
            kind: vr-vqfx
            image: vrnetlab/vr-vqfx:20.2R1.10
    leaf1:
            kind: vr-vqfx
            image: vrnetlab/vr-vqfx:20.2R1.10
    leaf2:
            kind: vr-vqfx
            image: vrnetlab/vr-vqfx:20.2R1.10
  links:
          - endpoints: ["leaf1:eth1", "spine1:eth1"]
          - endpoints: ["leaf2:eth1", "spine1:eth2"]
          - endpoints: ["leaf1:eth2", "spine2:eth1"]
          - endpoints: ["leaf2:eth2", "spine2:eth2"]
```

In its simplest form, you're telling Containerlab the name of your devices, the 'kind' for each device (vr-vqfx is the kind for vQFX) and the docker image it needs to use to build this lab.

Finally, you declare the device interconnections. You can boot up a lab without any links, but realistically, you'd want to have some connections. As you can see, all of this is written in yaml, which is super easy to consume and write. 

For vQFX, eth0 is the management interface, and shouldn't be used. eth1 corresponds to xe-0/0/0, eth2 corresponds to xe-0/0/1 and so on.

### Starting a lab using Containerlab

A lab can be started using 'containerlab deploy'. This command expects a topology as a reference, which is the .yml file we created earlier. Let's look at what happens when we start a lab, and more closely inspect a node bringup.

```
root@aninchat-ubuntu:~/clabs/juniper# containerlab deploy --topo 2L2S_vqfx.yml
INFO[0000] Containerlab v0.23.0 started
INFO[0000] Parsing & checking topology file: 2L2S_vqfx.yml
INFO[0000] Creating lab directory: /root/clabs/juniper/clab-2L2S_vqfx
INFO[0000] Creating docker network: Name='clab', IPv4Subnet='172.20.20.0/24', IPv6Subnet='2001:172:20:20::/64', MTU='1500'
INFO[0000] Creating container: leaf1
INFO[0000] Creating container: spine2
INFO[0000] Creating container: leaf2
INFO[0000] Creating container: spine1
INFO[0000] Creating virtual wire: leaf2:eth1 <--> spine1:eth2
INFO[0000] Creating virtual wire: leaf2:eth2 <--> spine2:eth2
INFO[0001] Creating virtual wire: leaf1:eth1 <--> spine1:eth1
INFO[0001] Creating virtual wire: leaf1:eth2 <--> spine2:eth1
INFO[0001] Adding containerlab host entries to /etc/hosts file
+---+-----------------------+--------------+----------------------------+---------+---------+----------------+----------------------+
| # |         Name          | Container ID |           Image            |  Kind   |  State  |  IPv4 Address  |     IPv6 Address     |
+---+-----------------------+--------------+----------------------------+---------+---------+----------------+----------------------+
| 1 | clab-2L2S_vqfx-leaf1  | 12f805d46bae | vrnetlab/vr-vqfx:20.2R1.10 | vr-vqfx | running | 172.20.20.5/24 | 2001:172:20:20::5/64 |
| 2 | clab-2L2S_vqfx-leaf2  | 0b8bfb802838 | vrnetlab/vr-vqfx:20.2R1.10 | vr-vqfx | running | 172.20.20.4/24 | 2001:172:20:20::4/64 |
| 3 | clab-2L2S_vqfx-spine1 | 99ed06e26513 | vrnetlab/vr-vqfx:20.2R1.10 | vr-vqfx | running | 172.20.20.3/24 | 2001:172:20:20::3/64 |
| 4 | clab-2L2S_vqfx-spine2 | 97edeb106888 | vrnetlab/vr-vqfx:20.2R1.10 | vr-vqfx | running | 172.20.20.2/24 | 2001:172:20:20::2/64 |
+---+-----------------------+--------------+----------------------------+---------+---------+----------------+----------------------+
```

Containerlab spins up all nodes in the topology as containers, along with the declared virtual wiring. You can look at a node bringup using 'docker logs' or watch it live using 'docker logs -f'. Let's look at the bringup for one of the nodes.

```
root@aninchat-ubuntu:~/clabs/juniper# docker logs clab-2L2S_vqfx-leaf1
2022-02-06 02:34:51,977: vrnetlab   DEBUG    Creating overlay disk image
2022-02-06 02:34:51,995: vrnetlab   DEBUG    Creating overlay disk image
2022-02-06 02:34:52,116: vrnetlab   DEBUG    Starting vrnetlab VQFX
2022-02-06 02:34:52,116: vrnetlab   DEBUG    VMs: [<__main__.VQFX_vcp object at 0x7fbbc91342e0>, <__main__.VQFX_vpfe object at 0x7fbbc90b7670>]
2022-02-06 02:34:52,122: vrnetlab   DEBUG    VM not started; starting!
2022-02-06 02:34:52,122: vrnetlab   INFO     Starting VQFX_vcp

*snip* 

2022-02-05 14:48:21,281: launch     INFO     matched login prompt
2022-02-05 14:48:21,282: launch     DEBUG    writing to serial console: root
2022-02-05 14:48:21,282: launch     TRACE    Waiting for Password:
2022-02-05 14:48:49,224: launch     TRACE    Read:  root
Password:
2022-02-05 14:48:49,224: launch     DEBUG    writing to serial console: Juniper
2022-02-05 14:48:52,228: launch     TRACE    OUTPUT VCP:

--- JUNOS 19.4R1.10 built 2019-12-19 03:54:05 UTC
aunch     DEBUG    writing to serial console: cli
2022-02-05 14:49:00,585: launch     TRACE    Waiting for >
2022-02-05 14:49:00,975: launch     TRACE    Read:  cli
{master:0}
root@vqfx-re>
2022-02-05 14:49:00,976: launch     DEBUG    writing to serial console: configure
2022-02-05 14:49:00,976: launch     TRACE    Waiting for #
2022-02-05 14:49:01,046: launch     TRACE    Read:  configure
Entering configuration mode

{master:0}[edit]
root@vqfx-re#
2022-02-05 14:49:01,046: launch     DEBUG    writing to serial console: set system services ssh
2022-02-05 14:49:01,046: launch     TRACE    Waiting for #
2022-02-05 14:49:01,090: launch     TRACE    Read:  set system services ssh

{master:0}[edit]
root@vqfx-re#
2022-02-05 14:49:01,091: launch     DEBUG    writing to serial console: set system services netconf ssh
2022-02-05 14:49:01,091: launch     TRACE    Waiting for #
2022-02-05 14:49:01,143: launch     TRACE    Read:  set system services netconf ssh

{master:0}[edit]
root@vqfx-re#
2022-02-05 14:49:01,143: launch     DEBUG    writing to serial console: set system services netconf rfc-compliant
2022-02-05 14:49:01,143: launch     TRACE    Waiting for #
2022-02-05 14:49:01,201: launch     TRACE    Read:  set system services netconf rfc-compliant

{master:0}[edit]
root@vqfx-re#
2022-02-05 14:49:01,201: launch     DEBUG    writing to serial console: delete system login user vagrant
2022-02-05 14:49:01,201: launch     TRACE    Waiting for #
2022-02-05 14:49:01,260: launch     TRACE    Read:  delete system login user vagrant

{master:0}[edit]
root@vqfx-re#
2022-02-05 14:49:01,260: launch     DEBUG    writing to serial console: set system login user admin class super-user authentication plain-text-password
2022-02-05 14:49:01,260: launch     TRACE    Waiting for New password:
root@vqfx-re# ...dmin class super-user authentication plain-text-password    in class super-user authentication
New password:
2022-02-05 14:49:04,250: launch     DEBUG    writing to serial console: admin@123
2022-02-05 14:49:04,250: launch     TRACE    Waiting for Retype new password:
2022-02-05 14:49:04,263: launch     TRACE    Read:
Retype new password:
2022-02-05 14:49:04,263: launch     DEBUG    writing to serial console: admin@123
2022-02-05 14:49:04,263: launch     TRACE    Waiting for #
2022-02-05 14:49:04,302: launch     TRACE    Read:

{master:0}[edit]
root@vqfx-re#
2022-02-05 14:49:04,302: launch     DEBUG    writing to serial console: set system root-authentication plain-text-password
2022-02-05 14:49:04,303: launch     TRACE    Waiting for New password:
2022-02-05 14:49:04,356: launch     TRACE    Read:  set system root-authentication plain-text-password
New password:
2022-02-05 14:49:04,356: launch     DEBUG    writing to serial console: admin@123
2022-02-05 14:49:04,356: launch     TRACE    Waiting for Retype new password:
2022-02-05 14:49:04,369: launch     TRACE    Read:
Retype new password:
2022-02-05 14:49:04,369: launch     DEBUG    writing to serial console: admin@123
2022-02-05 14:49:04,369: launch     TRACE    Waiting for #
2022-02-05 14:49:04,407: launch     TRACE    Read:

{master:0}[edit]
root@vqfx-re#
2022-02-05 14:49:04,408: launch     DEBUG    writing to serial console: delete interfaces
2022-02-05 14:49:04,408: launch     TRACE    Waiting for #
2022-02-05 14:49:04,877: launch     TRACE    Read:  delete interfaces

{master:0}[edit]
root@vqfx-re#
2022-02-05 14:49:04,877: launch     DEBUG    writing to serial console: set interfaces em0 unit 0 family inet address 10.0.0.15/24
2022-02-05 14:49:04,877: launch     TRACE    Waiting for #
2022-02-05 14:49:05,020: launch     TRACE    Read:  set interfaces em0 unit 0 family inet address 10.0.0.15/24

{master:0}[edit]
root@vqfx-re#
2022-02-05 14:49:05,020: launch     DEBUG    writing to serial console: set interfaces em1 unit 0 family inet address 169.254.0.2/24
2022-02-05 14:49:05,020: launch     TRACE    Waiting for #
root@vqfx-re#:49:05,076: launch     TRACE    Read:  set interfaces em1 unit 0 family inet address 169.254.0.2/24
2022-02-05 14:49:05,077: launch     DEBUG    writing to serial console: set system host-name leaf1
2022-02-05 14:49:05,077: launch     TRACE    Waiting for #
2022-02-05 14:49:05,119: launch     TRACE    Read:  ... 0 family inet address 169.254.0.2/24

{master:0}[edit]
root@vqfx-re#
2022-02-05 14:49:05,119: launch     DEBUG    writing to serial console: commit
2022-02-05 14:49:05,119: launch     TRACE    Waiting for #
2022-02-05 14:49:05,163: launch     TRACE    Read:  set system host-name leaf1

{master:0}[edit]
root@vqfx-re#
2022-02-05 14:49:05,163: launch     DEBUG    writing to serial console: exit
2022-02-05 14:49:05,163: launch     INFO     Startup complete in: 0:05:29.249487
```

As you can see, as part of the container bringup, the launch file bootstraps the configuration for it, which includes basic settings such as enabling SSH, setting a hostname and root password, and so on.

Make sure the containers are healthy:

```
root@aninchat-ubuntu:~/clabs/juniper# docker ps
CONTAINER ID   IMAGE                        COMMAND                  CREATED          STATUS                    PORTS                                                 NAMES
b6e669f3eede   vrnetlab/vr-vqfx:20.2R1.10   "/launch.py --userna…"   28 minutes ago   Up 28 minutes (healthy)   22/tcp, 830/tcp, 5000/tcp, 10000-10099/tcp, 161/udp   clab-2L2S_vqfx-spine2
31d62725210d   vrnetlab/vr-vqfx:20.2R1.10   "/launch.py --userna…"   28 minutes ago   Up 28 minutes (healthy)   22/tcp, 830/tcp, 5000/tcp, 10000-10099/tcp, 161/udp   clab-2L2S_vqfx-leaf2
33c01d3fdab8   vrnetlab/vr-vqfx:20.2R1.10   "/launch.py --userna…"   28 minutes ago   Up 28 minutes (healthy)   22/tcp, 830/tcp, 5000/tcp, 10000-10099/tcp, 161/udp   clab-2L2S_vqfx-spine1
ef1e6f245787   vrnetlab/vr-vqfx:20.2R1.10   "/launch.py --userna…"   28 minutes ago   Up 28 minutes (healthy)   22/tcp, 830/tcp, 5000/tcp, 10000-10099/tcp, 161/udp   clab-2L2S_vqfx-leaf1
```

If it remains unhealthy for a long period of time (some of these images can take 5-10 minutes to fully bootup), then most likely your container bringup is stalled somewhere (check docker logs to find out where and possibly why).

## Accessing the lab

Once Containerlab deploys a lab, it also assigns IP addresses for logging into the devices. You can view this any time using 'containerlab inspect --topo <>'

```
root@aninchat-ubuntu:~/clabs/juniper# containerlab inspect --topo 2L2S_vqfx.yml
INFO[0000] Parsing & checking topology file: 2L2S_vqfx.yml
+---+-----------------------+--------------+----------------------------+---------+---------+----------------+----------------------+
| # |         Name          | Container ID |           Image            |  Kind   |  State  |  IPv4 Address  |     IPv6 Address     |
+---+-----------------------+--------------+----------------------------+---------+---------+----------------+----------------------+
| 1 | clab-2L2S_vqfx-leaf1  | ef1e6f245787 | vrnetlab/vr-vqfx:20.2R1.10 | vr-vqfx | running | 172.20.20.4/24 | 2001:172:20:20::4/64 |
| 2 | clab-2L2S_vqfx-leaf2  | 31d62725210d | vrnetlab/vr-vqfx:20.2R1.10 | vr-vqfx | running | 172.20.20.3/24 | 2001:172:20:20::3/64 |
| 3 | clab-2L2S_vqfx-spine1 | 33c01d3fdab8 | vrnetlab/vr-vqfx:20.2R1.10 | vr-vqfx | running | 172.20.20.5/24 | 2001:172:20:20::5/64 |
| 4 | clab-2L2S_vqfx-spine2 | b6e669f3eede | vrnetlab/vr-vqfx:20.2R1.10 | vr-vqfx | running | 172.20.20.2/24 | 2001:172:20:20::2/64 |
+---+-----------------------+--------------+----------------------------+---------+---------+----------------+----------------------+
```

Let's login to leaf1 as an example now:

```
root@aninchat-ubuntu:~/clabs/juniper# ssh -l admin 172.20.20.4
The authenticity of host '172.20.20.4 (172.20.20.4)' can't be established.
ECDSA key fingerprint is SHA256:S5gTisaBW1vp3N2yiMYkQjVzTEbM/DAuFtPNPYrhHno.
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
Warning: Permanently added '172.20.20.4' (ECDSA) to the list of known hosts.
Password:
--- JUNOS 19.4R1.10 built 2019-12-19 03:54:05 UTC
{master:0}
admin@leaf1>
```

The login is admin/admin@123. And you're in! Let's actually configure something to make sure it's all good. Our goal is to enable LLDP between all peers, configure loopbacks on leaf1 and leaf2, configure OSPF in area 0 and advertise the loopbacks into OSPF. As a final check, leaf1 should be able to reach leaf2 via the loopbacks. 

```
admin@leaf1> configure
Entering configuration mode

{master:0}[edit]
admin@leaf1# set protocols lldp interface xe-0/0/0

{master:0}[edit]
admin@leaf1# set protocols lldp interface xe-0/0/1

{master:0}[edit]
admin@leaf1# commit
configuration check succeeds
commit complete
```

We do the same thing on all the other devices as well, and we see our LLDP peerings now:

```
// Leaf1

{master:0}[edit]
admin@leaf1# run show lldp neighbors
Local Interface    Parent Interface    Chassis Id          Port info          System Name
xe-0/0/1           -                   02:05:86:71:37:00   xe-0/0/0           spine2
xe-0/0/0           -                   02:05:86:71:e8:00   xe-0/0/0           spine1

// Leaf2

{master:0}[edit]
admin@leaf2# run show lldp neighbors
Local Interface    Parent Interface    Chassis Id          Port info          System Name
xe-0/0/1           -                   02:05:86:71:37:00   xe-0/0/1           spine2
xe-0/0/0           -                   02:05:86:71:e8:00   xe-0/0/1           spine1
```

Configure the loopbacks on both leaf1 and leaf2 now:

```
// Leaf1

{master:0}[edit]
admin@leaf1# set interfaces lo0 unit 0 family inet address 192.100.100.1/32

// Leaf2

{master:0}[edit]
admin@leaf2# set interfaces lo0 unit 0 family inet address 192.100.100.2/32
```

Configure the p2p connections between the leaf and spines, and enable OSPF for each p2p link, as well as the loopback. At the end of this, we see the expected OSPF neighbors:

```
{master:0}[edit]
admin@leaf1# run show ospf neighbor
Address          Interface              State           ID               Pri  Dead
10.100.100.1     xe-0/0/0.0             Full            11.11.11.11      128    38
10.100.100.3     xe-0/0/1.0             Full            22.22.22.22      128    38

{master:0}[edit]
admin@leaf2# run show ospf neighbor
Address          Interface              State           ID               Pri  Dead
10.100.100.5     xe-0/0/0.0             Full            11.11.11.11      128    36
10.100.100.7     xe-0/0/1.0             Full            22.22.22.22      128    38
```

On each leaf, we see the corresponding loopback as well, and we can confirm reachability by pinging between them:

```
// Leaf1

admin@leaf1# run show route protocol ospf

inet.0: 13 destinations, 14 routes (13 active, 0 holddown, 0 hidden)
+ = Active Route, - = Last Active, * = Both

10.100.100.4/31    *[OSPF/10] 00:04:17, metric 2
                    >  to 10.100.100.1 via xe-0/0/0.0
10.100.100.6/31    *[OSPF/10] 00:02:45, metric 2
                    >  to 10.100.100.3 via xe-0/0/1.0
192.100.100.1/32    [OSPF/10] 00:05:09, metric 0
                    >  via lo0.0
192.100.100.2/32   *[OSPF/10] 00:02:45, metric 2
                    >  to 10.100.100.1 via xe-0/0/0.0
                       to 10.100.100.3 via xe-0/0/1.0
224.0.0.5/32       *[OSPF/10] 00:05:14, metric 1
                       MultiRecv

inet6.0: 2 destinations, 2 routes (2 active, 0 holddown, 0 hidden)

// Leaf2 

admin@leaf2# run show route protocol ospf

inet.0: 13 destinations, 14 routes (13 active, 0 holddown, 0 hidden)
+ = Active Route, - = Last Active, * = Both

10.100.100.0/31    *[OSPF/10] 00:04:51, metric 2
                    >  to 10.100.100.5 via xe-0/0/0.0
10.100.100.2/31    *[OSPF/10] 00:03:25, metric 2
                    >  to 10.100.100.7 via xe-0/0/1.0
192.100.100.1/32   *[OSPF/10] 00:03:25, metric 2
                    >  to 10.100.100.5 via xe-0/0/0.0
                       to 10.100.100.7 via xe-0/0/1.0
192.100.100.2/32    [OSPF/10] 00:05:45, metric 0
                    >  via lo0.0
224.0.0.5/32       *[OSPF/10] 00:05:50, metric 1
                       MultiRecv

inet6.0: 2 destinations, 2 routes (2 active, 0 holddown, 0 hidden)

admin@leaf1# run ping 192.100.100.2 source 192.100.100.1
PING 192.100.100.2 (192.100.100.2): 56 data bytes
64 bytes from 192.100.100.2: icmp_seq=0 ttl=63 time=122.324 ms
64 bytes from 192.100.100.2: icmp_seq=1 ttl=63 time=129.199 ms
64 bytes from 192.100.100.2: icmp_seq=2 ttl=63 time=119.371 ms
64 bytes from 192.100.100.2: icmp_seq=3 ttl=63 time=120.535 ms
64 bytes from 192.100.100.2: icmp_seq=4 ttl=63 time=125.425 ms
^C
--- 192.100.100.2 ping statistics ---
5 packets transmitted, 5 packets received, 0% packet loss
round-trip min/avg/max/stddev = 119.371/123.371/129.199/3.559 ms
```

How awesome is this? vQFX as containers, and a fully functional lab! 

One last thing to add - to destroy the lab, you can simply use the 'containerlab destroy' command.

```
root@aninchat-ubuntu:~/clabs/juniper# containerlab destroy --topo 2L2S_vqfx.yml
INFO[0000] Parsing & checking topology file: 2L2S_vqfx.yml
INFO[0000] Destroying lab: 2L2S_vqfx
INFO[0000] Removed container: clab-2L2S_vqfx-leaf2
INFO[0001] Removed container: clab-2L2S_vqfx-leaf1
INFO[0001] Removed container: clab-2L2S_vqfx-spine1
INFO[0001] Removed container: clab-2L2S_vqfx-spine2
INFO[0001] Removing containerlab host entries from /etc/hosts file
```

I hope this was informative.