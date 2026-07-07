---
layout: web/course-reading
heading: "Troubleshooting: CrashLoopBackOff & Restarts"
title: "Kubernetes CrashLoopBackOff: Diagnose and Fix It Now"
subtitles:
  - "A step-by-step, symptom-first guide to diagnosing and fixing Kubernetes CrashLoopBackOff, OOMKilled pods, and restart loops, with copy-paste commands and plain-English explanations."
permalink: /kubernetes/troubleshooting-crashloop-and-restarts
seo:
  type: TechArticle
  date_published: false
description: "A step-by-step, symptom-first guide to diagnosing and fixing Kubernetes CrashLoopBackOff, OOMKilled pods, and restart loops, with copy-paste commands and plain-English explanations."
---


## Is this the right page?

You're here because `kubectl get pods` shows `CrashLoopBackOff`, or the `RESTARTS` column keeps climbing. This means the container **did** start successfully at least once, then exited, and Kubernetes is now retrying it on an increasing backoff delay. That's a different failure from a pod that never starts at all, if `kubectl get pods -o wide` shows no node assigned, go to [Pod Won't Start](/kubernetes/troubleshooting-pod-not-starting) instead.

`CrashLoopBackOff` is a **symptom, not a diagnosis**. It can mean at least four genuinely different root causes, and they require completely different fixes. Guessing costs you time; running the four steps below in order tells you which one you're dealing with in under a minute.

## Step 1: get the real reason, not just the badge

```bash
kubectl describe pod <pod> -n <namespace>
```

Scroll to the `Last State` block under the failing container, and the `Events` section at the bottom. The `Last State: Terminated` block has a `Reason` and an `Exit Code`, this is the single most important piece of information on this page, everything below branches off it.

## Step 2: get logs from the container that actually crashed

```bash
kubectl logs <pod> -n <namespace> --previous
```

This is the command people most often get wrong under pressure: plain `kubectl logs` (without `--previous`) shows logs from the **current, newly-restarted** container, which may have only been running for a few seconds and logged nothing useful yet. `--previous` shows logs from the container instance that just died, which is almost always what you actually want when investigating a crash.

## Step 3: check the restart count and trend

```bash
kubectl get pod <pod> -n <namespace>
# Or across every replica of a Deployment:
kubectl get pods -n <namespace> -l app=<your-app-label>
```

A `RESTARTS` count of 1 that hasn't grown in 20 minutes is a different situation (one transient failure, possibly already resolved) from a count climbing every 30-60 seconds (an active, ongoing crash loop). Check whether it's isolated to one replica or affecting all of them, one bad replica points at node-specific or random causes; every replica crashing simultaneously points at a bad deploy or a shared dependency being down.

## Step 4: check requested vs. actual memory limit

```bash
kubectl get pod <pod> -n <namespace> -o jsonpath='{.spec.containers[0].resources}'
```

Keep this number in mind for the OOMKilled branch below.

## Now branch on what Step 1 told you

### `Reason: OOMKilled` (exit code 137)

The kernel's cgroup OOM killer terminated your container for exceeding its memory limit. This is a resource-ceiling problem, not automatically a memory leak, don't jump straight to "there's a leak" before ruling out the simpler causes below.

```bash
# Compare the configured limit against what the JVM (or your runtime) actually thinks its ceiling is
kubectl exec -it <pod> -n <namespace> -- cat /sys/fs/cgroup/memory.max            # cgroup v2
kubectl exec -it <pod> -n <namespace> -- cat /sys/fs/cgroup/memory/memory.limit_in_bytes  # cgroup v1

# Live memory usage right now, per-container
kubectl top pod <pod> -n <namespace> --containers

# Process-level memory breakdown inside the container
kubectl exec -it <pod> -n <namespace> -- ps aux
```

Work through causes in this order, roughly most-to-least common:

1. **The container memory limit is simply too low** for what this workload actually needs. Compare the limit from Step 4 against the live usage from `kubectl top`, if usage was already close to the limit before the crash, this is a capacity-planning fix: raise the limit (and the node capacity to support it), not a code fix.
2. **A JVM or runtime that isn't aware of the container's memory ceiling.** Older JVMs, or an explicit heap flag (`-Xmx`) set higher than the container's memory limit, will happily allocate past what the container is allowed to use. If you're running a JVM-based app, `kubectl exec -it <pod> -n <namespace> -- jcmd 1 VM.flags` shows the heap flags actually in effect, compare `-Xmx` against the cgroup limit above.
3. **Off-heap memory isn't accounted for.** For JVM workloads specifically: thread stacks, metaspace, direct buffers (common with Netty/Kafka clients), and JIT-compiled code all consume memory the container limit counts against, but that a simple `-Xmx` comparison won't show. A container limit needs headroom above the heap size, not just enough for the heap alone.
4. **A genuine memory leak**: usage climbs steadily over hours/days rather than spiking suddenly. Confirm this pattern with `kubectl top pod <pod> -n <namespace> --containers` sampled every few minutes before concluding it's a leak, a one-time spike under load looks different from a steady climb.
5. **A sudden spike in concurrent requests** driving up thread count, and therefore thread-stack memory, all at once. Correlate the OOM timestamp against your traffic/request-rate dashboards if you have them.

If you've confirmed it's a genuine leak rather than a ceiling/config problem, deep heap-dump analysis is a separate, deeper investigation, that level of diagnosis is out of scope for a live-incident page like this one; for now, mitigate (raise the limit, or roll back a suspect deploy) to stop the bleeding, then schedule the deeper analysis separately.

### `Reason: Error`, exit code `1` (application error)

The process started and then exited on its own due to an application-level failure, most commonly an uncaught exception during startup.

```bash
kubectl logs <pod> -n <namespace> --previous | tail -100
```

Read for a startup failure banner or a stack trace. If the log is long, grep for the highest-signal phrases first:

```bash
kubectl logs <pod> -n <namespace> --previous | grep -iE "fatal|exception|error|failed to start|caused by|connection refused|unknownhostexception|address already in use|port already in use|no such file"
```

| Log signature | Likely cause |
|---|---|
| `Connection refused` to a dependency's host/port | The dependency pod isn't ready, the wrong port was configured, or a Service selector doesn't match any pod, see [Networking, DNS & 503s](/kubernetes/troubleshooting-networking-and-503s) |
| `UnknownHostException` / `no such host` / DNS resolution failure | The service name is wrong, or cluster DNS itself is unhealthy, see [Networking, DNS & 503s](/kubernetes/troubleshooting-networking-and-503s) |
| Missing required environment variable / config value | A ConfigMap or Secret didn't get mounted or wasn't updated, see [Storage, ConfigMaps & Secrets](/kubernetes/troubleshooting-storage-and-config) |
| `Address already in use` / `Port already in use` | Two processes competing for the same port, often a leftover process from a previous crash in the same container, or a misconfigured `command`/`args` |
| Out-of-memory error thrown by the *application/runtime itself* (not the kernel) | Different from `OOMKilled` above, this is the process catching its own allocation failure before the kernel intervenes, tune runtime memory flags, e.g. metaspace/heap settings for a JVM |

### `Reason: Completed`, exit code `0`

This is worth calling out because it's easy to misread: exit code `0` means the main process **exited on purpose**, this often isn't a crash at all. If your container's entrypoint is a short-lived script rather than a long-running server process, Kubernetes will keep restarting it because a container in a Deployment is expected to run forever, not run-once-and-exit.

```bash
kubectl get pod <pod> -n <namespace> -o jsonpath='{.spec.containers[0].command}{"\n"}{.spec.containers[0].args}{"\n"}'
```

Check whether the configured `command`/`args` actually launches a persistent server process. If it's meant to be a one-off job instead of a long-running service, it should be a Kubernetes `Job`, not a `Deployment`.

### No useful `Reason`, or `kubectl logs` is empty

The application may be crashing before its logging framework even initializes, or it's writing logs somewhere `kubectl logs` doesn't look (a file inside the container, instead of stdout/stderr).

```bash
# Confirm the real entrypoint/command
kubectl get pod <pod> -n <namespace> -o jsonpath='{.spec.containers[0].command}{"\n"}{.spec.containers[0].args}{"\n"}'

# Check whether the app is configured to log to a file instead of the console
kubectl exec -it <pod> -n <namespace> -- ls -la /app/logs 2>/dev/null

# Get an interactive shell inside a copy of this pod's environment to poke around before the next crash
kubectl debug -it <pod> -n <namespace> --image=busybox --target=<container-name>
```

If logs are genuinely going to a file instead of stdout/stderr, that's an application-level anti-pattern in a containerized environment, `kubectl logs` will always come up empty regardless of what's happening inside. The fix is to reconfigure the app's logging to write to the console; you can confirm this diagnosis by execing in and checking the file path your logging config points to before the next crash happens.

### It's actually a failing liveness probe, not a real crash

If `Events` shows `Liveness probe failed` or `Unhealthy` immediately before the restart, Kubernetes itself is killing a container that was otherwise fine, because it failed a health check.

```bash
kubectl describe pod <pod> -n <namespace> | grep -B2 -A5 -i "unhealthy\|liveness"

# Confirm the probe endpoint actually responds correctly from inside the pod right now
kubectl exec -it <pod> -n <namespace> -- curl -sv http://localhost:<port>/<probe-path>

# Compare probe timing against how long the app actually takes to become ready
kubectl get pod <pod> -n <namespace> -o yaml | grep -A8 "livenessProbe:\|readinessProbe:\|startupProbe:"
```

The classic version of this: an application that takes 45 seconds to fully start (large dependency injection context, database migrations on boot, etc.) but a `livenessProbe` with `initialDelaySeconds: 15`, the probe starts checking before the app is ready, fails repeatedly, and kubelet kills it, forever, before it ever gets a chance to finish starting. The fix is a `startupProbe` with a generous enough `failureThreshold * periodSeconds` to cover real startup time, letting liveness/readiness only take over once startup has actually succeeded once.

## Quick reference: exit code to likely cause

| Exit code | Meaning | Check first |
|---|---|---|
| `0` | Clean, intentional exit | Is this container meant to be long-running? Check `command`/`args` |
| `1` | Generic application error | `kubectl logs --previous`, grep for exceptions/stack traces |
| `137` | `SIGKILL` (128+9) | `Reason: OOMKilled`? Check memory limit vs actual usage |
| `143` | `SIGTERM` (128+15) | Graceful shutdown, if unexpected check `terminationGracePeriodSeconds` and whether the app handled the signal in time |

## Checkpoint

- [ ] I read `Last State: Reason` and the exact exit code before forming a theory.
- [ ] I used `--previous` to get logs from the container that actually crashed, not the newly restarted one.
- [ ] I ruled out a probe-induced kill (check `Events` for `Unhealthy`) before assuming it's a real application crash.
- [ ] For an OOMKilled pod, I compared the configured memory limit against live usage before assuming a leak.
- [ ] I know what to check when `kubectl logs` shows nothing useful at all.

**Related:** [Operations & Troubleshooting: Start Here](/kubernetes/operations-troubleshooting-start-here) · [Pod Won't Start](/kubernetes/troubleshooting-pod-not-starting) · [Networking, DNS & 503s](/kubernetes/troubleshooting-networking-and-503s) · [Storage, ConfigMaps & Secrets](/kubernetes/troubleshooting-storage-and-config) · [Command Cheat Sheet](/kubernetes/command-cheat-sheet)
