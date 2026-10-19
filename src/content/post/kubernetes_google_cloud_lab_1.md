---
title: "Building a Kubernetes Lab on Google Cloud: Part 1 — Preparing Nodes"
description: "Preparing one control-plane VM and two worker VMs for a kubeadm-based Kubernetes lab."
publishDate: 2025-04-07
updatedDate: 2026-10-19
tags: [kubernetes, google-cloud, linux, lab]
draft: false
---

> English translation and technical edit of the [original Korean lab note](https://tyge-ottesen-brahe.tistory.com/15). This is a record of a 2025 learning exercise, not a validated production installation guide. Command examples are illustrative and may require updates for the current Kubernetes, Ubuntu, and Google Cloud releases.

## Goal and environment

I decided to explore Kubernetes by building a small cluster, rather than studying only architectural diagrams. The intended topology was one control-plane node (`k8s-master`) and two worker nodes (`k8s-worker1` and `k8s-worker2`) on Google Cloud Compute Engine. Google Cloud Shell made it possible to issue the commands without preparing a separate local administration environment.

The original exercise used Ubuntu 20.04, `e2-medium` instances, a 30 GB boot disk per instance, and the `asia-northeast3-a` zone. Those are historical lab choices, not recommendations for a new deployment. Three running VMs and their disks can incur charges.

## 1. Create virtual machines

The following example captures the original approach. Confirm the project, zone, machine availability, image support, budget, and organization policies before running it.

```bash
gcloud config set project YOUR_PROJECT_ID

gcloud compute instances create k8s-master k8s-worker1 k8s-worker2 \
  --zone=asia-northeast3-a \
  --machine-type=e2-medium \
  --image-family=ubuntu-2004-lts \
  --image-project=ubuntu-os-cloud \
  --boot-disk-size=30GB \
  --tags=k8s
```

Each node should have a stable means of reaching the other nodes over the ports required by Kubernetes. A network tag by itself does **not** create an appropriate firewall policy, and exposing the Kubernetes API server or kubelet broadly to the Internet would be unsafe.

I connected to each machine through SSH, for example:

```bash
gcloud compute ssh k8s-master --zone=asia-northeast3-a
```

## 2. Name resolution between nodes

The initial lab used the instances' internal IP addresses in `/etc/hosts`, so that each VM could resolve the other nodes by name. A conceptual example looks like this:

```text
10.0.0.10 k8s-master
10.0.0.11 k8s-worker1
10.0.0.12 k8s-worker2
```

The addresses above are **placeholders**. The actual private addresses should be obtained from Compute Engine. Cloud DNS or another managed name-resolution approach is preferable when maintaining more than a short-lived lab.

## 3. Prepare the operating system

For the kubeadm setup in this series, I disabled swap on each node and checked that it would not be re-enabled on reboot. This followed the default kubelet configuration used in the exercise; swap support and configuration options have evolved across Kubernetes releases.

```bash
sudo swapoff -a
free -h
```

If swap is configured in `/etc/fstab`, edit the relevant entry deliberately after taking a backup. Avoid blindly deleting all swap-related entries on machines you do not control.

I then installed the prerequisite packages and prepared the Kubernetes package repository. The following commands show the version-scoped repository style used for the `v1.29` release track in the original lab; **check current supported versions and repository instructions before reuse**.

```bash
sudo apt-get update
sudo apt-get install -y apt-transport-https ca-certificates curl gpg
sudo install -m 0755 -d /etc/apt/keyrings

curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.29/deb/Release.key \
  | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg

echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.29/deb/ /' \
  | sudo tee /etc/apt/sources.list.d/kubernetes.list

sudo apt-get update
sudo apt-get install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl
```

`kubelet` is the node agent, `kubeadm` bootstraps the cluster, and `kubectl` is the command-line client for the Kubernetes API. Installing these packages alone does not make a functioning cluster: the container runtime, kernel networking settings, control plane, and pod network still need configuration.

## What I learned

I had assumed that installing Kubernetes would be a single package-management step. Instead, the exercise exposed several prerequisites: network reachability, operating-system settings, container runtime configuration, and version compatibility. Those dependencies are easier to see when constructing the cluster manually.

The next note continues with the container runtime, `kubeadm init`, cluster networking, and worker-node registration.

## Technical review notes

- **Historical compatibility:** Kubernetes v1.29 and Ubuntu 20.04 reflect the 2025 source, not a current security baseline. Review supported releases and security updates before using these commands.
- **Network security:** Keep control-plane and node-management interfaces on private, tightly scoped networks. Firewall rules must be designed explicitly.
- **Repeatability:** Pin supported package versions and document the container runtime before initializing the cluster.
- **Claim boundary:** These notes describe the intended setup and learning process. They do not assert successful validation in a fresh 2026 environment.

References: [Install kubeadm](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/), [Google Cloud Compute Engine documentation](https://cloud.google.com/compute/docs).