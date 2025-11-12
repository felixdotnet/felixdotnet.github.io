---
title: "Kubernetes Scheduling: Beyond the Basics - Taints, Tolerations, and the Art of Pod Placement"
date: 2025-05-15T16:45:00+01:00
draft: false
categories:
  - Kubernetes
tags:
  - Kubernetes
  - Scheduling
  - Infrastructure
  - Advanced
description: "Advanced Kubernetes scheduling techniques that most people never discover until they hit production issues."
---

Most Kubernetes tutorials cover the basics: Deployments, Services, maybe a bit about node selectors. But after running production clusters for years, I've learned that the real power-and the real headaches-come from the advanced scheduling features that most people never touch.

## The Scheduler's Decision Tree

Before diving into the advanced stuff, it's worth understanding what the scheduler actually does. When a pod is created, the scheduler evaluates every node against a series of predicates (filters) and then scores the remaining nodes. The highest-scoring node wins.

The predicates include things like:
- Node resources (CPU, memory)
- Node selectors and affinity
- Taints and tolerations
- Pod anti-affinity rules
- Volume zone restrictions

But here's what's not obvious: the scheduler runs these checks in a specific order, and if a node fails any predicate, it's immediately excluded. This means the order of your constraints matters.

## Taints and Tolerations: The Misunderstood Duo

Taints and tolerations are probably the most misunderstood scheduling feature. Most people think they're just for dedicated nodes, but they're actually a powerful way to create node "classes" and control pod placement.

### The Basics (That Everyone Gets Wrong)

A taint has three parts: key, value, and effect. The effect can be:
- `NoSchedule` - Don't schedule new pods (existing pods stay)
- `PreferNoSchedule` - Try not to schedule, but allow if necessary
- `NoExecute` - Don't schedule AND evict existing pods that don't tolerate

Here's the gotcha: `NoExecute` taints will evict pods that don't have matching tolerations, even if they were running before the taint was added. I've seen this cause unexpected pod evictions during node maintenance.

### Advanced Taint Patterns

**Pattern 1: Dedicated Workload Nodes**

```yaml
# Taint the node
kubectl taint nodes gpu-node-1 workload=gpu:NoSchedule

# Pod with toleration
apiVersion: v1
kind: Pod
metadata:
  name: gpu-workload
spec:
  tolerations:
  - key: "workload"
    operator: "Equal"
    value: "gpu"
    effect: "NoSchedule"
  containers:
  - name: gpu-app
    image: nvidia/cuda:latest
```

But here's the trick: you can use `operator: Exists` to tolerate any taint with a specific key, regardless of value. This is useful for "soft" node classes:

```yaml
tolerations:
- key: "workload"
  operator: "Exists"  # Tolerates any workload taint
  effect: "NoSchedule"
```

**Pattern 2: Time-Based Taints**

You can use taints to implement maintenance windows. Add a taint before maintenance, and only pods with the appropriate toleration will run:

```bash
# Before maintenance
kubectl taint nodes node-1 maintenance=yes:NoExecute

# After maintenance
kubectl taint nodes node-1 maintenance=yes:NoExecute-
```

The `-` at the end removes the taint. This pattern is useful for canary deployments or testing new node configurations.

**Pattern 3: Cost Optimization with Spot Instances**

AWS spot instances (and similar in other clouds) can be terminated with 2 minutes notice. Use taints to mark these nodes and ensure only fault-tolerant workloads run on them:

```yaml
# Taint spot instances
kubectl taint nodes spot-node-1 instance-type=spot:NoSchedule

# Pod that can handle interruptions
apiVersion: v1
kind: Pod
metadata:
  name: batch-job
spec:
  tolerations:
  - key: "instance-type"
    operator: "Equal"
    value: "spot"
    effect: "NoSchedule"
  # Use terminationGracePeriodSeconds for graceful shutdown
  terminationGracePeriodSeconds: 120
```

## Pod Affinity and Anti-Affinity: The Real Power

Pod affinity rules are where scheduling gets interesting. Most people know about `requiredDuringSchedulingIgnoredDuringExecution` (hard requirement) and `preferredDuringSchedulingIgnoredDuringExecution` (soft preference), but there are subtleties that bite you in production.

### The Topology Key Gotcha

Topology keys determine what "topology" means. Common keys include:
- `kubernetes.io/hostname` - Same node
- `topology.kubernetes.io/zone` - Same availability zone
- `topology.kubernetes.io/region` - Same region

But here's what's not obvious: if you use a topology key that doesn't exist on your nodes, the affinity rule is ignored. No error, no warning-it just silently fails. Always verify your topology keys:

```bash
kubectl get nodes --show-labels | grep topology
```

### Advanced Anti-Affinity Pattern: Spread Across Everything

Want to ensure pods are spread across nodes, zones, and even regions? You need multiple anti-affinity rules:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: distributed-service
spec:
  replicas: 6
  template:
    spec:
      affinity:
        podAntiAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
          - labelSelector:
              matchExpressions:
              - key: app
                operator: In
                values:
                - my-app
            topologyKey: kubernetes.io/hostname
          preferredDuringSchedulingIgnoredDuringExecution:
          - weight: 100
            podAffinityTerm:
              labelSelector:
                matchExpressions:
                - key: app
                  operator: In
                  values:
                  - my-app
              topologyKey: topology.kubernetes.io/zone
          - weight: 50
            podAffinityTerm:
              labelSelector:
                matchExpressions:
                - key: app
                  operator: In
                  values:
                  - my-app
              topologyKey: topology.kubernetes.io/region
