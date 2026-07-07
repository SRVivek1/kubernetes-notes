---
layout: web/course-reading
heading: "Troubleshooting: Pod Won't Start"
title: "Kubernetes Pod Stuck Pending or ImagePullBackOff: Fix It Now"
subtitles:
  - "A step-by-step, symptom-first guide to fixing Kubernetes pods stuck in Pending, ImagePullBackOff, or ContainerCreating, with copy-paste commands and plain-English explanations."
permalink: /kubernetes/troubleshooting-pod-not-starting
seo:
  type: TechArticle
  date_published: false
description: "A step-by-step, symptom-first guide to fixing Kubernetes pods stuck in Pending, ImagePullBackOff, or ContainerCreating, with copy-paste commands and plain-English explanations."
---


## Is this the right page?

You're here because `kubectl get pods` shows one of these, and the container inside has never successfully started:

- `Pending`
- `ImagePullBackOff` or `ErrImagePull`
- `ContainerCreating` stuck for more than a minute or two
- `Init:Error` or `Init:CrashLoopBackOff` (an init container, one that runs before your main container, is failing)

If instead your pod *did* start successfully at least once and is now restarting, go to [CrashLoopBackOff & Restarts](/kubernetes/troubleshooting-crashloop-and-restarts) instead, that's a different failure mode with a different fix. If you haven't run the baseline cluster-health checks yet, start at [Operations & Troubleshooting: Start Here](/kubernetes/operations-troubleshooting-start-here).

## Why this matters: two different failure points

A pod goes through two distinct phases before your application code runs: first the **scheduler** has to decide which node it runs on, then the **kubelet** on that node has to actually pull the image and create the container. Almost every "won't start" problem is one of these two phases failing, and the fix is completely different depending on which one it is. `Pending` with no node assigned yet means phase one failed. `ImagePullBackOff`/`ContainerCreating` means phase one succeeded and phase two is failing. Telling these apart in the first 30 seconds saves you from investigating the wrong layer entirely.

```bash
kubectl get pod <pod> -n <namespace> -o wide
```

