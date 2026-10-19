---
title: "Rolling, Blue–Green, and Canary Deployments: Understanding the Trade-offs"
description: "A practical introduction to three deployment strategies, motivated by the question of how to update production applications without interrupting users."
publishDate: 2025-05-27
updatedDate: 2026-10-19
tags:
  - kubernetes
  - deployment
  - reliability
draft: true
---

> **Translation note:** Originally published in Korean on [Tistory](https://tyge-ottesen-brahe.tistory.com/40) on May 27, 2025. Translated into English and technically reviewed in October 2026. This is a conceptual study note, not a report of a production deployment.

## Why I started learning about zero-downtime deployments

If someone asked me why I started studying Kubernetes, my answer would be simple: I wanted to understand **how zero-downtime deployments work**.

I have spent late nights working around deployment schedules, so the possibility of releasing changes without interrupting a service naturally caught my attention. Could such a deployment process also reduce the burden on engineers?

But I was curious about more than the convenience. *How does an application switch to a new version while people are using it? Can users really experience the transition without noticing anything?*

I wanted to understand the mechanism behind these claims rather than simply memorize deployment commands.

## Before we begin

Suppose we modify application code or prepare a patch for a security vulnerability. In a containerized application, we would typically build a new image and deploy Pods using that image. The service then needs to transition from the existing version to the updated one.

This creates an important question: **How can we replace a running application without making the service unavailable?**

Three common deployment patterns provide different answers:

1. **Rolling update:** Replace the application's running instances gradually.
2. **Blue–green deployment:** Prepare an entire new environment alongside the existing one, then switch traffic to it.
3. **Canary deployment:** Route a limited share of users or requests to the new version before expanding the rollout.

All three aim to manage the risk of software changes, but they differ in how they handle traffic, infrastructure resources, and rollback.

## 1. Rolling updates

A rolling update replaces application instances incrementally rather than stopping the entire service and starting it again.

When a Kubernetes Deployment's Pod template changes, Kubernetes creates a new ReplicaSet. It scales the new ReplicaSet up while scaling the old one down according to rollout settings such as `maxSurge` and `maxUnavailable`.

During the transition, the old and new versions can coexist:

1. The old version is serving users.
2. Kubernetes starts Pods running the new version.
3. Once new Pods are ready, older Pods are gradually removed.
4. The rollout finishes when the desired set of updated Pods is available.

This means that requests may reach either version while the rollout is in progress. The deployment therefore needs to tolerate a temporary mixture of versions.

**What I find useful about this approach:** We can update an application incrementally without maintaining a complete second environment. It is a natural starting point for understanding production rollouts.

However, **a rolling update alone does not guarantee zero downtime**. The application still needs appropriate readiness checks, sufficient capacity, graceful shutdown behavior, and compatibility between versions. A database-schema change, for example, may affect both old and new application instances during the rollout.

*Diagram placeholder: Old and new ReplicaSets coexist while Pods are replaced. The diagram from the original Korean post should be reviewed and migrated separately.*

## 2. Blue–green deployments

Sometimes we want to prepare an updated application in an environment that closely resembles production before directing ordinary users to it. In that situation, replacing Pods gradually may not be the most convenient option.

A blue–green deployment keeps two environments:

- **Blue:** The current version serving users.
- **Green:** The candidate version, deployed and checked separately.

The basic sequence is straightforward:

1. Keep blue available for existing traffic.
2. Deploy the new version to green.
3. Confirm that green is ready.
4. Change the routing configuration—for example, a Kubernetes Service selector or an external load balancer—to direct traffic to green.

If an issue appears while blue is still available, traffic can potentially be directed back to the previous version. Once blue has been removed, that quick rollback route no longer exists.

The main trade-off is **additional capacity**. Because two environments coexist for a period, blue–green deployment can require substantial extra resources. The exact increase depends on how the environments are designed; it is not necessarily precisely twice the original footprint.

*Diagram placeholder: Blue and green environments run in parallel, with the traffic route switching to green.*

## 3. Canary deployments

Rolling updates introduce new application instances gradually, but they can still expose many users to a faulty change as the rollout progresses. Blue–green deployments allow a separate environment to be prepared, but running two environments can be costly.

What if we wanted to let **only a small subset of real users** try the new version before deploying it more widely?

That is the idea behind a canary release.

For example, a system could initially route 5% of eligible traffic to a new application version while the other 95% continues to use the existing version. Engineers could then observe error rates, latency, and other application-specific indicators. If the candidate behaves as expected, the share of traffic directed to it could be increased.

The 5% figure is only an illustration, not a recommended default.

Staged feature rollouts in consumer applications illustrate why some users might see a feature earlier than others. That visible difference does not, by itself, prove that a particular service uses canary deployment; other feature-management mechanisms may produce similar results.

*Diagram placeholder: A small share of traffic reaches the candidate version while the remaining traffic stays on the stable version.*

### How is a rolling update different from a canary release?

Both methods can be gradual, but they control different things.

| | Rolling update | Canary release |
|---|---|---|
| Primary control | Replacement of running application instances | Exposure of users or traffic to a candidate version |
| Typical progression | Old Pods are replaced by new Pods | A small traffic share grows after evaluation |
| Main question | How do we replace instances safely? | How does the new version behave with limited real traffic? |
| Additional mechanisms | Supported by Kubernetes Deployment rollout behavior | Often requires traffic splitting, service-mesh, ingress, or feature-control tooling |

In other words, a rolling update is primarily about **instance replacement**, while a canary release is primarily about **controlled exposure and evaluation**. They are not mutually exclusive.

## What I want to explore next

Canary deployment is the approach I find hardest to visualize. How does the system actually decide which users—or which requests—should reach the new version? How can we verify that the traffic split is working as intended?

I want to answer these questions through a small experiment rather than stop at the conceptual explanation.

That will be the subject of the next post.