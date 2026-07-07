---
layout: web/course-reading
heading: "Operations & Troubleshooting: Start Here"
title: "Kubernetes Troubleshooting Guide: Start Here (Beginner-Friendly)"
subtitles:
  - "A self-contained, end-to-end Kubernetes troubleshooting guide. Start with a 5-minute triage checklist, then follow the symptom router to the exact fix, no prior lessons required."
permalink: /kubernetes/operations-troubleshooting-start-here
seo:
  type: TechArticle
  date_published: false
description: "A self-contained, end-to-end Kubernetes troubleshooting guide. Start with a 5-minute triage checklist, then follow the symptom router to the exact fix, no prior lessons required."
---


## What this section is for

Something is broken in your cluster right now and you need to fix it, not learn Kubernetes from scratch. This section is a standalone operations guide: every page is self-contained, explains what each command does and why you're running it, and gives you the full command up front, not a link to go read somewhere else first. You do **not** need to have read the rest of the course to use it.

If you'd rather build a complete mental model of Kubernetes first, the [full course](/kubernetes/) starting at [Welcome](/kubernetes/welcome) teaches everything here in depth, with labs. Come back to this section for live troubleshooting once something is actually on fire.

{% capture callout_body %}
Every command below assumes you have `kubectl` installed and pointed at the right cluster. If you're not sure, run `kubectl config current-context` first, it prints the cluster name you're about to run commands against. Running the right commands against the wrong cluster is a common, entirely avoidable mistake.
{% endcapture %}
{% include components/course-callout.html type="note" title="Before you run anything" body=callout_body %}

## The first 5 minutes of any incident

Run these four checks in order, every single time, regardless of what the symptom looks like. They take under five minutes combined and they tell you *where* the problem lives (your app, the cluster, or the network) before you go deep on any one theory.

### Step 1: Confirm you're pointed at the right cluster and it's healthy

```bash
kubectl config current-context
kubectl cluster-info
kubectl get nodes -o wide
```

- `kubectl config current-context` prints which cluster your commands will run against. If you manage more than one cluster (dev/staging/prod), this is the single most common source of wasted troubleshooting time, checking the wrong cluster and concluding "everything looks fine."
- `kubectl cluster-info` confirms the control plane (the Kubernetes "brain": API server, scheduler, etc.) is reachable at all. If this hangs or errors, nothing else below will work either, the problem is at the cluster level, not your application.
- `kubectl get nodes -o wide` lists every machine (physical or virtual) in the cluster. Look at the `STATUS` column: every node should say `Ready`. A node stuck on `NotReady` means every pod scheduled there is at risk, and explains outages that look random ("only some replicas are failing") until you notice they're all on one bad node.

### Step 2: Look at cluster-wide events, sorted by time

```bash
kubectl get events -A --sort-by='.lastTimestamp'
```

Kubernetes emits an "event" object every time something notable happens: a pod fails to schedule, an image fails to pull, a container crashes, a probe fails. This single command shows you the most recent ones across every namespace (`-A`), sorted oldest to newest. Scroll to the bottom for the newest events, this is often the fastest way to spot "oh, it's a `FailedScheduling` event for 15 pods, all at the same second" and immediately know you're looking at a cluster capacity problem, not 15 unrelated application bugs.

### Step 3: Check the health of the specific workload you suspect

```bash
kubectl get pods -n <namespace> -o wide
```

