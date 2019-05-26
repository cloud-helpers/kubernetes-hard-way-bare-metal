Kubernetes The Hard Way - Bare Metal with VM and Containers
===========================================================

![Containers on Docks](https://github.com/cloud-helpers/kubernetes-hard-way-bare-metal/blob/master/img/Containers%20on%20Docks%20-%20Pixabay.jpg)

# General
[This set of documents](https://github.com/cloud-helpers/kubernetes-hard-way-bare-metal/blob/master/README.md)
aims at providing full hands-on guides to set up
[Kubernetes (aka K8S)](https://kubernetes.io) clusters on bare metal servers,
which can actually be physical servers or virtual machines (VM),
on premises or in private and public clouds.

Those documents are adaptations of the excellent
["Kubernetes The Hard Way - Bare Metal"
guide](https://github.com/Praqma/LearnKubernetes/blob/master/kamran/Kubernetes-The-Hard-Way-on-BareMetal.md)
(by [Tobias Schöneberg](https://github.com/metas-ts)),
itself derived from the famous
["Kubernetes The Har Way"
guide](https://github.com/kelseyhightower/kubernetes-the-hard-way),
by [Kelsey Hightower](https://github.com/kelseyhightower).

Except for the Proxmox host itself, which is based on a
[Debian distribution](https://www.debian.org/), all the nodes
are installed with [CentOS distributions](https://www.centos.org).
Other distributions may of course be used. One distrubtion has to be chosen,
so let it be CentOS for these guides.

For the Kubernetes-related tools and binaries (_e.g._, Docker, Go,
`kubectl`), two variants are documented:
* CentOS 7 packaged versions, _e.g._:
  + [Docker packages](https://git.centos.org/rpms/docker/releases)
  + [Go packages](https://git.centos.org/rpms/golang/releases)
  + [Kubernetes packages](https://git.centos.org/rpms/kubernetes/releases)
* Upstream versions:
  + [Docker CE](https://download.docker.com/linux/centos/7/x86_64/stable/Packages/)
  + [Go latest releases](https://golang.org/doc/devel/release.html)
  + [Kubernetes latest release number](https://storage.googleapis.com/kubernetes-release/release/stable.txt)
  + [Go]()
  + [`kubectl` v1.14.2 binary](https://storage.googleapis.com/kubernetes-release/release/v1.14.2/bin/linux/amd64/kubectl)
* Kubernetes CentOS packages are fairly outdated (v1.5.2).
  There are various consequences, for instance:
  + The DNS-related services are still limited, like (old versions of)
    [Kube-DNS](https://github.com/kubernetes/dns) only
  + Most of the (recent) documentation on Kubernetes services
    does not work as is with those older versions. For instance,
	the `Service` part of
	[Nginx](https://kubernetes.io/docs/concepts/services-networking/connect-applications-service/)
	does not work with CentOS packages of Kubernetes
* Upgrading some versions of Kubernetes utilities on CentOS is not
  an esay task, as all the Docker, Go and Kubernetes tools have been
  packaged in a consistent way on the CentOS distribution. Either of
  those cannot be upgraded to newer versions without breaking the
  CentOS packaging contraint
* Both variants have been documented, because depending on the use case,
  either may be prefered:
  + CentOS packages for a quick setup, fully supported and consistent
    with the rest of the CentOS distribution
  + Upstream binaries for a more flexible and up-to-date setup,
    requiring more configuration and maintenance work


As many other documentations, that one will soon be outdated, and imperfect
for sure. Contributions are therefore welcome to complemenent that guide.
For instance, through
[issues](https://github.com/cloud-helpers/kubernetes-hard-way-bare-metal/issues) or
[pull requests](https://github.com/cloud-helpers/kubernetes-hard-way-bare-metal/pulls).

# [With KVM/QEMU Virtual Machines (VM)](https://github.com/cloud-helpers/kubernetes-hard-way-bare-metal/blob/master/kvm-qemu/README.md)
Guide to set up a [Kubernetes](https://kubernetes.io) cluster on
[Proxmox-based](https://www.proxmox.com/en/proxmox-ve/features)
[KVM/QEMU virtual machines (VM)](https://en.wikipedia.org/wiki/Kernel-based_Virtual_Machine).
Using any other
[virtualization framework](https://en.wikipedia.org/wiki/OS-level_virtualisation)
than Proxmox should not make much difference.

While full virtualization (with virtual machines) provides more flexibility
(_e.g._, choice of kernel and type of operating system, reserved CPU and
memory resources), it implies more overhead than containers.
The following section accommodates cases with more resource-conscious
requirements.

# [With LXC Containers](https://github.com/cloud-helpers/kubernetes-hard-way-bare-metal/blob/master/lxc/README.md)
Guide to set up a [Kubernetes](https://kubernetes.io) cluster on
[Proxmox-based](https://www.proxmox.com/en/proxmox-ve/features)
[LXC containers](https://linuxcontainers.org/#LXC).
Using [LXD](https://linuxcontainers.org/#LXD) or [OpenVZ](https://openvz.org)
rather than Proxmox should not make much difference.

While there are quite a few articles and solutions on Kubernetes and similar
clusters in Docker containers (_e.g._, [KinD](https://kind.sigs.k8s.io),
[Footloose](https://github.com/weaveworks/footloose)), the literature remains
scarce about Kubernetes deployed on container technologies
such as Linux containers (_e.g._, LXC, OpenVZ).
However, Linux containers allow for far more flexibility than Docker
containers. Basically, Linux containers aim at executing full-blown (Linux)
operating system (OS), whereas Docker containers aim at micro-services
and serverless platforms, mainly servicing single processes.

[This guide](https://github.com/cloud-helpers/kubernetes-hard-way-bare-metal/blob/master/lxc/README.md)
therefore aims at helping filling the gap of documentation in the
Linux container space for Kubernetes.

# [Proxmox Setup](https://github.com/cloud-helpers/kubernetes-hard-way-bare-metal/blob/master/proxmox/README.md)
That section provides some details on how to prepare a
[Proxmox distribution](https://pve.proxmox.com) in order
to setup Kubernetes clusters on it.

All the Kubernetes cluster nodes are insulated thanks to a gateway:
all the traffic from outside the cluster is channelled through the gateway.
The set up of such a gateway is also an addition to this set of guides.
It allows one to experiment with Kubernetes clusters, including operating
some in production-like settings, while keeping some good level of security.

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
* Kubernetes - Pod Networking, by
  [Marvin The Paranoid - Own work, CC BY-SA 4.0](https://commons.wikimedia.org/w/index.php?curid=75140812)
  ![Kubernetes - Pod Networking](https://github.com/cloud-helpers/kubernetes-hard-way-bare-metal/blob/master/lxc/img/Kubernetes%20-%20Pod%20Networking.png "Kubernetes - Pod Networking")