```

This ensures:
1. No two pods on the same node (hard requirement)
2. Prefer spreading across zones (soft preference, weight 100)
3. Prefer spreading across regions (soft preference, weight 50)

The weights matter: higher weights are preferred more strongly. The scheduler sums weights for nodes that match multiple preferred rules.

### Inter-Pod Affinity: The Hidden Feature

You can make pods prefer to be scheduled near other pods with specific labels. This is useful for co-locating related services:

```yaml
affinity:
  podAffinity:
    preferredDuringSchedulingIgnoredDuringExecution:
    - weight: 100
      podAffinityTerm:
        labelSelector:
          matchLabels:
            app: database
        topologyKey: kubernetes.io/hostname
```

This makes the pod prefer to be on the same node as pods labeled `app: database`. Useful for reducing network latency between tightly coupled services.

## Node Affinity: Beyond Simple Selectors

Node selectors are simple, but node affinity gives you more power:

```yaml
affinity:
  nodeAffinity:
    requiredDuringSchedulingIgnoredDuringExecution:
      nodeSelectorTerms:
      - matchExpressions:
        - key: instance-type
          operator: In
          values:
          - m5.large
          - m5.xlarge
    preferredDuringSchedulingIgnoredDuringExecution:
    - weight: 80
      preference:
        matchExpressions:
        - key: zone
          operator: In
          values:
          - us-east-1a
```

The `operator` field supports: `In`, `NotIn`, `Exists`, `DoesNotExist`, `Gt`, `Lt`. The `Gt` and `Lt` operators are useful for numeric labels like CPU count or memory size.

## The Scheduler's Scoring System

After filtering nodes with predicates, the scheduler scores them. The default scoring plugins include:
- `NodeResourcesFit` - Prefers nodes with more available resources
- `NodeAffinity` - Scores based on node affinity rules
- `InterPodAffinity` - Scores based on pod affinity/anti-affinity
- `ImageLocality` - Prefers nodes that already have the container image

You can see the scores (in debug mode) with:

```bash
kubectl get events --field-selector involvedObject.kind=Pod --sort-by='.lastTimestamp' | grep Scheduled
```

But to really understand why a pod was scheduled where it was, you need to enable scheduler logging or use a custom scheduler.

## Custom Schedulers: When the Default Isn't Enough

Sometimes the default scheduler isn't enough. You can run multiple schedulers and specify which one to use:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: custom-scheduled-pod
spec:
  schedulerName: my-custom-scheduler
  containers:
  - name: app
    image: nginx
```

I've used custom schedulers for:
- GPU workload scheduling (ensuring GPU memory constraints)
- Cost optimization (preferring cheaper instance types)
- Workload-specific placement (ML training jobs that need specific hardware)

Building a custom scheduler is non-trivial, but the Kubernetes scheduler framework makes it possible. You implement the `SchedulePlugin` interface and register your plugin.

## Real-World Gotchas

**Gotcha #1: Pending Pods and Resource Fragmentation**

If you have many small pods, you can end up with resource fragmentation where no node has enough resources for a large pod, even though the cluster has sufficient total resources. The solution is to use pod priority and preemption, or to defragment by draining and rescheduling nodes.

**Gotcha #2: Affinity Rules and Rolling Updates**

During a rolling update, new pods need to be scheduled. If your anti-affinity rules are too strict, new pods might not be schedulable because they conflict with existing pods. Use `preferredDuringScheduling` instead of `requiredDuringScheduling` for rolling update scenarios.

**Gotcha #3: Taint Effects and Existing Pods**

Remember: `NoExecute` taints evict existing pods. If you add a `NoExecute` taint to a node, all pods without matching tolerations will be evicted immediately. Use `NoSchedule` first, then drain manually if needed.

## Monitoring Scheduling Decisions

To understand why pods are pending or why they're scheduled on specific nodes:

```bash
# Describe the pod to see events
kubectl describe pod <pod-name>

# Check node conditions
kubectl get nodes -o wide

# See scheduler events
kubectl get events --all-namespaces --field-selector reason=Scheduled
```

For deeper debugging, enable scheduler profiling:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: scheduler-config
data:
  config.yaml: |
    profiles:
    - schedulerName: default-scheduler
      plugins:
        score:
          enabled:
          - name: NodeResourcesFit
            weight: 1
          - name: NodeAffinity
            weight: 1
```

## Best Practices

1. **Start with soft constraints** - Use `preferredDuringScheduling` before `requiredDuringScheduling` to avoid scheduling deadlocks.

2. **Test your scheduling rules** - Create test pods and verify they're scheduled as expected before deploying to production.

3. **Monitor pending pods** - Set up alerts for pods that are pending for more than a few minutes.

4. **Use pod priority** - Implement priority classes to ensure critical workloads are scheduled first.

5. **Document your taints** - If you're using custom taints, document what they mean and which workloads should tolerate them.

6. **Review scheduling decisions periodically** - Cluster topology changes, and your scheduling rules might need adjustment.

## Conclusion

Kubernetes scheduling is deceptively complex. The basics get you far, but understanding the advanced features-taints, tolerations, affinity rules, and the scoring system-is what separates production-ready clusters from development setups.

The key is to start simple and add complexity only when needed. But when you do need it, these advanced scheduling features give you the control to optimize for cost, performance, and reliability.

