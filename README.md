# Kubernetes Playground

This project contains a `Vagrantfile` to provisioning a 3 nodes Kubernetes cluster using `VirtualBox` and `Ubuntu 16.04`.

## Prerequisites

You need the following installed to use this playground.

- `Vagrant`, version 2.2 or better.
- `VirtualBox`, tested with Version 6.0
- Internet access, this playground pulls Vagrant boxes from the Internet as well
as installs Ubuntu application packages from the Internet.
- Tested on **Windows 10** Build 17763, **macOS Mojave** 10.14.6

## Bringing Up The cluster

To bring up the cluster, clone this repository to a working directory.

```bash
git clone https://github.com/filippobuletto/k8s-playground.git
```

Change into the working directory and `vagrant up`

```bash
cd k8s-playground
vagrant up
```

Vagrant will start three machines. Each machine will have a NAT-ed network
interface, through which it can access the Internet, and a `private-network`
interface in the subnet 192.168.205.0/24. The private network is used for
intra-cluster communication.

The machines created are:

| NAME | IP ADDRESS | ROLE |
| --- | --- | --- |
| k8s-head | `192.168.205.10` | Cluster Master |
| k8s-node-1 | `192.168.205.11` | Cluster Worker |
| k8s-node-2 | `192.168.205.12` | Cluster Worker |

As the cluster brought up the cluster master (**k8s-head**) will perform a `kubeadm init` and the cluster workers will perform a `kubeadmin join`.

### Import `kubeconfig` file

In order for `kubectl` to find and access a Kubernetes cluster, it needs a `kubeconfig` file, which is created by vagrant during master provisioning: `admin.conf`.

Move and rename the file to the default location: `%USERPROFILE%\.kube\config`.

Check that `kubectl` is properly configured by getting the cluster state: `kubectl cluster-info`.

## Useful Vagrant commands

```bash
#Create the cluster or start the cluster after a host reboot
vagrant up

#Execute provision in all the vagrant boxes
vagrant provision

#Execute provision in the Kubernetes node 1
vagrant provision k8s-node-1

#Open an ssh connection to the Kubernetes master
vagrant ssh k8s-head

#Open an ssh connection to the Kubernetes node 1
vagrant ssh k8s-node-1

#Open an ssh connection to the Kubernetes node 2
vagrant ssh k8s-node-2

#Stop all Vagrant machines (use vagrant up to start)
vagrant halt

#Destroy the cluster
vagrant destroy -f
```

## Other Resources

- [Starting A Sample Service / Deployment](https://github.com/filippobuletto/k8s-playground/blob/master/ServiceDeployment.md)
- [Enable Ingress Controller](https://github.com/filippobuletto/k8s-playground/blob/master/IngressController.md)
- [Forward all `.kube` host for local development](https://github.com/filippobuletto/k8s-playground/blob/master/LocalDevelopment.md)
- [Use internal registry](https://github.com/filippobuletto/k8s-playground/blob/master/InternalRegistry.md)
- [Windows Extra](https://github.com/filippobuletto/k8s-playground/blob/master/WindowsExtra.md)
