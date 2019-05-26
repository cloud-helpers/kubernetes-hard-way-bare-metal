Kubernetes The Hard Way - Bare Metal with KVM/QEMU Virtual Machines (VM)
========================================================================

![Creative container assembly](https://github.com/cloud-helpers/kubernetes-hard-way-bare-metal/blob/master/kvm-qemu/img/Container%20Ship%20Hamburg%20-%20Pixabay.jpg)

- [Host preparation](#host-preparation)
- [Node creations](#node-creations)
  * [Kubernetes master](#kubernetes-master)
  * [Kubernetes node 1](#kubernetes-node-1)
  * [Kubernetes node 2](#kubernetes-node-2)
  * [Supplementary configuration](#supplementary-configuration)
    + [General](#general)
    + [SSH](#ssh)
    + [Firewalls](#firewalls)
    + [Network](#network)
  * [On the clients (eg, laptops)](#on-the-clients--eg--laptops-)
    + [Kubernetes](#kubernetes)
      - [Kubernetes master](#kubernetes-master-1)
      - [Kubernetes any node](#kubernetes-any-node)
      - [Kubernetes node 1](#kubernetes-node-1-1)
      - [Kubernetes node 2](#kubernetes-node-2-1)
      - [Kubernetes master - Simple status](#kubernetes-master---simple-status)
      - [Smoke test with the simulator](#smoke-test-with-the-simulator)
      - [Smoke test with Nginx](#smoke-test-with-nginx)
      - [Kubernetes master - Troubleshooting](#kubernetes-master---troubleshooting)
      - [Kubernetes master - Deploy Docker images](#kubernetes-master---deploy-docker-images)

# Overview
[This document](https://github.com/cloud-helpers/kubernetes-hard-way-bare-metal/blob/master/kvm-qemu/README.md)
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
containers](https://github.com/cloud-helpers/kubernetes-hard-way-bare-metal/blob/master/lxc/README.md)
for more details on a more lightweight setup.

For instance, that latter is based on the 
["Kubernetes The Hard Way - Bare Metal"
guide](https://github.com/Praqma/LearnKubernetes/blob/master/kamran/Kubernetes-The-Hard-Way-on-BareMetal.md),
which suggests to create many nodes (_e.g._, controller plane, load balancers,
worker nodes). But creating all those nodes as KVM/QEMU VM would load the
(Proxmox) host quite heavily. Hence, for that KVM/QEMU guide, a lighter
setup is prefered, with only one master and two worker nodes.

That guide is therefore an adaptation of the
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

In order to install a KVM/QEMU VM, an ISO image is needed. A minimal version
of CentOS 7 (_e.g._, without graphical environment), released in October 2018,
is available from various sites, for instance from
[TecAdmin in the US](https://tecadmin.net/download-centos-7/) and
[Aachen University in Europe](http://ftp.halifax.rwth-aachen.de/centos/7/isos/x86_64/).
That CentOS 7 ISO image has a size of 920 MB, and needs to be saved in the
`/vz/template/iso` directory of the Proxmox host:
```bash
root@proxmox:~$ ln -s /var/lib/vz /vz && mkdir -p /vz/template/iso
# In the US:
root@proxmox:~$ wget http://isoredirect.centos.org/centos/7/isos/x86_64/CentOS-7-x86_64-Minimal-1810.iso -O /vz/template/iso/CentOS-7-x86_64-Minimal-1810.iso
# In Europe:
root@proxmox:~$ wget http://ftp.halifax.rwth-aachen.de/centos/7/isos/x86_64/CentOS-7-x86_64-Minimal-1810.iso -O /vz/template/iso/CentOS-7-x86_64-Minimal-1810.iso
```

With the ISO image, the virtual machine (VM) has to be installed, exactly
as it would be on a local computer with a DVD player. Since the server
is most proobably remote (be it on a private data center or on a private
cloud), a virtual interface can be used. Hopefully, Proxmox offers a few
options, among which [noVNC](https://novnc.com/info.html),
[Spice](https://www.spice-space.org) or [xterm.js](https://xtermjs.org).

The virtual machines (VM) may first be created through the command-line.
Examples on how to do that for each node are detailed in the sections below.

Once the VM have been created, they must be installed thanks to a virtual
interface (say [noVNC](https://novnc.com/info.html).
From the Proxmox UI (it depends on your hosting provider, but it should
be something like https://admin.example.com:8006):
* On the left hand side menu, right-click on the VM,
  select `>_ Console`, launch the VM, which boots
  on the DVD ISO image.
* Install CentOS 7 with the following network configuration:
  + Host name: `kub-{master,node1,node2}.example.com`
    Check on your DNS zone management that the DNS entry corresponds to
	that configuration
  + Manual (static) IPv4 address (go in the IPv4 tab and add an address):
    `10.240.0.20{0,1,2}`, netmask `255.255.255.0`, gateway `10.240.0.2`
  + DNS servers: `127.0.0.1,8.8.8.8`
  + Additional search domain: `example.com`
* Setup the `ttyS0` serial console to the guest through the same
  `>_ Console` UI, logging into the VM as `root`, editing the
  `/etc/default/grub` Grub configuration file (in order to add
  `console=tty0 console=ttyS0,115200` to the line ending by `quiet`),
  and re-installing the Grub boot-loader:
```bash
$ sed -i -e 's/quiet/quiet console=tty0 console=ttyS0,115200/g' /etc/default/grub
$ grub2-mkconfig --output=/boot/grub2/grub.cfg
$ reboot
```

The next sections give the details on how to create the VM, and once they
have been installed as just mentioned above, how they can be configured.
For the remaining of that guide, the command-line instructions may be
performed by logging into the Proxmox host through SSH. The Proxmox UI
is needed only for the installation of the VM, once they have been created,
and for the configuration of Grub. All the rest does not need any UI.

# Node creations

## Kubernetes master
* Create and start the VM (it also creates the disk image):
```bash
root@proxmox:~$ qm create 200 --bootdisk scsi0 --cores 2 --ide2 local:iso/CentOS-7-x86_64-Minimal-1810.iso,media=cdrom --memory 16384 --name kub-master.example.com --net0 virtio=AA:69:BC:1D:58:81,bridge=vmbr1 --numa 0 --ostype l26 --scsi0 local:200,format=qcow2,size=20G --scsihw virtio-scsi-pci --serial0 socket --sockets 1
root@proxmox:~$ ls -laFh /etc/pve/qemu-server/200.conf
root@proxmox:~$ ls -laFh /var/lib/vz/images/200/vm-200-disk-0.qcow2
root@proxmox:~$ qm start 200
```

* Install the VM and configure Grub as mentioned in the section above

* Connect to the VM (type `root` on the keyboard and `ENTER`, then the password):
```bash
root@proxmox:~$ qm set 200 -serial0 socket
root@proxmox:~$ qm terminal 200 # CTRL-o to exit that mode
```

## Kubernetes node 1
* Create and start the VM (it also creates the disk image)
```bash
root@proxmox:~$ qm create 201 --bootdisk scsi0 --cores 4 --ide2 local:iso/CentOS-7-x86_64-Minimal-1810.iso,media=cdrom --memory 16384 --name kub-node1.example.com --net0 virtio=AA:69:BC:1D:57:82,bridge=vmbr1 --numa 0 --ostype l26 --scsi0 local:201,format=qcow2,size=50G --scsihw virtio-scsi-pci --serial0 socket --sockets 1 --smbios uuid=9a3ff989-9792-4d86-acab-3350e5ab9906 --vmgenid 14195dac-46ae-48b7-9c79-bf047088a1b1
root@proxmox:~$ ls -laFh /etc/pve/qemu-server/201.conf
root@proxmox:~$ ls -laFh /var/lib/vz/images/201/vm-201-disk-0.qcow2
root@proxmox:~$ qm start 201
root@proxmox:~$ qm set 201 -serial0 socket
root@proxmox:~$ qm terminal 201 # CTRL-o to exit that mode
```

* Install the VM and configure Grub as mentioned in the section above

* Connect to the VM (type `root` on the keyboard and `ENTER`, then the password):
```bash
root@proxmox:~$ qm set 201 -serial0 socket
root@proxmox:~$ qm terminal 201 # CTRL-o to exit that mode
```

## Kubernetes node 2
* Create and start the VM (it also creates the disk image)
```bash
root@proxmox:~$ qm create 202 --bootdisk scsi0 --cores 4 --ide2 local:iso/CentOS-7-x86_64-Minimal-1810.iso,media=cdrom --memory 16384 --name kub-node2.example.com --net0 virtio=AA:69:BC:1D:57:83,bridge=vmbr1 --numa 0 --ostype l26 --scsi0 local:202,format=qcow2,size=50G --scsihw virtio-scsi-pci --serial0 socket --sockets 1 --smbios uuid=e6d02bbf-7367-4872-8947-2ac92cb4056b --vmgenid b8900448-0b9e-435e-9966-dad0ab532897
root@proxmox:~$ ls -laFh /etc/pve/qemu-server/202.conf
root@proxmox:~$ ls -laFh /var/lib/vz/images/202/vm-202-disk-0.qcow2
root@proxmox:~$ qm start 202
root@proxmox:~$ qm set 202 -serial0 socket
root@proxmox:~$ qm terminal 202 # CTRL-o to exit that mode
```

* Install the VM and configure Grub as mentioned in the section above

* Connect to the VM (type `root` on the keyboard and `ENTER`, then the password):
```bash
root@proxmox:~$ qm set 202 -serial0 socket
root@proxmox:~$ qm terminal 202 # CTRL-o to exit that mode
```

## Supplementary configuration

### General
```bash
root@proxmox:~$ yum -y update && yum -y install epel-release && \
  yum -y install rpmconf yum-utils htop wget file less net-tools whois bzip2 \
    bash-completion bash-completion-extras openssh-server ntp git tcpdump
root@proxmox:~$ rpmconf -a
root@proxmox:~$ ln -sf /usr/share/zoneinfo/Europe/Paris /etc/localtime
root@proxmox:~$ systemctl start ntpd && systemctl enable ntpd
root@proxmox:~$ setenforce 0
root@proxmox:~$ sed -i --follow-symlinks 's/SELINUX=enforcing/SELINUX=disabled/g' /etc/sysconfig/selinux
```

### SSH
```bash
root@proxmox:~$ mkdir ~/.ssh && chmod 700 ~/.ssh
root@proxmox:~$ cat >> ~/.ssh/authorized_keys << _EOF
ssh-rsa AAAA...AHIrV root@example.com
_EOF
root@proxmox:~$ chmod 600 ~/.ssh/authorized_keys
root@proxmox:~$ cat > ~/.ssh/id_rsa.pub << _EOF
ssh-rsa AAAA...AHIrV root@example.com
_EOF
root@proxmox:~$ systemctl start sshd.service && systemctl enable sshd.service
# Check that you can connect from outside, beginning by from the Proxmox host
root@proxmox:~$ passwd -d root
```

### Firewalls
* Check that the firewalls are not installed:
```bash
root@proxmox:~$ systemctl status firewalld.service && systemctl stop firewalld.service && systemctl disable firewalld.service
root@proxmox:~$ systemctl status iptables.service
```

### Network
* Set the `/etc/hosts` file:
```bash
root@proxmox:~$ cat > /etc/hosts << _EOF
# Local VM
127.0.0.1   localhost localhost.localdomain localhost4 localhost4.localdomain4
::1         localhost localhost.localdomain localhost6 localhost6.localdomain6

# Kubernetes cluster
10.240.0.200  kub-master.example.com kub-master
10.240.0.201  kub-node1.example.com kub-node1
10.240.0.202  kub-node2.example.com kub-node2
_EOF
```

## On the clients (eg, laptops)
* SSH configuration:
```bash
user@laptop$ cat >> ~/.ssh/config << _EOF

# Kubernetes cluster on Proxmox
Host gw.kublxc
  HostName gwkublxc.example.com
  Port 7022
Host kub-master
  HostName kub-master.example.com
  ProxyCommand ssh -W %h:22 root@gw.kublxc
Host kub-node1
  HostName kub-node11.example.com
  ProxyCommand ssh -W %h:22 root@gw.kublxc
Host kub-node2
  HostName kub-node22.example.com
  ProxyCommand ssh -W %h:22 root@gw.kublxc

_EOF
user@laptop$ chmod 600 ~/.ssh/config
```

* Upload the SSH keys onto the K8S gateway:
```bash
user@laptop$ rsync -av your-ssh-keys root@gw.kublxc:~/.ssh/
```

### Kubernetes
* Kubernetes configuration file:
```bash
root@proxmox:~$ cat > /etc/kubernetes/master-kubeconfig.yaml << _EOF
kind: Config
clusters:
- name: local
  cluster:
    server: http://kub-master:8080
users:
- name: kubelet
contexts:
- context:
    cluster: local
    user: kubelet
  name: kubelet-context
current-context: kubelet-context

_EOF
```

* Kubernetes configuration
```bash
root@proxmox:~$ swapoff -a
root@proxmox:~$ sed -i -e 's|/dev/mapper/centos-swap|#/dev/mapper/centos-swap|g' /etc/fstab
root@proxmox:~$ modprobe br_netfilter
root@proxmox:~$ echo '1' > /proc/sys/net/bridge/bridge-nf-call-iptables
root@proxmox:~$ yum -y install kubernetes flannel
root@proxmox:~$ cat >> ~/.bashrc << _EOF

# Kubernetes
export KUBECONFIG=/etc/kubernetes/master-kubeconfig.yaml
_EOF
root@proxmox:~$ . ~/.bashrc
```

#### Kubernetes master
* Set up the Kubernetes master:
```bash
root@proxmox:~$ ssh kub-master
root@kub-master:~# yum -y install etcd kubernetes
root@kub-master:~# sed -i -e 's/KUBE_MASTER="--master=http:\/\/127.0.0.1:8080"/KUBE_MASTER="--master=http:\/\/kub-master:8080"/g' /etc/kubernetes/config
```

* Set up the services (on the master)
```bash
root@kub-master:~# sed -i -e 's/KUBE_API_ADDRESS="--insecure-bind-address=127.0.0.1"/KUBE_API_ADDRESS="--address=0.0.0.0"/g' /etc/kubernetes/apiserver
root@kub-master:~# sed -i -e 's/^KUBE_ADMISSION_CONTROL=/#KUBE_ADMISSION_CONTROL=/g' /etc/kubernetes/apiserver
root@kub-master:~# sed -i -e 's/ETCD_LISTEN_CLIENT_URLS="http:\/\/localhost:2379"/ETCD_LISTEN_CLIENT_URLS="http:\/\/0.0.0.0:2379"/g' /etc/etcd/etcd.conf
root@kub-master:~# sed -i -e 's/KUBELET_ADDRESS="--address=127.0.0.1"/KUBELET_ADDRESS="--address=0.0.0.0"/g' /etc/kubernetes/kubelet
root@kub-master:~# sed -i -e 's/KUBELET_HOSTNAME="--hostname-override=127.0.0.1"/KUBELET_HOSTNAME="--hostname-override=kub-master"/g' /etc/kubernetes/kubelet
root@kub-master:~# sed -i -e 's/KUBELET_API_SERVER="--api-servers=http:\/\/127.0.0.1:8080"/KUBELET_ARGS="--cgroup-driver=cgroupfs --kubeconfig=\/etc\/kubernetes\/master-kubeconfig.yaml --require-kubeconfig"/g' /etc/kubernetes/kubelet
root@kub-master:~# sed -i -e 's/^KUBELET_ARGS=""/#KUBELET_ARGS=""/g' /etc/kubernetes/kubelet
```

* Start the appropriate services on master:
```bash
root@kub-master:~# declare -a kubsvc_list=(etcd kube-apiserver kube-controller-manager kube-scheduler)
root@kub-master:~# for kubsvc in ${kubsvc_list[*]}; do systemctl restart $kubsvc && systemctl enable $kubsvc && systemctl status $kubsvc -l; done
```

* Check `etcd`:
```bash
root@kub-master:~# etcdctl cluster-health
member 8e9e05c52164694d is healthy: got healthy result from http://10.240.0.200:2379
cluster is healthy
root@kub-master:~# etcdctl member list
8e9e05c52164694d: name=default peerURLs=http://localhost:2380 clientURLs=http://10.240.0.200:2379 isLeader=true
```

* Adda few aliases:
```bash
root@kub-master:~# cat > ~/.bash_aliases << _EOF

# Kubernetes
kubsvc_list=(etcd kube-apiserver kube-controller-manager kube-scheduler)
alias kubsvcrestart='for kubsvc in \${kubsvc_list[*]}; do systemctl restart \$kubsvc; done'
alias kubsvcstop='for kubsvc in \${kubsvc_list[*]}; do systemctl stop \$kubsvc; done'
alias kubsvcstart='for kubsvc in \${kubsvc_list[*]}; do systemctl start \$kubsvc; done'
alias kubsvcstatus='for kubsvc in \${kubsvc_list[*]}; do systemctl status \$kubsvc -l; done'

_EOF
```

* Flannel:
```bash
root@kub-master:~# cat > /etc/kubernetes/flannel.json << _EOF
{
    "Network": "172.17.0.0/16",
    "SubnetLen": 24,
    "Backend": {
        "Type": "vxlan",
        "VNI": 1
     }
}
_EOF
```

* Define flannel network configuration in `etcd`:
```bash
root@kub-master:~# etcdctl set /atomic.io/network/config < /etc/kubernetes/flannel.json
root@kub-master:~# etcdctl mk /atomic.io/network/config '{"Network":"172.17.0.0/16"}'
{"Network":"172.17.0.0/16"}
```

* Check a the flannel network configuration:
```bash
root@kub-master:~# grep "FLANNEL_ETCD_PREFIX" /etc/sysconfig/flanneld 
FLANNEL_ETCD_PREFIX="/atomic.io/network"
root@kub-master:~# etcdctl ls /atomic.io/network/subnets
/atomic.io/network/subnets/172.17.19.0-24
/atomic.io/network/subnets/172.17.45.0-24
/atomic.io/network/subnets/172.17.46.0-24
root@kub-master:~# etcdctl get /atomic.io/network/subnets/172.17.19.0-24
{"PublicIP":"10.240.0.200"}
root@kub-master:~# etcdctl get /atomic.io/network/subnets/172.17.45.0-24
{"PublicIP":"10.240.0.201"}
root@kub-master:~# etcdctl get /atomic.io/network/subnets/172.17.46.0-24
{"PublicIP":"10.240.0.202"}
```

#### Kubernetes any node
* Kubernetes configuration file:
```bash
root@proxmox:~$ ssh kub-nodeX
root@kub-nodeX:~# cat > /etc/kubernetes/master-kubeconfig.yaml << _EOF
kind: Config
clusters:
- name: local
  cluster:
    server: http://kub-master:8080
users:
- name: kubelet
contexts:
- context:
    cluster: local
    user: kubelet
  name: kubelet-context
current-context: kubelet-context

_EOF
```

* Set up the flannel service in the nodes:
```bash
root@kub-nodeX:~# sed -i -e 's/^FLANNEL_ETCD_ENDPOINTS="http:\/\/127.0.0.1:2379"/FLANNEL_ETCD_ENDPOINTS="http:\/\/kub-master:2379"/g' /etc/sysconfig/flanneld
```

* Add the `docker ` group:
```bash
root@kub-nodeX:~# groupadd docker
root@kub-nodeX:~# #usermod -aG docker $USER
```

* Specify the Kubernetes master on the node:
```bash
root@kub-nodeX:~# sed -i -e 's/KUBE_MASTER="--master=http:\/\/127.0.0.1:8080"/KUBE_MASTER="--master=http:\/\/kub-master:8080"/g' /etc/kubernetes/config
```

* Set up the services in the nodes:
```bash
root@kub-nodeX:~# sed -i -e 's/KUBELET_ADDRESS="--address=127.0.0.1"/KUBELET_ADDRESS="--address=0.0.0.0"/g' /etc/kubernetes/kubelet
root@kub-nodeX:~# sed -i -e 's/KUBELET_HOSTNAME="--hostname-override=127.0.0.1"/KUBELET_HOSTNAME="--hostname-override=kub-node1"/g' /etc/kubernetes/kubelet
root@kub-nodeX:~# sed -i -e 's/KUBELET_API_SERVER="--api-servers=http:\/\/127.0.0.1:8080"/KUBELET_ARGS="--cgroup-driver=cgroupfs --kubeconfig=\/etc\/kubernetes\/master-kubeconfig.yaml --require-kubeconfig"/g' /etc/kubernetes/kubelet
root@kub-nodeX:~# sed -i -e 's/^KUBELET_ARGS=""/#KUBELET_ARGS=""/g' /etc/kubernetes/kubelet
```

#### Kubernetes node 1
* Set up the Kubernetes node 1:
```bash
root@kub-node1:~# yum -y remove docker docker-common container-selinux docker-selinux docker-engine
root@kub-node1:~# yum -y install kubernetes flannel
root@kub-node1:~# cat > /etc/yum.repos.d/kubernetes.repo << _EOF
[kubernetes]
name=Kubernetes
baseurl=https://packages.cloud.google.com/yum/repos/kubernetes-el7-x86_64
enabled=0
gpgcheck=1
repo_gpgcheck=1
gpgkey=https://packages.cloud.google.com/yum/doc/yum-key.gpg
        https://packages.cloud.google.com/yum/doc/rpm-package-key.gpg
_EOF
```

* Start the appropriate services on the nodes:
```bash
root@kub-node1:~# kubsvc_list=(kube-proxy kube-apiserver kubelet flanneld docker)
root@kub-node1:~# for kubsvc in ${kubsvc_list[*]}; do systemctl restart $kubsvc && systemctl enable $kubsvc && systemctl status $kubsvc -l; done
```

* Add a few aliases:
```bash
root@kub-node1:~# cat >> ~/.bash_aliases << _EOF

# Kubernetes
kubsvc_list=(kube-proxy kube-apiserver kubelet flanneld docker)
alias kubsvcrestart='for kubsvc in \${kubsvc_list[*]}; do systemctl restart \$kubsvc; done'
alias kubsvcstop='for kubsvc in \${kubsvc_list[*]}; do systemctl stop \$kubsvc; done'
alias kubsvcstart='for kubsvc in \${kubsvc_list[*]}; do systemctl start \$kubsvc; done'
alias kubsvcstatus='for kubsvc in \${kubsvc_list[*]}; do systemctl status \$kubsvc -l; done'
_EOF
```

#### Kubernetes node 2
* Set up the Kubernetes node 2:
```bash
root@kub-node2:~# yum -y remove docker docker-common container-selinux docker-selinux docker-engine
root@kub-node2:~# yum -y install kubernetes flannel
root@kub-node2:~# cat > /etc/yum.repos.d/kubernetes.repo << _EOF
[kubernetes]
name=Kubernetes
baseurl=https://packages.cloud.google.com/yum/repos/kubernetes-el7-x86_64
enabled=0
gpgcheck=1
repo_gpgcheck=1
gpgkey=https://packages.cloud.google.com/yum/doc/yum-key.gpg
        https://packages.cloud.google.com/yum/doc/rpm-package-key.gpg
_EOF
```

* Start the appropriate services on the nodes:
```bash
root@kub-node2:~# kubsvc_list=(kube-proxy kube-apiserver kubelet flanneld docker)
root@kub-node2:~# for kubsvc in ${kubsvc_list[*]}; do systemctl restart $kubsvc && systemctl enable $kubsvc && systemctl status $kubsvc -l; done
```

* Add a few aliases:
```bash
root@kub-node2:~# cat >> ~/.bash_aliases << _EOF

# Kubernetes
kubsvc_list=(kube-proxy kube-apiserver kubelet flanneld docker)
alias kubsvcrestart='for kubsvc in \${kubsvc_list[*]}; do systemctl restart \$kubsvc; done'
alias kubsvcstop='for kubsvc in \${kubsvc_list[*]}; do systemctl stop \$kubsvc; done'
alias kubsvcstart='for kubsvc in \${kubsvc_list[*]}; do systemctl start \$kubsvc; done'
alias kubsvcstatus='for kubsvc in \${kubsvc_list[*]}; do systemctl status \$kubsvc -l; done'
_EOF
```

#### Kubernetes master - Simple status
* Check that the master sees the nodes:
```bash
root@kub-master:~# kubectl get nodes
NAME         STATUS     AGE
kub-node1    Ready      4m
kub-node2    Ready      25m
root@kub-master:~# kubectl get rs,deploy,po,svc,ds --all-namespaces
root@kub-master:~# kubectl get componentstatus
NAME                 STATUS    MESSAGE             ERROR
scheduler            Healthy   ok                  
controller-manager   Healthy   ok                  
etcd-0               Healthy   {"health":"true"}   
root@kub-master:~# kubectl run -it --generator=run-pod/v1 --image=hello-world hello-app
deployment "hello-app" created
root@kub-master:~# kubectl get deployments
NAME        DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
hello-app   1         0         0            0           2m
root@kub-master:~# kubectl get endpoints
NAME         ENDPOINTS          AGE
kubernetes   10.240.0.200:6443   2h
root@kub-master:~# kubectl get ns
NAME          STATUS    AGE
default       Active    2h
kube-system   Active    2h
root@kub-master:~# kubectl get svc
NAME         CLUSTER-IP   EXTERNAL-IP   PORT(S)   AGE
kubernetes   10.254.0.1   <none>        443/TCP   2h
root@kub-master:~# curl -s http://kub-master:2379/v2/keys/atomic.io/network/subnets | jq
{
  "action": "get",
  "node": {
    "key": "/atomic.io/network/subnets",
    "dir": true,
    "nodes": [
      {
        "key": "/atomic.io/network/subnets/172.17.45.0-24",
        "value": "{\"PublicIP\":\"10.240.0.201\"}",
        "expiration": "2019-05-26T18:11:11.357098719Z",
        "ttl": 85670,
        "modifiedIndex": 973,
        "createdIndex": 973
      },
      {
        "key": "/atomic.io/network/subnets/172.17.46.0-24",
        "value": "{\"PublicIP\":\"10.240.0.202\"}",
        "expiration": "2019-05-26T18:11:18.773635788Z",
        "ttl": 85678,
        "modifiedIndex": 997,
        "createdIndex": 997
      },
      {
        "key": "/atomic.io/network/subnets/172.17.19.0-24",
        "value": "{\"PublicIP\":\"10.240.0.200\"}",
        "expiration": "2019-05-26T18:10:20.049457838Z",
        "ttl": 85619,
        "modifiedIndex": 921,
        "createdIndex": 921
      }
    ],
    "modifiedIndex": 921,
    "createdIndex": 921
  }
}
root@kub-master:~# kubectl api-versions
apps/v1beta1
authentication.k8s.io/v1beta1
authorization.k8s.io/v1beta1
autoscaling/v1
batch/v1
certificates.k8s.io/v1alpha1
extensions/v1beta1
policy/v1beta1
rbac.authorization.k8s.io/v1alpha1
storage.k8s.io/v1beta1
v1
```

#### Smoke test with the simulator
```bash
root@kub-master:~# kubectl run -it --generator=run-pod/v1 --image=tvlsim/metasim:centos tvlsim-app
Waiting for pod default/tvlsim-app to be running, status is Pending, pod ready: false
If you don't see a command prompt, try pressing enter.
[build@tvlsim-app sim]$ workspace/install/airinv/bin/airinv
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
[build@tvlsim-app sim]$ exit
exit
Session ended, resume using 'kubectl attach tvlsim-app -c tvlsim-app -i -t' command when the pod is running
root@kub-master:~# kubectl get ds,deploy,po,rs,svc --all-namespaces
NAMESPACE   NAME            READY     STATUS    RESTARTS   AGE
default     po/tvlsim-app   1/1       Running   1          4m

NAMESPACE   NAME             CLUSTER-IP   EXTERNAL-IP   PORT(S)   AGE
default     svc/kubernetes   10.254.0.1   <none>        443/TCP   36m
root@kub-master:~# kubectl exec -it tvlsim-app -- bash
[build@tvlsim-app sim]$ exit
exit
root@kub-master:~# kubectl delete po tvlsim-app
pod "tvlsim-app" deleted
root@kub-master:~# kubectl get ds,deploy,po,rs,svc --all-namespaces
NAMESPACE   NAME         CLUSTER-IP   EXTERNAL-IP   PORT(S)   AGE
default     kubernetes   10.254.0.1   <none>        443/TCP   39m
```

#### Smoke test with Nginx
* On the master, create a deployment YAML file for the Nginx containers,
  and apply it:
```bash
root@kub-master:~# cat > /etc/kubernetes/nginx-pod.yaml << _EOF
apiVersion: v1
kind: Service
metadata:
  name: nginx-app
  labels:
    app: nginx
spec:
  selector:
    app: nginx
  type: NodePort
  ports:
  - port: 80
    targetPort: 80
---
apiVersion: v1
kind: Pod
metadata:
  name: nginx-app
  labels:
    app: nginx
spec:
  containers:
  - image: nginx:latest
    name: nginx-app
    ports:
    - containerPort: 80
  dnsPolicy: ClusterFirst
  securityContext
  - capabilities
    - add: NET_ADMIN
#  dnsConfig:
#    nameservers:
#      - 213.186.33.99
#      - 8.8.8.8
#      - 10.240.0.200
#      - 172.17.19.0
#    searches:
#      - svc.cluster.local
#      - example.com
_EOF
root@kub-master:~# kubectl create -f /etc/kubernetes/nginx-pod.yaml
service "nginx-app" created
pod "nginx-app" created
```

* As an alternative, the pod and service may also be create from
  the command-line:
```bash
root@kub-master:~# kubectl run --generator=run-pod/v1 --image=nginx:latest nginx-app --port 80 --labels app=nginx
pod "nginx-app" created
root@kub-master:~# kubectl get po -l app=nginx -o wide
NAME        READY     STATUS    RESTARTS   AGE       IP            NODE
nginx-app   1/1       Running   1          10m       172.17.46.2   kub-node2
root@kub-master:~# kubectl expose pod nginx-app -l app=nginx --type NodePort
service "nginx-app" exposed
root@kub-master:~# kubectl get svc -o wide
NAME         CLUSTER-IP      EXTERNAL-IP   PORT(S)        AGE       SELECTOR
kubernetes   10.254.0.1      <none>        443/TCP        5h        <none>
nginx-app    10.254.129.53   <nodes>       80:30597/TCP   7m        app=nginx
```

* From the master, launch a Shell prompt on the Nginx container. Kubernetes
  connects to the (Docker) container on the worker node (`kub-node2` above):
```bash
root@kub-master:~# kubectl exec -it nginx-app -- bash
root@nginx-app:/# apt-get update && apt-get -y install procps iproute2 iputils-ping wget less net-tools htop curl vim
root@nginx-app:/# ifconfig
eth0: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1472
        inet 172.17.46.2  netmask 255.255.255.0  broadcast 0.0.0.0
root@nginx-app:/# ip route show
default via 172.17.46.1 dev eth0 
172.17.46.0/24 dev eth0 proto kernel scope link src 172.17.46.2
# The following does not work
root@nginx-app:/# ip route add 172.17.19.0/24 via 172.17.19.2 dev eth0
root@nginx-app:/# route add -net 172.17.19.0 netmask 255.255.255.0 gw 172.17.19.2
root@nginx-app:/# ip route add 172.17.45.0/24 via 172.17.45.2 dev eth0
root@nginx-app:/# route add -net 172.17.45.0 netmask 255.255.255.0 gw 172.17.45.2
root@nginx-app:/# ip route add 172.17.46.0/24 via 172.17.46.2 dev eth0
root@nginx-app:/# route add -net 172.17.46.0 netmask 255.255.255.0 gw 172.17.46.2
```

* From the master, query Nginx through the workers (either from their IP
  or host name), specifying the node-port:
```bash
root@kub-master:~# NODE_PORT=$(kubectl get svc nginx-app --output=jsonpath='{range .spec.ports[0]}{.nodePort}')
root@kub-master:~# curl http://kub-node2:30597
```

* From the master, query Nginx through a POD, which does not require
  the knowledge of the port:
```bash
root@kub-master:~# declare -a pod_list=($(kubectl get pods -o go-template --template '{{range .items}}{{.metadata.name}}{{"\n"}}{{end}}' | grep nginx))
root@kub-master:~# for pod in "${pod_list[@]}"; do kubectl exec ${pod} -it -- ip addr show; done | grep inet | grep -v inet6 | grep -v 127
    inet 172.17.46.2/24 scope global eth0
root@kub-master:~# curl http://172.17.46.2
curl: (7) Failed connect to 172.17.46.2:80; No route to host
```

* From the worker node where the Nginx pod is running (`kub-node2` above):
```bash
$ ssh kub-node2
root@kub-nide2:~# docker ps
CONTAINER ID        IMAGE                                      COMMAND                  CREATED              STATUS              PORTS               NAMES
ed654398e6d1        nginx:latest                               "nginx -g 'daemon ..."   About a minute ago   Up About a minute                       k8s_nginx-app.cc7903ed_nginx-app_default_6310d5b1-7f41-11e9-bd04-aa69bc1d5781_14edda1d
440307ccd890        gcr.io/google_containers/pause-amd64:3.0   "/pause"                 About a minute ago   Up About a minute                       k8s_POD.b2390301_nginx-app_default_6310d5b1-7f41-11e9-bd04-aa69bc1d5781_40f22229
root@kub-node2:~# docker exec -it ed654398e6d1 bash
root@nginx-app:/# ifconfig
eth0: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1472
        inet 172.17.46.2  netmask 255.255.255.0  broadcast 0.0.0.0
root@nginx-app:/# exit
root@kub-node2:~$ exit
```

#### Kubernetes master - Troubleshooting
* Check the nodes:
```bash
root@kub-master:~# kubectl get nodes
NAME        STATUS    AGE
kub-node1   Ready     52s
kub-node2   Ready     45s
```

* Check the all the components:
```bash
root@kub-master:~# kubectl get componentstatus
NAME                 STATUS    MESSAGE             ERROR
controller-manager   Healthy   ok                  
scheduler            Healthy   ok                  
etcd-0               Healthy   {"health":"true"}   
```

* Check the status of the main components/resources:
```bash
root@kub-master:~# kubectl get ds,deploy,po,rs,svc --all-namespaces
NAMESPACE   NAME         CLUSTER-IP   EXTERNAL-IP   PORT(S)   AGE
default     kubernetes   10.254.0.1   <none>        443/TCP   1m
```

* If, for some reason, some resources cannot be deleted,
  the `etcd` database can be reset, and the service restarted:
```bash
root@kub-master:~# systemctl stop etcd.service
root@kub-master:~# rm -rf /var/lib/etcd/default.etcd/member
root@kub-master:~# systemctl start etcd.service && systemctl status etcd.service -l
```

#### Kubernetes master - Deploy Docker images
* On a worker node, check that the underlying Docker image is ready and works:
```bash
root@kub-nodeX:~# docker run -it --rm tvlsim/metasim:centos bash
[build@78b541cc6d5d sim]$ exit
```

* Deploy the travel simulator on the Kubernetes cluster:
```bash
root@kub-master:~# kubectl run --generator=run-pod/v1 --image=tvlsim/metasim:centos tvlsim-app --rm -it
If you don't see a command prompt, try pressing enter.
[root@tvlsim-app sim]# exit
exit
Session ended, resume using 'kubectl attach tvlsim-app -c tvlsim-app -i -t' command when the pod is running
pod "tvlsim-app" deleted
```

* Check the deployment status:
```bash
root@kub-master:~# kubectl get deployment tvlsim-app -o yaml
root@kub-master:~# kubectl get deployment
NAME         DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
tvlsim-app   1         1         1            0           1m
root@kub-master:~# kubectl get pods
NAME                        READY     STATUS              RESTARTS   AGE
tvlsim-app-75408578-cl5rp   0/1       ContainerCreating   0          52s
```

* Launch a Shell terminal within the simulator container:
```bash
root@kub-master:~# export POD_NAME=$(kubectl get pods -o go-template --template '{{range .items}}{{.metadata.name}}{{"\n"}}{{end}}')
root@kub-master:~# echo $POD_NAME 
tvlsim-app-75408578-cl5rp
root@kub-master:~# kubectl exec -ti $POD_NAME bash
```

* Delete the deployment:
```bash
root@kub-master:~# kubectl delete deployment tvlsim-app
```

