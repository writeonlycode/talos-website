---
title: "VMware"
description: "Creating Talos Kubernetes cluster using VMware."
---

## Creating a Cluster via the `govc` CLI

In this guide we will create an HA Kubernetes cluster with 3 worker nodes.
We will use the `govc` cli which can be downloaded [here](https://github.com/vmware/govmomi/tree/master/govc#installation).

### Prerequisites

Prior to starting, it is important to have the following infrastructure in place and available:

- DHCP server
- Load Balancer or DNS address for cluster endpoint
  - If using a load balancer, the most common setup is to balance `tcp/443` across the control plane nodes `tcp/6443`
  - If using a DNS address, the A record should return back the addresses of the control plane nodes

### Create the Machine Configuration Files

#### Generating Base Configurations

Using the DNS name or name of the loadbalancer used in the prereq steps, generate the base configuration files for the Talos machines:

```bash
$ talosctl gen config talos-k8s-vmware-tutorial https://<load balancer IP or DNS>:<port>
created controlplane.yaml
created worker.yaml
created talosconfig
```

```bash
$ talosctl gen config talos-k8s-vmware-tutorial https://<DNS name>:6443
created controlplane.yaml
created worker.yaml
created talosconfig
```

At this point, you can modify the generated configs to your liking.
Optionally, you can specify `--config-patch` with RFC6902 jsonpatch which will be applied during the config generation.

#### Validate the Configuration Files

```bash
$ talosctl validate --config controlplane.yaml --mode cloud
controlplane.yaml is valid for cloud mode
$ talosctl validate --config worker.yaml --mode cloud
worker.yaml is valid for cloud mode
```

### Set Environment Variables

`govc` makes use of the following environment variables

```bash
export GOVC_URL=<vCenter url>
export GOVC_USERNAME=<vCenter username>
export GOVC_PASSWORD=<vCenter password>
```

> Note: If your vCenter installation makes use of self signed certificates, you'll want to export `GOVC_INSECURE=true`.

There are some additional variables that you may need to set:

```bash
export GOVC_DATACENTER=<vCenter datacenter>
export GOVC_RESOURCE_POOL=<vCenter resource pool>
export GOVC_DATASTORE=<vCenter datastore>
export GOVC_NETWORK=<vCenter network>
```

### Download the OVA

A `talos.ova` asset is published with each [release](https://github.com/talos-systems/talos/releases).
We will refer to the version of the release as `$TALOS_VERSION` below.
It can be easily exported with `export TALOS_VERSION="v0.3.0-alpha.10"` or similar.

```bash
curl -LO https://github.com/talos-systems/talos/releases/download/$TALOS_VERSION/talos.ova
```

### Import the OVA into vCenter

We'll need to repeat this step for each Talos node we want to create.
In a typical HA setup, we'll have 3 control plane nodes and N workers.
In the following example, we'll setup a HA control plane with two worker nodes.

```bash
govc import.ova -name talos-$TALOS_VERSION /path/to/downloaded/talos.ova
```

#### Create the Control Plane Nodes

Talos makes use of the `guestinfo` facility of VMware to provide the machine/cluster configuration.
This can be set using the `govc vm.change` command.
To facilitate persistent storage using the vSphere cloud provider integration with Kubernetes, `disk.enableUUID=1` is used.

```bash
govc vm.clone -on=false -vm talos-$TALOS_VERSION control-plane-1
govc vm.change \
  -e "guestinfo.talos.config=$(base64 controlplane.yaml)" \
  -e "disk.enableUUID=1" \
  -vm /ha-datacenter/vm/control-plane-1
govc vm.clone -on=false -vm talos-$TALOS_VERSION control-plane-2
govc vm.change \
  -e "guestinfo.talos.config=$(base64 controlplane.yaml)" \
  -e "disk.enableUUID=1" \
  -vm /ha-datacenter/vm/control-plane-2
govc vm.clone -on=false -vm talos-$TALOS_VERSION control-plane-3
govc vm.change \
  -e "guestinfo.talos.config=$(base64 controlplane.yaml)" \
  -e "disk.enableUUID=1" \
  -vm /ha-datacenter/vm/control-plane-3
```

```bash
govc vm.change \
  -c 2 \
  -m 4096 \
  -vm /ha-datacenter/vm/control-plane-1
govc vm.change \
  -c 2 \
  -m 4096 \
  -vm /ha-datacenter/vm/control-plane-2
govc vm.change \
  -c 2 \
  -m 4096 \
  -vm /ha-datacenter/vm/control-plane-3
```

```bash
govc vm.disk.change -vm control-plane-1 -disk.name disk-1000-0 -size 10G
govc vm.disk.change -vm control-plane-2 -disk.name disk-1000-0 -size 10G
govc vm.disk.change -vm control-plane-3 -disk.name disk-1000-0 -size 10G
```

```bash
govc vm.power -on control-plane-1
govc vm.power -on control-plane-2
govc vm.power -on control-plane-3
```

#### Update Settings for the Worker Nodes

```bash
govc vm.clone -on=false -vm talos-$TALOS_VERSION worker-1
govc vm.change \
  -e "guestinfo.talos.config=$(base64 worker.yaml)" \
  -e "disk.enableUUID=1" \
  -vm /ha-datacenter/vm/worker-1
govc vm.clone -on=false -vm talos-$TALOS_VERSION worker-2
govc vm.change \
  -e "guestinfo.talos.config=$(base64 worker.yaml)" \
  -e "disk.enableUUID=1" \
  -vm /ha-datacenter/vm/worker-2
```

```bash
govc vm.change \
  -c 4 \
  -m 8192 \
  -vm /ha-datacenter/vm/worker-1
govc vm.change \
  -c 4 \
  -m 8192 \
  -vm /ha-datacenter/vm/worker-2
```

```bash
govc vm.disk.change -vm worker-1 -disk.name disk-1000-0 -size 50G
govc vm.disk.change -vm worker-2 -disk.name disk-1000-0 -size 50G
```

```bash
govc vm.power -on worker-1
govc vm.power -on worker-2
```

### Bootstrap Etcd

Set the `endpoints` and `nodes`:

```bash
talosctl --talosconfig talosconfig config endpoint <control plane 1 IP>,<control plane 2 IP>,<control plane 3 IP>
talosctl --talosconfig talosconfig config node <control plane 1 IP>
```

Bootstrap `etcd`:

```bash
talosctl --talosconfig talosconfig bootstrap
```

### Retrieve the `kubeconfig`

At this point we can retrieve the admin `kubeconfig` by running:

```bash
talosctl --talosconfig talosconfig config endpoint <control plane 1 IP>
talosctl --talosconfig talosconfig config node <control plane 1 IP>
talosctl --talosconfig talosconfig kubeconfig .
```