Replace `<namespace>` with wherever your application lives (`kubectl get namespaces` lists all of them if you're not sure). Look at the `STATUS` column. This one command tells you which broad category of problem you're in, which is exactly what the symptom router below is organized around:

| STATUS you see | What it means | Go to |
|---|---|---|
| `Pending` | Pod hasn't been scheduled onto a node yet, or is stuck before containers even start | [Pod Won't Start](/kubernetes/troubleshooting-pod-not-starting) |
| `ImagePullBackOff` / `ErrImagePull` | Kubernetes can't download the container image | [Pod Won't Start](/kubernetes/troubleshooting-pod-not-starting) |
| `CrashLoopBackOff` | Container starts, then dies, repeatedly | [CrashLoopBackOff & Restarts](/kubernetes/troubleshooting-crashloop-and-restarts) |
| `Running` but `READY` shows `0/1` | Container is running but failing its readiness check, so it receives no traffic | [Networking & 503s](/kubernetes/troubleshooting-networking-and-503s) |
| `Running` and `READY`, but users report errors or slowness | Likely networking, downstream dependency, or resource contention | [Networking & 503s](/kubernetes/troubleshooting-networking-and-503s) or [Slow Performance & Resources](/kubernetes/troubleshooting-slow-performance-and-resources) |
| `Terminating` and stuck there for minutes | Pod won't shut down cleanly, often a storage or finalizer issue | [Storage & Config Issues](/kubernetes/troubleshooting-storage-and-config) |

### Step 4: Check what changed recently

```bash
kubectl rollout history deployment/<name> -n <namespace>
kubectl get deployment <name> -n <namespace> -o jsonpath='{.spec.template.spec.containers[0].image}{"\n"}'
```

The single highest-value question in any incident is "what changed?" The first command shows the revision history of a Deployment (a Deployment is the object that manages a set of identical pod replicas for you), the second prints exactly which container image is currently configured to run. Cross-reference the timing against your deploy pipeline or GitOps tool. A huge fraction of production incidents trace back to a deploy that happened in the last hour, check this before assuming you have a novel, mysterious bug.

## Symptom router

Once you've run the four steps above, you should already have a strong hint about which category you're in. Pick the page that matches what you're actually seeing:

- **Pod stuck in `Pending`, `ImagePullBackOff`, or won't get scheduled at all** → [Pod Won't Start / Pending / ImagePullBackOff](/kubernetes/troubleshooting-pod-not-starting)
- **Pod starts, then restarts repeatedly, `CrashLoopBackOff`, OOMKilled, non-zero exit codes** → [CrashLoopBackOff & Restarts](/kubernetes/troubleshooting-crashloop-and-restarts)
- **503s, connection refused, timeouts, "works locally but not from another pod," DNS failures** → [Networking, DNS & 503s](/kubernetes/troubleshooting-networking-and-503s)
- **Requests are slow, CPU/memory look pegged, autoscaling isn't kicking in or is thrashing** → [Slow Performance & Resource Throttling](/kubernetes/troubleshooting-slow-performance-and-resources)
- **PersistentVolumeClaim stuck `Pending`, data missing/stale, ConfigMap or Secret change not taking effect** → [Storage, ConfigMaps & Secrets](/kubernetes/troubleshooting-storage-and-config)

Each of those pages is fully self-contained: it repeats the relevant context, explains every command inline, and doesn't assume you've read anything else first. If you land on the wrong page, each one links back here and sideways to the others, so you can course-correct without losing your place.

## If you're still stuck after the linked page

1. Re-run the [first 5 minutes](#the-first-5-minutes-of-any-incident) checks, sometimes the real signal appears only after you've already fixed the first, more obvious problem.
2. Widen your search: `kubectl get events -A --sort-by='.lastTimestamp'` again, a second, unrelated event may have appeared since you started.
3. If this is a live production incident with users affected, stop debugging solo and open the [Incident Response Runbook Template](/kubernetes/incident-runbook-template), it keeps you from missing a step under pressure and gives you a structured document to hand off or escalate with.

## Checkpoint

- [ ] I ran `kubectl config current-context` and confirmed I'm pointed at the correct cluster.
- [ ] I checked node health and cluster-wide events before diving into one specific theory.
- [ ] I identified my pod's `STATUS` and picked the matching symptom page above.
- [ ] I know to check "what changed recently" before assuming a novel root cause.

**Related:** [Command Cheat Sheet](/kubernetes/command-cheat-sheet) · [Incident Runbook Template](/kubernetes/incident-runbook-template)
