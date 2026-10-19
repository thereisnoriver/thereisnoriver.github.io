---
title: "Kubernetes Rolling Update Experiment Plan"
description: "Five failure-oriented exercises and a comparison of local and production container image workflows."
publishDate: 2025-08-03
updatedDate: 2026-10-19
tags: [kubernetes, testing, deployment]
draft: false
---

> Originally published in Korean: [Rolling Update Test](https://tyge-ottesen-brahe.tistory.com/42). This document records **an experiment plan and learning notes**, not verified results from all five experiments. The notes were translated, reorganized, and technically reviewed in October 2026.

## Five experiments to understand a rolling update

I wanted to test a rolling deployment step by step, progressing from a simple version change to several failure scenarios. The plan uses a small Spring application that exposes `/version`.

### 1. Basic version update

- Build and deploy a container image tagged `spring-version:v1`.
- Update the image to `spring-version:v2` using `kubectl set image` (the container name must match the Deployment specification).
- Repeatedly query `/version` and observe whether responses change from `v1` to `v2`.

### 2. Observe gradual Pod replacement

- Configure at least four replicas.
- Set `maxSurge` and `maxUnavailable` explicitly.
- Watch Pod states with `kubectl get pods -w` while repeatedly sending HTTP requests.
- Compare the number of old and new Pods during the transition.

### 3. Introduce an application error

- Deliberately make `v2` return a server error in some requests (the original plan proposed a `RuntimeException`).
- Deploy it without an application-specific `readinessProbe` and check whether requests can produce HTTP 500.
- Examine rollout behavior and manually restore the previous revision using `kubectl rollout undo deployment/<name>`.

**Technical distinction:** An HTTP 500 does not necessarily make a Pod *NotReady*. Without an appropriate probe, Kubernetes may keep directing traffic to an application that is running but responding incorrectly.

### 4. Make the candidate process terminate

- Intentionally terminate the `v2` application process after startup, as proposed in the original plan using `System.exit(1)`.
- Inspect the resulting restart behavior and possible `CrashLoopBackOff` state.
- Check `kubectl rollout status deployment/<name>` and restore a working revision if necessary.

Kubernetes can report a stalled rollout, but a Deployment does **not automatically roll back** solely because its progress deadline is exceeded. Higher-level deployment tools may implement such recovery. See the [Kubernetes Deployment documentation](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/).

### 5. Make only some instances fail

- Introduce intermittent failures in `v2` and use at least five replicas.
- Compare Pod restart states and readiness status.
- Observe whether traffic reaches unhealthy instances with and without an application-specific readiness probe.

**Review note:** A random application-level error is different from a container crash; the experiment should deliberately choose which behavior to induce. Do not interpret `0/1 Ready` and `CrashLoopBackOff` as interchangeable outcomes.

At the time of the original note, I described myself as being around the beginning of steps 2–3, with four or more replicas and no readiness probe. That is a progress note, not evidence that the later failure experiments were completed.

## How the container gets deployed

The underlying build sequence is:

1. Write Java or Kotlin application code.
2. Use `./gradlew build` to produce an executable JAR.
3. Write a Dockerfile and build an image from the application artifact.
4. Make that image accessible to the Kubernetes nodes.
5. Apply the Kubernetes manifests to create or update the workload.

### A local Minikube setup

In an environment configured to use Minikube's Docker daemon, the original workflow was:

```bash
./gradlew build
eval $(minikube docker-env)
docker build -t spring-version:v1 .
kubectl apply -f deployment.yaml
```

The benefit is that an external registry push can sometimes be skipped when the cluster's container runtime can access the locally built image. This depends on the Minikube driver and runtime configuration; alternatives include `minikube image load`. An appropriate `imagePullPolicy` is also necessary to avoid unexpectedly fetching an image from a remote registry.

### A typical remote cluster workflow

For managed Kubernetes, such as GKE, EKS, or AKS, the original notes compare this with a build–push–deploy sequence:

```bash
./gradlew build
docker build -t registry.example.com/team/service:v1 .
docker push registry.example.com/team/service:v1
kubectl apply -f deployment.yaml
```

The registry URL above is illustrative. The actual steps should be automated and authenticated appropriately in a production pipeline.

| Aspect | Local Minikube example | Remote cluster example |
|---|---|---|
| Image availability | Local load or shared daemon | Cluster-accessible registry |
| Registry push | Potentially unnecessary | Usually part of deployment pipeline |
| Primary goal | Fast learning and iteration | Repeatable and controlled release |

The main idea I wanted to remember was the separation between **building an application artifact, packaging it as an image, and instructing Kubernetes to run it**.

## Questions for a future experiment report

Which versions actually answered each request? Did any requests fail? How did readiness affect routing? How long did rollback take? To turn this plan into experimental evidence, I would record commands, cluster versions, probes, response samples, and a timestamped rollout timeline.