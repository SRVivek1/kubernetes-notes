---
layout: web/course-reading
heading: "Troubleshooting: Slow Performance & Resource Throttling"
title: "Kubernetes Slow Pods & CPU Throttling: Diagnose and Fix"
subtitles:
  - "A step-by-step, symptom-first guide to diagnosing slow Kubernetes pods, CPU throttling, memory pressure, and autoscaling problems, with copy-paste commands and plain-English explanations."
permalink: /kubernetes/troubleshooting-slow-performance-and-resources
seo:
  type: TechArticle
  date_published: false
description: "A step-by-step, symptom-first guide to diagnosing slow Kubernetes pods, CPU throttling, memory pressure, and autoscaling problems, with copy-paste commands and plain-English explanations."
---


## Is this the right page?

You're here because of one of these:

- Requests are slower than usual, but the pod is `Running` and not crashing or restarting
- `kubectl top` shows CPU or memory usage near, or at, the configured limit
- Autoscaling (HPA) isn't adding replicas when you expect it to, or is scaling up and down repeatedly ("thrashing")
- Latency spikes happen in short, regular bursts rather than being constantly slow

If the pod is actually crashing or restarting, go to [CrashLoopBackOff & Restarts](/kubernetes/troubleshooting-crashloop-and-restarts) instead, that page covers `OOMKilled` (memory limit exceeded, container killed). This page covers the case where the container survives, but performs badly. If requests are failing outright rather than just slow, check [Networking, DNS & 503s](/kubernetes/troubleshooting-networking-and-503s) first, a `504 Gateway Timeout` from an edge proxy often means "reached this page's territory but gave up before you noticed"; both are worth checking together.

## Step 1: confirm what's actually being consumed right now

```bash
kubectl top pod <pod> -n <namespace> --containers
kubectl top nodes
```

`kubectl top` requires the metrics-server add-on to be installed in the cluster; if this command errors out entirely, that's your first finding, ask whoever manages the cluster to confirm metrics-server is running before continuing. Otherwise, this gives you live CPU/memory usage per container, and per node. Compare the per-container numbers against the pod's configured requests/limits:

```bash
kubectl get pod <pod> -n <namespace> -o jsonpath='{.spec.containers[0].resources}'
```

