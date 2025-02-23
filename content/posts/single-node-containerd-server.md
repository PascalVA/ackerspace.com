---
date: '2025-02-27T09:49:29Z'
draft: true
title: 'Building a single node rootless container host with containerd'
author: Pascal Van Acker
ShowToc: true
cover:
  image: /single-node-rootless-container-host-with-containerd-banner.png
tags:
- containerd
- containers
- nerdctl
- rootless containers
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


## Installing Containerd and the CNI

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

Check the installation output for any errors (You can ignore the CRI Service error).


## Testing the Waters

With nerdctl, containerd and the CNI installed, it is time to test the waters by running some (rootless) containers:
```shell
nerdctl run -ti --rm alpine:latest -- id
```
The output of the command above will show that the latest alpine image is pulled from the registry and that the user inside of the container is **root**.

We can test if we have root privileges on the host system by mounting the root filesystem as a volume and trying to read a protected file:
```
nerdctl run -v /:/rootdir -ti --rm alpine:latest -- cat /rootdir/etc/shadow
```
This command will output: `cat: can't open '/rootdir/etc/shadow': Permission denied`.

That is exactly what we want. Even though we are **root** inside of the container,
we do not have root privileges on the host system.
We can see exactly what is going on if we use a container to create a file in a mounted volume:
```shell
# create an empty file in the containerd home directory
nerdctl run -v ${HOME}:/homedir/ -ti --rm alpine:latest -- touch /homedir/test.txt

# list the file that was just created outside of the container
ls -l
# -rw-r--r-- 1 containerd containerd 0 Feb 27 22:14 test.txt

# list the file that was just created inside of the container
nerdctl run -v ${HOME}:/homedir/ -ti --rm alpine:latest -- ls -l /homedir/
# -rw-r--r--    1 root     root             0 Feb 27 22:14 test.txt
```

The user **root** inside the container is actually mapped to the user **containerd** on the host.
As a final test, we can start an nginx web server and publish port 80 on the host machine:
```shell
# this will output a unique container ID and start the nginx container in the background
nerdctl run -d -p 80:80 --name nginx nginx:latest
```

If we open a browser and navigate to the server's IP address (In my case `64.225.66.108`),
we will see the nginx welcome page, confirming that our rootless container is running and is published on port 80.
![Welcome to nginx screenshot](/single-node-rootless-container-host-with-containerd-screenshot.png)

When we have finished proudly staring at our rootless containerized service, we can list and clean up the nginx container with the following commands:
```shell
# list running containers
nerdctl ps
```
```shell
# stops and removes the container
nerdctl rm -f nginx
```


## User Namespaces and ID Mappings

You might ask yourself: _"What happens if I try to use a non-root user account inside of a container?"_ and that would be an excellent question!
We can try it by running an ubuntu 24.04 container that has a user account called **ubuntu** built-in.
```shell
nerdctl run --name ubuntu --user ubuntu -ti --rm ubuntu:24.04 -- id
```
This works as expected and will output the information of the `ubuntu` user.
We can see that the user id and group id of the ubuntu user within the container are `1000`:

`uid=1000(ubuntu) gid=1000(ubuntu) groups=1000(ubuntu),4(adm),20(dialout),...`

Now, let's try to use this containter to create a file in our home directory:
```shell
nerdctl run --name ubuntu --user ubuntu -ti --rm \
    -v ${HOME}:/homedir ubuntu:24.04 -- touch /homedir/test-non-root.txt
```

Running this command will lead to the following error message:

`touch: cannot touch '/homedir/test-non-root.txt': Permission denied`

What happened here? If we run the `id` command outside of the container,
we can see that our containerd user is also using a user id and group id of `1000`:

`uid=1000(containerd) gid=1000(containerd) groups=1000(containerd)`

That is where **User Namespaces** come in. It allows a non-root user to map user and group ids on the host system
to a different range of user and group ids within the namespace.
The ubuntu user within the container is mapped to a predefined range of ids on the host system.
You can find the configuration of these ranges in the following files:
```
# /etc/subuid
containerd:100000:65536
```
```
# /etc/subgid
containerd:100000:65536
```

This means that the containerd user account is allowed to map user and group ids from `100000` up to and including `165535`.
The mapped ids inside of containers are calculated as an offset from the start of the range.
However, since root (uid `0`) is mapped to the containerd user (uid `1000`) on the host, the mapping starts from uid `1`.
This means that the user ubuntu with the uid of `1000` inside of the container is mapped to the uid of `100999` on the host.

{{< notice tip >}}
If this offset irritates you, simply modify the start of the range in the configuration files mentioned above to `100001`.

Keep in mind that if you change this, you will also need to restart the containerd service with the following command:
```shell
# run this as the containerd user
systemctl --user restart containerd
```

For the purpose of this article, we will leave the settings at the default.

{{< /notice >}}

The solution to this problem, is to create the volumes for our containers in advance and change the ownership to the ids mapped on the host:
{{< notice >}}
We have to run the commands below as a user that is allowed to run sudo (or as root)
because the containerd user is not allowed to change ownership to the mapped ids.
I understand that this whole process is quite painful and
that is why deployments like these should most definitely be automated.
{{< /notice >}}
```shell
# change to the home directory path of the containerd user
CONTAINERD_HOMEDIR=/home/containerd

# create the volumes directory owned by containerd
sudo install -d ${CONTAINERD_HOMEDIR}/volumes -o containerd -g containerd

# create the ubuntu volume with owner 100999 and group 100999
sudo install -d ${CONTAINERD_HOMEDIR}/volumes/ubuntu -o 100999 -g 100999
```

