---
title: "Why Kubernetes? From Virtual Machines to Container Orchestration"
description: "A beginner's attempt to understand why virtualization, containers, Docker, and Kubernetes exist and how they relate."
publishDate: 2025-05-11
updatedDate: 2026-10-19
tags: [kubernetes, docker, virtualization]
draft: false
---

> Translated and technically edited from the [original Korean article](https://tyge-ottesen-brahe.tistory.com/35). The source is an extended beginner's explanation. This version preserves its conceptual progression while correcting several generalizations and explicitly separating common patterns from guarantees.

## Before learning Kubernetes

Many introductions to Kubernetes begin by comparing virtual machines, Docker, and Kubernetes. For someone who does not yet know what a container image or hypervisor is, that comparison can create an immediate barrier.

I wanted to start with the more basic question: **Why did these technologies appear in the first place?** I tend to get stuck when I cannot explain the reason behind a concept, so this is a deliberately beginner-oriented discussion.

## Virtual machines

### Why virtualize a physical server?

Dedicated physical servers can waste resources when applications use only a fraction of the available CPU and memory. Running multiple workloads on a single operating system helps with utilization, but different applications may compete for resources and require different configurations or operating systems.

Virtualization provides a way to run multiple isolated guest environments on one physical machine. A **hypervisor** presents virtual CPU, memory, storage, and other devices to each guest operating system, while coordinating access to the underlying hardware.

A virtual machine is thus a software-defined computer that can boot its own operating system. Multiple VMs on one host can improve utilization and provide stronger isolation boundaries than ordinary processes, although isolation is never absolute.

The technology also makes it easier to capture and reproduce environments using images and snapshots. As a student, I remember operating-system experiments feeling painfully slow on an underpowered laptop; that experience made the resource overhead of full VMs tangible.

### Virtualization versus emulation

Virtualization usually runs guest instructions on compatible underlying hardware with hardware assistance. Emulation can reproduce a different instruction-set architecture by translating or interpreting instructions, often with additional overhead. They can be combined in some systems, but they are not the same mechanism.

**Security note:** A hypervisor vulnerability can undermine isolation across guests. That does not mean sensitive organizations generally avoid virtualization; financial institutions and other security-conscious organizations do use virtualized infrastructure with appropriate controls. The claim in the original note was too broad.

## Containers

A VM typically runs a separate guest kernel and user-space environment. A container instead isolates processes while sharing the host operating system kernel. Linux mechanisms such as **namespaces** and **cgroups** provide different aspects of isolation and resource control.

Containers can start quickly and often have less overhead than full VMs, making them convenient for packaging applications. Sharing the host kernel also creates a security consideration: a kernel-level vulnerability may affect the isolation boundary.

### Container image layers

A container image is built from layers, often corresponding to build steps. Unchanged layers can be cached and shared, so a build does not always need to redo every earlier step. Registries can also avoid retransferring layers that a destination already possesses.

For example, a base runtime layer may be shared among many application images. A change to application code might rebuild only the relevant later layers if the Dockerfile and build cache are structured appropriately. Layering helps storage and distribution efficiency, but it does not by itself provide a complete rollback policy for a running service.

### VM versus container

| | Virtual machine | Container |
|---|---|---|
| Isolation boundary | Virtualized hardware and guest OS | Processes sharing a kernel |
| Startup | Often includes guest OS boot | Usually starts as a process |
| Operating systems | Can run distinct compatible guest OSs | Shares host kernel family |
| Overhead | Usually higher per environment | Often lower |
| Security trade-off | Hypervisor and guest attack surfaces | Shared host kernel and runtime attack surfaces |

## Docker

Containers existed before Docker, but managing them directly could be cumbersome. Docker made common image-build, distribution, and execution workflows accessible through a consistent set of tools.

### Client–server architecture

The Docker CLI sends requests to the Docker Engine API. The Docker daemon performs image and container management, while components such as containerd and lower-level runtimes execute containers. A CLI and daemon can run on the same machine or communicate remotely through a secured connection.

A simplified `docker run` operation involves resolving an image, downloading missing image layers if necessary, creating a container from that image and configuration, and starting its process.

### Images and containers are different

An **image** is an immutable, reusable package of application files and runtime metadata. A **container** is an instance created from an image, with its own runtime state and writable layer where configured. Changes inside a running container do not automatically change the underlying image. Reproducible deployment normally involves changing source/configuration, building a new image, and releasing that image rather than treating live container edits as the deployment artifact.

Docker helped make it easier to build and run containers consistently across compatible environments. However, a single host's container tools do not by themselves solve every problem of managing many instances across several machines.

## Why an orchestrator?

As a system grows, its operators must handle service discovery, networking, placement, scaling, health checks, recovery, and upgrades. Manually tracking every container and host becomes difficult. **Container orchestration** coordinates those responsibilities.

Kubernetes is a widely used orchestrator. Among its major capabilities are workload scheduling, declarative configuration, service discovery, rolling updates, resource management, and automated recovery of certain failure states. Autoscaling and load balancing depend on appropriately configured components and metrics; they are not magic properties of every cluster.

### Declarative desired state

The idea I find central is the declarative API. Rather than telling the system every individual action, we describe a desired state—for example, that a Deployment should maintain ten replicas.

Controllers continually compare observed state with desired state and attempt to reconcile the difference. If two managed Pods disappear, the relevant controllers can create replacements. A desired replica count is not a guarantee that capacity is immediately available; scheduling constraints and failures can prevent the target from being reached.

### How the components cooperate

1. A user or automation sends API requests through the **API server**, which authenticates and authorizes them.
2. Persistent cluster objects are stored through the control-plane storage system, typically **etcd**.
3. **Controllers** watch state and create or update objects toward the desired configuration.
4. The **scheduler** selects suitable nodes for unscheduled Pods.
5. Each node's **kubelet** coordinates with a container runtime, commonly through the Container Runtime Interface, to start and monitor containers.

Kubernetes removed its integrated *dockershim* integration; it did **not** stop supporting Docker-built OCI-compatible container images. A runtime such as containerd can run those images without the Docker daemon being Kubernetes' node runtime.

## Conclusion

Kubernetes is not just a patch for something Docker forgot to include. It is a system for coordinating containerized workloads across infrastructure through APIs, scheduling, and control loops.

I still had many questions after learning these concepts. The next step was to experiment with actual clusters and observe how the objects and controllers behaved.