If usage is sitting right at (or bouncing off) the CPU or memory limit, you've likely found your answer already, continue to whichever branch below matches: CPU near its limit → [CPU throttling](#cause-cpu-is-being-throttled-not-genuinely-slow), memory near its limit but not yet OOMKilled → [memory pressure](#cause-memory-pressure-short-of-a-full-oomkill). If usage looks comfortably below both limits, the bottleneck is likely elsewhere (a downstream dependency, database, or the application's own code), and Kubernetes-level resource tuning won't fix it, jump to [It's not resource-bound at all](#its-not-resource-bound-at-all-checking-downstream-dependencies).

## Cause: CPU is being throttled, not genuinely slow

This is the single most commonly misdiagnosed performance problem in containerized applications, because it produces symptoms that look exactly like a garbage-collection pause or general sluggishness, but the fix is completely different.

### Why this happens

A CPU **limit** on a container is enforced by the Linux kernel's CFS (Completely Fair Scheduler) using cgroup quotas: your container is given a fixed amount of CPU time within each fixed time window (commonly 100ms). Once it has used up that window's quota, the kernel stops scheduling it entirely until the next window starts, no matter what it was doing, mid-request, mid-garbage-collection, doesn't matter. From outside the container, this looks like a stall followed by a resume, indistinguishable at a glance from a GC pause or a slow downstream call.

### Check for it directly

```bash
kubectl exec -it <pod> -n <namespace> -- cat /sys/fs/cgroup/cpu.stat
```

Look at `nr_throttled` (how many scheduling periods this container has been throttled in) and `throttled_time` (cumulative time spent throttled). If these are non-zero and climbing while you reproduce the slowness, CPU throttling is confirmed as at least part of the problem.

```bash
# Sample it twice, a minute apart, under load, to see the *rate* of throttling, not just the total
kubectl exec -it <pod> -n <namespace> -- cat /sys/fs/cgroup/cpu.stat
# wait ~60s, ideally while generating load
kubectl exec -it <pod> -n <namespace> -- cat /sys/fs/cgroup/cpu.stat
```

### The definitive way to rule GC in or out (JVM workloads)

If this is a JVM-based application (Java/Spring Boot), the fastest way to tell CFS throttling apart from a genuine GC pause is that **GC pauses always appear in the GC log at the exact moment they happen; CFS throttling never appears in the GC log at all**, the JVM has no way of knowing the kernel paused it, from its own perspective time simply skipped forward.

```bash
# If GC logging is enabled, check whether pause timestamps line up with your observed latency spikes
kubectl exec -it <pod> -n <namespace> -- tail -100 /dumps/gc.log
```

If your latency spikes do **not** correlate with any GC log entry at the same timestamp, stop looking at GC tuning entirely and treat it as CPU throttling.

### The fix

Two options, and they solve different aspects of the same problem:

1. **Raise (or remove) the CPU limit** so the container has enough quota per period to avoid throttling under real load. Check current usage patterns with `kubectl top pod --containers` over time first to pick a sane new limit rather than guessing.
2. **For JVM workloads specifically**, set `-XX:ActiveProcessorCount` to match the container's *actual* CPU budget. Without this, the JVM sizes its internal thread pools (including GC worker threads) based on the number of CPU cores it sees on the node, which may be far more than its cgroup quota actually allows, making it oversubscribe itself and throttle harder than necessary.

```bash
# Apply a higher CPU limit
kubectl set resources deployment/<name> -n <namespace> -c=<container-name> --limits=cpu=1000m
```

## Cause: memory pressure short of a full OOMKill

If `kubectl top pod --containers` shows memory usage consistently near the configured limit, but the container hasn't actually been killed yet, you may be seeing degraded performance from the application (or JVM) working hard to stay under the ceiling, frequent garbage collection cycles as the heap fills up repeatedly, for example, rather than an outright crash.

```bash
# For a JVM workload: check GC frequency and whether the heap is recovering space after each collection
kubectl exec -it <pod> -n <namespace> -- jstat -gcutil 1 1000 10
```

Watch the `FGC`/`FGCT` columns (full garbage collection count and cumulative time). Frequent full GCs where heap occupancy stays high even right after a collection indicate the working set genuinely doesn't fit in the available heap, this is the same resource-ceiling problem covered on the [CrashLoopBackOff & Restarts](/kubernetes/troubleshooting-crashloop-and-restarts#reason-oomkilled-exit-code-137) page, just caught before it escalates to a full kill. The fix is the same: raise the memory limit with real headroom, or address a genuine leak if usage climbs steadily rather than sawtoothing normally between collections.

## Autoscaling (HPA) isn't behaving as expected

### HPA isn't scaling up even though load is high

```bash
kubectl get hpa -n <namespace>
kubectl describe hpa <hpa-name> -n <namespace>
```

`kubectl get hpa` shows current vs. target metric value directly in its output, if the current value never crosses the target threshold, the HPA is behaving correctly, your scaling target/threshold itself may just be set too high or measuring the wrong thing. `kubectl describe hpa` shows recent scaling events and, importantly, any errors it hit trying to read metrics at all:

```bash
# If describe shows "unable to get metrics" or similar, metrics-server itself is the problem
kubectl top pods -n <namespace>
```

If `kubectl top pods` also fails or returns nothing, the HPA has no data to scale on regardless of real load, fix the metrics pipeline first (confirm metrics-server is running: `kubectl get deployment metrics-server -n kube-system`), before touching the HPA configuration itself.

### HPA is thrashing (scaling up and down repeatedly)

```bash
kubectl describe hpa <hpa-name> -n <namespace> | grep -A10 Events
```

This usually means the target utilization threshold is too tight relative to natural traffic variance, causing the HPA to react to normal fluctuation as if it were a sustained trend. The fix is typically raising the `--horizontal-pod-autoscaler-downscale-stabilization` window (how long the HPA waits after a spike before scaling back down) or widening the target metric threshold so brief spikes don't trigger an immediate scale event.

## It's not resource-bound at all: checking downstream dependencies

If CPU and memory both look comfortably under their limits, the slowness is very likely coming from something the application is waiting on, not the pod's own compute resources.

```bash
# Check if this pod is spending time waiting on a downstream call
kubectl logs <pod> -n <namespace> --tail=200 | grep -iE "timeout|slow|retry|circuit"

# Check connection pool exhaustion signatures (common with a DB or HTTP client pool that's too small)
kubectl logs <pod> -n <namespace> --tail=200 | grep -iE "pool exhausted|connection pool|timed out waiting for connection"
```

If you find evidence of a downstream dependency (a database, cache, or another microservice) being slow or refusing connections, that dependency needs to be diagnosed directly, following the same approach on whichever pod it's running as. If it's an external, non-Kubernetes dependency (a managed database, third-party API), this is now outside what `kubectl`-based troubleshooting alone can diagnose, check that dependency's own metrics/dashboard next.

## Quick reference: symptom to cause

| Symptom | Check first |
|---|---|
| Latency spikes in short, regular bursts | `cpu.stat` `nr_throttled`, compare against GC log timestamps if JVM |
| Steady degraded performance, not bursty | `kubectl top` vs. requests/limits, then downstream dependency logs |
| Memory usage near limit, frequent GCs, not yet killed | `jstat -gcutil`, same fix path as OOMKilled but caught earlier |
| HPA never scales up | `kubectl describe hpa` for metric errors, confirm metrics-server is healthy |
| HPA scales up/down repeatedly | Widen target threshold or increase downscale stabilization window |
| Fast pod, slow overall request | Check downstream dependency logs for timeouts/pool exhaustion |

## Checkpoint

- [ ] I compared live `kubectl top` usage against the pod's configured requests/limits before forming a theory.
- [ ] I checked `cpu.stat`'s `nr_throttled` before assuming a JVM latency spike was a GC problem.
- [ ] I know that CFS throttling never appears in a GC log, and used that to rule GC in or out with evidence.
- [ ] I checked `kubectl describe hpa` for metric-collection errors before assuming an autoscaling threshold problem.
- [ ] I checked downstream dependency logs before concluding the pod's own resources were the bottleneck.

**Related:** [Operations & Troubleshooting: Start Here](/kubernetes/operations-troubleshooting-start-here) · [CrashLoopBackOff & Restarts](/kubernetes/troubleshooting-crashloop-and-restarts) · [Networking, DNS & 503s](/kubernetes/troubleshooting-networking-and-503s) · [Command Cheat Sheet](/kubernetes/command-cheat-sheet)