After creating the volume directories, our ubuntu user can read and write in this directory:

```shell
# create an empty file in the data volume
nerdctl run --name ubuntu --user ubuntu -ti --rm \
    -v ${HOME}/volumes/ubuntu:/data ubuntu:24.04 -- touch /data/test-non-root.txt
```

```shell
# list the data directory contents
nerdctl run --name ubuntu --user ubuntu -ti --rm \
    -v ${HOME}/volumes/ubuntu:/data ubuntu:24.04 -- ls -l /data/
```

```shell
# list the data directory contents outside of the container
ls -l ${HOME}/volumes/ubuntu
```

You will notice that the new file `test-non-root.txt` is owned by the `ubuntu` user within the container
and that it is owned by the user `100999` outside of the container.


## Exploring the Configuration

{{< notice >}}
The file trees below are created using the `tree` command.
We did not install this command in this article because it is
just used for demonstration purposes.
If you want to install it yourself, you can use the following command:
```shell
sudo apt -y install tree
```
{{< /notice >}}

Now that we have started a few containers and understand how everything works, let's have a look at what actually happened under the hood.

While running the commands in this article, the `.config` directory in the **containerd** user's home directory
was populated with the following files:

```shell
tree ~/.config
```
```
.config/
├── cni
│   └── net.d
│       ├── default
│       └── nerdctl-bridge.conflist
├── containerd
└── systemd
    └── user
        ├── containerd.service
        └── default.target.wants
            └── containerd.service -> /home/containerd/.config/systemd/user/containerd.service

8 directories, 3 files
```

* The `systemd` folder contains the user systemd service that was created during the installation of the rootless containerd
* The `containerd` folder remains empty since we do not have any custom configuration for containerd
* The `cni` folder contains network configuration for the CNI. It has the default network and a bridge network that was automatically created by nerdctl while running our containers.


On top of that the `.local` directory has was also populated with files needed to run our containers.
It contains the storage for containerd and includes things such as container image layers, container volumes, the overlayfs and snapshots.

```shell
# we limit the level to 5 to avoid printing the container overlayfs directories
tree -L 5 ~/.local
```
```
containerd@ackerspace:~$ tree -L 5 .local/
.local/
└── share
    ├── cni
    │   ├── networks
    │   │   └── bridge
    │   │       ├── last_reserved_ip.0
    │   │       └── lock
    │   └── results
    ├── containerd
    │   ├── io.containerd.content.v1.content
    │   │   ├── blobs
    │   │   │   └── sha256
    │   │   └── ingest
    │   ├── io.containerd.metadata.v1.bolt
    │   │   └── meta.db
    │   ├── io.containerd.runtime.v1.linux
    │   ├── io.containerd.runtime.v2.task
    │   │   └── default
    │   ├── io.containerd.snapshotter.v1.blockfile
    │   ├── io.containerd.snapshotter.v1.btrfs
    │   ├── io.containerd.snapshotter.v1.native
    │   │   └── snapshots
    │   ├── io.containerd.snapshotter.v1.overlayfs
    │   │   ├── metadata.db
    │   │   └── snapshots
    │   │       ├── 1
    │   │       ├── 10
    │   │       ├── 11
    │   │       ├── 12
    │   │       ├── 13
    │   │       ├── 14
    │   │       ├── 15
    │   │       └── 16
    │   └── tmpmounts
    └── nerdctl
        └── 1935db59
            ├── containers
            │   └── default
            ├── default
            ├── etchosts
            │   └── default
            └── volumes
                └── default
```


## Limiting Network Access

All of the steps we have followed so far are fine if we want our services to be publicly accessible but,
if you want to restrict access to the server, we will have to configure the firewall and CNI to use a whitelist of IP addresses.

We will start by installing the tools required to facilitate this:
```shell
sudo apt -y install iptables-persistent ipset
```

* The package `ipset` makes it easy to create sets of IPs to use with iptables
* The package `iptables-persistent` persists changes to iptables and ipset after a reboot

Before creating a whitelist, we must first will create a new network for our firewalled containers:
```shell
# create a new bridged network called "whitelist" using the subnet "10.0.0.0/24"
nerdctl network create whitelist --driver bridge --subnet 10.0.0.0/24

# list the current networks
nerdctl network ls
```


We can now use ip set to create a new set of IPs. I will call it whitelist:
```shell
# create an ip set consisting of hash of network cidrs
ipset create whitelist hash:net

# list ip sets
ipset list

# add ip address ranges to the whitelist ip set
ip set add whitelist 10.0.0.0/24
```

## What's Next?

I hope you have enjoyed yourself and have learned something while reading this article.
In the future I plan to write some more articles detailing how to automate this setup with Ansible.
I will also cover how to use nerdctl compose to deploy containers and I will most likely also write
an article on how to properly manage and monitor these container deployments.


## Sources

* https://access.redhat.com/articles/5946151
* https://github.com/containerd/containerd/blob/main/docs/getting-started.md
* https://github.com/containerd/containerd/blob/main/docs/rootless.md
* https://github.com/containerd/nerdctl/blob/main/docs/rootless.md
* https://github.com/rootless-containers/bypass4netns
* https://github.com/rootless-containers/rootlesskit/blob/v0.14.1/docs/port.md#exposing-privileged-ports
* https://github.com/rootless-containers/slirp4netns
* https://rootlesscontaine.rs/getting-started/common/login/
