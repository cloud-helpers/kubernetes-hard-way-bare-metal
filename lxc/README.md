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
  * [Propagation of certificates on the newly created LXC containers](#propagation-of-certificates-on-the-newly-created-lxc-containers)
  * [Supplementary configuration of the LXC containers](#supplementary-configuration-of-the-lxc-containers)
    + [General](#general)
    + [SSH](#ssh)
    + [Basic checks for the configuration of the LXC containers](#basic-checks-for-the-configuration-of-the-lxc-containers)
- [Set up of the configuration management (`etcd`) servers](#set-up-of-the-configuration-management---etcd---servers)
  * [Check the setup](#check-the-setup)
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
  * [Installation of complementary packages](#installation-of-complementary-packages-1)
  * [Kubernetes certificates](#kubernetes-certificates-1)
  * [Kubernetes binaries](#kubernetes-binaries-1)
  * [Docker](#docker)
  * [CNI network plugin](#cni-network-plugin)
  * [API server](#api-server)
  * [Kubelet configuration](#kubelet-configuration)
    + [Kubernetes configuration file (`KUBECONFIG`)](#kubernetes-configuration-file---kubeconfig--)
    + [Kubelet service](#kubelet-service)
  * [`kube-proxy` configuration](#-kube-proxy--configuration)
  * [Check the Kubernetes services on the worker nodes](#check-the-kubernetes-services-on-the-worker-nodes)
  * [CoreDNS](#coredns)
    + [Smoke test with busybox](#smoke-test-with-busybox)
    + [CoreDNS from the sources (optional)](#coredns-from-the-sources--optional-)
- [Kube DNS](#kube-dns)
- [Kubernetes client on the gateway](#kubernetes-client-on-the-gateway)
- [Smoke tests with replicas on Alpine network multi-tool](#smoke-tests-with-replicas-on-alpine-network-multi-tool)
- [Smoke tests with replicas of Nginx](#smoke-tests-with-replicas-of-nginx)
- [Smoke tests with the travel simulator](#smoke-tests-with-the-travel-simulator)
  * [Create a single simulator pod](#create-a-single-simulator-pod)
  * [Create a full deployment of several simulator pods](#create-a-full-deployment-of-several-simulator-pods)

# Overview
[This document](https://github.com/cloud-helpers/cloud-helpers/kubernetes-hard-way-bare-metal/blob/master/lxc/README.md)
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
to that the (LXC) containers may run Docker inside.

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
setup](https://github.com/cloud-helpers/cloud-helpers/kubernetes-hard-way-bare-metal/blob/master/proxmox/README.md).

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
    "10.240.0.103",
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
root@proxmox:~$ pct create 211 local:vztmpl/centos-7-default_20190510_amd64.tar.xz --arch amd64 --cores 1 --hostname etcd1.example.com --memory 1024 --swap 2048 --net0 name=eth0,bridge=vmbr3,gw=10.240.0.2,ip=10.240.0.11/24,type=veth --onboot 1 --ostype centos
root@proxmox:~$ pct resize 211 rootfs 10G
root@proxmox:~$ pce enter 211
root@etcd1:~# yum -y install net-tools openssh-server wget file less htop git
root@etcd1:~# systemctl start sshd && systemctl enable sshd
root@etcd1:~# exit
```

* `etcd2`:
```bash
root@proxmox:~$ pct create 212 local:vztmpl/centos-7-default_20190510_amd64.tar.xz --arch amd64 --cores 1 --hostname etcd2.example.com --memory 1024 --swap 2048 --net0 name=eth0,bridge=vmbr3,gw=10.240.0.2,ip=10.240.0.12/24,type=veth --onboot 1 --ostype centos
root@proxmox:~$ pct resize 212 rootfs 10G
root@proxmox:~$ pce enter 212
root@etcd2:~# yum -y install net-tools openssh-server wget file less htop git
root@etcd2:~# systemctl start sshd && systemctl enable sshd
root@etcd2:~# exit
```

## Controller plane servers
* `controller`:
```bash
root@proxmox:~$ pct create 220 local:vztmpl/centos-7-default_20190510_amd64.tar.xz --arch amd64 --cores 2 --hostname controller.example.com --memory 2048 --swap 4096 --net0 name=eth0,bridge=vmbr3,gw=10.240.0.2,ip=10.240.0.20/24,type=veth --onboot 1 --ostype centos
root@proxmox:~$ pct resize 220 rootfs 10G
```

* `controller1`:
```bash
root@proxmox:~$ pct create 221 local:vztmpl/centos-7-default_20190510_amd64.tar.xz --arch amd64 --cores 2 --hostname controller1.example.com --memory 2048 --swap 4096 --net0 name=eth0,bridge=vmbr3,gw=10.240.0.2,ip=10.240.0.21/24,type=veth --onboot 1 --ostype centos
root@proxmox:~$ pct resize 221 rootfs 10G
```

* `controller2`:
```bash
root@proxmox:~$ pct create 222 local:vztmpl/centos-7-default_20190510_amd64.tar.xz --arch amd64 --cores 2 --hostname controller2.example.com --memory 2048 --swap 4096 --net0 name=eth0,bridge=vmbr3,gw=10.240.0.2,ip=10.240.0.22/24,type=veth --onboot 1 --ostype centos
root@proxmox:~$ pct resize 222 rootfs 10G
```

## Worker nodes
* `worker1`:
```bash
root@proxmox:~$ pct create 231 local:vztmpl/centos-7-default_20190510_amd64.tar.xz --arch amd64 --cores 8 --hostname worker1.example.com --memory 16384 --swap 16384 --net0 name=eth0,bridge=vmbr3,gw=10.240.0.2,ip=10.240.0.31/24,type=veth --onboot 1 --ostype centos
root@proxmox:~$ pct resize 231 rootfs 20G
```

* `worker2`:
```bash
root@proxmox:~$ pct create 232 local:vztmpl/centos-7-default_20190510_amd64.tar.xz --arch amd64 --cores 8 --hostname worker2.example.com --memory 16384 --swap 16384 --net0 name=eth0,bridge=vmbr3,gw=10.240.0.2,ip=10.240.0.32/24,type=veth --onboot 1 --ostype centos
root@proxmox:~$ pct resize 232 rootfs 20G
```

## Load balancers (lb)
* `lb`:
```bash
root@proxmox:~$ pct create 240 local:vztmpl/centos-7-default_20190510_amd64.tar.xz --arch amd64 --cores 1 --hostname lb.example.com --memory 512 --swap 1024 --net0 name=eth0,bridge=vmbr3,gw=10.240.0.2,ip=10.240.0.40/24,type=veth --onboot 1 --ostype centos
root@proxmox:~$ # pct resize 240 rootfs 4G
```

* `lb1`:
```bash
root@proxmox:~$ pct create 241 local:vztmpl/centos-7-default_20190510_amd64.tar.xz --arch amd64 --cores 1 --hostname lb1.example.com --memory 512 --swap 1024 --net0 name=eth0,bridge=vmbr3,gw=10.240.0.2,ip=10.240.0.41/24,type=veth --onboot 1 --ostype centos
root@proxmox:~$ # pct resize 241 rootfs 4G
```

* `lb2`:
```bash
root@proxmox:~$ pct create 242 local:vztmpl/centos-7-default_20190510_amd64.tar.xz --arch amd64 --cores 1 --hostname lb2.example.com --memory 512 --swap 1024 --net0 name=eth0,bridge=vmbr3,gw=10.240.0.2,ip=10.240.0.42/24,type=veth --onboot 1 --ostype centos
root@proxmox:~$ # pct resize 242 rootfs 4G
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

## Propagation of certificates on the newly created LXC containers
* Push the certificates to every K8S node:
```bash
root@gwkublxc:~# chmod 644 /opt/cfssl/etc/kubernetes-key.pem
root@gwkublxc:~# declare -a node_list=("lb" "lb1" "lb2" "etcd1" "etcd2" \
 "controller" "controller1" "controller2" "worker1" "worker2")
root@gwkublxc:~# declare -a cert_list=("/opt/cfssl/etc/ca.pem" \
 "/opt/cfssl/etc/kubernetes-key.pem" "/opt/cfssl/etc/kubernetes.pem")
root@gwkublxc:~# for node in "${node_list[@]}"
do
  for cert in "${cert_list[@]}"
  do
    rsync -av ${cert} root@${node}:/root/
  done
done
```

## Supplementary configuration of the LXC containers
* The following sub-sections have to be performed not only on the gateway,
  but also on every LXC container composing the Kubernetes cluster.
  Those steps cannot be easily automated, as SSH is not yet setup
  on those containers.

### General
```bash
root@container:~# yum -y update && \
 yum -y install epel-release && \
 yum -y install rpmconf yum-utils htop wget curl file less net-tools whois \
  bzip2 rsync bash-completion bash-completion-extras openssh-server ntp
root@container:~# rpmconf -a
root@container:~# ln -sf /usr/share/zoneinfo/Europe/Paris /etc/localtime && \
 systemctl start ntpd && systemctl enable ntpd && \
 setenforce 0 && \
 sed -i --follow-symlinks 's/SELINUX=enforcing/SELINUX=disabled/g' \
  /etc/sysconfig/selinux
```

### SSH
```bash
root@container:~# mkdir ~/.ssh && chmod 700 ~/.ssh && \
 cat >> ~/.ssh/authorized_keys << _EOF
ssh-rsa AAAxxxZZZ k8s@example.com
_EOF
root@container:~# chmod 600 ~/.ssh/authorized_keys
root@container:~# systemctl start sshd.service && systemctl enable sshd.service
```
* Check that you can connect from outside, beginning by the Proxmox host.
  Then, remove the password for `root`:
```bash
root@container:~# passwd -d root
```

* Check that the firewalls are not installed:
```bash
root@container:~# systemctl status firewalld.service && \
 systemctl stop firewalld.service && systemctl disable firewalld.service
root@container:~# systemctl status iptables.service
```

* Set the `/etc/hosts` file:
```bash
root@container:~# cat > /etc/hosts << _EOF
# Local VM
127.0.0.1   localhost localhost.localdomain localhost4 localhost4.localdomain4
::1         localhost localhost.localdomain localhost6 localhost6.localdomain6

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
```

### Basic checks for the configuration of the LXC containers
* On the gateway:
```bash
root@gwkublxc:~# declare -a node_list=("lb" "lb1" "lb2" "etcd1" "etcd2" \
 "controller" "controller1" "controller2" "worker1" "worker2")
root@gwkublxc:~# for node in "${node_list[@]}"; do \
 ssh root@${node} "hostname; getenforce"; done
lb.example.com
Disabled
lb1.example.com
Disabled
lb2.example.com
Disabled
etcd1.example.com
Disabled
etcd2.example.com
Disabled
controller.example.com
Disabled
controller1.example.com
Disabled
controller2.example.com
Disabled
worker1.example.com
Disabled
worker2.example.com
Disabled
```

# Set up of the configuration management (`etcd`) servers
* References:
 + https://github.com/etcd-io/etcd/blob/master/Documentation/demo.md
 + https://github.com/etcd-io/etcd/blob/master/Documentation/op-guide/security.md

* Define the `etcd` nodes:
```bash
root@gwkublxc:~# declare -a node_list=("etcd1" "etcd2")
root@gwkublxc:~# declare -a nodeip_list=("10.240.0.11" "10.240.0.12")
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
ExecStart=/bin/bash -c "GOMAXPROCS=\$(nproc) /usr/bin/etcd \
  --name \"ETCD_NAME\" \
  --data-dir=\"\${ETCD_DATA_DIR}\" \
  --cert-file=\"/etc/etcd/kubernetes.pem\" \
  --key-file=\"/etc/etcd/kubernetes-key.pem\" \
  --peer-cert-file=\"/etc/etcd/kubernetes.pem\" \
  --peer-key-file=\"/etc/etcd/kubernetes-key.pem\" \
  --trusted-ca-file=\"/etc/etcd/ca.pem\" \
  --peer-trusted-ca-file=\"/etc/etcd/ca.pem\" \
  --initial-advertise-peer-urls \"https://INTERNAL_IP:2380\" \
  --listen-peer-urls \"https://INTERNAL_IP:2380\" \
  --advertise-client-urls \"https://INTERNAL_IP:2379\" \
  --listen-client-urls \"https://INTERNAL_IP:2379,http://127.0.0.1:2379\" \
  --initial-cluster-token \"etcd-cluster-0\" \
  --initial-cluster \"etcd1=https://10.240.0.11:2380,etcd2=https://10.240.0.12:2380\" \
  --initial-cluster-state \"new\""
Restart=on-failure
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target
_EOF
```

* Deploy and intanciate the `etcd` service scripts:
```bash
[root@gwkublxc ~]# for node in "${node_list[@]}"; do ssh root@${node} "mkdir -p /etc/etcd/ && mv ca.pem kubernetes-key.pem kubernetes.pem /etc/etcd/"; done
[root@gwkublxc ~]# for node in "${node_list[@]}"; do ssh root@${node} "yum -y install etcd"; done
[root@gwkublxc ~]# for node in "${node_list[@]}"; do rsync -av -p /opt/etcd/etcd.service root@${node}:/usr/lib/systemd/system/; done
[root@gwkublxc ~]# for i in "${!node_list[@]}"; do ssh root@${nodeip_list[$i]} "sed -i -e s/INTERNAL_IP/nodeip_list[$i]/g -e s/ETCD_NAME/node_list[$i]/g /usr/lib/systemd/system/etcd.service"; done
[root@gwkublxc ~]# for nodeip in "${nodeip_list[@]}"; do ssh root@${nodeip} "systemctl daemon-reload && systemctl enable etcd && systemctl start etcd && systemctl status etcd -l"; done
```

* Check the setup from the gateway:
```bash
root@gwkublxc:~# ENDPOINTS="https://10.240.0.11:2379,https://10.240.0.12:2379"
root@gwkublxc:~# etcdctl --ca-file=/etc/etcd/ca.pem --endpoints=$ENDPOINTS cluster-health
member 3a57933972cb5131 is healthy: got healthy result from https://10.240.0.12:2379
member ffed16798470cab5 is healthy: got healthy result from https://10.240.0.11:2379
cluster is healthy
root@gwkublxc:~# etcdctl --ca-file=/etc/etcd/ca.pem --endpoints=$ENDPOINTS member list
3a57933972cb5131: name=etcd2 peerURLs=https://10.240.0.12:2380 clientURLs=https://10.240.0.12:2379 isLeader=true
ffed16798470cab5: name=etcd1 peerURLs=https://10.240.0.11:2380 clientURLs=https://10.240.0.11:2379 isLeader=false
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

## Check the setup
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
tcp        0      0 10.240.0.11:22          10.240.0.103:42792      ESTABLISHED 4419/sshd: root@pts 
tcp        0      0 10.240.0.11:39722       10.240.0.11:2379        ESTABLISHED 4746/etcd           
tcp        0      0 10.240.0.11:2379        10.240.0.11:39722       ESTABLISHED 4746/etcd           
tcp        0      0 127.0.0.1:2379          127.0.0.1:40872         ESTABLISHED 4746/etcd           
tcp6       0      0 :::22                   :::*                    LISTEN      266/sshd            
root@etcd1:~# etcdctl --ca-file=/etc/etcd/ca.pem cluster-health
member 8e9e05c52164694d is healthy: got healthy result from https://10.240.0.11:2379
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
tcp        0      0 10.240.0.12:22          10.240.0.103:58082      ESTABLISHED 635/sshd: root@pts/ 
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
* Specify the node IP addresses and host names for the following sub-sections:
```bash
root@gwkublxc:~# declare -a nodeip_list=("10.240.0.20" "10.240.0.21" \
 "10.240.0.22")
root@gwkublxc:~# declare -a node_list=("controller" "controller1" "controller2")
root@gwkublxc:~# declare -a node_ext_list=("lb" "lb1" "lb2" "etcd1" "etcd2" \
 "controller" "controller1" "controller2" "worker1" "worker2")
```

## Installation of complementary packages
```bash
[root@gwkublxc ~]# for node in "${nodeip_ext_list[@]}"; do \
 ssh root@${node} "yum -y update && yum -y install file less man-db htop screen jq golang"; done
```

## Go
At least for [CoreDNS](https://github.com/coredns/coredns),
version 1.12 of Go is required, whereas most of the distributions have older
versions of Go, for instance 1.11.

* Versions:
```bash
VERSION="1.12.5"
OS="linux"
ARCH="amd64"
GOPKG="go${VERSION}.${OS}-${ARCH}.tar.gz"
```

* Download and install the Go binary package on the gateway:
```bash
root@gwkublxc:~# mkdir -p /opt/go/tarballs
root@gwkublxc:~# wget https://dl.google.com/go/${GOPKG} -O /opt/go/tarballs/${GOPKG}
root@gwkublxc:~# tar -C /usr/local -zxf /opt/go/tarballs/${GOPKG} && mv /usr/local/go /usr/local/go${VERSION} && ln -sf /usr/local/go${VERSION} /usr/local/go && mkdir ${HOME}/go
```

* Install Go on the controller nodes:
```bash
root@gwkublxc:~# declare -a node_ctrl_list=("controller1" "controller2")
root@gwkublxc:~# for node in "${node_ctrl_list[@]}"; do rsync -av /opt/go root@${node}:/opt/; done
root@gwkublxc:~# for node in "${node_ctrl_list[@]}"; do ssh root@${node} "tar -C /usr/local -zxf /opt/go/tarballs/${GOPKG} && mv /usr/local/go /usr/local/go${VERSION} && ln -sf /usr/local/go${VERSION} /usr/local/go && mkdir ${HOME}/go"; done
```

* On every controller node, the `~/.bashrc` file has to be updated, like:
```bash
root@controller1:~# cat >> ~/.bashrc << _EOF

# Go
export PATH="\${PATH}:/usr/local/go/bin"
_EOF
root@controller1:~# . ~/.bashrc
```

## Kubernetes binaries
* On the LXC K8S gateway, download Kubernetes:
```bash
root@gwkublxc:~# K8S_VER=$(curl -s https://storage.googleapis.com/kubernetes-release/release/stable.txt)
root@gwkublxc:~# echo $K8S_VER 
v1.14.2
root@gwkublxc:~# declare -a kubbin_list=("kube-apiserver" \
 "kube-controller-manager" "kube-scheduler" "kubectl")
root@gwkublxc:~# for kubbin in "${kubbin_list[@]}"; do \
 curl -LO https://storage.googleapis.com/kubernetes-release/release/${K8S_VER}/bin/linux/amd64/${kubbin} && \
 chmod +x ${kubbin} && mv ${kubbin} /usr/local/bin/; done
```

* Upload the Kubernetes binaries to all the nodes:
```bash
root@gwkublxc:~# for node in "${node_ext_list[@]}"; do \
 rsync -av /usr/local/bin/kube* root@${node}:/usr/local/bin/; done
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
ExecStart=/usr/local/bin/kube-apiserver \
  --admission-control=NamespaceLifecycle,LimitRanger,SecurityContextDeny,ServiceAccount,ResourceQuota \
  --advertise-address=INTERNAL_IP \
  --allow-privileged=true \
  --apiserver-count=3 \
  --authorization-mode=ABAC \
  --authorization-policy-file=/var/lib/kubernetes/authorization-policy.jsonl \
  --bind-address=0.0.0.0 \
  --enable-swagger-ui=true \
  --etcd-cafile=/var/lib/kubernetes/ca.pem \
  --insecure-bind-address=0.0.0.0 \
  --kubelet-certificate-authority=/var/lib/kubernetes/ca.pem \
  --etcd-servers=https://10.240.0.11:2379,https://10.240.0.12:2379 \
  --service-account-key-file=/var/lib/kubernetes/kubernetes-key.pem \
  --service-cluster-ip-range=10.32.0.0/24 \
  --service-node-port-range=30000-32767 \
  --tls-cert-file=/var/lib/kubernetes/kubernetes.pem \
  --tls-private-key-file=/var/lib/kubernetes/kubernetes-key.pem \
  --token-auth-file=/var/lib/kubernetes/token.csv \
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
ExecStart=/usr/local/bin/kube-controller-manager \
  --allocate-node-cidrs=true \
  --cluster-cidr=10.200.0.0/16 \
  --cluster-name=kubernetes \
  --leader-elect=true \
  --master=http://INTERNAL_IP:8080 \
  --root-ca-file=/var/lib/kubernetes/ca.pem \
  --service-account-private-key-file=/var/lib/kubernetes/kubernetes-key.pem \
  --service-cluster-ip-range=10.32.0.0/24 \
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
ExecStart=/usr/local/bin/kube-scheduler \
  --leader-elect=true \
  --master=http://INTERNAL_IP:8080 \
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
* Kubernetes context helper (`kubectx`):
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

* Kubernetes namespace helper (`kubens`):
```bash
root@gwkublxc:~# git clone https://github.com/jonmosco/kube-ps1.git ${HOME}/.kube-ps1
root@gwkublxc:~# cat >> ~/.bashrc << _EOF

# Kubernetes prompt
source ~/.kube-ps1/kube-ps1.sh
PS1='[\u@\h \W \$(kube_ps1)]\\$ '
source <(kubectl completion bash)

_EOF
root@gwkublxc:~# . ~/.bashrc
```

## Check the components
```bash
root@gwkublxc:~# for nodeip in "${nodeip_list[@]}"; do \
 echo "Node: ${nodeip}" && ssh root@${nodeip} "kubectl get componentstatuses"; \
 done
Node: 10.240.0.20
NAME                 STATUS    MESSAGE             ERROR
controller-manager   Healthy   ok                  
scheduler            Healthy   ok                  
etcd-0               Healthy   {"health":"true"}   
etcd-1               Healthy   {"health":"true"}   
Node: 10.240.0.21
NAME                 STATUS    MESSAGE             ERROR
controller-manager   Healthy   ok                  
scheduler            Healthy   ok                  
etcd-0               Healthy   {"health":"true"}   
etcd-1               Healthy   {"health":"true"}   
Node: 10.240.0.22
NAME                 STATUS    MESSAGE             ERROR
controller-manager   Healthy   ok                  
scheduler            Healthy   ok                  
etcd-0               Healthy   {"health":"true"}   
etcd-1               Healthy   {"health":"true"}   
```

# Kubernetes worker nodes
* The software and configuration files are downloaded and setup on the
  Kubernetes gateway (`gwkublxc`), and then sent to the workers from there
```bash
root@gwkublxc:~# $ K8S_VER=$(curl -s https://storage.googleapis.com/kubernetes-release/release/stable.txt)
root@gwkublxc:~# echo $K8S_VER 
v1.14.1
root@gwkublxc:~# declare -a node_list=("worker1" "worker2")
root@gwkublxc:~# declare -a nodeip_list=("10.240.0.31" "10.240.0.32")
```

## Installation of complementary packages
* Install a few complementary packages:
```bash
root@gwkublxc:~# for node in "${nodeip_list[@]}"; do \
 ssh root@${node} "yum -y install file less man-db htop screen jq golang"; done
```

## Kubernetes certificates
* If not already done:
```bash
root@gwkublxc:~# for node in "${nodeip_list[@]}"; do \
 ssh root@${node} "mkdir -p /var/lib/kubernetes && \
  mv ca.pem kubernetes-key.pem kubernetes.pem /var/lib/kubernetes/"; done
```

## Kubernetes binaries
* Download and install the Kubernetes binaries (which are slightly different
  from the ones on the control plane: only `kubectl` is common):
```bash
root@gwkublxc:~# declare -a kubbin_list=("kube-proxy" "kubelet" "kubectl")
root@gwkublxc:~# for kubbin in "${kubbin_list[@]}"; do \
 curl -LO https://storage.googleapis.com/kubernetes-release/release/${K8S_VER}/bin/linux/amd64/${kubbin} && \
 chmod +x ${kubbin} && mv ${kubbin} /usr/local/bin; done
root@gwkublxc:~# for node in "${nodeip_list[@]}"; do \
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
  latest release (as of May 2019, it was `v0.8.0`).
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
root@gwkublxc:~# for node in "${nodeip_list[@]}"; do \
 rsync -av /var/lib/kubelet root@${node}:/var/lib/; done
```

### Kubelet service
* Create the service file:
```bash
root@gwkublxc:~# cat > /etc/systemd/system/kubelet.service << _EOF
[Unit]
Description=Kubernetes Kubelet
Documentation=https://github.com/GoogleCloudPlatform/kubernetes
After=docker.service
Requires=docker.service

[Service]
ExecStart=/usr/local/bin/kubelet \
 --allow-privileged=true \
 --cloud-provider= \
 --cluster-dns=10.32.0.10 \
 --cluster-domain=cluster.local \
 --container-runtime=docker \
 --docker=unix:///var/run/docker.sock \
 --network-plugin=kubenet \
 --kubeconfig=/var/lib/kubelet/kubeconfig \
 --serialize-image-pulls=false \
 --fail-swap-on=false \
 --tls-cert-file=/var/lib/kubernetes/kubernetes.pem \
 --tls-private-key-file=/var/lib/kubernetes/kubernetes-key.pem \
 --v=2

Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
_EOF
```

* Distribute the service file to the nodes:
```bash
root@gwkublxc:~# for node in "${nodeip_list[@]}"; do \
 rsync -av /etc/systemd/system/kubelet.service \
  root@${node}:/etc/systemd/system/; done
root@gwkublxc:~# for node in "${nodeip_list[@]}"; do \
 ssh root@${node} "systemctl daemon-reload && \
  systemctl start kubelet.service && \
  systemctl enable kubelet.service && \
  systemctl status kubelet.service -l"; done
```

## `kube-proxy` configuration
* Create the service file:
```bash
root@gwkublxc:~# cat > /etc/systemd/system/kube-proxy.service << _EOF
[Unit]
Description=Kubernetes Kube Proxy
Documentation=https://github.com/GoogleCloudPlatform/kubernetes

[Service]
ExecStart=/usr/local/bin/kube-proxy \
  --master=https://10.240.0.21:6443 \
  --kubeconfig=/var/lib/kubelet/kubeconfig \
  --proxy-mode=iptables \
  --v=2

Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
_EOF
```

* Distribute the service file to the nodes:
```bash
root@gwkublxc:~# for node in "${nodeip_list[@]}"; do \
 rsync -av /etc/systemd/system/kube-proxy.service \
  root@${node}:/etc/systemd/system/; done
root@gwkublxc:~# for node in "${nodeip_list[@]}"; do \
 ssh root@${node} "systemctl daemon-reload && \
  systemctl start kube-proxy.service && \
  systemctl enable kube-proxy.service && \
  systemctl status kube-proxy.service -l"; done
```

## Check the Kubernetes services on the worker nodes
* On the gateway:
```bash
root@gwkublxc:~# for node in "${nodeip_list[@]}"; do \
 ssh root@${node} "kubectl get componentstatuses"; done
root@gwkublxc:~# for node in "${nodeip_list[@]}"; do \
 ssh root@${node} "kubectl get nodes"; done
```

* On one of the controllers:
```bash
root@controller1:~# kubectl run --generator=run-pod/v1 --rm mytest \
 --image=yauritux/busybox-curl -it
/home # uname -a
Linux mytest 4.15.18-14-pve #1 SMP PVE 4.15.18-38 (Tue, 30 Apr 2019 10:51:33 +0200) x86_64 GNU/Linux
/home # exit
root@controller1:~# kubectl get deployment
NAME     READY   UP-TO-DATE   AVAILABLE   AGE
mytest   1/1     1            1           3m32s
root@controller1:~# kubectl get pods
NAME                      READY   STATUS    RESTARTS   AGE
mytest-589f6788ff-g4vsf   1/1     Running   0          2m26s
root@controller1:~# kubectl delete deployment mytest
deployment.extensions "mytest" deleted
root@controller1:~# kubectl run --generator=run-pod/v1 --rm tvlsim \
 --image=tvlsim/metasim:centos -it
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
root@controller1:~# kubectl delete deployment tvlsim
root@controller1:~# kubectl run --generator=run-pod/v1 tvlsim \
 --image=tvlsim/metasim:centos --replicas=2
pod/tvlsim created
root@controller1:~# kubectl run --generator=run-pod/v1 nginx --image=nginx \
 --port=80 --replicas=2
root@controller1:~# kubectl exec nginx -it -- bash
root@controller1:~# kubectl run --generator=run-pod/v1 alpinenet \
 --image=praqma/network-multitool --replicas=2
root@controller1:~# kubectl exec alpinenet -it -- bash
bash-4.4# ifconfig|grep inet|grep -v 127
          inet addr:10.200.1.9  Bcast:10.200.1.255  Mask:255.255.255.0
bash-4.4# ping 10.200.0.1
bash-4.4# ping 10.200.1.1
bash-4.4# exit
```

* On the Proxmox host, add the routes to the workers:
```bash
root@proxmox:~$ route add -net 10.200.0.0 netmask 255.255.255.0 gw 10.240.0.31
root@proxmox:~$ route add -net 10.200.1.0 netmask 255.255.255.0 gw 10.240.0.32
root@proxmox:~$ route -n
Kernel IP routing table
Destination     Gateway         Genmask         Flags Metric Ref    Use Iface
0.0.0.0         149.202.75.254  0.0.0.0         UG    0      0        0 vmbr0
149.202.75.0    0.0.0.0         255.255.255.0   U     0      0        0 vmbr0
10.30.1.0       0.0.0.0         255.255.255.0   U     0      0        0 vmbr1
10.30.2.0       0.0.0.0         255.255.255.0   U     0      0        0 vmbr2
10.240.0.0      0.0.0.0         255.255.255.0   U     0      0        0 vmbr3
10.200.0.0      10.240.0.31     255.255.255.0   UG    0      0        0 vmbr3
10.200.1.0      10.240.0.32     255.255.255.0   UG    0      0        0 vmbr3
```

* On a controller, test that the pod can ping the workers:
```bash
root@controller1:~# kubectl exec alpinenet -it -- bash
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
bash-4.4# exit
root@controller1:~# kubectl delete pod alpinenet
```

## CoreDNS
* On a controller, download the configuration file of CoreDNS and deploy
  the service:
```bash
root@controller1:~# wget https://storage.googleapis.com/kubernetes-the-hard-way/coredns.yaml -O /var/lib/kubernetes/coredns.yaml -O /var/lib/kubernetes/coredns.yaml
root@controller1:~# kubectl apply -f /var/lib/kubernetes/coredns.yaml
serviceaccount/coredns created
clusterrole.rbac.authorization.k8s.io/system:coredns created
clusterrolebinding.rbac.authorization.k8s.io/system:coredns created
configmap/coredns created
deployment.extensions/coredns created
service/kube-dns created
```

* Check that the service is running:
```bash
root@controller1:~# kubectl get pods -l k8s-app=kube-dns -n kube-system
NAME                       READY   STATUS    RESTARTS   AGE
coredns-699f8ddd77-94qv9   1/1     Running   0          20s
coredns-699f8ddd77-gtcgb   1/1     Running   0          20s
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

