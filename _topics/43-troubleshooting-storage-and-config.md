---
layout: web/course-reading
heading: "Troubleshooting: Storage, ConfigMaps & Secrets"
title: "Kubernetes PVC Pending & Config Not Updating: Fix It Now"
subtitles:
  - "A step-by-step, symptom-first guide to diagnosing stuck PersistentVolumeClaims, volume mount failures, and ConfigMap or Secret changes that aren't taking effect, with copy-paste commands."
permalink: /kubernetes/troubleshooting-storage-and-config
seo:
  type: TechArticle
  date_published: false
description: "A step-by-step, symptom-first guide to diagnosing stuck PersistentVolumeClaims, volume mount failures, and ConfigMap or Secret changes that aren't taking effect, with copy-paste commands."
---


## Is this the right page?

You're here because of one of these:

- A `PersistentVolumeClaim` (PVC) is stuck in `Pending` and the pod that needs it won't start
- A pod is stuck `ContainerCreating` with a mount-related event
- You updated a ConfigMap or Secret and the application isn't picking up the new value
- A pod shows `status.reason: Evicted` rather than a normal crash

If the pod itself is crashing with an application error unrelated to storage or config, check [CrashLoopBackOff & Restarts](/kubernetes/troubleshooting-crashloop-and-restarts) first, the log-signature table there covers config-related startup failures like a missing environment variable specifically. This page covers the storage and config-propagation mechanics themselves.

## PersistentVolumeClaim stuck `Pending`

### Background: how PV, PVC, and StorageClass relate

Three objects work together to get your pod a disk: a **PersistentVolumeClaim (PVC)** is what your pod's manifest actually references, it's a namespaced request for storage ("give me 10Gi, read-write, at least this fast"). A **PersistentVolume (PV)** is the actual piece of provisioned storage backing that claim. A **StorageClass** is a template telling Kubernetes how to automatically create a new PV when a PVC asks for one, which storage backend to use, what disk type, what happens to the disk when the claim is deleted. In almost every modern cluster, a default StorageClass means you never create PVs by hand, a PVC alone is enough to trigger one.

### Step 1: read the actual reason

```bash
kubectl describe pvc <pvc-name> -n <namespace>
```

Look at the `Events` section at the bottom, same as troubleshooting a pod. A `Pending` PVC almost always falls into one of three causes:

### Cause: no StorageClass matches what was requested

```bash
kubectl get storageclass
kubectl get pvc <pvc-name> -n <namespace> -o jsonpath='{.spec.storageClassName}{"\n"}'
```

