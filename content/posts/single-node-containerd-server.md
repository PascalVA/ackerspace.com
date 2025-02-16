---
date: '2025-02-22T09:49:29Z'
draft: true
title: 'Building a single node container host with containerd'
author: Pascal Van Acker
ShowToc: true
cover:
  image: /single-node-container-host-with-containerd-banner.png
tags:
- containers
- containerd
- nerdctl
---

In this article I will explain how to setup a single node container host using [containerd](https://github.com/containerd/containerd).
The Docker-compatible command-line interface [nerdctl](https://github.com/containerd/nerdctl) will be used to interact with containerd.
Additionally, we will ensure that our containers do not expose ports worldwide by adding an ip whitelist _(using iptables with ipset)_.

This article is targeted at anyone who wants to set up stand-alone servers using containerd instead of Docker,
anyone that is interested in learning more about containerd as a container runtime and anyone that is just interested in reading about this use-case.

This article is **not** targeted at developers just starting out with containers.
If you are just starting out it is work looking in to [Docker](https://www.docker.com/get-started/) or [Podman](https://podman.io/get-started).


## What Is Containerd and Why Not Use Docker or Podman?

To be completely honest, the main reason I chose to use containerd instead of Docker or Podman,
is to learn and experiment with new things and this seemed like fun and challenging topic.

Besides that, here are some actual reasons on why you might want to do this:
* The containerd runtime is [OCI Compliant](https://opencontainers.org/) and [CRI compliant](https://kubernetes.io/docs/concepts/architecture/cri/)
* The containerd runtime is is more lightweight and more [performant](https://medium.com/norma-dev/benchmarking-containerd-vs-dockerd-performance-efficiency-and-scalability-64c9043924b1) than Docker
* You want to develop an application or interface that uses the [CRI specification](https://github.com/kubernetes/cri-api/blob/master/pkg/apis/runtime/v1/api.proto)
* You want to run a single node with a [standalone Kubelet](https://kubernetes.io/docs/tutorials/cluster-management/kubelet-standalone/) for development

If you want to get started quickly and favor ease of use and an extensive feature set, Docker or Podman will do just fine.

However, since performance matters more than usability on our servers, containerd is the better choice.

Container deployments on our server will be automated and we will rarely interact with them manually so,
the usability and user-friendliness of the container runtime are not that important to us.

On our local machine, we can still use Docker to develop, build and test images.
Docker builds produce [OCI Compliant](https://opencontainers.org/) container images which containerd can run without issue.


## Limitations

Unfortunately, at the time of writing this article, remote management with nerdctl is not possible.
I searched for alternatives such as [running a standalone kubelet](https://kubernetes.io/docs/tutorials/cluster-management/kubelet-standalone/),
[nerdctld](https://github.com/afbjorklund/nerdctld), exposing the containerd socket with ssh or tcp (not supported in nerdctl), and using [ctr](https://github.com/projectatomic/containerd/blob/master/docs/cli.md) directly.

None of those solutions was appealing to me so, I decided to just install nerdctl on the target machines for now.
That way, we can use nerdctl compose to start our containers similar to the way we would run docker compose.

If I come across a better solution in the future, I will surely write an article about it.


## Server Setup

We will be using [ubuntu 24.04 LTS server](https://releases.ubuntu.com/noble/) as our base server.
I will not be covering the installation of Ubuntu. You can have a look at [Creating a Professional Home-Lab Setup Part 1 - Installing the Operating System](https://medium.com/tadaweb/creating-a-professional-home-lab-setup-part-1-installing-the-operating-system-cfb4839fc796) on medium for instructions on how to do that.
For convenience I will be using a [Digital Ocean](https://www.digitalocean.com/) droplet.
However, the instructions should work on any virtual machine or bare-metal host.


Before starting, I like to make sure my system is up to date with the latest packages:

* Login to your (new) server as root or run commands using sudo.

* Update the system packages
    ```shell
    sudo apt update && sudo apt -y upgrade
    ```

## Installing Containerd

To get started, we will need to install [containerd](https://github.com/containerd/containerd/blob/main/docs/getting-started.md)


## Installing the Container Network Interface (CNI)

Next we need to install a [Container Network Interface](). We will be using [CNI](https://github.com/containernetworking/cni).


