Kubernetes The Hard Way - Setup of the Proxmox Host
===================================================

![Lake Tisza](https://github.com/cloud-helpers/kubernetes-hard-way-bare-metal/blob/master/proxmox/img/Lake%20Tisza%20-%20Pixabay.jpg)

- [Overview](#overview)
- [Host preparation](#host-preparation)
  * [Get the latest CentOS templates](#get-the-latest-centos-templates)
  * [Overlay module](#overlay-module)
- [Kubernetes cluster gateway](#kubernetes-cluster-gateway)
  * [Complement the installation on the gateway](#complement-the-installation-on-the-gateway)

# Overview
[This document](https://github.com/cloud-helpers/cloud-helpers/kubernetes-hard-way-bare-metal/blob/master/proxmox/README.md)
aims at providing a full hands-on guide to set up a
[Proxmox distribution](https://www.proxmox.com/en/proxmox-ve/features)
in order to accommodate [Kubernetes (aka K8S)](https://kubernetes.io)
clusters, both on [LXC containers](https://linuxcontainers.org/#LXC)
and
[KVM/QEMU virtual machines (VM)](https://en.wikipedia.org/wiki/Kernel-based_Virtual_Machine).

All the Kubernetes cluster nodes, be them full KVM/QEMU virtual machines (VM)
or LXC containers, are insulated thanks to an LXC gateway.
All the traffic from outside the cluster is channelled through the gateway.

The set up of such a gateway is also an addition to this set of guides.

It allows one to experiment with Kubernetes clusters, including operating
some in production-like settings, while keeping some good level of security.

# Host preparation
In that section, it is assumed that we are logged on the Proxmox host
as `root`.

Though it is not strictly necessary, the cluster may be accessible from
outside (from the Internet).

The following parameters are used in the remaining of the guide, and may be
adapted according to your configuration:
* IP of the routing gateway on the host (typically ends with `.254`:
  `HST_GTW_IP`
* (Potentially virtual) MAC address of the Kubernetes cluster gateway:
  `GTW_MAC`
* IP address of the Kubernetes cluster gateway: `GTW_IP`
* VM ID of that Kubernetes cluster gateway: `103`
* For the KVM/QEMU, the setup of the virtual machines (VM)
  reflects in a loosely way the
  [Getting started guide on installing a multi-node Kubernetes cluster
  on Fedora with flannel](https://kubernetes.io/docs/getting-started-guides/fedora/flannel_multi_node_cluster/),
  and is summarized below:

| VM ID | Private IP  |    Host name (full)     | Short name  |
| ----- | ----------- | ----------------------- | ----------- |
|  103  | 10.240.0.2  | gwkublxc.example.com    | gwkublxc    |
|  200  | 10.240.0.200| kub-master.example.com  | kub-master  |
|  201  | 10.240.0.201| kub-node1.example.com   | kub-node1   |
|  202  | 10.240.0.202| kub-node2.example.com   | kub-node2   |

* For the LXC container-based setup, the private IP addresses
  and host names of all the nodes correspond to
  [Tobias' guide](https://github.com/Praqma/LearnKubernetes/blob/master/kamran/Kubernetes-The-Hard-Way-on-BareMetal.md#provision-vms-in-kvm),
  and a summary is provided below:

| VM ID | Private IP  |    Host name (full)     | Short name  |
| ----- | ----------- | ----------------------- | ----------- |
|  103  | 10.240.0.2  | gwkublxc.example.com    | gwkublxc    |
|  211  | 10.240.0.11 | etcd1.example.com       | etcd1       |
|  212  | 10.240.0.12 | etcd2.example.com       | etcd2       |
|  220  | 10.240.0.20 | controller.example.com  | controller  |
|  221  | 10.240.0.21 | controller1.example.com | controller1 |
|  222  | 10.240.0.22 | controller2.example.com | controller2 |
|  231  | 10.240.0.31 | worker1.example.com     | worker1     |
|  232  | 10.240.0.32 | worker2.example.com     | worker2     |
|  240  | 10.240.0.40 | lb.example.com          | lb          |
|  241  | 10.240.0.41 | lb1.example.com         | lb1         |
|  242  | 10.240.0.42 | lb2.example.com         | lb2         |

* Extract of the host network configuration:
```bash
root@proxmox:~$ cat /etc/network/interfaces
# This file describes the network interfaces available on your system
# and how to activate them. For more information, see interfaces(5).

# The loopback network interface
auto lo
iface lo inet loopback

auto eno1
iface eno1 inet manual

auto eno2
iface eno2 inet manual

auto bond0
iface bond0 inet manual
        bond-slaves eno1 eno2
        bond-miimon 100
        bond-mode active-backup

# vmbr0: Bridging. Make sure to use only MAC adresses that were assigned to you.
auto vmbr0
iface vmbr0 inet static
        address ${HST_IP}
        netmask 255.255.255.0
        gateway ${HST_GTW_IP}
        bridge_ports bond0
        bridge_stp off
        bridge_fd 0

auto vmbr3
iface vmbr3 inet static
        address 10.240.0.2
        netmask 255.255.255.0
        bridge-ports none
        bridge-stp off
        bridge-fd 0
        post-up echo 1 > /proc/sys/net/ipv4/ip_forward
        post-up iptables -t nat -A POSTROUTING -s '10.240.0.0/24' -o vmbr0 -j MASQUERADE
        post-down iptables -t nat -D POSTROUTING -s '10.240.0.0/24' -o vmbr0 -j MASQUERADE
root@proxmox:~$ cat /etc/systemd/network/50-default.network
# This file sets the IP configuration of the primary (public) network device.
# You can also see this as "OSI Layer 3" config.
# It was created by the OVH installer, please be careful with modifications.
# Documentation: man systemd.network or https://www.freedesktop.org/software/systemd/man/systemd.network.html

[Match]
Name=vmbr0

[Network]
Description=network interface on public network, with default route
DHCP=no
Address=${HST_IP}/24
Gateway=${HST_GTW_IP}
IPv6AcceptRA=no
NTP=ntp.ovh.net
DNS=127.0.0.1
DNS=8.8.8.8

[Address]
Address=${HST_IPv6}

[Route]
Destination=2001:0000:0000:34ff:ff:ff:ff:ff
Scope=link
root@proxmox:~$ cat /etc/systemd/network/50-public-interface.link
# This file configures the relation between network device and device name.
# You can also see this as "OSI Layer 2" config.
# It was created by the OVH installer, please be careful with modifications.
# Documentation: man systemd.link or https://www.freedesktop.org/software/systemd/man/systemd.link.html

[Match]
Name=vmbr0

[Link]
Description=network interface on public network, with default route
MACAddressPolicy=persistent
NamePolicy=kernel database onboard slot path mac
#Name=eth0	# name under which this interface is known under OVH rescue system
#Name=eno1	# name under which this interface is probably known by systemd
```

## Get the latest CentOS templates
* Download the latest template from the
  [Linux containers site](https://us.images.linuxcontainers.org/images/centos/7/amd64/default/)
  (change the date and time-stamp according to the time you download that
  template):
```bash
root@proxmox:~$ wget https://us.images.linuxcontainers.org/images/centos/7/amd64/default/20190510_07:08/rootfs.tar.xz -O /vz/template/cache/centos-7-default_20190510_amd64.tar.xz
```

## Overlay module
```bash
root@proxmox:~$ modprobe overlay && \
 cat > /etc/modules-load.d/docker-overlay.conf << _EOF
overlay
_EOF
```

# Kubernetes cluster gateway
* Create the LXC container for the Kubernetes cluster gateway:
```bash
root@proxmox:~$ pct create 103 local:vztmpl/centos-7-default_20190510_amd64.tar.xz --arch amd64 --cores 1 --hostname gwkublxc.example.com --memory 1024 --swap 2048 --net0 name=eth0,bridge=vmbr0,firewall=1,gw=${HST_GTW_IP},hwaddr=${GTW_MAC},ip=${GTW_IP}/32,type=veth --net1 name=eth1,bridge=vmbr3,ip=10.240.0.103/24,type=veth --onboot 1 --ostype centos
```

* Start and enter the gateway:
```bash
root@proxmox:~$ pct start 103
root@proxmox:~$ pct enter 103
root@gwkublxc:~#
```

## Complement the installation on the gateway
* For security reason, it may be a good idea to change the SSH port
  from `22` to, say `7022`:
```bash
root@gwkublxc:~# yum -y update && yum -y install epel-release
root@gwkublxc:~# yum -y install openssh-server man-db file rsync openssl \
 wget curl less htop yum-utils net-tools git etcd
root@gwkublxc:~# sed -ie 's/#Port 22/Port 7022/g' /etc/ssh/sshd_config
root@gwkublxc:~# systemctl restart sshd && systemctl enable sshd
```

* Settings:
```bash
root@gwkublxc:~# cat > /etc/hosts << _EOF
# Local VM
127.0.0.1   localhost localhost.localdomain localhost4 localhost4.localdomain4
::1         localhost localhost.localdomain localhost6 localhost6.localdomain6

# Kubernetes on KVM/QEMU
10.240.0.200    kub-master.example.com  kub-master
10.240.0.201    kub-node1.example.com   kub-node1
10.240.0.202    kub-node2.example.com   kub-node2

# Kubernetes on LXC
10.240.0.11     etcd1.example.com       etcd1
10.240.0.12     etcd2.example.com       etcd2
10.240.0.20     controller.example.com  controller
10.240.0.21     controller1.example.com controller1
10.240.0.22     controller2.example.com controller2
10.240.0.31     worker1.example.com     worker1
10.240.0.32     worker2.example.com     worker2
10.240.0.40     lb.example.com          lb
10.240.0.41     lb1.example.com         lb1
10.240.0.42     lb2.example.com         lb2

_EOF
root@gwkublxc:~# cat >> ~/.bashrc << _EOF

# Kubernetes
export KUBERNETES_PUBLIC_IP_ADDRESS="10.240.0.20"

# Source aliases
if [ -f ~/.bash_aliases ]
then
        . ~/.bash_aliases
fi

_EOF
root@gwkublxc:~$ cat ~/.bash_alises << _EOF
# User specific aliases and functions
alias dir='ls -laFh --color'
alias rm='rm -i'
alias cp='cp -i'
alias mv='mv -i'

_EOF
root@gwkublxc:~# . ~/.bashrc
```

