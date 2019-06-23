Kubernetes The Hard Way - Bare Metal with LXC Containers
========================================================

![Creative container assembly](https://github.com/cloud-helpers/kubernetes-hard-way-bare-metal/blob/master/lxc/img/Creative%20container%20-%20Pixabay.jpg)

- [Overview](#overview)
- [References](#references)
- [Host preparation](#host-preparation)
- [CFSSL](#cfssl)
  * [CFSSL software](#cfssl-software)
  * [CA certificates](#ca-certificates)
  * [Node certificates](#node-certificates)
- [Creation and basic setup of the LXC containers](#creation-and-basic-setup-of-the-lxc-containers)
  * [Configuration management (`etcd`) servers](#configuration-management---etcd---servers)
  * [Controller plane servers](#controller-plane-servers)
  * [Worker nodes](#worker-nodes)
  * [Load balancers (lb)](#load-balancers--lb-)
  * [On the clients (eg, laptops)](#on-the-clients--eg--laptops-)
  * [List of the Kubernetes nodes](#list-of-the-kubernetes-nodes)
  * [Install the Kubernetes cluster certificates on the Kubernetes nodes](#install-the-kubernetes-cluster-certificates-on-the-kubernetes-nodes)
  * [Complementary packages for the Kubernetes nodes](#complementary-packages-for-the-kubernetes-nodes)
  * [Basic checks](#basic-checks)
- [Set up of the configuration management (`etcd`) servers](#set-up-of-the-configuration-management---etcd---servers)
  * [Check the setup on each `etcd` node](#check-the-setup-on-each--etcd--node)
- [Kubernetes API, controller and scheduler servers](#kubernetes-api--controller-and-scheduler-servers)
  * [Installation of complementary packages](#installation-of-complementary-packages)
  * [Go](#go)
  * [Kubernetes binaries](#kubernetes-binaries)
  * [Kubernetes certificates](#kubernetes-certificates)
  * [Kubernetes authentication token](#kubernetes-authentication-token)
  * [Access Control Lists (ACL)](#access-control-lists--acl-)
  * [Kubernetes API service](#kubernetes-api-service)
  * [Kubernetes controller manager](#kubernetes-controller-manager)
  * [Kubernetes scheduler](#kubernetes-scheduler)
  * [Restarting the services on the control plane](#restarting-the-services-on-the-control-plane)
  * [Helper tools (`kubectx` / `kubens`)](#helper-tools---kubectx-----kubens--)
  * [Check the components](#check-the-components)
- [Kubernetes worker nodes](#kubernetes-worker-nodes)
  * [Kubernetes binaries](#kubernetes-binaries-1)
  * [Docker](#docker)
  * [CNI network plugin](#cni-network-plugin)
  * [API server](#api-server)
  * [Kubelet configuration](#kubelet-configuration)
    + [Kubernetes configuration file (`KUBECONFIG`)](#kubernetes-configuration-file---kubeconfig--)
    + [Kubelet service](#kubelet-service)
  * [`kube-proxy` configuration](#-kube-proxy--configuration)
  * [Checks](#checks)
  * [CoreDNS](#coredns)
    + [Query the cluster](#query-the-cluster)
    + [Smoke test with busybox](#smoke-test-with-busybox)
  * [Remove CoreDNS](#remove-coredns)
    + [CoreDNS from the sources (optional)](#coredns-from-the-sources--optional-)
- [Kube DNS](#kube-dns)
  * [Remove KubeDNS](#remove-kubedns)
- [Kubernetes client on the gateway](#kubernetes-client-on-the-gateway)
- [Smoke tests with replicas on Alpine network multi-tool](#smoke-tests-with-replicas-on-alpine-network-multi-tool)
- [Smoke tests with replicas of Nginx](#smoke-tests-with-replicas-of-nginx)
- [Smoke tests with the travel simulator](#smoke-tests-with-the-travel-simulator)
  * [Create a single simulator pod](#create-a-single-simulator-pod)
  * [Create a full deployment of several simulator pods](#create-a-full-deployment-of-several-simulator-pods)

# Overview
[This document](https://github.com/cloud-helpers/kubernetes-hard-way-bare-metal/blob/master/lxc/README.md)
aims at providing a full hands-on guide to set up a
[Kubernetes (aka K8S)](https://kubernetes.io) cluster on
[Proxmox-based](https://www.proxmox.com/en/proxmox-ve/features)
[LXC containers](https://linuxcontainers.org/#LXC).
Using [LXD](https://linuxcontainers.org/#LXD) or [OpenVZ](https://openvz.org)
rather than Proxmox should not make much difference.

It is an adaptation of the excellent
["Kubernetes The Hard Way - Bare Metal"
guide](https://github.com/Praqma/LearnKubernetes/blob/master/kamran/Kubernetes-The-Hard-Way-on-BareMetal.md),
itself derived from the famous
["Kubernetes The Har Way"
guide](https://github.com/kelseyhightower/kubernetes-the-hard-way),
by [Kelsey Hightower](https://github.com/kelseyhightower).

The "Bare Metal" version of the "Kubernetes The Hard Way" guide explains
how to install a Kubernetes cluster on Vagrant/`libvirt`-based
[KVM](https://en.wikipedia.org/wiki/Kernel-based_Virtual_Machine)/[QEMU](https://en.wikipedia.org/wiki/QEMU)
virtual machines (VM), and begins to show its age. At the time of writing
(May 2019), most of the details are outdated.

This guide also makes use of the
["Run kubernetes inside LXC container"
article](https://medium.com/@kvaps/run-kubernetes-in-lxc-container-f04aa94b6c9c),
for the adaptation from KVM/QEMU virtual machines (VM) to LXC containers.
Basically, a few more options have to be set up on the (Proxmox) host
so that the (LXC) containers may run Docker inside.

Overall, this guide is therefore both an update for the
"Kubernetes The Hard Way" guides and an adpation for light-weight
containers (rather than full virtual machines (VM)).

While there are quite a few articles and solutions on Kubernetes and similar
clusters in Docker containers (_e.g._, [KinD](https://kind.sigs.k8s.io),
[Footloose](https://github.com/weaveworks/footloose)), the literature remains
scarce about Kubernetes deployed on container technologies
such as Linux containers (_e.g._, LXC, OpenVZ).
However, Linux containers allow for far more flexibility than Docker
containers. Basically, Linux containers aim at executing full-blown (Linux)
operating system (OS), whereas Docker containers aim at micro-services
and serverless platforms, mainly servicing single processes.

This guide therefore aims at helping filling the gap of documentation in
the Linux container space.

All the nodes are setup with [CentOS distributions](https://www.centos.org),
and insulated thanks to a gateway: all the traffic from outside
the cluster is channelled through the gateway. The [set up of such
a gateway](https://github.com/cloud-helpers/kubernetes-hard-way-bare-metal/blob/master/proxmox/README.md)
is also an addition to this guide. It allows
one to experiment with Kubernetes clusters, including operating some
in production-like settings, while keeping some good level of security.

As many other documentations, that one will soon be outdated, and imperfect
for sure. Contributions are therefore welcome to complemenent that guide.
For instance, through
[issues](https://github.com/cloud-helpers/kubernetes-hard-way-bare-metal/issues) or
[pull requests](https://github.com/cloud-helpers/kubernetes-hard-way-bare-metal/pulls).

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

# CFSSL

## CFSSL software
```bash
root@gwkublxc:~# mkdir -p /opt/cfssl
root@gwkublxc:~# wget https://pkg.cfssl.org/R1.2/cfssl_linux-amd64 -O /opt/cfssl/cfssl_linux-amd64
root@gwkublxc:~# wget https://pkg.cfssl.org/R1.2/cfssljson_linux-amd64 -O /opt/cfssl/cfssljson_linux-amd64
root@gwkublxc:~# chmod +x /opt/cfssl/cfssl_linux-amd64 /opt/cfssl/cfssljson_linux-amd64
root@gwkublxc:~# install /opt/cfssl/cfssl_linux-amd64 /usr/local/bin/cfssl
root@gwkublxc:~# install /opt/cfssl/cfssljson_linux-amd64 /usr/local/bin/cfssljson
```

## CA certificates
```bash
root@gwkublxc:~# mkdir -p /opt/cfssl/etc && cd /opt/cfssl/etc
root@gwkublxc:~# cat > /opt/cfssl/etc/ca-config.json << _EOF
{
  "signing": {
    "default": {
      "expiry": "8760h"
    },
    "profiles": {
      "kubernetes": {
        "usages": ["signing", "key encipherment", "server auth", "client auth"],
        "expiry": "8760h"
      }
    }
  }
}
_EOF
```

```bash
root@gwkublxc:~# cat > /opt/cfssl/etc/ca-csr.json << _EOF
{
  "CN": "Kubernetes",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "NO",
      "L": "Oslo",
      "O": "Kubernetes",
      "OU": "CA",
      "ST": "Oslo"
    }
  ]
}
_EOF
```

```bash
root@gwkublxc:~# cfssl gencert -initca /opt/cfssl/etc/ca-csr.json | \
 cfssljson -bare ca
2019/05/11 14:39:26 [INFO] generating a new CA key and certificate from CSR
2019/05/11 14:39:26 [INFO] generate received request
2019/05/11 14:39:26 [INFO] received CSR
2019/05/11 14:39:26 [INFO] generating key: rsa-2048
2019/05/11 14:39:26 [INFO] encoded CSR
2019/05/11 14:39:26 [INFO] signed certificate with serial number 53..53
root@gwkublxc:~# openssl x509 -in /opt/cfssl/etc/ca.pem -text -noout
Certificate:
    Data:
        Version: 3 (0x2)
        Serial Number:
            5d:..:71
    Signature Algorithm: sha256WithRSAEncryption
        Issuer: C=NO, ST=Oslo, L=Oslo, O=Kubernetes, OU=CA, CN=Kubernetes
        Validity
            Not Before: May 11 12:34:00 2019 GMT
            Not After : May  9 12:34:00 2024 GMT
        Subject: C=NO, ST=Oslo, L=Oslo, O=Kubernetes, OU=CA, CN=Kubernetes
        Subject Public Key Info:
            Public Key Algorithm: rsaEncryption
                Public-Key: (2048 bit)
                Modulus:
                    00:e1:42:3b:8b:96:81:bf:3a:00:80:17:8c:8e:48:
					...
                    d8:99
                Exponent: 65537 (0x10001)
        X509v3 extensions:
            X509v3 Key Usage: critical
                Certificate Sign, CRL Sign
            X509v3 Basic Constraints: critical
                CA:TRUE, pathlen:2
            X509v3 Subject Key Identifier: 
                3B:3D:FA:5A:CA:4D:7E:5A:66:72:92:34:0D:9D:CB:E8:38:C3:99:86
            X509v3 Authority Key Identifier: 
                keyid:3B:3D:FA:3A:CA:7D:7E:5A:66:72:92:34:0D:9D:CB:E8:38:C3:99:86

    Signature Algorithm: sha256WithRSAEncryption
         c5:fa:d8:50:f7:ec:13:f1:2c:68:d5:dd:c9:67:b9:d9:47:cd:
		 ...
         37:93:6e:fd
```

## Node certificates
```bash
root@gwkublxc:~# cat > /opt/cfssl/etc/kubernetes-csr.json << _EOF
{
  "CN": "*.example.com",
  "hosts": [
    "10.32.0.1",
    "10.240.0.200",
    "10.240.0.201",
    "10.240.0.202",
    "kub-master.example.com",
    "kub-node1.example.com",
    "kub-node2.example.com",
    "etcd1",
    "etcd2",
    "etcd1.example.com",
    "etcd2.example.com",
    "10.240.0.11",
    "10.240.0.12",
    "controller1",
    "controller2",
    "controller1.example.com",
    "controller2.example.com",
    "10.240.0.21",
    "10.240.0.22",
    "worker1",
    "worker2",
    "worker3",
    "worker4",
    "worker1.example.com",
    "worker2.example.com",
    "worker3.example.com",
    "worker4.example.com",
    "10.240.0.31",
    "10.240.0.32",
    "10.240.0.33",
    "10.240.0.34",
    "controller.example.com",
    "kubernetes.example.com",
    "${KUBERNETES_PUBLIC_IP_ADDRESS}",
    "lb",
    "lb1",
    "lb2",
    "lb.example.com",
    "lb1.example.com",
    "lb2.example.com",
    "10.240.0.40",
    "10.240.0.41",
    "10.240.0.42",
    "gwkublxc",
    "gwkublxc.example.com",
    "10.240.0.2",
    "147.135.185.243",
    "localhost",
    "127.0.0.1"
  ],
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "NO",
      "L": "Oslo",
      "O": "Kubernetes",
      "OU": "Cluster",
      "ST": "Oslo"
    }
  ]
}
_EOF
```

```bash
root@gwkublxc:~# cfssl gencert \
  -ca=/opt/cfssl/etc/ca.pem \
  -ca-key=/opt/cfssl/etc/ca-key.pem \
  -config=/opt/cfssl/etc/ca-config.json \
  -profile=kubernetes \
  /opt/cfssl/etc/kubernetes-csr.json | cfssljson -bare kubernetes
2019/05/11 15:48:07 [INFO] generate received request
2019/05/11 15:48:07 [INFO] received CSR
2019/05/11 15:48:07 [INFO] generating key: rsa-2048
2019/05/11 15:48:07 [INFO] encoded CSR
2019/05/11 15:48:07 [INFO] signed certificate with serial number 31..24
2019/05/11 15:48:07 [WARNING] This certificate lacks a "hosts" field. This makes it unsuitable for
websites. For more information see the Baseline Requirements for the Issuance and Management
of Publicly-Trusted Certificates, v.1.1.6, from the CA/Browser Forum (https://cabforum.org);
specifically, section 10.2.3 ("Information Requirements").
root@gwkublxc:~# openssl x509 -in /opt/cfssl/etc/kubernetes.pem -text -noout
Certificate:
    Data:
        Version: 3 (0x2)
        Serial Number:
            37:79:27:bc:bf:35:e8:f7:40:58:f9:03:73:ac:38:86:18:bd:ee:68
    Signature Algorithm: sha256WithRSAEncryption
        Issuer: C=NO, ST=Oslo, L=Oslo, O=Kubernetes, OU=CA, CN=Kubernetes
        Validity
            Not Before: May 11 13:43:00 2019 GMT
            Not After : May 10 13:43:00 2020 GMT
        Subject: C=NO, ST=Oslo, L=Oslo, O=Kubernetes, OU=Cluster, CN=*.example.com
        Subject Public Key Info:
            Public Key Algorithm: rsaEncryption
                Public-Key: (2048 bit)
                Modulus:
                    00:d9:49:54:5d:4b:81:55:20:13:ff:61:a3:a3:79:
					...
                    5a:95
                Exponent: 65537 (0x10001)
        X509v3 extensions:
            X509v3 Key Usage: critical
                Digital Signature, Key Encipherment
            X509v3 Extended Key Usage: 
                TLS Web Server Authentication, TLS Web Client Authentication
            X509v3 Basic Constraints: critical
                CA:FALSE
            X509v3 Subject Key Identifier: 
                A6:3F:76:BE:4E:C1:42:6E:43:F2:13:79:A1:B8:54:0D:B8:BB:48:C2
            X509v3 Authority Key Identifier: 
                keyid:3B:5D:FA:5A:CA:7D:7E:5A:66:72:92:34:0D:9D:CB:E8:38:C3:99:86

            X509v3 Subject Alternative Name: 
                DNS:etcd1, ..., DNS:localhost, IP Address:10.32.0.1, ..., IP Address:127.0.0.1
    Signature Algorithm: sha256WithRSAEncryption
         c9:ad:3a:16:c7:8c:56:f0:9a:ec:4c:77:72:18:c7:26:34:ae:
		 ...
         ae:84:75:d7
```

# Creation and basic setup of the LXC containers

* Back to the (Proxmox) host:
```bash
root@gwkublxc:~# exit
root@proxmox:~$
```

## Configuration management (`etcd`) servers
* `etcd1`:
```bash
root@proxmox:~$ pct create 211 local:vztmpl/centos-7-default_20171212_amd64.tar.xz --arch amd64 --cores 1 --hostname etcd1.example.com --memory 1024 --swap 2048 --net0 name=eth0,bridge=vmbr3,gw=10.240.0.2,ip=10.240.0.11/24,type=veth --onboot 1 --ostype centos
root@proxmox:~$ pct resize 211 rootfs 10G
root@proxmox:~$ pct start 211 && pct enter 211
root@etcd1# yum -y update && yum -y install epel-release
root@etcd1# yum -y install net-tools bind-utils whois file less htop git \
    screen rpmconf man-db wget curl rsync openssh-server
root@etcd1# rpmconf -a
root@etcd1# systemctl start sshd.service && systemctl enable sshd.service
root@etcd1# mkdir -p ~/.ssh && chmod 700 ~/.ssh && \
cat >> ~/.ssh/authorized_keys << _EOF
ssh-rsa AAAxxxZZZ k8s@example.com
_EOF
chmod 600 ~/.ssh/authorized_keys
root@etcd1# exit
```

* `etcd2`:
```bash
root@proxmox:~$ pct create 212 local:vztmpl/centos-7-default_20171212_amd64.tar.xz --arch amd64 --cores 1 --hostname etcd2.example.com --memory 1024 --swap 2048 --net0 name=eth0,bridge=vmbr3,gw=10.240.0.2,ip=10.240.0.12/24,type=veth --onboot 1 --ostype centos
root@proxmox:~$ pct resize 212 rootfs 10G
root@proxmox:~$ pct start 212 && pct enter 212
root@etcd1# yum -y update && yum -y install epel-release
root@etcd1# yum -y install net-tools bind-utils whois file less htop git \
    screen rpmconf man-db wget curl rsync openssh-server
root@etcd1# rpmconf -a
root@etcd1# systemctl start sshd.service && systemctl enable sshd.service
root@etcd1# mkdir -p ~/.ssh && chmod 700 ~/.ssh && \
cat >> ~/.ssh/authorized_keys << _EOF
ssh-rsa AAAxxxZZZ k8s@example.com
_EOF
chmod 600 ~/.ssh/authorized_keys
root@etcd1# exit
```

## Controller plane servers
* `controller`:
```bash
root@proxmox:~$ pct create 220 local:vztmpl/centos-7-default_20171212_amd64.tar.xz --arch amd64 --cores 2 --hostname controller.example.com --memory 2048 --swap 4096 --net0 name=eth0,bridge=vmbr3,gw=10.240.0.2,ip=10.240.0.20/24,type=veth --onboot 1 --ostype centos
root@proxmox:~$ pct resize 220 rootfs 10G
```

* `controller1`:
```bash
root@proxmox:~$ pct create 221 local:vztmpl/centos-7-default_20171212_amd64.tar.xz --arch amd64 --cores 2 --hostname controller1.example.com --memory 2048 --swap 4096 --net0 name=eth0,bridge=vmbr3,gw=10.240.0.2,ip=10.240.0.21/24,type=veth --onboot 1 --ostype centos
root@proxmox:~$ pct resize 221 rootfs 10G
root@proxmox:~$ pct start 221 && pct enter 221
root@controller1# yum -y update && yum -y install epel-release
root@controller1# yum -y install net-tools bind-utils whois file less htop git \
    screen rpmconf man-db wget curl rsync openssh-server
root@controller1# rpmconf -a
root@controller1# systemctl start sshd.service && systemctl enable sshd.service
root@controller1# mkdir -p ~/.ssh && chmod 700 ~/.ssh && \
cat >> ~/.ssh/authorized_keys << _EOF
ssh-rsa AAAxxxZZZ k8s@example.com
_EOF
chmod 600 ~/.ssh/authorized_keys
root@controller1# exit
```

* `controller2`:
```bash
root@proxmox:~$ pct create 222 local:vztmpl/centos-7-default_20171212_amd64.tar.xz --arch amd64 --cores 2 --hostname controller2.example.com --memory 2048 --swap 4096 --net0 name=eth0,bridge=vmbr3,gw=10.240.0.2,ip=10.240.0.22/24,type=veth --onboot 1 --ostype centos
root@proxmox:~$ pct resize 222 rootfs 10G
root@proxmox:~$ pct start 222 && pct enter 222
root@controller2# yum -y update && yum -y install epel-release
root@controller2# yum -y install net-tools bind-utils whois file less htop git \
    screen rpmconf man-db wget curl rsync openssh-server
root@controller2# rpmconf -a
root@controller2# systemctl start sshd.service && systemctl enable sshd.service
root@controller2# mkdir -p ~/.ssh && chmod 700 ~/.ssh && \
cat >> ~/.ssh/authorized_keys << _EOF
ssh-rsa AAAxxxZZZ k8s@example.com
_EOF
chmod 600 ~/.ssh/authorized_keys
root@controller2# exit
```

## Worker nodes
* From Kubernetes v1.15 (June 2019), it seems that `kubelet` needs to access
  `/dev/kmsg` to work properly. As that device has been dropped from Proxmox LXC,
  it needs to be created for every worker container. First, a `mount-hook.sh`
  is added, and that helper script is invoked from the LXC ocnfiguration files.
```bash
root@proxmox$ ls -laFh /dev/kmsg
crw-r--r-- 1 root root 1, 11 Jun 23 01:10 /dev/kmsg
root@proxmox$ cat > /var/lib/lxc/231/mount-hook.sh << _EOF
#!/bin/sh

mknod -m 777 ${LXC_ROOTFS_MOUNT}/dev/kmsg c 1 11
_EOF
```

* Derive the short version of the hostname:
```bash
root@proxmox$ myhost=$(hostname|cut -d'.' -f1,1)
```

* `worker1`:
```bash
root@proxmox:~$ pct create 231 local:vztmpl/centos-7-default_20171212_amd64.tar.xz --arch amd64 --cores 8 --hostname worker1.example.com --memory 16384 --swap 16384 --net0 name=eth0,bridge=vmbr3,gw=10.240.0.2,ip=10.240.0.31/24,type=veth --onboot 1 --ostype centos
root@proxmox:~$ pct resize 231 rootfs 20G
root@proxmox$ cat >> /etc/pve/nodes/${myhost}/lxc/231.conf << _EOF
lxc.apparmor.profile: unconfined
lxc.cap.drop: 
lxc.cgroup.devices.allow: a
lxc.mount.auto: proc:rw sys:rw
lxc.cgroup.devices.allow: c 1:11 rwm
lxc.hook.autodev: /var/lib/lxc/231/mount-hook.sh
lxc.mount.entry: /dev/kmsg dev/kmsg none bind,optional 0 0
_EOF
root@proxmox:~$ pct start 231 && pct enter 231
root@worker1# yum -y update && yum -y install epel-release
root@worker1# yum -y install net-tools bind-utils whois file less htop git \
    screen rpmconf man-db wget curl rsync openssh-server
root@worker1# rpmconf -a
root@worker1# systemctl start sshd.service && systemctl enable sshd.service
root@worker1# mkdir -p ~/.ssh && chmod 700 ~/.ssh && \
cat >> ~/.ssh/authorized_keys << _EOF
ssh-rsa AAAxxxZZZ k8s@example.com
_EOF
chmod 600 ~/.ssh/authorized_keys
root@worker1# exit
root@proxmox$ # lxc-device add -n 231 /dev/kmsg
```

* `worker2`:
```bash
root@proxmox:~$ pct create 232 local:vztmpl/centos-7-default_20171212_amd64.tar.xz --arch amd64 --cores 8 --hostname worker2.example.com --memory 16384 --swap 16384 --net0 name=eth0,bridge=vmbr3,gw=10.240.0.2,ip=10.240.0.32/24,type=veth --onboot 1 --ostype centos
root@proxmox:~$ pct resize 232 rootfs 20G
root@proxmox$ cat >> /etc/pve/nodes/${myhost}/lxc/232.conf << _EOF
lxc.apparmor.profile: unconfined
lxc.cap.drop: 
lxc.cgroup.devices.allow: a
lxc.mount.auto: proc:rw sys:rw
lxc.cgroup.devices.allow: c 1:11 rwm
lxc.hook.autodev: /var/lib/lxc/231/mount-hook.sh
lxc.mount.entry: /dev/kmsg dev/kmsg none bind,optional 0 0
_EOF
root@proxmox:~$ pct start 232 && pct enter 232
root@worker2# yum -y update && yum -y install epel-release
root@worker2# yum -y install net-tools bind-utils whois file less htop git \
    screen rpmconf man-db wget curl rsync openssh-server
root@worker2# rpmconf -a
root@worker2# systemctl start sshd.service && systemctl enable sshd.service
root@worker2# mkdir -p ~/.ssh && chmod 700 ~/.ssh && \
cat >> ~/.ssh/authorized_keys << _EOF
ssh-rsa AAAxxxZZZ k8s@example.com
_EOF
chmod 600 ~/.ssh/authorized_keys
root@worker2# exit
root@proxmox$ # lxc-device add -n 232 /dev/kmsg
```

## Load balancers (lb)
* `lb`:
```bash
root@proxmox:~$ pct create 240 local:vztmpl/centos-7-default_20171212_amd64.tar.xz --arch amd64 --cores 1 --hostname lb.example.com --memory 512 --swap 1024 --net0 name=eth0,bridge=vmbr3,gw=10.240.0.2,ip=10.240.0.40/24,type=veth --onboot 1 --ostype centos
root@proxmox:~$ # pct resize 240 rootfs 4G
root@proxmox:~$ pct start 240 && pct enter 240
root@lb# yum -y update && yum -y install epel-release
root@lb# yum -y install net-tools bind-utils whois file less htop git \
    screen rpmconf man-db wget curl rsync openssh-server
root@lb# rpmconf -a
root@lb# systemctl start sshd.service && systemctl enable sshd.service
root@lb# mkdir -p ~/.ssh && chmod 700 ~/.ssh && \
cat >> ~/.ssh/authorized_keys << _EOF
ssh-rsa AAAxxxZZZ k8s@example.com
_EOF
chmod 600 ~/.ssh/authorized_keys
root@lb# exit
```

* `lb1`:
```bash
root@proxmox:~$ pct create 241 local:vztmpl/centos-7-default_20171212_amd64.tar.xz --arch amd64 --cores 1 --hostname lb1.example.com --memory 512 --swap 1024 --net0 name=eth0,bridge=vmbr3,gw=10.240.0.2,ip=10.240.0.41/24,type=veth --onboot 1 --ostype centos
root@proxmox:~$ # pct resize 241 rootfs 4G
root@proxmox:~$ pct start 241 && pct enter 241
root@lb1# yum -y update && yum -y install epel-release
root@lb1# yum -y install net-tools bind-utils whois file less htop git \
    screen rpmconf man-db wget curl rsync openssh-server
root@lb1# rpmconf -a
root@lb1# systemctl start sshd.service && systemctl enable sshd.service
root@lb1# mkdir -p ~/.ssh && chmod 700 ~/.ssh && \
cat >> ~/.ssh/authorized_keys << _EOF
ssh-rsa AAAxxxZZZ k8s@example.com
_EOF
chmod 600 ~/.ssh/authorized_keys
root@lb1# exit
```

* `lb2`:
```bash
root@proxmox:~$ pct create 242 local:vztmpl/centos-7-default_20171212_amd64.tar.xz --arch amd64 --cores 1 --hostname lb2.example.com --memory 512 --swap 1024 --net0 name=eth0,bridge=vmbr3,gw=10.240.0.2,ip=10.240.0.42/24,type=veth --onboot 1 --ostype centos
root@proxmox:~$ # pct resize 242 rootfs 4G
root@proxmox:~$ pct start 242 && pct enter 242
root@lb2# yum -y update && yum -y install epel-release
root@lb2# yum -y install net-tools bind-utils whois file less htop git \
    screen rpmconf man-db wget curl rsync openssh-server
root@lb2# rpmconf -a
root@lb2# systemctl start sshd.service && systemctl enable sshd.service
root@lb2# mkdir -p ~/.ssh && chmod 700 ~/.ssh && \
cat >> ~/.ssh/authorized_keys << _EOF
ssh-rsa AAAxxxZZZ k8s@example.com
_EOF
chmod 600 ~/.ssh/authorized_keys
root@lb2# exit
```

## On the clients (eg, laptops)
* SSH configuration:
```bash
user@laptop$ cat >> ~/.ssh/config << _EOF

# Kubernetes cluster on Proxmox/LXC
Host gw.kublxc
  HostName gwkublxc.example.com
  Port 7022
Host etcd1
  HostName etcd1.example.com
  ProxyCommand ssh -W %h:22 root@gw.kublxc
Host etcd2
  HostName etcd2.example.com
  ProxyCommand ssh -W %h:22 root@gw.kublxc
Host controller
   HostName controller.example.com
   ProxyCommand ssh -W %h:22 root@gw.kublxc
Host controller1
   HostName controller1.example.com
   ProxyCommand ssh -W %h:22 root@gw.kublxc
Host controller2
   HostName controller2.example.com
   ProxyCommand ssh -W %h:22 root@gw.kublxc
Host worker1
  HostName worker1.example.com
  ProxyCommand ssh -W %h:22 root@gw.kublxc
Host worker2
  HostName worker2.example.com
  ProxyCommand ssh -W %h:22 root@gw.kublxc
Host lb
  HostName lb.example.com
  ProxyCommand ssh -W %h:22 root@gw.kublxc
Host lb1
  HostName lb1.example.com
  ProxyCommand ssh -W %h:22 root@gw.kublxc
Host lb2
  HostName lb2.example.com
  ProxyCommand ssh -W %h:22 root@gw.kublxc

_EOF
```

* Upload the SSH keys onto the K8S gateway:
```bash
user@laptop$ rsync -av your-ssh-keys root@gw.kublxc:~/.ssh/
```

## List of the Kubernetes nodes
```bash
root@gwkublxc# declare -a node_list=("lb" "lb1" "lb2" "etcd1" "etcd2" "controller" "controller1" "controller2" "worker1" "worker2")
```

## Install the Kubernetes cluster certificates on the Kubernetes nodes
* Push the certificates to every K8S node, from the gateway:
```bash
root@gwkublxc# chmod 644 /opt/cfssl/etc/kubernetes-key.pem
root@gwkublxc# declare -a cert_list=("/opt/cfssl/etc/ca.pem" "/opt/cfssl/etc/kubernetes-key.pem" "/opt/cfssl/etc/kubernetes.pem")
root@gwkublxc# for node in "${node_list[@]}"
do
  for cert in "${cert_list[@]}"
  do
    rsync -av ${cert} root@${node}:/root/
  done
done
```

## Complementary packages for the Kubernetes nodes
* Install a few complementary packages:
```bash
root@gwkublxc# for node in "${node_list[@]}"; do \
  ssh root@${node} "echo \"Updating ${node}...\" && \
    yum -y update && yum -y install epel-release && \
    yum -y install rpmconf yum-utils htop wget curl file less man-db net-tools \
      whois bzip2 rsync bash-completion bash-completion-extras screen htop \
	  openssh-server ntp git jq"; done
root@gwkublxc# for node in "${node_list[@]}"; do \
  ssh root@${node} "ln -sf /usr/share/zoneinfo/Europe/Paris /etc/localtime && \
    systemctl start ntpd && systemctl enable ntpd && \
    setenforce 0 && \
    sed -i --follow-symlinks 's/SELINUX=enforcing/SELINUX=disabled/g' /etc/sysconfig/selinux"; done
```

## Basic checks
* On the host:
```bash
root@gwkublxc# for node in "${node_list[@]}"; do \
  ssh root@${node} "hostname; getenforce"; done
lb.telecoms-intelligence.com
Disabled
lb1.telecoms-intelligence.com
Disabled
lb2.telecoms-intelligence.com
Disabled
etcd1.telecoms-intelligence.com
Disabled
etcd2.telecoms-intelligence.com
Disabled
controller.telecoms-intelligence.com
Disabled
controller1.telecoms-intelligence.com
Disabled
controller2.telecoms-intelligence.com
Disabled
worker1.telecoms-intelligence.com
Disabled
worker2.telecoms-intelligence.com
Disabled
```

# Set up of the configuration management (`etcd`) servers
* References:
 + https://github.com/etcd-io/etcd/blob/master/Documentation/demo.md
 + https://github.com/etcd-io/etcd/blob/master/Documentation/op-guide/security.md

* Define the `etcd` nodes:
```bash
root@gwkublxc:~# declare -a node_etc_list=("etcd1" "etcd2")
root@gwkublxc:~# declare -a nodeip_etc_list=("10.240.0.11" "10.240.0.12")
```

* Write an `etcd` service script on the gateway, with place holders
  for the variables (depending on actual nodes):
```bash
root@gwkublxc:~# mkdir -p /opt/etcd && cat > /opt/etcd/etcd.service << _EOF
[Unit]
Description=Etcd Server
After=network.target
After=network-online.target
Wants=network-online.target

[Service]
Type=notify
WorkingDirectory=/var/lib/etcd/
EnvironmentFile=-/etc/etcd/etcd.conf
User=etcd
# set GOMAXPROCS to number of processors
ExecStart=/bin/bash -c "GOMAXPROCS=\$(nproc) /usr/bin/etcd \\
  --name \"ETCD_NAME\" \\
  --data-dir=\"\${ETCD_DATA_DIR}\" \\
  --cert-file=\"/etc/etcd/kubernetes.pem\" \\
  --key-file=\"/etc/etcd/kubernetes-key.pem\" \\
  --peer-cert-file=\"/etc/etcd/kubernetes.pem\" \\
  --peer-key-file=\"/etc/etcd/kubernetes-key.pem\" \\
  --trusted-ca-file=\"/etc/etcd/ca.pem\" \\
  --peer-trusted-ca-file=\"/etc/etcd/ca.pem\" \\
  --initial-advertise-peer-urls \"https://INTERNAL_IP:2380\" \\
  --listen-peer-urls \"https://INTERNAL_IP:2380\" \\
  --advertise-client-urls \"https://INTERNAL_IP:2379\" \\
  --listen-client-urls \"https://INTERNAL_IP:2379,http://127.0.0.1:2379\" \\
  --initial-cluster-token \"etcd-cluster-0\" \\
  --initial-cluster \"etcd1=https://10.240.0.11:2380,etcd2=https://10.240.0.12:2380\" \\
  --initial-cluster-state \"new\""
Restart=on-failure
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target
_EOF
```

* Deploy and intanciate the `etcd` service scripts:
```bash
root@gwkublxc:~# for node in "${node_etc_list[@]}"; do ssh root@${node} "\
  mkdir -p /etc/etcd/ && \
  mv ca.pem kubernetes-key.pem kubernetes.pem /etc/etcd/"; done
root@gwkublxc:~# for node in "${node_etc_list[@]}"; do ssh root@${node} "\
  yum -y install etcd"; done
root@gwkublxc:~# for node in "${node_etc_list[@]}"; do \
  rsync -av -p /opt/etcd/etcd.service root@${node}:/usr/lib/systemd/system/; done
root@gwkublxc:~# for i in "${!node_etc_list[@]}"; do \
  ssh root@${nodeip_etc_list[$i]} "\
    sed -i -e s/INTERNAL_IP/${nodeip_etc_list[$i]}/g \
	       -e s/ETCD_NAME/${node_etc_list[$i]}/g /usr/lib/systemd/system/etcd.service"; done
root@gwkublxc:~# for node in "${node_etc_list[@]}"; do ssh root@${node} "\
  systemctl daemon-reload && systemctl enable etcd && \
  systemctl start etcd && systemctl status etcd -l"; done
```

* Check the setup from the gateway:
```bash
root@gwkublxc:~# cp -a /opt/cfssl/etc/ca.pem /etc/etcd/
root@gwkublxc:~# ENDPOINTS="https://10.240.0.11:2379,https://10.240.0.12:2379"
root@gwkublxc:~# etcdctl --ca-file=/etc/etcd/ca.pem --endpoints=$ENDPOINTS cluster-health
member 3a57933972cb5131 is healthy: got healthy result from https://10.240.0.12:2379
member ffed16798470cab5 is healthy: got healthy result from https://10.240.0.11:2379
cluster is healthy
root@gwkublxc:~# etcdctl --ca-file=/etc/etcd/ca.pem --endpoints=$ENDPOINTS member list
3a57933972cb5131: name=etcd2 peerURLs=https://10.240.0.12:2380 clientURLs=https://10.240.0.12:2379 isLeader=true
ffed16798470cab5: name=etcd1 peerURLs=https://10.240.0.11:2380 clientURLs=https://10.240.0.11:2379 isLeader=false
root@gwkublxc:~# cat >> ~/.bashrc << _EOF

# etcd cluster
ENDPOINTS="https://10.240.0.11:2379,https://10.240.0.12:2379"

_EOF
root@gwkublxc:~# cat >> ~/.bash_aliases << _EOF

# etcd cluster
alias etcdgethealth='etcdctl --ca-file=/etc/etcd/ca.pem --endpoints=$ENDPOINTS cluster-health'
alias etcdgetlistmembers='etcdctl --ca-file=/etc/etcd/ca.pem --endpoints=$ENDPOINTS member list'
alias etcdget='etcdctl --ca-file=/etc/etcd/ca.pem --endpoints=$ENDPOINTS get '
alias etcdset='etcdctl --ca-file=/etc/etcd/ca.pem --endpoints=$ENDPOINTS set '

_EOF
```

* Check setting and getting values from the gateway:
```bash
root@gwkublxc:~# cp /var/lib/kubernetes/*.pem /etc/etcd/
root@gwkublxc:~# curl --cacert /etc/etcd/ca.pem --cert /etc/etcd/kubernetes.pem --key /etc/etcd/kubernetes-key.pem -L https://10.240.0.11:2379/v2/keys/foo -XPUT -d value=bar -v
* About to connect() to 10.240.0.11 port 2379 (#0)
*   Trying 10.240.0.11...
* Connected to 10.240.0.11 (10.240.0.11) port 2379 (#0)
* Initializing NSS with certpath: sql:/etc/pki/nssdb
*   CAfile: /etc/etcd/ca.pem
  CApath: none
* SSL connection using TLS_ECDHE_RSA_WITH_AES_256_GCM_SHA384
* Server certificate:
* 	subject: CN=*.example.com,OU=Cluster,O=Kubernetes,L=Oslo,ST=Oslo,C=NO
* 	start date: May 11 13:43:00 2019 GMT
* 	expire date: May 10 13:43:00 2020 GMT
* 	common name: *.example.com
* 	issuer: CN=Kubernetes,OU=CA,O=Kubernetes,L=Oslo,ST=Oslo,C=NO
> PUT /v2/keys/foo HTTP/1.1
> User-Agent: curl/7.29.0
> Host: 10.240.0.11:2379
> Accept: */*
> Content-Length: 9
> Content-Type: application/x-www-form-urlencoded
> 
* upload completely sent off: 9 out of 9 bytes
< HTTP/1.1 201 Created
< Content-Type: application/json
< X-Etcd-Cluster-Id: cdeaba18114f0e16
< X-Etcd-Index: 14
< X-Raft-Index: 20816
< X-Raft-Term: 22
< Date: Thu, 23 May 2019 18:17:23 GMT
< Content-Length: 90
< 
{"action":"set","node":{"key":"/foo","value":"bar","modifiedIndex":14,"createdIndex":14}}
* Connection #0 to host 10.240.0.11 left intact
root@gwkublxc:~# curl --cacert /etc/etcd/ca.pem --cert /etc/etcd/kubernetes.pem --key /etc/etcd/kubernetes-key.pem https://10.240.0.11:2379/v2/keys/foo
{"action":"get","node":{"key":"/foo","value":"bar","modifiedIndex":14,"createdIndex":14}}
```

## Check the setup on each `etcd` node
* On `etcd1`:
```bash
root@gwkublxc:~# ssh etcd1
root@etcd1:~# systemctl status etcd --no-pager
root@etcd1:~# netstat -antlp
Active Internet connections (servers and established)
Proto Recv-Q Send-Q Local Address           Foreign Address         State       PID/Program name    
tcp        0      0 10.240.0.11:2379        0.0.0.0:*               LISTEN      4746/etcd           
tcp        0      0 127.0.0.1:2379          0.0.0.0:*               LISTEN      4746/etcd           
tcp        0      0 10.240.0.11:2380        0.0.0.0:*               LISTEN      4746/etcd           
tcp        0      0 0.0.0.0:22              0.0.0.0:*               LISTEN      266/sshd            
tcp        0      0 127.0.0.1:40872         127.0.0.1:2379          ESTABLISHED 4746/etcd           
tcp        0      0 10.240.0.11:22          10.240.0.2:42792        ESTABLISHED 4419/sshd: root@pts 
tcp        0      0 10.240.0.11:39722       10.240.0.11:2379        ESTABLISHED 4746/etcd           
tcp        0      0 10.240.0.11:2379        10.240.0.11:39722       ESTABLISHED 4746/etcd           
tcp        0      0 127.0.0.1:2379          127.0.0.1:40872         ESTABLISHED 4746/etcd           
tcp6       0      0 :::22                   :::*                    LISTEN      266/sshd            
root@etcd1:~# etcdctl --ca-file=/etc/etcd/ca.pem cluster-health
member 8e9e05c52164694d is healthy: got healthy result from https://10.240.0.11:2
379
cluster is healthy
root@etcd1:~# etcdctl cluster-health
failed to check the health of member 8e9e05c52164694d on https://10.240.0.11:2379: Get https://10.240.0.11:2379/health: x509: certificate signed by unknown authority
member 8e9e05c52164694d is unreachable: [https://10.240.0.11:2379] are all unreachable
cluster is unavailable
root@etcd1:~# exit
```

* On `etcd2`:
```bash
root@gwkublxc:~# ssh etcd2
root@etcd2:~# netstat -antlp
Active Internet connections (servers and established)
Proto Recv-Q Send-Q Local Address           Foreign Address         State       PID/Program name    
tcp        0      0 10.240.0.12:2379        0.0.0.0:*               LISTEN      1179/etcd           
tcp        0      0 127.0.0.1:2379          0.0.0.0:*               LISTEN      1179/etcd           
tcp        0      0 10.240.0.12:2380        0.0.0.0:*               LISTEN      1179/etcd           
tcp        0      0 0.0.0.0:22              0.0.0.0:*               LISTEN      389/sshd            
tcp        0      0 127.0.0.1:2379          127.0.0.1:40864         ESTABLISHED 1179/etcd           
tcp        0      0 127.0.0.1:40864         127.0.0.1:2379          ESTABLISHED 1179/etcd           
tcp        0      0 10.240.0.12:22          10.240.0.2:58082        ESTABLISHED 635/sshd: root@pts/ 
tcp        0      0 10.240.0.12:2379        10.240.0.12:60890       ESTABLISHED 1179/etcd           
tcp        0      0 10.240.0.12:60890       10.240.0.12:2379        ESTABLISHED 1179/etcd           
tcp6       0      0 :::22                   :::*                    LISTEN      389/sshd            
root@etcd2:~# etcdctl --ca-file=/etc/etcd/ca.pem cluster-health
member 8e9e05c52164694d is healthy: got healthy result from https://10.240.0.12:2379
cluster is healthy
root@etcd2:~# etcdctl cluster-health
failed to check the health of member 8e9e05c52164694d on https://10.240.0.12:2379: Get https://10.240.0.12:2379/health: x509: certificate signed by unknown authority
member 8e9e05c52164694d is unreachable: [https://10.240.0.12:2379] are all unreachable
cluster is unavailable
root@etcd2:~# exit
```

# Kubernetes API, controller and scheduler servers
* The software and configuration files are downloaded and setup on the
  Kubernetes gateway (`gwkublxc`), and then sent to the nodes from there
```bash
root@gwkublxc:~# K8S_VER=$(curl -s https://storage.googleapis.com/kubernetes-release/release/stable.txt)
root@gwkublxc:~# echo $K8S_VER 
v1.14.3
root@gwkublxc:~# declare -a node_ctrl_list=("controller" "controller1" "controller2")
root@gwkublxc:~# declare -a nodeip_ctrl_list=("10.240.0.20" "10.240.0.21" "10.240.0.22")
root@gwkublxc:~# declare -a node_ext_list=("lb" "lb1" "lb2" "etcd1" "etcd2" "controller" "controller1" "controller2" "worker1" "worker2" "worker3")
root@gwkublxc:~# declare -a nodeip_ext_list=("10.240.0.40" "10.240.0.41" "10.240.0.42" "10.240.0.11" "10.240.0.12" "10.240.0.20" "10.240.0.21" "10.240.0.22" "10.240.0.31" "10.240.0.32" "10.240.0.33")
root@gwkublxc:~# mkdir -p /var/lib/kubernetes && cp -a /opt/cfssl/etc/*.pem /var/lib/kubernetes/
```

## Installation of complementary packages
* Install `etcd` and `jq` on the controller nodes:
```bash
root@gwkublxc:~# for node in "${node_ctrl_list[@]}"; do \
  ssh root@${node} "yum -y update && yum -y install etcd jq"; done
```

* Upload the Kubernetes certificates:
```bash
root@gwkublxc:~# for node in "${node_ext_list[@]}"; do \
  rsync -av -p /opt/cfssl/etc/*.pem root@${node}:/etc/etcd/; done
root@gwkublxc:~# for node in "${node_ext_list[@]}"; do \
  ssh root@${node} "mkdir -p /var/lib/kubernetes && \
    cp -a /etc/etcd/*.pem /var/lib/kubernetes/"; done
```

## Go
At least for [CoreDNS](https://github.com/coredns/coredns),
version 1.12 of Go is required, whereas most of the distributions have older
versions of Go, for instance 1.11.

* Versions (from https://golang.org/dl/):
```bash
VERSION="1.12.6"
OS="linux"
ARCH="amd64"
GOPKG="go${VERSION}.${OS}-${ARCH}.tar.gz"
```

* Download and install the Go binary package on the gateway:
```bash
root@gwkublxc:~# mkdir -p /opt/go/tarballs
root@gwkublxc:~# wget https://dl.google.com/go/${GOPKG} \
  -O /opt/go/tarballs/${GOPKG}
root@gwkublxc:~# tar -C /usr/local -zxf /opt/go/tarballs/${GOPKG} && \
  mv /usr/local/go /usr/local/go${VERSION} && \
  ln -sf /usr/local/go${VERSION} /usr/local/go && mkdir ${HOME}/go
```

* Install Go on all the Kubernetes nodes:
```bash
root@gwkublxc:~# for node in "${node_ext_list[@]}"; do \
  rsync -av -p /opt/go root@${node}:/opt/; done
root@gwkublxc:~# for node in "${node_ext_list[@]}"; do \
  ssh root@${node} "echo \"${node}\" && \
    tar -C /usr/local -zxf /opt/go/tarballs/${GOPKG} && \
    mv /usr/local/go /usr/local/go${VERSION} && \
    ln -sf /usr/local/go${VERSION} /usr/local/go && mkdir ${HOME}/go"; done
```

* On all the nodes, the `~/.bashrc` file has to be updated, like:
```bash
root@gwkublxc# for node in "${node_ext_list[@]}"; do \
  ssh root@${node} "echo \"${node}\" && \
    cat >> ~/.bashrc << _EOF

# Go
export PATH="\${PATH}:/usr/local/go/bin"

_EOF
"; done
root@gwkublxc# for node in "${node_ext_list[@]}"; do \
  ssh root@${node} "echo \"${node}\" && go version"; done
lb
go version go1.12.6 linux/amd64
...
worker2
go version go1.12.6 linux/amd64
```

## Kubernetes binaries
* On the LXC K8S gateway, download Kubernetes:
```bash
root@gwkublxc:~# K8S_VER=$(curl -s https://storage.googleapis.com/kubernetes-release/release/stable.txt)
root@gwkublxc:~# echo $K8S_VER 
v1.15.0
root@gwkublxc:~# declare -a kubbin_list=("kube-apiserver" \
 "kube-controller-manager" "kube-scheduler" "kubectl")
root@gwkublxc:~# for kubbin in "${kubbin_list[@]}"; do \
 curl -LO https://storage.googleapis.com/kubernetes-release/release/${K8S_VER}/bin/linux/amd64/${kubbin} && \
 chmod +x ${kubbin} && mv ${kubbin} /usr/local/bin/; done
```

* Upload the Kubernetes binaries to all the controller nodes:
```bash
root@gwkublxc:~# for node in "${node_ext_list[@]}"; do \
 rsync -av /usr/local/bin/kube* root@${node}:/usr/local/bin/; done
root@gwkublxc:~# for node in "${node_ctrl_list[@]}"; do \
  ssh root@${node} "echo \"${node}\" && kubectl version 2>&1 | \
    grep -v \"The connection\""; done
controller
Client Version: version.Info{Major:"1", Minor:"15", GitVersion:"v1.15.0", GitCommit:"e8462b5b5dc2584fdcd18e6bcfe9f1e4d970a529", GitTreeState:"clean", BuildDate:"2019-06-19T16:40:16Z", GoVersion:"go1.12.5", Compiler:"gc", Platform:"linux/amd64"}
Server Version: version.Info{Major:"1", Minor:"14", GitVersion:"v1.14.3", GitCommit:"5e53fd6bc17c0dec8434817e69b04a25d8ae0ff0", GitTreeState:"clean", BuildDate:"2019-06-06T01:36:19Z", GoVersion:"go1.12.5", Compiler:"gc", Platform:"linux/amd64"}
...
controller2
Client Version: version.Info{Major:"1", Minor:"15", GitVersion:"v1.15.0", GitCommit:"e8462b5b5dc2584fdcd18e6bcfe9f1e4d970a529", GitTreeState:"clean", BuildDate:"2019-06-19T16:40:16Z", GoVersion:"go1.12.5", Compiler:"gc", Platform:"linux/amd64"}
Server Version: version.Info{Major:"1", Minor:"14", GitVersion:"v1.14.3", GitCommit:"5e53fd6bc17c0dec8434817e69b04a25d8ae0ff0", GitTreeState:"clean", BuildDate:"2019-06-06T01:36:19Z", GoVersion:"go1.12.5", Compiler:"gc", Platform:"linux/amd64"}
```

## Kubernetes certificates
```bash
root@gwkublxc:~# for node in "${node_list[@]}"; do \
 ssh root@${node} "mkdir -p /var/lib/kubernetes/ && \
 mv ca.pem kubernetes-key.pem kubernetes.pem /var/lib/kubernetes/"; done
```

## Kubernetes authentication token
* Create the token on the gateway:
```bash
root@gwkublxc:~# mkdir -p /var/lib/kubernetes && \
 cat > /var/lib/kubernetes/token.csv << _EOF
chAng3m3,admin,admin
chAng3m3,scheduler,scheduler
chAng3m3,kubelet,kubelet
_EOF
```

* Upload the Kubernetes token to all the nodes:
```bash
root@gwkublxc:~# for node in "${node_ext_list[@]}"; do \
 rsync -av /var/lib/kubernetes/token.csv root@${node}:/var/lib/kubernetes/; \
 done
```

## Access Control Lists (ACL)
* Create the JSON file for the Kubernetes authorization policy:
```bash
root@gwkublxc:~# cat > /var/lib/kubernetes/authorization-policy.jsonl << _EOF
{"apiVersion": "abac.authorization.kubernetes.io/v1beta1", "kind": "Policy", "spec": {"user":"*", "nonResourcePath": "*", "readonly": true}}
{"apiVersion": "abac.authorization.kubernetes.io/v1beta1", "kind": "Policy", "spec": {"user":"admin", "namespace": "*", "resource": "*", "apiGroup": "*"}}
{"apiVersion": "abac.authorization.kubernetes.io/v1beta1", "kind": "Policy", "spec": {"user":"scheduler", "namespace": "*", "resource": "*", "apiGroup": "*"}}
{"apiVersion": "abac.authorization.kubernetes.io/v1beta1", "kind": "Policy", "spec": {"user":"kubelet", "namespace": "*", "resource": "*", "apiGroup": "*"}}
{"apiVersion": "abac.authorization.kubernetes.io/v1beta1", "kind": "Policy", "spec": {"group":"system:serviceaccounts", "namespace": "*", "resource": "*", "apiGroup": "*", "nonResourcePath": "*"}}
_EOF
```

* Distribute the JSON policy file:
```bash
root@gwkublxc:~# for node in "${nodeip_ext_list[@]}"; do \
 rsync -av /var/lib/kubernetes/authorization-policy.jsonl root@${node}:/var/lib/kubernetes/; done
```

## Kubernetes API service
* Create the API service file:
```bash
root@gwkublxc:~# cat > /var/lib/kubernetes/kube-apiserver.service << _EOF
[Unit]
Description=Kubernetes API Server
Documentation=https://github.com/GoogleCloudPlatform/kubernetes

[Service]
ExecStart=/usr/local/bin/kube-apiserver \\
  --admission-control=NamespaceLifecycle,LimitRanger,SecurityContextDeny,ServiceAccount,ResourceQuota \\
  --advertise-address=INTERNAL_IP \\
  --allow-privileged=true \\
  --apiserver-count=3 \\
  --authorization-mode=ABAC \\
  --authorization-policy-file=/var/lib/kubernetes/authorization-policy.jsonl \\
  --bind-address=0.0.0.0 \\
  --enable-swagger-ui=true \\
  --etcd-cafile=/var/lib/kubernetes/ca.pem \\
  --insecure-bind-address=0.0.0.0 \\
  --kubelet-certificate-authority=/var/lib/kubernetes/ca.pem \\
  --etcd-servers=https://10.240.0.11:2379,https://10.240.0.12:2379 \\
  --service-account-key-file=/var/lib/kubernetes/kubernetes-key.pem \\
  --service-cluster-ip-range=10.32.0.0/24 \\
  --service-node-port-range=30000-32767 \\
  --tls-cert-file=/var/lib/kubernetes/kubernetes.pem \\
  --tls-private-key-file=/var/lib/kubernetes/kubernetes-key.pem \\
  --token-auth-file=/var/lib/kubernetes/token.csv \\
  --v=2
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
_EOF
```

* Distribute the API service file:
```bash
root@gwkublxc:~# for nodeip in "${nodeip_ext_list[@]}"; do \
 rsync -av /var/lib/kubernetes/kube-apiserver.service root@${nodeip}:/etc/systemd/system/; done
root@gwkublxc:~# for nodeip in "${nodeip_ext_list[@]}"; do \
 ssh root@${nodeip} "sed -ie s/INTERNAL_IP/${nodeip}/g /etc/systemd/system/kube-apiserver.service"; done
root@gwkublxc:~# for nodeip in "${nodeip_list[@]}"; do \
 ssh root@${nodeip} "systemctl daemon-reload && \
  systemctl enable kube-apiserver && \
  systemctl start kube-apiserver && \
  systemctl status kube-apiserver -l"; done
```

## Kubernetes controller manager
* Create the service file:
```bash
root@gwkublxc:~# cat > /var/lib/kubernetes/kube-controller-manager.service << _EOF
[Unit]
Description=Kubernetes Controller Manager
Documentation=https://github.com/GoogleCloudPlatform/kubernetes

[Service]
ExecStart=/usr/local/bin/kube-controller-manager \\
  --allocate-node-cidrs=true \\
  --cluster-cidr=10.200.0.0/16 \\
  --cluster-name=kubernetes \\
  --leader-elect=true \\
  --master=http://INTERNAL_IP:8080 \\
  --root-ca-file=/var/lib/kubernetes/ca.pem \\
  --service-account-private-key-file=/var/lib/kubernetes/kubernetes-key.pem \\
  --service-cluster-ip-range=10.32.0.0/24 \\
  --v=2
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
_EOF
```

* Distribute the service file:
```bash
root@gwkublxc:~# for node in "${nodeip_list[@]}"; do \
 rsync -av /var/lib/kubernetes/kube-controller-manager.service \
  root@${node}:/etc/systemd/system/; done
root@gwkublxc:~# for nodeip in "${nodeip_list[@]}"; do \
 ssh root@${nodeip} "sed -ie s/INTERNAL_IP/${nodeip}/g \
  /etc/systemd/system/kube-controller-manager.service"; done
root@gwkublxc:~# for nodeip in "${nodeip_list[@]}"; do \
 ssh root@${nodeip} "systemctl daemon-reload && \
  systemctl enable kube-controller-manager && \
  systemctl start kube-controller-manager && \
  systemctl status kube-controller-manager -l"; done
```

## Kubernetes scheduler
* Create the service file:
```bash
root@gwkublxc:~# cat > /var/lib/kubernetes/kube-scheduler.service << _EOF
[Unit]
Description=Kubernetes Scheduler
Documentation=https://github.com/GoogleCloudPlatform/kubernetes

[Service]
ExecStart=/usr/local/bin/kube-scheduler \\
  --leader-elect=true \\
  --master=http://INTERNAL_IP:8080 \\
  --v=2
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
_EOF
```

* Distribute the service file:
```bash
root@gwkublxc:~# for node in "${nodeip_list[@]}"; do \
 rsync -av /var/lib/kubernetes/kube-scheduler.service \
  root@${node}:/etc/systemd/system/; done
root@gwkublxc:~# for nodeip in "${nodeip_list[@]}"; do \
 ssh root@${nodeip} "sed -ie s/INTERNAL_IP/${nodeip}/g \
  /etc/systemd/system/kube-scheduler.service"; done
root@gwkublxc:~# for nodeip in "${nodeip_list[@]}"; do \
 ssh root@${nodeip} "systemctl daemon-reload && \
  systemctl enable kube-scheduler && \
  systemctl start kube-scheduler && \
  systemctl status kube-scheduler -l"; done
```

## Restarting the services on the control plane
```bash
root@gwkublxc:~# declare -a kubsvc_list=("kube-apiserver" \
 "kube-controller-manager" "kube-scheduler")
root@gwkublxc:~# for nodeip in "${nodeip_list[@]}"; do \
 for kubsvc in "${kubsvc_list[@]}"; do \
  ssh root@${nodeip} "systemctl restart ${kubsvc} && \
   systemctl status ${kubsvc} -l"; done; done
```

## Helper tools (`kubectx` / `kubens`)
* Kubernetes context helper (`kubectx`) and namespace helper (`kubens`):
```bash
root@gwkublxc:~# git clone https://github.com/ahmetb/kubectx ${HOME}/.kubectx
root@gwkublxc:~# COMPDIR=$(pkg-config --variable=completionsdir bash-completion)
root@gwkublxc:~# ln -sf ${HOME}/.kubectx/completion/kubens.bash ${COMPDIR}/kubens
root@gwkublxc:~# ln -sf ${HOME}/.kubectx/completion/kubectx.bash ${COMPDIR}/kubectx
root@gwkublxc:~# cat >> ~/.bashrc << _EOF

# Kubernetes helper tools
export PATH="\${HOME}/.kubectx:\${PATH}"

_EOF
root@gwkublxc:~# . ~/.bashrc
```

* Install `kubectx` on all the Kubernetes nodes:
```bash
root@gwkublxc:~# for node in "${node_ext_list[@]}"; do \
  rsync -av -p ${HOME}/.kubectx root@${node}:~/; done
root@gwkublxc:~# for node in "${node_ext_list[@]}"; do \
  ssh root@${node} "echo \"${node}\" && \
    COMPDIR=$(pkg-config --variable=completionsdir bash-completion) && \
    ln -sf ~/.kubectx/completion/kubens.bash ${COMPDIR}/kubens && \
    ln -sf ~/.kubectx/completion/kubectx.bash ${COMPDIR}/kubectx"; done
root@gwkublxc2# for node in "${node_ext_list[@]}"; do \
  ssh root@${node} "echo \"${node}\" && \
    cat >> ~/.bashrc << _EOF

# Kubernetes helper tools
export PATH="\${HOME}/.kubectx:\${PATH}"

_EOF
"; done
root@gwkublxc2# for node in "${node_ext_list[@]}"; do \
  ssh root@${node} "echo \"${node}\" && type kubectx"; done
lb
kubectx is /root/.kubectx/kubectx
...
worker3
kubectx is /root/.kubectx/kubectx
```

* Kubernetes prompt (`kube-ps1`):
```bash
root@gwkublxc:~# git clone https://github.com/jonmosco/kube-ps1.git ${HOME}/.kube-ps1
root@gwkublxc:~# cat >> ~/.bashrc << _EOF

# Kubernetes prompt
source ~/.kube-ps1/kube-ps1.sh
PS1='[\u@\h \W \$(kube_ps1)]\$ '
source <(kubectl completion bash)

_EOF
root@gwkublxc:~# . ~/.bashrc
```

* Install `kube-ps1` on all the Kubernetes nodes:
```bash
root@gwkublxc:~# for node in "${node_ext_list[@]}"; do \
  rsync -av -p ${HOME}/.kube-ps1 root@${node}:~/; done
root@gwkublxc:~# for node in "${node_ext_list[@]}"; do \
  ssh root@${node} "echo \"${node}\" && \
    cat >> ~/.bashrc << _EOF

# Kubernetes prompt
source ~/.kube-ps1/kube-ps1.sh
PS1='[\u@\h \W \\\$(kube_ps1)]\$ '
source <(kubectl completion bash)

_EOF
"; done
```

## Check the components
```bash
root@gwkublxc:~# for i in "${!node_ctrl_list[@]}"; do \
  echo "Node: ${node_ctrl_list[$i]} (${nodeip_ctrl_list[$i]})" && \
  ssh root@${node_list[$i]} "kubectl get componentstatuses"; done
Node: controller (10.240.0.20)
NAME                 STATUS    MESSAGE             ERROR
scheduler            Healthy   ok                  
controller-manager   Healthy   ok                  
etcd-0               Healthy   {"health":"true"}   
etcd-1               Healthy   {"health":"true"}   
Node: controller1 (10.240.0.21)
NAME                 STATUS    MESSAGE             ERROR
scheduler            Healthy   ok                  
controller-manager   Healthy   ok                  
etcd-1               Healthy   {"health":"true"}   
etcd-0               Healthy   {"health":"true"}   
Node: controller2 (10.240.0.22)
NAME                 STATUS    MESSAGE             ERROR
scheduler            Healthy   ok                  
controller-manager   Healthy   ok                  
etcd-0               Healthy   {"health":"true"}   
etcd-1               Healthy   {"health":"true"}   
```

# Kubernetes worker nodes
* The software and configuration files are downloaded and setup on the
  Kubernetes gateway (`gwkublxc`), and then sent to the workers from there
```bash
root@gwkublxc:~# $ K8S_VER=$(curl -s https://storage.googleapis.com/kubernetes-release/release/stable.txt)
root@gwkublxc:~# echo $K8S_VER 
v1.15.0
root@gwkublxc:~# declare -a node_list=("worker1" "worker2")
root@gwkublxc:~# declare -a nodeip_list=("10.240.0.31" "10.240.0.32")
```

## Kubernetes binaries
* Download and install the Kubernetes binaries (which are slightly different
  from the ones on the control plane: only `kubectl` is common):
```bash
root@gwkublxc:~# declare -a kubbin_list=("kube-proxy" "kubelet" "kubectl")
root@gwkublxc:~# for kubbin in "${kubbin_list[@]}"; do \
  curl -LO https://storage.googleapis.com/kubernetes-release/release/${K8S_VER}/bin/linux/amd64/${kubbin} && \
  chmod +x ${kubbin} && mv ${kubbin} /usr/local/bin/; done
root@gwkublxc:~# for node in "${node_list[@]}"; do \
  rsync -av /usr/local/bin/kube{-proxy,let,ctl} root@${node}:/usr/local/bin/; done
```

## Docker
* Install Docker CE (Community Edition) from the Docker CentOS repository:
```bash
root@gwkublxc:~# for node in "${nodeip_list[@]}"; do \
 ssh root@${node} "yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo && \
 yum -y install docker-ce"; done
```

* Start the Docker services:
```bash
root@gwkublxc:~# for node in "${nodeip_list[@]}"; do \
 ssh root@${node} "systemctl start docker.service && \
  systemctl enable docker.service && systemctl status docker.service -l && \
  docker version"; done
```

* Check that Docker works:
```bash
root@gwkublxc:~# for node in "${nodeip_list[@]}"; do \
 ssh root@${node} "docker run hello-world"; done
worker1
Hello from Docker!
worker2
Hello from Docker!
```

* Download and run the Docker container featuring the
  [travel simulator](https://cloud.docker.com/u/tvlsim/repository/docker/tvlsim/metasim):
```bash
root@gwkublxc:~# ssh worker1
root@worker1:~# docker run --rm -it tvlsim/metasim:centos
[build@1..70 sim]$ workspace/install/airinv/bin/airinv
The BOM should be built-in? no
The BOM should be built from schedule? no
Input inventory filename is: /home/build/dev/sim/workspace/install/stdair/share/stdair/samples/invdump01.csv
Log filename is: airinv.log
airinv SV5 / 2010-Mar-11> quit
End of the session. Exiting.
[build@125e6bb84570 sim]$ exit
root@worker1:~# exit
root@gwkublxc:~# ssh worker2
root@worker2:~# docker run --rm -it tvlsim/metasim:centos
[build@d02d6aba83c1 sim]$ workspace/install/airinv/bin/airinv
airinv SV5 / 2010-Mar-11> quit
End of the session. Exiting.
[build@d0..3c1 sim]$ exit
root@worker2:~# exit
```

## CNI network plugin
* Look on https://github.com/containernetworking/plugins/releases for the
  latest release (as of June 2019, it was `v0.8.1`).
* Install the CNI plugin on the worker nodes:
```bash
root@gwkublxc:~# mkdir -p ~/bin && \
 wget https://gist.githubusercontent.com/denisarnaud/1152fc54d017847d8cee02074a2b43df/raw/72f864c154ef3850a9f20fb9aaf0469f7b3ff464/getGitHubLatestRelease.sh \
 -O ~/bin/getGitHubLatestRelease.sh && chmod +x ~/bin/getGitHubLatestRelease.sh
root@gwkublxc:~# CNI_VER=$(~/bin/getGitHubLatestRelease.sh containernetworking/plugins|head -1)
root@gwkublxc:~# mkdir -p /opt/network && \
 wget https://github.com/containernetworking/plugins/releases/download/${CNI_VER}/cni-plugins-linux-amd64-${CNI_VER}.tgz \
  -O /opt/network/cni-plugins-linux-amd64-${CNI_VER}.tgz
root@gwkublxc:~# for node in "${nodeip_list[@]}"; do \
 ssh root@${node} "mkdir -p /opt/{network,cni/bin} && \
  wget https://github.com/containernetworking/plugins/releases/download/${CNI_VER}/cni-plugins-linux-amd64-${CNI_VER}.tgz \
   -O /opt/network/cni-plugins-linux-amd64-${CNI_VER}.tgz && \
   tar xvf /opt/network/cni-plugins-linux-amd64-${CNI_VER}.tgz -C /opt/cni/bin"; done
```

## API server
```bash
root@gwkublxc:~# for node in "${nodeip_list[@]}"; do \
 ssh root@${node} "systemctl daemon-reload && \
  systemctl start kube-apiserver.service && \
  systemctl enable kube-apiserver.service && \
  systemctl status kube-apiserver.service -l"; done
```

## Kubelet configuration

### Kubernetes configuration file (`KUBECONFIG`)
* Create the Kubernetes configuration file (`KUBECONFIG`):
```bash
root@gwkublxc:~# mkdir -p /var/lib/kubelet && \
 cat > /var/lib/kubelet/kubeconfig << _EOF
apiVersion: v1
kind: Config
clusters:
- cluster:
    certificate-authority: /var/lib/kubernetes/ca.pem
    server: https://10.240.0.21:6443
  name: kubernetes
contexts:
- context:
    cluster: kubernetes
    user: kubelet
  name: kubelet
current-context: kubelet
users:
- name: kubelet
  user:
    token: chAng3m3

_EOF
```

* Distribute the Kubernetes configuration file to the nodes:
```bash
root@gwkublxc:~# for node in "${node_list[@]}"; do \
 rsync -av /var/lib/kubelet root@${node}:/var/lib/; done
```

### Kubelet service
* Create the [`kubelet` configuration
  file](https://kubernetes.io/docs/tasks/administer-cluster/kubelet-config-file/),
  [specifying a `KubeletConfiguration`
  struct](https://github.com/kubernetes/kubernetes/blob/master/staging/src/k8s.io/kubelet/config/v1beta1/types.go):
```bash
root@gwkublxc:~# cat > /var/lib/kubelet/kubelet-config.yaml << _EOF
apiVersion: kubelet.config.k8s.io/v1beta1
kind: KubeletConfiguration
clusterDNS:
- 10.32.0.10
clusterDomain: cluster.local
tlsCertFile: /var/lib/kubernetes/kubernetes.pem
tlsPrivateKeyFile: /var/lib/kubernetes/kubernetes-key.pem
serializeImagePulls: false
failSwapOn: false
_EOF
```

* Upload the configuration file to all the worker nodes:
```bash
root@gwkublxc:~# for node in "${node_list[@]}"; do \
  rsync -av /var/lib/kubelet/ root@${node}:/var/lib/kubelet/; done
```

* Create the Kubelet service file:
```bash
root@gwkublxc:~# cat > /etc/systemd/system/kubelet.service << _EOF
[Unit]
Description=Kubernetes Kubelet
Documentation=https://github.com/GoogleCloudPlatform/kubernetes
After=docker.service
Requires=docker.service

[Service]
ExecStart=/usr/local/bin/kubelet \\
 --config=/var/lib/kubelet/kubelet-config.yaml \\
 --container-runtime=docker \\
 --network-plugin=kubenet \\
 --kubeconfig=/var/lib/kubelet/kubeconfig \\
 --v=2

Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
_EOF
```

* Upload the service file to all the worker nodes:
```bash
root@gwkublxc:~# for node in "${node_list[@]}"; do \
  rsync -av /etc/systemd/system/kubelet.service root@${node}:/etc/systemd/system/; done
```

* Distribute the service file to the nodes:
```bash
root@gwkublxc:~# for node in "${node_list[@]}"; do ssh root@${node} "\
  systemctl daemon-reload && \
  systemctl start kubelet.service && \
  systemctl enable kubelet.service && \
  systemctl status kubelet.service -l"; done
```

## `kube-proxy` configuration
* Create the `kube-proxy` configuration file:
```bash
root@gwkublxc:~# cat > /var/lib/kubelet/kube-proxy-config.yaml << _EOF
apiVersion: kubeproxy.config.k8s.io/v1alpha1
kind: KubeProxyConfiguration
clientConnection:
  kubeconfig: "/var/lib/kubelet/kubeconfig"
mode: "iptables"
clusterCIDR: "10.200.0.0/16"
_EOF
```

* Upload the configuration file to all the worker nodes:
```bash
root@gwkublxc:~# for node in "${node_list[@]}"; do \
  rsync -av /var/lib/kubelet/ root@${node}:/var/lib/kubelet/; done
```

* Create the `kube-proxy` service file:
```bash
root@gwkublxc:~# cat > /etc/systemd/system/kube-proxy.service << _EOF
[Unit]
Description=Kubernetes Kube Proxy
Documentation=https://github.com/GoogleCloudPlatform/kubernetes

[Service]
ExecStart=/usr/local/bin/kube-proxy \\
  --config=/var/lib/kubelet/kube-proxy-config.yaml

Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
_EOF
```

* Upload the `kube-proxy` service file on all the worker nodes:
```bash
root@gwkublxc:~# for node in "${node_list[@]}"; do \
  rsync -av /etc/systemd/system/kube-proxy.service root@${node}:/etc/systemd/system/; done
```

* Start the `kube-proxy` service on the worker nodes:
```bash
root@gwkublxc:~# for node in "${node_list[@]}"; do ssh root@${node} "\
  systemctl daemon-reload && systemctl start kube-proxy.service && \
  systemctl enable kube-proxy.service && \
  systemctl status kube-proxy.service -l"; done
```

## Checks
* On the gateway:
```bash
root@gwkublxc:~# declare -a node_ctrl_list=("controller" "controller1" "controller2")
root@gwkublxc:~# declare -a nodeip_ctrl_list=("10.240.0.20" "10.240.0.21" "10.240.0.22")
root@gwkublxc:~# for node in "${node_ctrl_list[@]}"; do \
  ssh root@${node} "kubectl get componentstatuses"; done
NAME                 STATUS    MESSAGE             ERROR
scheduler            Healthy   ok                  
controller-manager   Healthy   ok                  
etcd-0               Healthy   {"health":"true"}   
etcd-1               Healthy   {"health":"true"}   
NAME                 STATUS    MESSAGE             ERROR
scheduler            Healthy   ok                  
etcd-0               Healthy   {"health":"true"}   
etcd-1               Healthy   {"health":"true"}   
controller-manager   Healthy   ok                  
NAME                 STATUS    MESSAGE             ERROR
scheduler            Healthy   ok                  
etcd-1               Healthy   {"health":"true"}   
etcd-0               Healthy   {"health":"true"}   
controller-manager   Healthy   ok                  
root@gwkublxc:~# for node in "${node_ctrl_list[@]}"; do \
  ssh root@${node} "kubectl get nodes"; done
NAME                                STATUS   ROLES    AGE   VERSION
worker1.telecoms-intelligence.com   Ready    <none>   17m   v1.14.3
worker2.telecoms-intelligence.com   Ready    <none>   17m   v1.14.3
NAME                                STATUS   ROLES    AGE   VERSION
worker1.telecoms-intelligence.com   Ready    <none>   17m   v1.14.3
worker2.telecoms-intelligence.com   Ready    <none>   17m   v1.14.3
NAME                                STATUS   ROLES    AGE   VERSION
worker1.telecoms-intelligence.com   Ready    <none>   17m   v1.14.3
worker2.telecoms-intelligence.com   Ready    <none>   17m   v1.14.3
```

* The API may be called on the 8080 port on one of the controllers:
```bash
root@controller# curl -s http://localhost:8080/api/v1/namespaces/default/endpoints
{
  "kind": "EndpointsList",
  "apiVersion": "v1",
  "metadata": {
    "selfLink": "/api/v1/namespaces/default/endpoints",
    "resourceVersion": "14860"
  },
  "items": [
    {
      "metadata": {
        "name": "kubernetes",
        "namespace": "default",
        "selfLink": "/api/v1/namespaces/default/endpoints/kubernetes",
        "uid": "5fbb0868-9049-11e9-b0e0-eaf8cf2a0d6a",
        "resourceVersion": "4812",
        "creationTimestamp": "2019-06-16T15:14:02Z"
      },
      "subsets": [
        {
          "addresses": [
            {
              "ip": "10.240.0.20"
            },
            {
              "ip": "10.240.0.21"
            },
            {
              "ip": "10.240.0.22"
            }
          ],
          "ports": [
            {
              "name": "https",
              "port": 6443,
              "protocol": "TCP"
            }
          ]
        }
      ]
    }
  ]
root@controller1# kubectl get ds,rc,deploy,ep,po,rs,svc --all-namespaces -o wide
NAMESPACE     NAME                                ENDPOINTS                                            AGE
default       endpoints/kubernetes                10.240.0.20:6443,10.240.0.21:6443,10.240.0.22:6443   3h27m
kube-system   endpoints/kube-controller-manager   <none>                                               3h11m
kube-system   endpoints/kube-scheduler            <none>                                               3h6m

NAMESPACE   NAME                 TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)   AGE     SELECTOR
default     service/kubernetes   ClusterIP   10.32.0.1    <none>        443/TCP   3h27m   <none>
```

* On one of the controllers, smoke tests with a busy box:
```bash
root@controller1# kubectl run --generator=run-pod/v1 --rm mytest --image=yauritux/busybox-curl -it
If you don't see a command prompt, try pressing enter.
/home # uname -a
Linux mytest 4.15.18-15-pve #1 SMP PVE 4.15.18-40 (Tue, 21 May 2019 17:43:20 +0200) x86_64 GNU/Linux
/home # exit
Session ended, resume using 'kubectl attach mytest -c mytest -i -t' command when the pod is running
pod "mytest" deleted
root@controller1# kubectl get deployment
NAME     READY   UP-TO-DATE   AVAILABLE   AGE
mytest   1/1     1            1           3m32s
root@controller1# kubectl get pods
NAME                      READY   STATUS    RESTARTS   AGE
mytest-589f6788ff-g4vsf   1/1     Running   0          2m26s
```

* On one of the controllers, smoke tests with Nginx:
```bash
root@controller1# kubectl run --generator=run-pod/v1 nginx --image=nginx --port=80 --replicas=2
pod/nginx created
root@controller1# kubectl exec nginx -it -- bash
root@nginx:/# uname -a
Linux nginx 4.15.18-15-pve #1 SMP PVE 4.15.18-40 (Tue, 21 May 2019 17:43:20 +0200) x86_64 GNU/Linux
root@nginx:/# exit
exit
root@controller1$ kubectl delete pod nginx
pod "nginx" deleted
```

* On one of the controllers, smoke tests with TvlSim. Note that, as the
  [Docker image of TvlSim](https://github.com/airsim/metasim/blob/master/docker/centos/Dockerfile#L45)
  does not provide a server endpoint, the image constantly terminates and
  Kubernetes re-launches it. In order to keep it running, an alternate
  endpoint must be provided.
```bash
root@controller1# kubectl run --generator=run-pod/v1 --rm tvlsim --image=tvlsim/metasim:centos -it
If you don't see a command prompt, try pressing enter.
[root@tvlsim sim]# workspace/install/airinv/bin/airinv
The BOM should be built-in? no
The BOM should be built from schedule? no
Input inventory filename is: /home/build/dev/sim/workspace/install/stdair/share/stdair/samples/invdump01.csv
Log filename is: airinv.log
airinv SV5 / 2010-Mar-11> quit
End of the session. Exiting.
[root@tvlsim sim]# exit
exit
Session ended, resume using 'kubectl attach tvlsim -c tvlsim -i -t' command when the pod is running
pod "tvlsim" deleted
root@controller1# kubectl run --generator=run-pod/v1 tvlsim --image=tvlsim/metasim:centos --replicas=2
pod/tvlsim created
root@controller1# kubectl exec tvlsim -it -- bash
[root@tvlsim sim]# exit
root@controller1$ kubectl delete pod tvlsim
pod "tvlsim" deleted
```

* From the `praqma/network-multitool` pod (named `alpinenet` here),
  test the network connectivity with the other pods and other nodes.
  Normally, as the routes are not yet added on the Proxmox host,
  only the ping to the same network will work; the other
  pings are blocked. `kubectl get pods -o wide` shows that the `alpinenet` pod
  runs on the `worker3` node, and the subnet IP is `10.200.2.0/24`;
  only the `ping 10.200.2.1` will therefore works at that stage:
```bash
root@controller1# kubectl run --generator=run-pod/v1 alpinenet --image=praqma/network-multitool --replicas=2
pod/alpinenet created
root@controller1# kubectl get pods -o wide
NAME        READY   STATUS             RESTARTS   AGE   IP           NODE                                NOMINATED NODE   READINESS GATES
alpinenet   1/1     Running            0          50m   10.200.2.3   worker3.telecoms-intelligence.com   <none>           <none>
root@controller1# kubectl exec alpinenet -it -- bash
bash-4.4# ifconfig|grep inet|grep -v 127
          inet addr:10.200.2.3  Bcast:10.200.2.255  Mask:255.255.255.0
bash-4.4# ping 10.200.0.1
bash-4.4# ping 10.200.1.1
bash-4.4# ping 10.200.2.1
bash-4.4# exit
root@controller1# exit
```

* On the Proxmox host, add the routes to the workers:
```bash
root@proxmox:~$ route add -net 10.200.0.0 netmask 255.255.255.0 gw 10.240.0.31&&\
  route add -net 10.200.1.0 netmask 255.255.255.0 gw 10.240.0.32
root@proxmox:~$ route -n
Kernel IP routing table
Destination     Gateway         Genmask         Flags Metric Ref    Use Iface
0.0.0.0         xx.xx.xx.254    0.0.0.0         UG    0      0        0 vmbr0
10.200.0.0      10.240.0.31     255.255.255.0   UG    0      0        0 vmbr1
10.200.1.0      10.240.0.32     255.255.255.0   UG    0      0        0 vmbr1
10.200.2.0      10.240.0.33     255.255.255.0   UG    0      0        0 vmbr1
10.240.0.0      0.0.0.0         255.255.255.0   U     0      0        0 vmbr1
xx.xx.xx.0      0.0.0.0         255.255.255.0   U     0      0        0 vmbr0
root@proxmox:~$ route del -net 10.200.0.0 netmask 255.255.255.0 gw 10.240.0.31
root@proxmox:~$ route del -net 10.200.1.0 netmask 255.255.255.0 gw 10.240.0.32
```

* Back on a controller, test that the pod can now ping all the workers:
```bash
root@controller1# kubectl exec alpinenet -it -- bash
bash-4.4# ping 10.200.0.1
PING 10.200.0.1 (10.200.0.1) 56(84) bytes of data.
64 bytes from 10.200.0.1: icmp_seq=1 ttl=62 time=0.068 ms
64 bytes from 10.200.0.1: icmp_seq=3 ttl=62 time=0.060 ms
^C
--- 10.200.0.1 ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 2034ms
rtt min/avg/max/mdev = 0.054/0.060/0.068/0.010 ms
bash-4.4# ping 10.200.1.1
PING 10.200.1.1 (10.200.1.1) 56(84) bytes of data.
64 bytes from 10.200.1.1: icmp_seq=1 ttl=64 time=0.028 ms
64 bytes from 10.200.1.1: icmp_seq=2 ttl=64 time=0.049 ms
^C
--- 10.200.1.1 ping statistics ---
2 packets transmitted, 2 received, 0% packet loss, time 1012ms
rtt min/avg/max/mdev = 0.028/0.038/0.049/0.012 ms
bash-4.4# ping 10.200.2.1
PING 10.200.2.1 (10.200.2.1) 56(84) bytes of data.
64 bytes from 10.200.2.1: icmp_seq=1 ttl=64 time=0.038 ms
64 bytes from 10.200.2.1: icmp_seq=2 ttl=64 time=0.053 ms
^C
--- 10.200.2.1 ping statistics ---
2 packets transmitted, 2 received, 0% packet loss, time 1005ms
rtt min/avg/max/mdev = 0.038/0.045/0.053/0.010 ms
bash-4.4# exit
root@controller1# kubectl delete pod alpinenet
```

## CoreDNS
* Note that CoreDNS and KubeDNS are competing services. Either may be installed,
  but not both. In
  [Kubernetes the hard way](https://github.com/Praqma/LearnKubernetes/blob/master/kamran/Kubernetes-The-Hard-Way-on-BareMetal.md#deploying-the-cluster-add-on-dns-skydns),
  KubeDNS is used. So, CoreDNS is mentioned here for completeness.

* On a controller, download the configuration file of CoreDNS and add
  a few DNS servers to the configuration map:
```bash
root@controller1# wget https://storage.googleapis.com/kubernetes-the-hard-way/coredns.yaml -O /var/lib/kubernetes/coredns.yaml
root@controller1# sed -i -e 's/upstream/upstream 8.8.8.8/g' \
  -e 's/proxy ./proxy . 8.8.8.8/g' /var/lib/kubernetes/coredns.yaml
```

* On a controller, deploy the CoreDNS services:
```bash
root@controller1# kubectl apply -f /var/lib/kubernetes/coredns.yaml
serviceaccount/coredns created
clusterrole.rbac.authorization.k8s.io/system:coredns created
clusterrolebinding.rbac.authorization.k8s.io/system:coredns created
configmap/coredns created
deployment.extensions/coredns created
service/kube-dns created
root@controller1$ kubectl get ds,rc,deploy,ep,po,rs,svc --all-namespaces -o wide
NAMESPACE     NAME                            READY   UP-TO-DATE   AVAILABLE   AGE    CONTAINERS   IMAGES                  SELECTOR
kube-system   deployment.extensions/coredns   2/2     2            2           103s   coredns      coredns/coredns:1.2.2   k8s-app=kube-dns

NAMESPACE     NAME                                ENDPOINTS                                               AGE
default       endpoints/kubernetes                10.240.0.20:6443,10.240.0.21:6443,10.240.0.22:6443      5h26m
kube-system   endpoints/kube-controller-manager   <none>                                                  5h9m
kube-system   endpoints/kube-dns                  10.200.0.4:53,10.200.2.4:53,10.200.0.4:53 + 1 more...   102s
kube-system   endpoints/kube-scheduler            <none>                                                  5h5m

NAMESPACE     NAME                           READY   STATUS    RESTARTS   AGE    IP           NODE                                NOMINATED NODE   READINESS GATES
kube-system   pod/coredns-75bf455cdf-8zxxq   1/1     Running   0          102s   10.200.0.4   worker1.telecoms-intelligence.com   <none>           <none>
kube-system   pod/coredns-75bf455cdf-fwcgm   1/1     Running   0          101s   10.200.1.4   worker2.telecoms-intelligence.com   <none>           <none>

NAMESPACE     NAME                                       DESIRED   CURRENT   READY   AGE    CONTAINERS   IMAGES                  SELECTOR
kube-system   replicaset.extensions/coredns-75bf455cdf   2         2         2       102s   coredns      coredns/coredns:1.2.2   k8s-app=kube-dns,pod-template-hash=75bf455cdf

NAMESPACE     NAME                 TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)         AGE     SELECTOR
default       service/kubernetes   ClusterIP   10.32.0.1    <none>        443/TCP         5h26m   <none>
kube-system   service/kube-dns     ClusterIP   10.32.0.10   <none>        53/UDP,53/TCP   102s    k8s-app=kube-dns
```

* Check that the service is running:
```bash
root@controller1:~# kubectl get pods -l k8s-app=kube-dns -n kube-system
NAME                       READY   STATUS    RESTARTS   AGE
coredns-699f8ddd77-94qv9   1/1     Running   0          20s
coredns-699f8ddd77-gtcgb   1/1     Running   0          20s
```

* Check the logs of the CoreDNS pod:
```bash
root@controller1# kubectl logs pod/coredns-75bf455cdf-8zxxq -n kube-system
E0620 18:56:32.257786       1 reflector.go:205] github.com/coredns/coredns/plugin/kubernetes/controller.go:355: Failed to list *v1.Namespace: Get https://10.32.0.1:443/api/v1/namespaces?limit=500&resourceVersion=0: dial tcp 10.32.0.1:443: i/o timeout
E0620 18:56:32.258927       1 reflector.go:205] github.com/coredns/coredns/plugin/kubernetes/controller.go:350: Failed to list *v1.Endpoints: Get https://10.32.0.1:443/api/v1/endpoints?limit=500&resourceVersion=0: dial tcp 10.32.0.1:443: i/o timeout
E0620 18:56:32.260040       1 reflector.go:205] github.com/coredns/coredns/plugin/kubernetes/controller.go:348: Failed to list *v1.Service: Get https://10.32.0.1:443/api/v1/services?limit=500&resourceVersion=0: dial tcp 10.32.0.1:443: i/o timeout
```

* (Optional) Edit the configuration map of CoreDNS (to quit the VI editor,
  type the `ESC` key and then the `:wq` command/sequence):
```bash
root@controller1# kubectl edit cm coredns -n kube-system
```

* Check the configuration map of CoreDNS:
```bash
root@controller1# kubectl describe cm coredns -n kube-system
Name:         coredns
Namespace:    kube-system
Labels:       <none>
Annotations:  <none>

Data
====
Corefile:
----
.:53 {
    errors
    health
    kubernetes cluster.local in-addr.arpa ip6.arpa {
      pods insecure
      upstream 8.8.8.8
      fallthrough in-addr.arpa ip6.arpa
    }
    prometheus :9153
    proxy . 8.8.8.8 /etc/resolv.conf
    cache 30
    loop
    reload
    loadbalance
}

Events:  <none>
```

### Query the cluster
* Through the `etcd` cluster:
```bash
root@gwkublxc:~# etcdctl --ca-file=/etc/etcd/ca.pem --endpoints=$ENDPOINTS ls
/foo
root@gwkublxc:~# etcdctl --ca-file=/etc/etcd/ca.pem --endpoints=$ENDPOINTS get foo
bar
```

* Through the API:
```bash
root@gwkublxc:~# curl http://controller1:8080/api/v1/namespaces
root@gwkublxc:~# curl http://controller1:8080/api/v1/pods
root@gwkublxc:~# curl http://controller1:8080/api/v1/nodes
```

### Smoke test with busybox
Create a busybox deployment:
```bash
root@controller1:~# kubectl run busybox --generator=run-pod/v1 --image=busybox:1.28 --command -- sleep 3600
pod/busybox created
```

* List the pod created by the busybox deployment:
```bash
root@controller1:~# kubectl get pods -l run=busybox
NAME                      READY   STATUS    RESTARTS   AGE
busybox-bd8fb7cbd-vflm9   1/1     Running   0          10s
```

* Retrieve the full name of the busybox pod:
```bash
root@controller1:~# POD_NAME=$(kubectl get pods -l run=busybox -o jsonpath="{.items[0].metadata.name}")
```

* Execute a DNS lookup for the kubernetes service inside the busybox pod:
```bash
root@controller1:~# kubectl exec -ti $POD_NAME -- nslookup kubernetes
Server:    10.32.0.10
Address 1: 10.32.0.10 kube-dns.kube-system.svc.cluster.local

Name:      kubernetes
Address 1: 10.32.0.1 kubernetes.default.svc.cluster.local
```

## Remove CoreDNS
```bash
root@controller1# kubectl delete -f /var/lib/kubernetes/coredns.yaml
serviceaccount "coredns" deleted
clusterrole.rbac.authorization.k8s.io "system:coredns" deleted
clusterrolebinding.rbac.authorization.k8s.io "system:coredns" deleted
configmap "coredns" deleted
deployment.extensions "coredns" deleted
service/kube-dns deleted
```

### CoreDNS from the sources (optional)
* References:
  + https://github.com/coredns/coredns
  + https://github.com/coredns/deployment/tree/master/kubernetes

* Install CoreDNS from sources on a controller. The C++ compiler is needed:
```bash
root@controller1:~# yum -y install gcc-c++ cmake3 zlib-devel bzip2-devel \
 libffi-devel readline-devel doxygen protobuf-devel protobuf-compiler
root@controller1:~# mkdir -p ~/dev/infra && git clone https://github.com/coredns/coredns ~/dev/infra/coredns
root@controller1:~# cd ~/dev/infra/coredns && make
```

# Kube DNS
* Again, if CoreDNS is working, do not install KubeDNS

* On a controller:
```bash
root@controller1:~# wget \
 https://github.com/kelseyhightower/kubernetes-the-hard-way/raw/master/deployments/kube-dns.yaml \
 -O /var/lib/kubernetes/kubedns.yaml
root@controller1:~# kubectl create -f /var/lib/kubernetes/kubedns.yaml
service/kube-dns created
serviceaccount/kube-dns created
configmap/kube-dns created
deployment.apps/kube-dns created
root@controller1:~# kubectl get svc --namespace=kube-system
NAME       TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)         AGE
kube-dns   ClusterIP   10.32.0.10   <none>        53/UDP,53/TCP   27s
root@controller1:~# kubectl get pods --namespace=kube-system
NAME                        READY   STATUS    RESTARTS   AGE
kube-dns-6bfbdd666c-vcvgb   3/3     Running   0          82s
root@controller1:~# kubectl create deployment alpinenet-deployment \
 --image=praqma/network-multitool
root@controller1:~# kubectl exec alpinenet-deployment-646478768-jrfs7 -it -- bash
bash-4.4# cat /etc/resolv.conf 
nameserver 10.32.0.10
search default.svc.cluster.local svc.cluster.local cluster.local example.com
options ndots:5
bash-4.4# dig kubernetes.default.svc.cluster.local

; <<>> DiG 9.12.3 <<>> kubernetes.default.svc.cluster.local
;; global options: +cmd
;; Got answer:
;; WARNING: .local is reserved for Multicast DNS
;; You are currently testing what happens when an mDNS query is leaked to DNS
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 20424
;; flags: qr aa rd ra; QUERY: 1, ANSWER: 1, AUTHORITY: 0, ADDITIONAL: 0

;; QUESTION SECTION:
;kubernetes.default.svc.cluster.local. IN A

;; ANSWER SECTION:
kubernetes.default.svc.cluster.local. 30 IN A	10.32.0.1
bash-4.4# 
```

## Remove KubeDNS
```bash
root@controller1# kubectl delete -f /var/lib/kubernetes/kubedns.yaml
```

# Kubernetes client on the gateway
* Setup the `KUBECONFIG` environment variable:
```bash
root@gwkublxc:~# cat >> ~/.bashrc << _EOF

# Kubernetes
export KUBECONFIG="/var/lib/kubelet/kubeconfig"

_EOF
root@gwkublxc:~# . ~/.bashrc
```

* Check that the gateway can interact with the Kubernetes cluster:
```bash
root@gwkublxc:~# kubectl get nodes
NAME                                STATUS   ROLES    AGE     VERSION
worker1.example.com   Ready    <none>   8h      v1.14.2
worker2.example.com   Ready    <none>   7h39m   v1.14.2
```

# Smoke tests with replicas on Alpine network multi-tool
* On a controller, create a deployment YAML file for the Alpine network
  multi-tool containers:
```bash
root@controller1:~# cat > /var/lib/kubernetes/alpinenet-deployment.yaml << _EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: alpinenet-deployment
  labels:
    app: alpinenet
spec:
  replicas: 2
  selector:
    matchLabels:
      app: alpinenet
  template:
    metadata:
      labels:
        app: alpinenet
    spec:
      containers:
      - name: alpinenet
        image: praqma/network-multitool

_EOF
```

* Create the Kubernetes deployment:
```bash
root@controller1:~# kubectl create -f /var/lib/kubernetes/alpinenet-deployment.yaml
deployment.apps/alpinenet-deployment created
```

* List the pods:
```bash
root@controller1:~# kubectl get pods -o go-template --template '{{range .items}}{{.metadata.name}}{{"\n"}}{{end}}'
alpinenet-deployment-84c8c587c-28vgv
alpinenet-deployment-84c8c587c-crfl8
```

* List the IP addresses of the pods, and check that they can interact
  one with the other:
```bash
root@controller1:~# declare -a pod_list=($(kubectl get pods -o go-template --template '{{range .items}}{{.metadata.name}}{{"\n"}}{{end}}'))
root@controller1:~# for pod in "${pod_list[@]}"; do kubectl exec ${pod} -it -- ifconfig; done | grep inet | grep -v 127
          inet addr:10.200.0.10  Bcast:10.200.0.255  Mask:255.255.255.0
          inet addr:10.200.1.11  Bcast:10.200.1.255  Mask:255.255.255.0
root@controller1:~# kubectl exec alpinenet-deployment-84c8c587c-28vgv -it -- bash
bash-4.4# ifconfig | grep inet | grep -v 127
          inet addr:10.200.0.10  Bcast:10.200.0.255  Mask:255.255.255.0
bash-4.4# ping 10.200.1.11
PING 10.200.1.11 (10.200.1.11) 56(84) bytes of data.
64 bytes from 10.200.1.11: icmp_seq=1 ttl=61 time=0.073 ms
64 bytes from 10.200.1.11: icmp_seq=2 ttl=61 time=0.074 ms
bash-4.4# exit
root@controller1:~# 
```

# Smoke tests with replicas of Nginx
* On a controller, create a deployment YAML file for the Nginx containers:
```bash
root@controller1:~# cat > /var/lib/kubernetes/nginx-deployment.yaml << _EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  labels:
    app: nginx
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.7.9
        ports:
        - containerPort: 80

_EOF
```

* Create the Kubernetes deployment:
```bash
root@controller1:~# kubectl create -f /var/lib/kubernetes/nginx-deployment.yaml
deployment.apps/nginx-deployment created
```

* List the pods:
```bash
root@controller1:~# kubectl get pods -o go-template --template '{{range .items}}{{.metadata.name}}{{"\n"}}{{end}}' | grep nginx
nginx-deployment-6dd86d77d-5r4nv
nginx-deployment-6dd86d77d-j4tgh
nginx-deployment-6dd86d77d-n842m
```

* Create a `NodePort` service:
```bash
root@controller1:~# kubectl expose deployment nginx-deployment --type NodePort
service/nginx-deployment exposed
root@controller1:~# kubectl get svc
NAME               TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)        AGE
kubernetes         ClusterIP   10.32.0.1    <none>        443/TCP        6d1h
nginx-deployment   NodePort    10.32.0.98   <none>        80:31233/TCP   3s
```

* Query Nginx through the workers (from either their IP or host name),
  specifying the node port:
```bash
root@controller1:~# NODE_PORT=$(kubectl get svc nginx-deployment \
 --output=jsonpath='{range .spec.ports[0]}{.nodePort}')
root@controller1:~# curl http://10.240.0.31:31233
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
...
</html>
root@controller1:~# curl http://worker2.example.com:31233
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
...
</html>
```

* Query Nginx through a POD, which does not require the knowledge of the port:
```bash
root@controller1:~# declare -a pod_list=($(kubectl get pods -o go-template \
 --template '{{range .items}}{{.metadata.name}}{{"\n"}}{{end}}' | grep nginx))
root@controller1:~# for pod in "${pod_list[@]}"; do \
 kubectl exec ${pod} -it -- ip addr show; done | grep inet | grep -v 127
    inet 10.200.0.12/24 brd 10.200.0.255 scope global eth0
    inet 10.200.0.13/24 brd 10.200.0.255 scope global eth0
    inet 10.200.1.13/24 brd 10.200.1.255 scope global eth0
root@controller1:~# curl http://10.200.0.12
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
...
</html>
root@controller1:~# curl http://10.200.1.13
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
...
</html>
```

# Smoke tests with the travel simulator

## Create a single simulator pod
* On one of the controllers, create a pod configuration file:
```bash
root@controller1:~# cat > /var/lib/kubernetes/tvlsim-pod.yaml << _EOF
apiVersion: v1
kind: Pod
metadata:
  name: tvlsim
  labels:
    purpose: bootstrap-sim
spec:
  containers:
  - name: tvlsim-container
    image: tvlsim/metasim:centos
    command: ["/home/build/dev/sim/workspace/install/airinv/bin/AirInvServer"]
  restartPolicy: OnFailure

_EOF
```

* Launch the simulator pod:
```bash
root@controller1:~# kubectl create -f /var/lib/kubernetes/tvlsim-pod.yaml
pod/tvlsim created
root@controller1:~# kubectl get pods
NAME        READY   STATUS    RESTARTS   AGE
tvlsim      1/1     Running   0          32s
```

* Attach to a simulator pod:
```bash
root@controller1:~# kubectl exec tvlsim -it -- bash
[root@tvlsim sim]# ps -edf
UID        PID  PPID  C STIME TTY          TIME CMD
root         1     0  0 21:54 ?        00:00:00 /home/build/dev/sim/workspace/install/airinv/bin/AirInvServer
root         9     0  0 21:56 pts/0    00:00:00 bash
root       136     9  0 21:56 pts/0    00:00:00 ps -edf
[root@tvlsim sim]# workspace/install/airinv/bin/AirInvClient 
Connecting to hello world server…
Sending Hello 0…
Received World 0
...
Sending Hello 9…
Received World 9
[root@tvlsim sim]# exit
root@controller1:~# kubectl exec tvlsim -it -- workspace/install/airinv/bin/AirInvClient
Connecting to hello world server…
Sending Hello 0…
Received World 0
...
Sending Hello 9…
Received World 9
root@controller1:~# kubectl exec tvlsim -it -- workspace/install/airinv/bin/airinv
The BOM should be built-in? no
The BOM should be built from schedule? no
Input inventory filename is: /home/build/dev/sim/workspace/install/stdair/share/stdair/samples/invdump01.csv
Log filename is: airinv.log
airinv SV5 / 2010-Mar-11> list
List of flights for all 0 (all)
1. SV
  1.1. SV5
    1.1.1. SV5 / 2010-Mar-11
    1.1.2. SV5 / 2010-May-11

airinv SV5 / 2010-Mar-11> quit
End of the session. Exiting.
```

* Delete the simulator pod:
```bash
root@controller1:~# kubectl delete -f /var/lib/kubernetes/tvlsim-pod.yaml
pod "tvlsim" deleted
```

## Create a full deployment of several simulator pods
* On one of the controllers, create a deployment configuration file:
```bash
root@controller1:~# cat > /var/lib/kubernetes/tvlsim-deployment.yaml << _EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: tvlsim-deployment
  labels:
    app: tvlsim
spec:
  replicas: 2
  selector:
    matchLabels:
      app: tvlsim
  template:
    metadata:
      labels:
        app: tvlsim
    spec:
      containers:
      - name: tvlsim
        image: tvlsim/metasim:centos
        command: ["/home/build/dev/sim/workspace/install/airinv/bin/AirInvServer"]
        ports:
        - containerPort: 5555

_EOF
```

* Launch the simulator deployment:
```bash
root@controller1:~# kubectl create -f /var/lib/kubernetes/tvlsim-deployment.yaml
deployment.apps/tvlsim-deployment created
root@controller1:~# kubectl get pods
NAME        READY   STATUS    RESTARTS   AGE
tvlsim-deployment-5f78c6c86b-qjrhk   1/1     Running   0          16s
tvlsim-deployment-5f78c6c86b-vfl5f   1/1     Running   0          16s
```

* Attach to a simulator pod:
```bash
root@controller1:~# kubectl exec tvlsim-deployment-5f78c6c86b-qjrhk -it -- workspace/install/airinv/bin/AirInvClient
Connecting to hello world server…
Sending Hello 0…
Received World 0
...
Sending Hello 9…
Received World 9
```

* Delete the simulator pod:
```bash
root@controller1:~# kubectl delete -f /var/lib/kubernetes/tvlsim-deployment.yaml
deployment.apps "tvlsim-deployment" deleted
```

