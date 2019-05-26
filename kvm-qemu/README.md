Kubernetes The Hard Way - Bare Metal with KVM/QEMU Virtual Machines (VM)
========================================================================

![Creative container assembly](https://github.com/cloud-helpers/kubernetes-hard-way-bare-metal/blob/master/kvm-qemu/img/Container%20Ship%20Hamburg%20-%20Pixabay.jpg)

- [Host preparation](#host-preparation)

# Overview
[This document](https://github.com/cloud-helpers/cloud-helpers/kubernetes-hard-way-bare-metal/blob/master/kvm-qemu/README.md)
aims at providing a full hands-on guide to set up a
[Kubernetes (aka K8S)](https://kubernetes.io) cluster on
[Proxmox-based](https://www.proxmox.com/en/proxmox-ve/features)

Guide to set up a [Kubernetes](https://kubernetes.io) cluster on
[Proxmox-based](https://www.proxmox.com/en/proxmox-ve/features)
[KVM/QEMU virtual machines (VM)](https://en.wikipedia.org/wiki/Kernel-based_Virtual_Machine).
Using any other
[virtualization framework](https://en.wikipedia.org/wiki/OS-level_virtualisation)
(_e.g._, [OpenVZ](https://openvz.org)) than Proxmox should not make
much difference.

While full virtualization (with virtual machines) provides more flexibility
(_e.g._, choice of kernel and type of operating system, reserved CPU and
memory resources), it implies more overhead than containers.
See [the guide on Kubernetes setup on LXC
containers](https://github.com/cloud-helpers/cloud-helpers/kubernetes-hard-way-bare-metal/blob/master/lxc/README.md)
for more details on a more lightweight setup.

It is an adaptation of the
[Getting started guide on installing a multi-node Kubernetes cluster
  on Fedora with flannel](https://kubernetes.io/docs/getting-started-guides/fedora/flannel_multi_node_cluster/).

All the nodes are setup with [CentOS distributions](https://www.centos.org),
and insulated thanks to a gateway: all the traffic from outside
the cluster is channelled through the gateway. The set up of such
a gateway is also an addition to this guide. It allows
one to experiment with Kubernetes clusters, including operating some
in production-like settings, while keeping some good level of security.

As many other documentations, that one will soon be outdated, and imperfect
for sure. Contributions are therefore welcome to complemenent that guide.
For instance, through
[issues](https://github.com/cloud-helpers/kubernetes-hard-way-proxmox-lxc/issues) or
[pull requests](https://github.com/cloud-helpers/kubernetes-hard-way-proxmox-lxc/pulls).

# References
* [Kubernetes The Hard Way - Bare Metal](https://github.com/Praqma/LearnKubernetes/blob/master/kamran/Kubernetes-The-Hard-Way-on-BareMetal.md),
  by [Tobias Schöneberg](https://github.com/metas-ts),
  February 2018, GitHub
* [Kubernetes The Hard Way](https://github.com/kelseyhightower/kubernetes-the-hard-way),
  by [Kelsey Hightower](https://github.com/kelseyhightower),
  2017-2018, GitHub
* [Run kubernetes inside LXC container](https://medium.com/@kvaps/run-kubernetes-in-lxc-container-f04aa94b6c9c),
  by [Andrei Kvapil (aka kvaps)](https://medium.com/@kvaps),
  August 2018, Medium
* [Kubernetes reference documentation](https://kubernetes.io/docs/reference/)
* [Getting started guide on installing a multi-node Kubernetes cluster
  on Fedora with flannel](https://kubernetes.io/docs/getting-started-guides/fedora/flannel_multi_node_cluster/)
* Kubernetes - Architecture, by
  [Khtan66 - Own work, CC BY-SA 4.0](https://commons.wikimedia.org/w/index.php?curid=53571935)
  ![Kubernetes - Architecture](https://github.com/cloud-helpers/kubernetes-hard-way-bare-metal/blob/master/lxc/img/Kubernetes%20-%20Architecture.png "Kubernetes - Architecture")
* Kubernetes - Pod Networking by, by
  [Marvin The Paranoid - Own work, CC BY-SA 4.0](https://commons.wikimedia.org/w/index.php?curid=75140812)
  ![Kubernetes - Pod Networking](https://github.com/cloud-helpers/kubernetes-hard-way-bare-metal/blob/master/lxc/img/Kubernetes%20-%20Pod%20Networking.png "Kubernetes - Pod Networking")
* [Documentation, generate the table of content (TOC)](https://ecotrust-canada.github.io/markdown-toc/)

# Host preparation
See [the guide dedicated to the Proxmox
setup](https://github.com/cloud-helpers/kubernetes-hard-way-bare-metal/blob/master/proxmox/README.md).

