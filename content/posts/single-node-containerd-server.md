---
date: '2025-02-27T09:49:29Z'
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

The article is targeted at anyone who wants to learn how to set up a rootless stand-alone server using containerd instead of Docker.
It is not targeted at people who are just starting out with containers.
If you are just starting out and want to get started quickly,
[Docker](https://www.docker.com/get-started/) or [Podman](https://podman.io/get-started) will do fine.

For this article, I will assume that you have some experience on the command-line and know how to login to a Linux server on the command-line via the console or [SSH](https://en.wikipedia.org/wiki/Secure_Shell).


## Why Containerd?

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

Unfortunately, at the time of writing this article, remote management of containerd is not that simple.
I searched for solutions such as [running a standalone kubelet](https://kubernetes.io/docs/tutorials/cluster-management/kubelet-standalone/),
using [nerdctld](https://github.com/afbjorklund/nerdctld), exposing the containerd socket over the network with SSH or TCP using tools like [socat](https://linux.die.net/man/1/socat) and [stunnel](https://www.stunnel.org/),
and using [ctr](https://github.com/projectatomic/containerd/blob/master/docs/cli.md) directly.

Due to their complexity, none of these solutions seemed appealing to me so, I decided to just install nerdctl on the target machine for now.
That way, we can use nerdctl compose to start our containers similar to the way we would run docker compose.

If I find a good solution in the future, I will surely write an article about it.


## Server Setup

I will be using [ubuntu 24.04 LTS server](https://releases.ubuntu.com/noble/) as the operating system.
For convenience, I will be using a [Digital Ocean](https://digitalocean.com) droplet while writing this article.
You can use my [referral link](https://m.do.co/c/bc29bbb69f3e) to get some free credit and get started.
I like Digital Ocean because it is fast, cheap and you can top up your account without the need for a credit card.
The droplets spin up in under a minute and have a fast internet connection which makes downloading and installing packages a breeze.

The instructions in this article should work on any virtual machine or bare-metal server with Ubuntu 24.04 installed so, you can use whatever infrastructure you like.
I will not be covering the installation of Ubuntu itself here (You can have a look at my post [Creating a Professional Home Lab Setup Part 1 - Installing the Operating System](https://medium.com/tadaweb/creating-a-professional-home-lab-setup-part-1-installing-the-operating-system-cfb4839fc796) on Medium for instructions on how to do that).


## Containerd Setup

After we have installed the operating system (or created the droplet),
we can log in to the server using the system console or SSH. 

{{< notice >}}
If you are using a Digital Ocean droplet, you will need to login as **root**.
{{< /notice >}}

Let's start by updating the server and installing [rootlesskit](https://github.com/rootless-containers/rootlesskit),
containerd and [slirp4netns](https://github.com/rootless-containers/slirp4netns):
```shell
# update system packages
sudo apt update && sudo apt -y upgrade

# install rootless containerd requirements
sudo apt -y install rootlesskit slirp4netns containerd
```

If there was a kernel update, it is a good idea to reboot the machine:
```shell
# only needed if there was a kernel update
sudo reboot
```

The containerd runtime does not come with the [CNI](https://github.com/containernetworking/cni) plugins installed. We will need to install them separately.
The CNI plugins will be installed in `/opt/cni/bin/`:
```shell
# change this to the latest release on https://github.com/containernetworking/plugins/releases
CNI_PLUGINS_VERSION=1.6.2

curl -LO https://github.com/containernetworking/plugins/releases/download/v${CNI_PLUGINS_VERSION}/cni-plugins-linux-amd64-v${CNI_PLUGINS_VERSION}.tgz
sudo mkdir -p /opt/cni/bin
sudo tar -C /opt/cni/bin -xvzf cni-plugins-linux-amd64-v${CNI_PLUGINS_VERSION}.tgz
```

While we are at it, we will also install nerdctl. It will be installed in `/usr/local/bin`:
{{< notice >}}
**nerdctl** includes scripts that facilitate installing and running a rootless containerd.
If you want to know how this is works, take a look at the [nerdctl documentation](https://github.com/containerd/nerdctl/blob/main/docs/rootless.md).
{{< /notice >}}
```shell
# change this to the latest release on https://github.com/containerd/nerdctl/releases/
NERDCTL_VERSION=2.0.3

curl -LO https://github.com/containerd/nerdctl/releases/download/v${NERDCTL_VERSION}/nerdctl-2.0.3-linux-amd64.tar.gz
sudo tar -C /usr/local/bin/ -xvzf nerdctl-${NERDCTL_VERSION}-linux-amd64.tar.gz
```

After installing the required the packages and tools,
the first thing we need to do is stop and disable the non-rootless containerd service that was automatically enabled while installing containerd:
```shell
# disable and stop containerd service
sudo systemctl disable --now containerd
```

We also need to make sure that rootlesskit is allowed to bind reserved [well-known ports](https://en.wikipedia.org/wiki/List_of_TCP_and_UDP_port_numbers#Well-known_ports). You can find more information about that [here](https://github.com/rootless-containers/rootlesskit/blob/v0.14.1/docs/port.md#exposing-privileged-ports).
```shell
sudo setcap cap_net_bind_service=ep /usr/bin/rootlesskit
```

Now, we can create an unprivileged user account that will be used to run containerd:
```shell
# we don't want this user to be able to login because it is a system account
# change the user home directory to wherever you want your containerd storage to be
sudo useradd containerd -d /home/containerd -m -s /usr/sbin/nologin
```

We will also need to enable lingering for this user account:
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
{{< notice >}}
We need to use **sudo** to spawn a bash shell in this way because the containerd user is not allowed to log in.
{{< /notice >}}
```shell
# XDG_RUNTIME_DIR is required for systemd and is normally set during login
sudo -u containerd bash -c 'export XDG_RUNTIME_DIR=/run/user/${UID}; cd; bash'
```
```shell
# this will check some prerequisites and create and start the rootless containerd service
# this script was installed with nerdctl in /usr/local/bin
containerd-rootless-setuptool.sh install
```

Check the installation output for any errors (You can ignore the CRI Service error). If it installed successfully, you will be able to run the following commands:
```shell
nerdctl run -ti --rm alpine:latest whoami
```
The output will show that the latest alpine image is pulled from the registry and that the user inside of the container is **root**.

```
nerdctl run -v /:/rootdir -ti --rm alpine:latest cat /rootdir/etc/shadow
```
This command will output: `cat: can't open '/rootdir/etc/shadow': Permission denied`.

That is exactly what we want. Even though we are **root** inside of the container,
we do not have root privileges on the host system.
We can see exactly what is going on if we use a container to create a file in a mounted volume:
```shell
# create an empty file in the containerd home directory
nerdctl run -v ${HOME}:/homedir/ -ti --rm alpine:latest touch /homedir/test.txt

# list the file that was just created outside of the container
ls -l
# -rw-r--r-- 1 containerd containerd 0 Feb 27 22:14 test.txt

# list the file that was just created inside of the container
nerdctl run -v ${HOME}:/homedir/ -ti --rm alpine:latest ls -l /homedir/
# -rw-r--r--    1 root     root             0 Feb 27 22:14 test.txt
```
You can see that the root user inside of the container is mapped to the containerd user on the host system. Neat!

As a final test, we can start an nginx web server and publish port 80 on the host machine:
```shell
# this will output a unique container ID and start the nginx container in the background
nerdctl run -d -p 80:80 --name nginx nginx:latest
```

If we open a browser and navigate to the server's IP address (In my case `64.225.66.108`),
we will see the nginx welcome page, confirming that our rootless container is running and is published on port 80.
![Welcome to nginx screenshot](/single-node-rootless-container-host-with-containerd-screenshot.png)

When we are done staring proudly at our rootless containerized service, we can clean up the nginx container with the following command:
```shell
# stops and removes the container
nerdctl rm -f nginx
```


## Limiting Network Access

All the steps we have followed so far are fine if you want your services to be publicly accessible but,
If you want to restrict access to the server, we will have to configure the firewall and CNI to use a whitelist of IP addresses.

TODO

## What's Next?

In the future, I plan two write articles describing the following topics:

* Create services using nerdctl compose
* Automate the setup described in this post with [Ansible](https://docs.ansible.com/ansible/latest/index.html)
* Monitor our containerd service using [Prometheus](https://prometheus.io/docs/introduction/overview/)


## Sources

* https://github.com/containerd/containerd/blob/main/docs/getting-started.md
* https://github.com/containerd/containerd/blob/main/docs/rootless.md
* https://github.com/containerd/nerdctl/blob/main/docs/rootless.md
* https://github.com/rootless-containers/bypass4netns
* https://rootlesscontaine.rs/getting-started/common/login/
* https://github.com/rootless-containers/slirp4netns
* https://github.com/rootless-containers/rootlesskit/blob/v0.14.1/docs/port.md#exposing-privileged-ports

