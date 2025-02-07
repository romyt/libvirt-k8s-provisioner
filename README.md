[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)

# libvirt-k8s-provisioner - Automate your cluster provisioning from 0 to k8s!

Welcome to the home of the project!

With this project, you can build up in minutes a fully working k8s cluster (single master/HA) with as many worker nodes as you want.

# DISCLAIMER

It is a hobby project, so it's not supported for production usage, but feel free to open issues and/or contributing to it!

# How does it work?

Kubernetes version that is installed can be choosen between:

- **1.27** - Latest 1.27 release (1.27.4)
- **1.26** - Latest 1.26 release (1.26.7)
- **1.25** - Latest 1.25 release (1.25.12)
- **1.24** - Latest 1.24 release (1.24.16)

Terraform will take care of the provisioning via terraform of:

- Loadbalancer machine with **haproxy** installed and configured for **HA** clusters
- k8s Master(s) VM(s)
- k8s Worker(s) VM(s)

It also takes care of preparing the host machine with needed packages, configuring:

- dedicated libvirt dnsmasq configuration
- dedicated libvirt network (fully customizable)
- dedicated libvirt storage pool (fully customizable)
- libvirt-terraform-provider ( compiled and initialized based on [https://github.com/dmacvicar/terraform-provider-libvirt](https://github.com/dmacvicar/terraform-provider-libvirt))

You can customize the setup choosing:

- **container runtime** that you want to use (**cri-o, containerd**).
- **schedulable master** if you want to schedule on your master nodes or leave the taint.
- **service CIDR** to be used during installation.
- **pod CIDR** to be used during installation.
- **network plugin** to be used, based on the documentation. **[Project Calico](https://www.projectcalico.org/calico-networking-for-kubernetes/)** **[Flannel](https://github.com/coreos/flannel)** **[Project Cilium](https://cilium.io/)**
- **additional SANS** to be added to api-server
- **[nginx-ingress-controller](https://kubernetes.github.io/ingress-nginx/)**, **[haproxy-ingress-controller](https://github.com/haproxytech/kubernetes-ingress)** or **[Project Contour](https://projectcontour.io/)** if you want to enable ingress management.
- **[metalLB](https://metallb.universe.tf/)** to manage bare-metal LoadBalancer services - **WIP** - Only L2 configuration can be set-up via playbook.
- **[Rook-Ceph](https://rook.io/docs/rook/v1.4/ceph-storage.html)** - To manage persistent storage, also configurable with single storage node.

## All VMs are specular,prepared with:

- OS:

  - Ubuntu 22.04 LTS Cloud base image [https://cloud-images.ubuntu.com/releases/jammy/release/](https://cloud-images.ubuntu.com/releases/jammy/release/)
  - Centos Stream 8 Generic Cloud base image [https://cloud.centos.org/centos/8-stream/x86_64/images/](https://cloud.centos.org/centos/8-stream/x86_64/images/)

- cloud-init:
  - user: **kube**
  - pass: **kuberocks**
  - ssh-key: generated during vm-provisioning and stores in the project folder

The user is capable of logging via SSH too.

## Quickstart

The playbook is meant to be ran against a local host or a remote host that has access to subnets that will be created, defined under **vm_host** group, depending on how many clusters you want to configure at once.

First of all, you need to install required collections to get started:

    ansible-galaxy collection install -r requirements.yml

Once the collections are installed, you can simply run the playbook:

    ansible-playbook main.yml

You can quickly make it work by configuring the needed vars, but you can go straight with the defaults!

You can also install your cluster using the **Makefile** with:

To install collections:

    make setup

To install the cluster:

    make create

## Quickstart with Execution Environment

The playbooks are compatible with the newly introduced **Execution environments (EE)**. To use them with an execution environment you need to have [ansible-builder](https://ansible-builder.readthedocs.io/en/stable/) and [ansible-navigator](https://ansible-navigator.readthedocs.io/en/latest/) installed.

### Build EE image

To build the EE image, jump in the _execution-environment_ folder and run the build:

    ansible-builder build -f execution-environment/execution-environment.yml -t k8s-ee

### Run playbooks

To run the playbooks use ansible navigator:

    ansible-navigator run main.yml -m stdout

## Recommended sizing

Recommended sizings are:

| Role   | vCPU | RAM |
| ------ | ---- | --- |
| master | 2    | 2G  |
| worker | 2    | 2G  |

**vars/k8s_cluster.yml**

# General configuration

    k8s:
      cluster_name: k8s-test
      cluster_os: Ubuntu
      cluster_version: 1.24
      container_runtime: crio
      master_schedulable: false

    # Nodes configuration

      control_plane:
        vcpu: 2
        mem: 2
        vms: 3
        disk: 30

      worker_nodes:
        vcpu: 2
        mem: 2
        vms: 1
        disk: 30

    # Network configuration

      network:
        network_cidr: 192.168.200.0/24
        domain: k8s.test
        additional_san: ""
        pod_cidr: 10.20.0.0/16
        service_cidr: 10.110.0.0/16
        cni_plugin: cilium

    rook_ceph:
      install_rook: false
      volume_size: 50
          rook_cluster_size: 1

    # Ingress controller configuration [nginx/haproxy]

    ingress_controller:
      install_ingress_controller: true
      type: haproxy
          node_port:
            http: 31080
            https: 31443

    # Section for metalLB setup

    metallb:
      install_metallb: false

l2:
iprange: 192.168.200.210-192.168.200.250

Size for **disk** and **mem** is in GB.
**disk** allows to provision space in the cloud image for pod's ephemeral storage.

**cluster_version** can be 1.20, 1.21, 1.22, 1.23, 1.24, 1.25 to install the corresponding latest version for the release

VMS are created with these names by default (customizing them is work in progress):

    - **cluster_name**-loadbalancer.**domain**
    - **cluster_name**-master-N.**domain**
    - **cluster_name**-worker-N.**domain**

It is possible to choose **CentOS**/**Ubuntu** as **kubernetes hosts OS**

## Multiple clusters - Thanks to @3rd-st-ninja for the input

Since last release, it is now possible to provision multiple clusters on the same host. Each cluster will be self consistent and will have its own folder under the /**/home/user/k8ssetup/clusters** folder in playbook root folder.

    clusters
    └── k8s-provisioner
    	├── admin.kubeconfig
    	├── haproxy.cfg
    	├── id_rsa
    	├── id_rsa.pub
    	├── libvirt-resources
    	│   ├── libvirt-resources.tf
    	│   └── terraform.tfstate
    	├── loadbalancer
    	│   ├── cloud_init.cfg
    	│   ├── k8s-loadbalancer.tf
    	│   └── terraform.tfstate
    	├── masters
    	│   ├── cloud_init.cfg
    	│   ├── k8s-master.tf
    	│   └── terraform.tfstate
    	├── workers
    	│   ├── cloud_init.cfg
    	│   ├── k8s-workers.tf
    	│   └── terraform.tfstate
    	└── workers-rook
    	    ├── cloud_init.cfg
    	    └── k8s-workers.tf

In the main folder will be provided a custom script for removing the single cluster, without touching others.

    k8s-provisioner-cleanup-playbook.yml

As well as a separated inventory for each cluster:

    k8s-provisioner-inventory-k8s

In order to keep clusters separated, ensure that you use a different **k8s.cluster_name**,**k8s.network.domain** and **k8s.network.network_cidr** variables.

## Rook

**Rook** setup actually creates a dedicated kind of worker, with an additional volume on the VMs that are required. Now it is possible to select the size of Rook cluster using **rook_ceph.rook_cluster_size** variable in the settings.

## MetalLB

Basic setup taken from the documentation. At the moment, the parameter **l2** reports the IPs that can be used (defaults to some IPs in the same subnet of the hosts) as 'external' IPs for accessing the applications

Suggestion and improvements are highly recommended!
Alex



=========================

Running the Ansible playbook to create and configure virtual machines on the virtualization host 1/2
From a root shell on the virtualization server, enter the following commands:


ansible-playbook main.yml
The task sequence will end with this error:


fatal: [k8s-test-worker-0.k8s.test]: FAILED! => {"changed": false, "elapsed": 600, "msg": "timed out waiting for ping module test: Failed to connect to the host via ssh: ssh: Could not resolve hostname k8s-test-worker-0.k8s.test: Name or service not known"}
fatal: [k8s-test-master-0.k8s.test]: FAILED! => {"changed": false, "elapsed": 600, "msg": "timed out waiting for ping module test: Failed to connect to the host via ssh: ssh: Could not resolve hostname k8s-test-master-0.k8s.test: Name or service not known"}
Note: we will recover from this error in a later step.

Preparing the virtualization server 3/3
----------------------------------------
From a root shell on the virtualization server, enter the following command:


virsh net-dhcp-leases k8s-test
Information about the virtual machines in the k8s-test network will be displayed:


root@henderson:/home/desktop# virsh net-dhcp-leases k8s-test
Expiry Time MAC address Protocol IP address Hostname Client ID or DUID
2022-07-29 07:21:42 52:54:00:4a:20:99 ipv4 192.168.200.99/24 k8s-test-master-0 ff:b5:5e:67:ff:00:02:00:00:ab:11:28:1f:a1:fb:24:5c:f5:70
2022-07-29 07:21:42 52:54:00:86:29:8f ipv4 192.168.200.28/24 k8s-test-worker-0 ff:b5:5e:67:ff:00:02:00:00:ab:11:9e:22:e1:40:72:21:cf:9d
Take note of the IP addresses starting with 192.168.200, these values will be needed in a later configuration step.


=========================
Understanding the need for IP forwarding on the virtualization server
By default, virtual machines are created with IP addresses in the 192.168.200.x subnet. This subnet is accessible within the virtualization server.

In order to make the 192.168.200.x subnet accessible to the automation server, we need to create a gateway router using iptables directives on the virtualization server.

In a later step, we will add a default route for the 192.168.200.x subnet on the automation server, allowing it to resolve IP addresses in that subnet.

Enabling IP forwarding for the 192.168.200.x subnet
From a root shell on the virtualization server, enter the following commands:

Use the nano editor to create the following text file:

cd /etc
nano sysctl.conf
Add the following line to the end of the sysctl.conf file:

net.ipv4.ip_forward = 1
Enter this command:

sysctl -p
Use the nano editor to create the following text file (substitute the wanadaptername and wanadapterip for those of the virtualization server in your setup):

nano forward.sh
contents:

#!/usr/bin/bash
# values
kvmsubnet="192.168.200.0/24"
wanadaptername="eno1"
wanadapterip="192.168.56.60"
kvmadaptername="k8s-test"
kvmadapterip="192.168.200.1"
# allow virtual adapter to accept packets from outside the host
iptables -I FORWARD -i $wanadaptername -o $kvmadaptername -d $kvmsubnet -j ACCEPT
iptables -I FORWARD -i $kvmadapterip -o $wanadaptername -s $kvmsubnet -j ACCEPT
iptables --table nat --append POSTROUTING --out-interface eth1 -j MASQUERADE

Enter the following commands:

chmod 755 forward.sh
bash forward.sh
Note: add invocation to /etc/rc.local for persistence.

Preparing the automation server 2/2
Adding IP addresses for the VM hosts created in “Preparing the virtualization server 3/3”
From a root shell on the automation server, enter the following commands:

Use the nano editor to create the following text file:

cd /etc
nano sysctl.conf
Add the following line to the end of the sysctl.conf file:

net.ipv4.ip_forward = 1
Enter this command:

sysctl -p
Use the nano editor to modify the /etc/hosts file:

nano hosts
Add the following lines (substitute the IP addresses observed earlier in “Preparing the virtualization server 3/3”):

192.168.200.99 k8s-test-master-0.k8s.test
192.168.200.28 k8s-test-worker-0.k8s.test
Adding a route for for the 192.168.200.x subnet:
From a root shell on the automation server, enter the following command (substitute the wanadaptername (dev) and wanadapterip for those of the virtualization server in your setup):

ip route add 192.168.200.0/24 via 192.168.56.60 dev enp0s3
Note: add invocation to /etc/rc.local for persistence.


On Mac OS X add route
sudo sysctl -w net.inet.ip.forwarding=1
route -n add -net 192.168.200.0/24 172.24.1.225

ansible-playbook main.yml --ask-become-pass 


echo "rdr pass on lo0 inet proto tcp from any to self port 5432 -> 192.168.1.103 port 543

rdr pass on en0 inet proto tcp from any to any port 5432 -> 192.168.1.103 port 543

rdr pass on en1 inet proto tcp from any to any port 5432 -> 192.168.1.103 port 543

" | sudo pfctl -ef -

=============================================================================
Resolving the local DNS issue in Ubuntu 22.04
=============================================================================

working within the systemd paradigm add a DNS to a link / device

systemd.network manual page
using ubuntu 17.10+ add a *.network file:

sudo nano /lib/systemd/network/100-somecustom.network:

100-somecustom.network ( 100 can be any number for priority, and it requires the .network file extension ):

[Match]
Name=wlo1 # the device name here

[Network] # add multiple DNS 
DNS=8.8.8.8
DNS=208.67.222.222
Then restart:

sudo service systemd-networkd restart
Also look into:

netplan apply
Then check:

resolvectl status eno1

Example of Netplan config:
root@yalim:/home/romyt# cat /etc/netplan/00-installer-config.yaml
# This is the network config written by 'subiquity'
network:
  renderer: NetworkManager
  ethernets:
    enp5s0:
      dhcp4: false
      addresses:
      - 172.24.1.225/24
    eno1:
      dhcp4: false
      addresses:
      - 172.24.1.223/24
      routes:
      - to: default
        via: 172.24.1.1
      nameservers:
        search: [yalim.cloud]
        addresses: [192.168.200.1,8.8.8.8,8.8.4.4]
  version: 2
!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!
Final resolution:
/etc/resolv.conf was pointing to a file not managed by Netplan
 /etc/resolv.conf -> ../run/systemd/resolve/stub-resolv.conf

 The solution:

 sudo rm -f /etc/resolv.conf
 sudo ln -sv /run/systemd/resolve/resolv.conf /etc/resolv.conf

