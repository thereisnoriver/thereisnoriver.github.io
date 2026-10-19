---
title: "Building a Kubernetes Lab on Google Cloud: Part 3 — Pods, Deployments, and Operations"
description: "Applying Kubernetes workload manifests and learning what controllers and health probes actually guarantee."
publishDate: 2025-04-07
updatedDate: 2026-10-19
tags: [kubernetes, pods, deployments, operations, lab]
draft: true
---

> English translation and technical edit of the [original Korean lab note](https://tyge-ottesen-brahe.tistory.com/18), following [Part 1](11_kubernetes_google_cloud_lab_1.md) and [Part 2](12_kubernetes_google_cloud_lab_2.md). These examples communicate the learning steps and have not been independently executed for this revision.

## Working with Pods

After building the lab cluster, I began experimenting with the workload objects that Kubernetes manages. The most basic deployable unit is the **Pod**: one or more tightly coupled containers sharing a network namespace and certain storage resources.

To make the idea tangible, I could create a simple nginx Pod and inspect it:

```bash
kubectl run mypod --image=nginx
kubectl get pods -o wide
kubectl describe pod mypod
```

An equivalent declarative starting point is:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: mypod
spec:
  containers:
    - name: nginx
      image: nginx:stable
      ports:
        - containerPort: 80
```

Save it as `pod.yaml` and apply it with `kubectl apply -f pod.yaml`. The `containerPort` declaration documents the port used by the container; it does not automatically expose the Pod to the Internet.

## Why use a Deployment?

A standalone Pod is useful for a quick experiment but does not provide the desired replication and rollout management for most stateless applications. A Deployment manages ReplicaSets, which in turn maintain the specified set of Pods.

```bash
kubectl create deployment dpy-mypod --image=nginx:stable
kubectl scale deployment dpy-mypod --replicas=3
kubectl get deployments,replicasets,pods
```

If a managed Pod disappears, the controller works toward the declared replica count. This is **reconciliation**, not a guarantee of uninterrupted availability: scheduling capacity, image-pull failures, application readiness, network failures, and other conditions can prevent replacements from serving traffic.

The core objects I encountered included:

- **Pod:** one or more co-scheduled containers.
- **ReplicaSet:** maintains a desired number of matching Pods.
- **Deployment:** declares desired stateless application rollout and ReplicaSet state.
- **Service:** gives clients a relatively stable endpoint for a group of workloads; exact reachability depends on Service type and networking.
- **ConfigMap and Secret:** separate configuration and sensitive values from workload definitions. Secrets require appropriate encryption-at-rest, access controls, and operational handling; base64 encoding is not encryption.
- **Namespace:** an organizational and policy-scoping boundary, not a hard security isolation boundary by itself.

## Requests and limits

Kubernetes can make scheduling decisions based on resource requests and enforce certain resource limits. For example:

```yaml
resources:
  requests:
    cpu: "100m"
    memory: "128Mi"
  limits:
    cpu: "200m"
    memory: "256Mi"
```

CPU requests influence scheduling, while CPU limits typically cause throttling when exceeded. A container that exceeds its memory limit may be terminated due to an out-of-memory condition. These effects differ, so the two resources should not be described as if they fail identically.

## Updating a workload

With a Deployment, I could update the image and observe the rollout:

```bash
kubectl set image deployment/dpy-mypod nginx=nginx:stable
kubectl rollout status deployment/dpy-mypod
kubectl rollout history deployment/dpy-mypod
```

To restart Pods under the Deployment's rollout strategy:

```bash
kubectl rollout restart deployment/dpy-mypod
```

If I intentionally decide to revert to a prior rollout revision, I can issue:

```bash
kubectl rollout undo deployment/dpy-mypod
```

**Technical correction:** A Deployment does not automatically execute `rollout undo` just because a rollout stalls. Kubernetes records rollout progress and may report `ProgressDeadlineExceeded`; detection, alerting, and rollback automation need to be designed separately.

## Health checks

I also learned why a running process is not necessarily a healthy or traffic-ready application:

- A **readiness probe** controls whether a Pod is considered ready for Service traffic.
- A **liveness probe** helps Kubernetes decide when to restart a stuck container.
- A **startup probe** can allow slower applications to initialize before liveness checks apply.

These checks should exercise meaningful application conditions without causing failure loops under temporary load. Readiness and liveness are distinct; using identical aggressive probes for both can make an incident worse.

## Scheduling and maintenance

The original lab also touched on operational commands such as `kubectl cordon` and `kubectl drain`. `cordon` prevents new ordinary workload placement on a node. `drain` attempts to evict eligible workloads so the node can be maintained. A drain operation can be blocked by PodDisruptionBudgets, unmanaged Pods, DaemonSets, or insufficient capacity; it should not be equated with risk-free live migration.

## What I took away

The key shift in thinking was from issuing one-off commands against a server to declaring desired state and watching controllers attempt to converge on it. Kubernetes offers powerful automation, but the application's availability still depends on correct manifests, adequate resources, network configuration, and operational safeguards.

## Technical review notes

- Avoid interpreting desired replicas as guaranteed live replicas.
- Explicit rollback is different from rollout status monitoring.
- A Service is not automatically a public load balancer; its type and environment matter.
- Treat resource limits and health probes as application-specific engineering choices.
- The commands here illustrate concepts; no new experiment results are claimed.

References: [Deployments](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/), [Pod lifecycle and probes](https://kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle/), [Resource management](https://kubernetes.io/docs/concepts/configuration/manage-resources-containers/).