If the PVC references a StorageClass name that doesn't exist in this cluster (a common cause: a manifest copied from a different cluster/cloud provider where the class names differ), or if no `storageClassName` is set and the cluster has no default StorageClass, provisioning can never start. Compare the two outputs directly, fix the PVC's `storageClassName` to one that actually exists (`kubectl get storageclass` shows `(default)` next to the cluster's default, if any).

### Cause: the CSI provisioner controller itself is unhealthy

Dynamic provisioning is done by a controller (a CSI driver) running as pods in the cluster, commonly in `kube-system`. If that controller is down, PVCs queue up and never bind, regardless of how correct your PVC/StorageClass configuration is.

```bash
kubectl describe storageclass <storageclass-name>
# Example for AWS EBS; substitute the label matching your cluster's actual CSI driver
kubectl -n kube-system get pods -l app=ebs-csi-controller
kubectl -n kube-system logs -l app=ebs-csi-controller -c csi-provisioner --tail=100
```

If the provisioner pods aren't `Running`, or their logs show repeated errors, this is a cluster-infrastructure problem to escalate, not something fixable from the application manifest side.

### Cause: requested zone/topology can't be satisfied (cloud clusters)

In cloud-managed clusters, block storage is often zone-specific, a disk provisioned in one availability zone can only attach to nodes in that same zone. If your PVC's StorageClass has topology constraints and no node in the required zone has room to schedule the pod, the PVC can stay `Pending` indefinitely even though the StorageClass and provisioner are both healthy.

```bash
kubectl describe storageclass <storageclass-name> | grep -i "volumeBindingMode\|allowedTopologies"
kubectl get nodes --show-labels | grep -i zone
```

If `volumeBindingMode` is `WaitForFirstConsumer` (the common, recommended setting), the PV isn't actually provisioned until a pod is scheduled, so check the pod's own scheduling status too, this can look like a storage problem when it's really the same kind of scheduling constraint covered on the [Pod Won't Start](/kubernetes/troubleshooting-pod-not-starting#pod-is-stuck-in-pending-scheduling-failed) page.

## Pod stuck `ContainerCreating` with a mount failure

If the PVC itself shows `Bound` (so provisioning succeeded) but the pod still won't start, check the pod's own events for a mount-specific failure.

```bash
kubectl describe pod <pod> -n <namespace> | grep -A5 -i mount
```

### `FailedAttachVolume`

The underlying cloud API couldn't attach the disk to the node. The most common cause: the volume is still attached to a **different** node from a previous scheduling, especially after an unclean node failure or a pod being rescheduled quickly.

```bash
kubectl get pod <pod> -n <namespace> -o wide      # note the CURRENT node
kubectl get volumeattachments | grep <pv-name>
```

If `volumeattachments` shows the volume still attached to a node other than where the pod is now scheduled, the CSI driver hasn't finished detaching it from the old node yet. This is usually self-resolving but can take longer than expected after a node failure specifically, the control plane has to wait out a timeout before it can safely force-detach a volume from a node that's no longer responding, rather than doing it immediately.

### `FailedMount`

The attach succeeded, but the actual filesystem mount inside the node/container failed. This is more likely to be a real, non-self-resolving problem: filesystem permissions, filesystem corruption, or a mismatched filesystem type between what the volume was formatted with and what's being requested.

```bash
kubectl describe pod <pod> -n <namespace> | grep -A10 -i "FailedMount"
```

Read the specific error message in the event, it typically names the exact permission or filesystem issue rather than requiring further guessing.

### Multiple replicas, one PVC, and the RWO trap

If you scaled a Deployment to more than one replica and only one pod ever successfully starts while the others sit in `ContainerCreating` indefinitely, check the PVC's access mode:

```bash
kubectl get pvc <pvc-name> -n <namespace> -o jsonpath='{.spec.accessModes}{"\n"}'
```

`ReadWriteOnce` (RWO) storage, by far the most common and cheapest kind (cloud block storage like EBS/GCE PD/Azure Disk), can only be mounted read-write by a single node at a time. Trying to scale a Deployment that shares one RWO-backed PVC across multiple replicas will always produce exactly this symptom, it's not a transient failure to retry, it's a structural mismatch between the storage type and the scaling model.

Two real fixes, depending on what you actually need:
- If each replica needs its **own independent** volume (the common, better-performing pattern, e.g. each replica gets its own working directory), convert the workload to a `StatefulSet` with a `volumeClaimTemplate`, this gives each replica (`app-0`, `app-1`, `app-2`, ...) its own automatically-provisioned PVC that follows that specific replica across rescheduling.
- If replicas genuinely need concurrent write access to the **same** files, you need `ReadWriteMany` (RWX)-capable storage instead (NFS, CephFS, or a cloud file-storage service like EFS/Filestore/Azure Files), generally slower and pricier than block storage, so only reach for it when concurrent shared writes are a real requirement.

## Pod shows `status.reason: Evicted`

This is a different failure from a crash or an OOMKill, and easy to misdiagnose as one if you don't specifically check the reason field.

```bash
kubectl get pod <pod> -n <namespace> -o jsonpath='{.status.reason}{"\n"}'
```

If this prints `Evicted`, the kubelet proactively removed the pod because the **node** was under resource pressure, commonly disk pressure from ephemeral storage (the container's writable layer, `emptyDir` volumes without a size limit, or accumulated logs), not because your PVC-backed persistent storage had a problem.

```bash
# Check actual disk usage inside the pod before it was evicted (or on a currently-running replica)
kubectl exec -it <pod> -n <namespace> -- df -h
kubectl exec -it <pod> -n <namespace> -- du -sh /app/* 2>/dev/null | sort -rh | head -10

# Confirm the node itself reported disk pressure
kubectl describe node <node> | grep -A5 "ephemeral-storage\|DiskPressure"
```

The fix depends on what's actually consuming the disk: unbounded log files (add rotation or ship logs off-node instead of accumulating them), an `emptyDir` without a `sizeLimit` being filled by the application, or simply too many pods sharing too little ephemeral storage on that node, which is a capacity-planning problem for whoever manages node sizing.

## ConfigMap or Secret change isn't taking effect

This is one of the most common "config isn't working" tickets, and it almost always comes down to one specific distinction: **how the pod consumes the ConfigMap/Secret determines whether it updates automatically at all.**

### Step 1: check which mechanism this pod uses

```bash
kubectl get pod <pod> -n <namespace> -o yaml | grep -B5 -A5 "configMapKeyRef\|configMapRef\|secretKeyRef\|secretRef"
kubectl get pod <pod> -n <namespace> -o yaml | grep -B3 -A3 "configMap:\|secret:"
```

- If the ConfigMap/Secret is referenced via `env` / `envFrom` (an environment variable), go to [Env var consumption: requires a restart, always](#env-var-consumption-requires-a-restart-always).
- If it's referenced via `volumeMounts` (mounted as a file), go to [Volume mount consumption: auto-updates, with a delay](#volume-mount-consumption-auto-updates-with-a-delay).

### Env var consumption: requires a restart, always

An environment variable sourced from a ConfigMap or Secret is injected into the container's process environment exactly once, at container start. This is not a Kubernetes limitation to work around, it's how process environments work at the operating-system level: a running Linux process cannot have its environment variables changed from outside after it starts. There is no delay to wait out here, the new value will **never** appear until the pod restarts, no matter how long you wait.

```bash
# Confirm what the pod actually has right now (not what you think you deployed)
kubectl exec -it <pod> -n <namespace> -- env | sort

# Compare against the ConfigMap/Secret's current content
kubectl get configmap <cm-name> -n <namespace> -o yaml
kubectl get secret <secret-name> -n <namespace> -o jsonpath='{.data}' | jq 'map_values(@base64d)'
```

Fix by restarting the pods so they pick up the new value on next start:

```bash
kubectl rollout restart deployment/<name> -n <namespace>
```

### Volume mount consumption: auto-updates, with a delay

A ConfigMap or Secret mounted as a volume **does** update automatically inside the running container, the kubelet periodically re-syncs the mounted files from the API server's current view of the object. The default sync period is roughly 60-90 seconds (the exact interval depends on kubelet configuration and internal caching, treat it as "eventually, usually under two minutes," not a guaranteed SLA), so if you just made the change, the first troubleshooting step is simply to wait a bit longer before assuming something is broken.

```bash
# Confirm the ConfigMap's resourceVersion changed (proves your edit actually landed on the API server)
kubectl get configmap <cm-name> -n <namespace> -o jsonpath='{.metadata.resourceVersion}{"\n"}'

# Check the mounted file's modification time inside the pod
kubectl exec -it <pod> -n <namespace> -- stat /config/<filename>
kubectl exec -it <pod> -n <namespace> -- cat /config/<filename>
```

If the file's content on disk **has** updated but the application still behaves as if it hasn't, the remaining gap is on the application side: the file changing on disk does not automatically mean your application re-reads it. An app that reads its config once at startup and holds it in memory will keep using the old in-memory value indefinitely, even though the underlying file is now correct. For Spring Boot specifically, this requires either a Spring Cloud Kubernetes ConfigMap/Secret watcher, or a `@RefreshScope` bean combined with hitting the `/actuator/refresh` endpoint, to actually pick up a changed file without a full restart.

```bash
# If none of that is wired up, confirm by checking whether the app logs any config reload attempt at all
kubectl logs <pod> -n <namespace> --tail=100 | grep -iE "refresh|reload|config"
```

If nothing in the logs indicates the app ever re-reads config after startup, treat it the same as the env-var case: restart the pod to pick up the change.

### If you're using Spring Cloud Config Server or Vault instead of native ConfigMaps

Some Spring Boot applications don't consume Kubernetes ConfigMaps/Secrets directly at all, they call out to a **Spring Cloud Config Server** (backed by a Git repo) or **HashiCorp Vault** at startup. This adds a layer that can fail independently of anything covered above.

```bash
# Check startup logs for the actual config-fetch attempt
kubectl logs <pod> -n <namespace> --previous | grep -iE "Config Server|vault|Fetching config"

# Verify the config server is serving what you expect, directly
curl -s http://<config-server-host>:8888/<app-name>/<profile>
```

If the app's logs show no config-fetch attempt at all, or the direct `curl` returns unexpected values, the problem is in that separate service (reachability, wrong profile/branch requested, or a Vault authentication failure via the pod's ServiceAccount token), not in any Kubernetes-native ConfigMap or Secret object. Diagnose the config server or Vault itself as its own dependency, using [Networking, DNS & 503s](/kubernetes/troubleshooting-networking-and-503s) if it's unreachable, rather than continuing to look at ConfigMap propagation mechanics that don't apply here.

## Quick reference: symptom to cause

| Symptom | Check first |
|---|---|
| PVC stuck `Pending` | `describe pvc` events, StorageClass name match, CSI provisioner health |
| Pod `ContainerCreating`, `FailedAttachVolume` | `volumeattachments`, still attached to old node? |
| Pod `ContainerCreating`, `FailedMount` | Exact error in pod events, usually permissions/filesystem |
| Only one of several replicas starts, rest stuck | PVC access mode, likely an RWO-with-multiple-replicas conflict |
| `status.reason: Evicted` | Node `DiskPressure`, ephemeral storage usage via `df`/`du` |
| Config change not applied, env var | Always requires a pod restart, this is expected, not a bug |
| Config change not applied, volume mount | Wait ~60-90s first, then check file `mtime`, then check if the app re-reads the file at all |

## Checkpoint

- [ ] I checked whether a `Pending` PVC's `storageClassName` actually matches an existing StorageClass before assuming provisioner failure.
- [ ] I distinguished `FailedAttachVolume` (cloud-level attach problem) from `FailedMount` (filesystem-level problem).
- [ ] I checked a PVC's access mode before assuming a multi-replica mount failure was a transient bug.
- [ ] I checked `status.reason` for `Evicted` before treating a stopped pod as a crash or OOMKill.
- [ ] I identified whether a ConfigMap/Secret is consumed via env var (needs restart, always) or volume mount (auto-updates, with delay) before troubleshooting a "config not applied" report.

**Related:** [Operations & Troubleshooting: Start Here](/kubernetes/operations-troubleshooting-start-here) · [Pod Won't Start](/kubernetes/troubleshooting-pod-not-starting) · [CrashLoopBackOff & Restarts](/kubernetes/troubleshooting-crashloop-and-restarts) · [Command Cheat Sheet](/kubernetes/command-cheat-sheet)
