---
title: "Building a Kubernetes Lab on Google Cloud: Part 2 — Initializing the Cluster"
description: "Configuring containerd, bootstrapping the control plane, and joining worker nodes with kubeadm."
publishDate: 2025-04-07
updatedDate: 2026-10-19
tags: [kubernetes, kubeadm, containerd, networking, lab]
draft: true
---

> English translation and technical edit of the [original Korean lab note](https://tyge-ottesen-brahe.tistory.com/16), continuing [Part 1](11_kubernetes_google_cloud_lab_1.md). Commands are explanatory, historically grounded examples. They have **not** been rerun against a current cloud environment.

## From prepared servers to a cluster

After preparing the VMs, I needed to turn them into Kubernetes nodes. The main tasks were configuring kernel networking, installing a container runtime, initializing a control plane, installing a Container Network Interface (CNI) plugin, and joining workers.

I learned that Kubernetes is not itself the program that directly launches every container. The kubelet communicates through the Container Runtime Interface to a runtime such as containerd. A working kubelet therefore depends on a compatible runtime and correct operating-system configuration.

## 1. Enable required packet forwarding

On every node, the original exercise enabled the bridge netfilter module and the relevant kernel parameters:

```bash
sudo modprobe br_netfilter

sudo tee /etc/modules-load.d/k8s.conf >/dev/null <<'EOF'
br_netfilter
EOF

sudo tee /etc/sysctl.d/k8s.conf >/dev/null <<'EOF'
net.bridge.bridge-nf-call-iptables = 1
net.ipv4.ip_forward = 1
EOF

sudo sysctl --system
```

These settings support the routing and filtering behavior required by common Linux Kubernetes networking setups. Exact requirements depend on the CNI and software versions.

## 2. Configure containerd

Each node needs a functioning container runtime before kubeadm initialization or joining. For an Ubuntu lab, containerd can be installed from a suitable trusted repository and configured with a CRI-enabled configuration. An illustrative workflow is:

```bash
sudo apt-get install -y containerd
sudo mkdir -p /etc/containerd
containerd config default | sudo tee /etc/containerd/config.toml >/dev/null
```

For kubeadm clusters using the systemd cgroup driver, set `SystemdCgroup = true` in the appropriate runc configuration section of `/etc/containerd/config.toml`; then restart containerd:

```bash
sudo systemctl restart containerd
sudo systemctl enable containerd
sudo systemctl status containerd
```

**Important:** The configuration key's nesting and package defaults can change by containerd version. Verify that the CRI plugin is enabled and that the kubelet and runtime use matching cgroup drivers. A command that merely creates a default configuration does not establish those conditions.

## 3. Initialize the control plane

On `k8s-master`, the lab initialized the control plane using a pod network CIDR associated with the planned CNI:

```bash
sudo kubeadm init --pod-network-cidr=10.244.0.0/16
```

The subnet must match the chosen networking plugin's configuration and must not conflict with the VM network. For a multi-homed VM, it may also be necessary to select the advertised API-server address deliberately.

After successful initialization, an administrator can configure `kubectl` with the generated kubeconfig. On a personal lab control-plane machine:

```bash
mkdir -p "$HOME/.kube"
sudo cp -i /etc/kubernetes/admin.conf "$HOME/.kube/config"
sudo chown "$(id -u):$(id -g)" "$HOME/.kube/config"
kubectl get nodes
```

The `admin.conf` file is a sensitive cluster-administrator credential. Do not commit it to a repository, email it, or share it with untrusted users.

## 4. Install a compatible pod network

At this stage, nodes often appear as `NotReady` until a compatible CNI plugin is installed. The original exploration considered Flannel and Calico. These are alternative networking choices with different features and setup requirements; a simple lab should select **one** matching CNI configuration rather than install both indiscriminately.

I would verify the supported CNI manifest for the chosen Kubernetes version, inspect what it changes, and apply that version-specific manifest. It is easy to make a misleading tutorial by pasting an unpinned `latest` YAML URL, so this draft intentionally does not prescribe one.

## 5. Join the worker nodes

Successful `kubeadm init` prints a worker join command. On each worker, it has a structure like:

```bash
sudo kubeadm join CONTROL_PLANE_PRIVATE_IP:6443 \
  --token TOKEN_FROM_KUBEADM \
  --discovery-token-ca-cert-hash sha256:CA_CERTIFICATE_HASH
```

These values are placeholders, not credentials. If the original token has expired, a cluster administrator can generate an appropriate new join command. The token and discovery hash should be handled as operational secrets, not published in a blog post.

On the control-plane node, I could then inspect the registered machines and their status:

```bash
kubectl get nodes -o wide
kubectl get pods -A
```

`Ready` indicates that Kubernetes considers a node able to accept workloads according to its health checks; it does not by itself prove that every desired network path or application works.

## What the exercise clarified

The cluster is a collaboration among layers: Linux networking, the container runtime, kubelet, API server, scheduler, controllers, and CNI. I also understood the distinction between **bootstrapping** the control plane with kubeadm and **operating** the resulting cluster with kubectl.

## Technical review notes

- The original sequence blurred a few prerequisites. In this edit, containerd is configured **before** cluster initialization and worker joining.
- The pod CIDR and CNI choice must agree; cluster connectivity requires appropriate private firewall rules.
- `admin.conf` and join tokens should not be checked into public source control.
- This is a learning lab, not a highly available production cluster: a single control-plane VM remains a single point of failure.

References: [Creating a cluster with kubeadm](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/create-cluster-kubeadm/), [Container runtimes](https://kubernetes.io/docs/setup/production-environment/container-runtimes/).