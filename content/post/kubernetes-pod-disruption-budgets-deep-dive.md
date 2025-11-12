---
title: "Pod Disruption Budgets: The Kubernetes Feature That Saved My Production Cluster"
date: 2025-04-25T14:30:00+01:00
draft: false
categories:
  - Kubernetes
tags:
  - Kubernetes
  - DevOps
  - Production
  - Reliability
description: "Deep dive into Pod Disruption Budgets and how they prevent cascading failures during voluntary disruptions."
---

I learned about Pod Disruption Budgets (PDBs) the hard way. It was 3 AM, and our entire payment processing service went down during what should have been a routine node drain. The cluster autoscaler decided to scale down, Kubernetes evicted pods left and right, and suddenly we had zero healthy replicas. That's when I discovered PDBs aren't just a nice-to-have-they're essential for production workloads.

## What Are Pod Disruption Budgets, Really?

At its core, a Pod Disruption Budget tells Kubernetes: "Hey, you can evict pods from this set, but never drop below X healthy replicas." It sounds simple, but the devil is in the details. PDBs only apply to *voluntary* disruptions-things like node drains, cluster autoscaling, or manual pod deletions. They don't protect against involuntary disruptions like node failures or OOM kills.

Here's the thing most people miss: PDBs work with the `disruption` subresource, which is separate from the actual pod lifecycle. When you drain a node, Kubernetes checks all relevant PDBs before evicting pods. If evicting a pod would violate a PDB, the eviction is blocked.

## The Math Behind PDBs

PDBs support two ways to specify availability:

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: payment-service-pdb
spec:
  minAvailable: 2  # Absolute number
  # OR
  minAvailable: 50%  # Percentage
  selector:
    matchLabels:
      app: payment-service
```

But here's the gotcha: `minAvailable` and `maxUnavailable` are mutually exclusive, and the calculation isn't always intuitive. If you have 5 replicas and set `minAvailable: 2`, Kubernetes will allow up to 3 pods to be disrupted. However, if you set `minAvailable: 50%` with 5 replicas, it rounds down to 2 (since 50% of 5 is 2.5, which becomes 2).

The real complexity comes when you have multiple PDBs affecting the same pods, or when pods are in different states (Pending, Running, Terminating). Kubernetes counts pods in the `Ready` condition, not just Running pods.

## Real-World Gotchas

### Gotcha #1: PDBs and Rolling Updates

During a rolling update, Kubernetes creates new pods before terminating old ones. But if you have a PDB that's too restrictive, your rolling update can get stuck. I've seen this happen:

```yaml
# This PDB will block rolling updates if you have 3 replicas
spec:
  minAvailable: 3  # With 3 replicas, this means 0 can be disrupted
```

The solution? Use `maxUnavailable` instead for workloads that need rolling updates:

```yaml
spec:
  maxUnavailable: 1  # Allow 1 pod to be unavailable during updates
```

### Gotcha #2: PDBs Don't Respect Readiness Probes Immediately

Here's something that bit me: when a pod's readiness probe starts failing, it's removed from the Service endpoints, but it might still count toward your PDB's `minAvailable` for a brief window. The PDB controller checks pod conditions, but there's a reconciliation delay.

If you're doing a node drain and a pod's readiness probe is flapping, you might see evictions blocked even though the pod isn't serving traffic. The workaround is to ensure your readiness probes are stable, or use preStop hooks to gracefully drain connections.

### Gotcha #3: Cluster Autoscaler Interactions

Cluster autoscaler respects PDBs, but the interaction can be subtle. When autoscaler wants to scale down a node, it checks PDBs. If evicting pods would violate a PDB, it won't drain the node. However, if you have pods that can't be evicted due to PDBs, and those pods are the only ones on a node, that node can't be scaled down.

This creates a situation where you might have underutilized nodes that can't be removed. The solution is to use pod anti-affinity rules to spread pods across nodes, or adjust your PDBs to allow some disruption.

## Advanced PDB Patterns

### Pattern 1: Tiered Availability

For microservices with different criticality levels, use tiered PDBs:

```yaml
# Critical service - must maintain 80% availability
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: critical-service-pdb
spec:
  minAvailable: 80%
  selector:
    matchLabels:
      tier: critical
---
# Standard service - can tolerate 50% disruption
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: standard-service-pdb
spec:
  maxUnavailable: 50%
  selector:
    matchLabels:
      tier: standard
```

### Pattern 2: PDBs with Pod Anti-Affinity

Combine PDBs with pod anti-affinity to ensure availability across zones:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: multi-zone-service
spec:
  replicas: 6
  template:
    spec:
      affinity:
        podAntiAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
          - weight: 100
            podAffinityTerm:
              labelSelector:
                matchLabels:
                  app: my-app
              topologyKey: topology.kubernetes.io/zone
---
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: multi-zone-pdb
spec:
  minAvailable: 4  # Ensures at least 4 pods across zones
  selector:
    matchLabels:
      app: my-app
```

This ensures that even if an entire zone goes down, you still have pods running in other zones.

## Monitoring PDBs

PDBs have a status field that's often overlooked:

```yaml
status:
  currentHealthy: 5
  desiredHealthy: 3
  disruptedPods: {}
  disruptionsAllowed: 2
  expectedPods: 5
  observedGeneration: 1
```

The `disruptionsAllowed` field tells you how many pods can currently be evicted. If this is 0 and you're trying to drain a node, the drain will be blocked. I've set up alerts on this metric to catch situations where PDBs are too restrictive.

## The Unhealthy Pod Problem

Here's a scenario that's not well-documented: what happens when you have unhealthy pods that count toward your PDB? If you have 5 replicas, 2 are unhealthy (not Ready), and your PDB requires `minAvailable: 3`, Kubernetes will only allow 0 disruptions because it counts the 3 healthy pods.

This can create a situation where you can't drain nodes or perform maintenance because unhealthy pods are blocking evictions. The solution is to either fix the unhealthy pods first, or temporarily adjust the PDB (though this is risky in production).

## Best Practices

1. **Always use PDBs for stateful services** - Even if you think you don't need them, they prevent accidental data loss during maintenance.

2. **Use `maxUnavailable` for stateless services** - It's more intuitive for rolling updates and allows better control.

3. **Set PDBs conservatively at first** - Start with higher availability requirements and relax them as you understand your workload's behavior.

4. **Monitor PDB status** - Set up alerts when `disruptionsAllowed` is 0 for extended periods.

5. **Test your PDBs** - Don't wait for a production incident. Test node drains in a staging environment with your PDBs in place.

6. **Document your PDB strategy** - Make it clear why each PDB is configured the way it is, so future engineers understand the trade-offs.

## Conclusion

Pod Disruption Budgets are one of those Kubernetes features that seem simple but have surprising depth. They're essential for production reliability, but they require careful tuning. The key is understanding how they interact with other Kubernetes features like rolling updates, cluster autoscaling, and pod scheduling.

After that 3 AM incident, I made PDBs mandatory for all production services. We haven't had a similar outage since. Sometimes the best lessons come from the worst nights.