Run this first. Look at the `NODE` column: if it shows `<none>`, you're in phase one (scheduling), skip to [Pod is stuck in `Pending`](#pod-is-stuck-in-pending-scheduling-failed) below. If a node name is shown, scheduling succeeded and you're in phase two, skip to [Pod shows `ImagePullBackOff`, `ErrImagePull`, or stuck `ContainerCreating`](#pod-shows-imagepullbackoff-errimagepull-or-stuck-containercreating).

## Pod is stuck in `Pending` (scheduling failed)

`Pending` means the Kubernetes scheduler looked at every node in the cluster and couldn't find one that satisfies this pod's requirements. The pod object exists, but it has nowhere to run yet.

### Step 1: read the actual reason

```bash
kubectl describe pod <pod> -n <namespace>
```

`describe` prints a full human-readable dump of the pod, but for a `Pending` pod the part you want is at the very bottom, the `Events` section. Look for a line with `Reason: FailedScheduling`. Unlike most Kubernetes errors, this message is specific and literal, it tells you exactly which constraint every node failed to satisfy. Read the whole message before doing anything else; guessing at this stage wastes time. You'll see one of the patterns below.

### Pattern: `Insufficient cpu` or `Insufficient memory`

Example message: `0/3 nodes are available: 3 Insufficient memory.`

This means every node's *unclaimed* capacity (total capacity minus what's already requested by other pods, not what's actually in use) is smaller than what this pod is asking for. This is a capacity problem, not a bug in your pod spec.

```bash
# See how much of each node's capacity is already spoken for
kubectl describe nodes | grep -A5 "Allocated resources"

# See exactly what this pod is requesting
kubectl get pod <pod> -n <namespace> -o jsonpath='{.spec.containers[*].resources}'
```

The first command shows, per node, how much CPU/memory is already requested by existing pods as a percentage of the node's total. If every node is near 100% allocated, the cluster genuinely needs more capacity (more nodes, or bigger nodes), or you need to reduce this pod's requested resources, or reduce/evict lower-priority pods elsewhere. If the numbers look like there should be room, double check the second command's output against what you expect, a resource request that's an accidental 100x too large (e.g. a typo like `500Gi` instead of `500Mi`) is a very common cause of this exact message.

### Pattern: node(s) had taint that the pod didn't tolerate

Example message: `0/3 nodes are available: 3 node(s) had taint {dedicated: gpu}, that the pod didn't tolerate.`

A **taint** is a marker an operator puts on a node to repel pods, unless the pod explicitly declares it "tolerates" that taint. This is intentional, common patterns are dedicating nodes to GPU workloads or keeping certain nodes free for a specific team, so this usually isn't a bug, it's a cluster policy your pod isn't configured to comply with.

```bash
# List taints on every node
kubectl get nodes -o json | jq '.items[] | {name: .metadata.name, taints: .spec.taints}'

# List what this pod tolerates
kubectl get pod <pod> -n <namespace> -o jsonpath='{.spec.tolerations}'
```

Compare the two outputs. If you don't recognize why a taint exists, ask whoever manages the cluster before adding a toleration, tolerations bypass an intentional guardrail, so the fix is usually "deploy to the right node pool," not "add a toleration to force it onto this one."

### Pattern: node(s) didn't match Pod's node affinity/selector

Example message: `0/3 nodes are available: 3 node(s) didn't match Pod's node affinity/selector.`

Your pod spec is asking to run only on nodes with specific labels (via `nodeSelector` or `affinity`), and no node currently has that label.

```bash
# What labels is this pod demanding?
kubectl get pod <pod> -n <namespace> -o jsonpath='{.spec.nodeSelector}'
kubectl get pod <pod> -n <namespace> -o jsonpath='{.spec.affinity}' | jq .

# What labels do nodes actually have?
kubectl get nodes --show-labels
```

Either the label on the pod spec has a typo (e.g. `disktype: ssd` vs `disk-type: ssd`), or the node that should have this label was never labeled, or that node pool doesn't exist in this cluster/environment. Fix whichever is actually wrong; don't just delete the constraint unless you're sure it's safe to run this pod anywhere.

### Pattern: pod has unbound immediate PersistentVolumeClaims

Example message: `0/3 nodes are available: 3 pod has unbound immediate PersistentVolumeClaims.`

Your pod references a PersistentVolumeClaim (a request for durable storage) that hasn't been fulfilled yet. Kubernetes won't schedule the pod until its storage is ready, since some volume types are tied to a specific node's availability zone.

```bash
kubectl get pvc -n <namespace>
```

Check the `STATUS` column for the claim this pod uses. This is common enough to warrant its own dedicated page, full diagnosis steps are in [Storage, ConfigMaps & Secrets](/kubernetes/troubleshooting-storage-and-config#persistentvolumeclaim-stuck-pending).

## Pod shows `ImagePullBackOff`, `ErrImagePull`, or stuck `ContainerCreating`

If you got here, the pod *has* been assigned to a node (the `NODE` column in `kubectl get pod -o wide` shows a real node name), so scheduling isn't the problem. The kubelet (the agent running on that node) is now trying to actually pull the container image and start it, and failing.

### Step 1: read the exact error

```bash
kubectl describe pod <pod> -n <namespace>
```

Again, look at the `Events` section at the bottom. `ErrImagePull` is the first failed attempt; `ImagePullBackOff` means Kubernetes has failed enough times that it's now waiting longer between retries (exponential backoff) before trying again, both indicate the exact same underlying problem, just at different points in the retry cycle. The event message itself tells you which of the three causes below you're dealing with, read it literally.

### Cause: wrong image name or tag

Message contains something like `manifest unknown` or `repository does not exist`.

```bash
# Confirm exactly what image reference this pod was configured with
kubectl get pod <pod> -n <namespace> -o jsonpath='{.spec.containers[*].image}{"\n"}'
```

Compare this character-for-character against an image you know exists (check your CI/CD pipeline logs, or your container registry's UI directly). A wrong tag (`v1.2` instead of `v1.2.0`) or a typo'd repository path is the single most common cause of this whole failure category. Fix the image reference in your Deployment/manifest and reapply.

### Cause: private registry authentication failure

Message contains something like `unauthorized`, `pull access denied`, or `401 Unauthorized`.

```bash
# Check whether a pull secret is attached to the pod at all
kubectl get pod <pod> -n <namespace> -o jsonpath='{.spec.imagePullSecrets}'

# If one is attached, inspect its actual decoded content
kubectl get secret <pull-secret-name> -n <namespace> -o jsonpath='{.data.\.dockerconfigjson}' | base64 -d
```

If the first command prints nothing, no pull secret is attached at all, either add one to the pod spec or to the pod's ServiceAccount (`kubectl get serviceaccount <name> -n <namespace> -o yaml`, look for `imagePullSecrets`) so every pod using that ServiceAccount picks it up automatically. If a secret is attached, the decoded output from the second command should contain valid credentials for the exact registry host in your image reference, a credential for `docker.io` won't authenticate against `ghcr.io` or a private ECR/GCR/ACR registry, the hostnames must match.

### Cause: network egress to the registry is blocked

Message is a raw connection timeout or `dial tcp: i/o timeout` rather than an authentication or "not found" error.

```bash
# Run a throwaway pod on the same node/namespace and try to reach the registry directly
kubectl run debug-pull --rm -it --image=<same-image-reference> --restart=Never -n <namespace> -- sh
```

If this throwaway pod also hangs trying to pull, the problem is network-level: a firewall, egress proxy, or an air-gapped cluster that can't reach the public internet (or your private registry's internal address). This is usually a cluster-infrastructure issue to escalate to whoever manages network policy or the cluster's egress rules, rather than something fixable from the application side.

### Init container is failing instead of the main container

If `kubectl get pods` shows `Init:Error`, `Init:CrashLoopBackOff`, or a number like `Init:1/2`, an **init container** (a container that must run to completion before your main application container starts, commonly used for waiting on a dependency or running a migration) is the thing actually failing, not your application.

```bash
# List init containers and check their individual status
kubectl get pod <pod> -n <namespace> -o jsonpath='{.spec.initContainers[*].name}'

# Get logs from a SPECIFIC init container by name
kubectl logs <pod> -n <namespace> -c <init-container-name>
```

Everything in this page's `ImagePullBackOff` and image-related sections applies equally to init containers, just target the specific init container name with `-c` on your `describe`/`logs` commands instead of assuming the failure is in your main container.

## Quick reference: symptom to cause

| What `kubectl get pods` shows | `NODE` column | Root cause category |
|---|---|---|
| `Pending` | `<none>` | Scheduler can't place the pod: capacity, taints, affinity, or unbound PVC |
| `ImagePullBackOff` / `ErrImagePull` | a real node | kubelet can't pull the image: wrong reference, auth, or network |
| `ContainerCreating` stuck for minutes | a real node | Usually a slow/stuck volume mount, check PVC/PV status next |
| `Init:Error` / `Init:CrashLoopBackOff` | a real node | An init container is failing, diagnose it directly with `-c <init-container-name>` |

## Checkpoint

- [ ] I checked the `NODE` column first to tell a scheduling failure apart from an image/runtime failure.
- [ ] I read the literal `FailedScheduling` or image-pull event message instead of guessing at the cause.
- [ ] I know the difference between an unclaimed-capacity problem and a taint/affinity mismatch, and checked the right one.
- [ ] I confirmed the exact image reference and any pull secret's registry host match what the pod actually needs.
- [ ] If an init container was involved, I diagnosed it directly with `-c <init-container-name>` instead of assuming the main container was at fault.

**Related:** [Operations & Troubleshooting: Start Here](/kubernetes/operations-troubleshooting-start-here) · [CrashLoopBackOff & Restarts](/kubernetes/troubleshooting-crashloop-and-restarts) · [Storage, ConfigMaps & Secrets](/kubernetes/troubleshooting-storage-and-config) · [Command Cheat Sheet](/kubernetes/command-cheat-sheet)
