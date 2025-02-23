---
date: '2025-02-24T09:49:29Z'
draft: true
title: 'Building a single node rootless container host with containerd'
author: Pascal Van Acker
ShowToc: true
cover:
  image: /single-node-rootless-container-host-with-containerd-banner.png
tags:
- containers
- containerd
- nerdctl
- system administration
---

In this article I will explain how to setup a single node, rootless container host using [containerd](https://github.com/containerd/containerd).
The Docker-compatible command-line interface [nerdctl](https://github.com/containerd/nerdctl) will be used to interact with containerd.
Additionally, we will ensure that our containers do not expose ports worldwide by adding an ip whitelist _(using iptables with ipset)_.

This article is targeted at anyone who wants to set up stand-alone servers using containerd instead of Docker,
anyone that is interested in learning more about containerd as a container runtime and anyone that is just interested in reading about this use-case.
This article is not targeted at people who are just starting out with containers.
If you are just starting out and want to get started quickly,
[Docker](https://www.docker.com/get-started/) or [Podman](https://podman.io/get-started) will do fine.

For this article, I will assume that you have some experience on the command-line and know how to login to a Linux server on the command-line via the console or [SSH](https://en.wikipedia.org/wiki/Secure_Shell).


## Why Not Use Docker or Podman?

To be completely honest, the main reason I chose to use containerd instead of Docker or Podman,
is to learn new things and experiment. This seemed like fun and challenging topic.

That said, there are some reasons why you might want to do this:
* Containerd is the default container runtime for [Kubernetes](https://kubernetes.io)
* Containerd is [OCI compliant](https://opencontainers.org/) and [CRI compliant](https://kubernetes.io/docs/concepts/architecture/cri/)
* Containerd is more lightweight and more [performant](https://medium.com/norma-dev/benchmarking-containerd-vs-dockerd-performance-efficiency-and-scalability-64c9043924b1) than Docker
* You want to develop an application or interface that uses the [CRI specification](https://github.com/kubernetes/cri-api/blob/master/pkg/apis/runtime/v1/api.proto)
* You want to run a single node with a [standalone Kubelet](https://kubernetes.io/docs/tutorials/cluster-management/kubelet-standalone/) for development

While containerd is more performant, it is a bit less user friendly and requires a bit more work to setup.
We can solve the ease of use issue by using [nerdctl](https://github.com/containerd/nerdctl),
a Docker compatible command-line interface and we can automate the containerd setup using [Ansible](https://docs.ansible.com/ansible/latest/index.html).

On our local machine, we can still use Docker to develop, build and test images.
Docker builds produce [OCI compliant](https://opencontainers.org/) container images which containerd can use without issue.


## Limitations

Unfortunately, at the time of writing this article, remote management of containerd is not easily possible.
I searched for alternatives such as [running a standalone kubelet](https://kubernetes.io/docs/tutorials/cluster-management/kubelet-standalone/),
[nerdctld](https://github.com/afbjorklund/nerdctld), exposing the containerd socket with ssh or tcp (not supported with nerdctl), and using [ctr](https://github.com/projectatomic/containerd/blob/master/docs/cli.md) directly. I also thought about using [socat]() and [stunnel]() to expose the containerd socket over the network but,
this would quickly unnecessarily complicated.

In short, none of these solutions was appealing to me so, I decided to just install nerdctl on the target machines for now.
That way, we can use nerdctl compose to start our containers similar to the way we would run docker compose.

If I come across good solution in the future, I will surely write an article about it.


## Server Setup

I will be using [ubuntu 24.04 LTS server](https://releases.ubuntu.com/noble/) as the operating system.
For convenience, I will be using a [Digital Ocean](https://digitalocean.com) droplet for this article.
You can use my [referral link](https://m.do.co/c/bc29bbb69f3e) to get some free credit and get started.
Digital Ocean is nice because you can just top up your account and you don't need to add a credit card.
The service is pretty cheap and has a fast internet connection which makes downloading and installing packages very fast.
You can top up your account without providing a credit card.

The instructions should work on any virtual machine or bare-metal server with Ubuntu 24.04 installed so, you can use whatever infrastructure you like.
I will not be covering the installation of Ubuntu itself here (You can have a look at my post [Creating a Professional Home Lab Setup Part 1 - Installing the Operating System](https://medium.com/tadaweb/creating-a-professional-home-lab-setup-part-1-installing-the-operating-system-cfb4839fc796) on Medium for instructions on how to do that).

After we have installed the operating system (or in my case created the droplet) we can login to the server using the system console or SSH. 

{{< notice >}}
If you are using a Digital Ocean droplet, you will need to SSH as **root**.
If you  have installed the server yourself, login with the user account you have created during the setup.
{{< /notice >}}

Let's start by updating the server and installing [rootlesskit](https://github.com/rootless-containers/rootlesskit), containerd and some other dependencies.
```shell
# update system packages
sudo apt update && sudo apt -y upgrade

# install rootlesskit and containerd (required for rootless containerd)
sudo apt -y install rootlesskit newuidmap newgidmap slirp4netns containerd
```

If there was a kernel update, reboot:
```shell
# only needed if there was a kernel update
sudo reboot
```

Stop and disable and the non-rootless containerd service:
```shell
# disable and stop containerd service
sudo systemctl disable --now containerd
```

We will need to install the [CNI](https://github.com/containernetworking/cni) plugins for container networking separately:
```shell
# change this to the latest release on https://github.com/containernetworking/plugins/releases
CNI_PLUGINS_VERSION=1.6.2

curl -LO https://github.com/containernetworking/plugins/releases/download/v${CNI_PLUGINS_VERSION}/cni-plugins-linux-amd64-v${CNI_PLUGINS_VERSION}.tgz
sudo mkdir -p /opt/cni/bin
sudo tar -C /opt/cni/bin -xvzf cni-plugins-linux-amd64-v${CNI_PLUGINS_VERSION}.tgz
```

Next, we need to create a system user account for containerd:
```shell
# we don't want this user to be able to login because it is a system account
sudo useradd containerd -d /home/containerd -m -s /usr/sbin/nologin
```

Then, it is time to install **nerdctl**. This includes a script that facilitates installing a rootless containerd (`containerd-rootless-setuptool.sh`).
If you are want to know how this is achieved, you can take a look at the [containerd documentation](https://github.com/containerd/nerdctl/blob/main/docs/rootless.md).
```shell
# change this to the latest release on https://github.com/containerd/nerdctl/releases/
NERDCTL_VERSION=2.0.3

curl -LO https://github.com/containerd/nerdctl/releases/download/v${NERDCTL_VERSION}/nerdctl-2.0.3-linux-amd64.tar.gz
sudo tar -C /usr/local/bin/ -xvzf nerdctl-${NERDCTL_VERSION}-linux-amd64.tar.gz
```

User lingering will need to be enabled for the containerd user:
{{< notice >}}
From the man page:
```
enable-linger [USER...], disable-linger [USER...]
    Enable/disable user lingering for one or more users. If enabled for a specific
    user, a user manager is spawned for the user at boot and kept around after
    logouts. This allows users who are not logged in to run long-running services.
    Takes one or more user names or numeric UIDs as argument. If no argument is
    specified, enables/disables lingering for the user of the session of the
    caller.
```
{{< /notice >}}
```shell
# this does not require a reboot
sudo loginctl enable-linger containerd
```

Finally, we will install and run the rootless containerd service as the containerd user:
```shell
sudo -u containerd bash
```
```shell
# this is not set and required when we create a bash shell using sudo
export XDG_RUNTIME_DIR=/run/user/${UID}

# this will check some prerequisites and create and start the rootless containerd service
containerd-rootless-setuptool.sh install
```

Check the installation output for any errors. If it installed successfully, you will be able to run the following commands:
```shell
nerdctl run -v /:/rootdir -ti --rm alpine:latest whoami
# output:
#   root

nerdctl run -v /:/rootdir -ti --rm alpine:latest cat /rootdir/etc/shadow
# output:
#  cat: can't open '/rootdir/etc/shadow': Permission denied
```

As you can tell by that final command, our container does not have root permissions on the host system.

That pretty much covers the installation of a rootless single node containerd service.
You can test that the containers are proberly started after a reboot with the following commands:
```shell
# as the containerd user
nerdctl run -d --name alpine alpine:latest -- sleep infinity

# as root or sudo
reboot
```

## Going Forward


## Summary


## TL;DR


## Sources

* https://github.com/containerd/containerd/blob/main/docs/getting-started.md
* https://github.com/containerd/containerd/blob/main/docs/rootless.md
* https://github.com/containerd/nerdctl/blob/main/docs/rootless.md
* https://github.com/rootless-containers/bypass4netns
* https://rootlesscontaine.rs/getting-started/common/login/

