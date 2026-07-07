---
layout: web/course-reading
heading: "Troubleshooting: Networking, DNS & 503s"
title: "Kubernetes 503, Connection Refused, DNS Failures: Fix It Now"
subtitles:
  - "A step-by-step, symptom-first guide to diagnosing Kubernetes 503s, connection refused errors, timeouts, and DNS failures, with copy-paste commands and plain-English explanations."
permalink: /kubernetes/troubleshooting-networking-and-503s
seo:
  type: TechArticle
  date_published: false
description: "A step-by-step, symptom-first guide to diagnosing Kubernetes 503s, connection refused errors, timeouts, and DNS failures, with copy-paste commands and plain-English explanations."
---


## Is this the right page?

You're here because of one of these:

- Users or another service get `503 Service Unavailable`, `502 Bad Gateway`, or `504 Gateway Timeout`
- A service-to-service call fails with `Connection refused`, `Connection reset`, or a timeout
- `UnknownHostException`, `nslookup: NXDOMAIN`, or another DNS resolution failure
- "It works from my laptop / when I port-forward, but not from inside another pod"

If instead the pod itself is crashing or won't start at all, go to [Pod Won't Start](/kubernetes/troubleshooting-pod-not-starting) or [CrashLoopBackOff & Restarts](/kubernetes/troubleshooting-crashloop-and-restarts). This page assumes the pod is `Running` and shows as healthy in `kubectl get pods`, but traffic still isn't getting through correctly.

## Why this matters: work from the inside out

A request from one pod to another passes through several independent layers: DNS resolution, the Service object, the Endpoints list behind that Service, kube-proxy's network rules, and finally the target pod itself. Each layer can fail independently and produces a different symptom. The fastest way to find the actual break is to test each layer in order, from the pod outward, rather than guessing based on the symptom alone, several completely different root causes produce an identical-looking timeout from the caller's point of view.

## Step 1: is the target pod actually ready to receive traffic?

```bash
kubectl get pods -n <namespace> -o wide
```

Look at the `READY` column. A pod showing `Running` with `READY` at `0/1` (or similarly, not all containers ready) has **failed its readiness probe** and has been deliberately removed from receiving traffic, this is Kubernetes working as intended, not a bug, but it produces exactly the same symptom to a caller as a broken Service: nothing responds.

```bash
kubectl describe pod <pod> -n <namespace> | grep -B2 -A5 -i "readiness\|unhealthy"
```

If you see `Readiness probe failed`, the fix is to figure out why the app itself isn't passing its own health check, check application logs, dependency availability (a readiness probe that checks DB connectivity will fail cluster-wide if the database is down), and whether the probe's timeout/path is even correct:

```bash
# Confirm the probe endpoint actually responds correctly right now
kubectl exec -it <pod> -n <namespace> -- curl -sv http://localhost:<port>/<probe-path>
```

If this manual curl also fails or hangs, the problem is in the application (or a dependency it checks as part of readiness), not in Kubernetes networking, stop here and investigate the app/dependency directly rather than continuing down this networking checklist.

## Step 2: can the calling pod resolve DNS at all?

```bash
kubectl exec -it <calling-pod> -n <namespace> -- nslookup kubernetes.default
```

This tests DNS resolution against a name that always exists in every cluster, so it isolates whether DNS itself works at all, independent of your specific target service. If this fails, cluster DNS is broken cluster-wide and every service is affected, skip straight to [Cause: CoreDNS itself is unhealthy](#cause-coredns-itself-is-unhealthy) below.

If that works, resolve your actual target next:

```bash
kubectl exec -it <calling-pod> -n <namespace> -- nslookup <target-service>.<target-namespace>.svc.cluster.local
```

Three possible outcomes:

- **`NXDOMAIN` / "can't find" / no answer** → the Service name, namespace, or spelling is wrong, or the Service object doesn't exist. Go to [Cause: the Service doesn't exist or the name is wrong](#cause-the-service-doesnt-exist-or-the-name-is-wrong).
- **Resolves to an IP, but the wrong/stale one** → especially for long-running JVM processes, go to [Cause: stale cached DNS inside the calling application](#cause-stale-cached-dns-inside-the-calling-application).
- **Resolves to the correct ClusterIP** → DNS is fine, the problem is downstream. Continue to Step 3.

## Step 3: does the Service actually have healthy endpoints behind it?

```bash
kubectl get svc <target-service> -n <target-namespace>
kubectl get endpoints <target-service> -n <target-namespace>
```

A Service is only a stable name and virtual IP, it does no work itself. The **Endpoints** object is the live list of actual pod IPs currently receiving traffic for that Service. If `kubectl get endpoints` returns an empty list (`<none>`) while pods that should back this service appear healthy in `kubectl get pods`, this is the single most common root cause of "DNS resolves fine but nothing responds": **the Service's selector doesn't match any pod's labels.**

```bash
# What labels does the Service expect?
kubectl get svc <target-service> -n <target-namespace> -o jsonpath='{.spec.selector}'

# What labels do the pods actually have?
kubectl get pods -n <target-namespace> --show-labels
```

Compare these two outputs directly, character for character. A mismatch here (often introduced by a refactor that renamed a label, or a copy-pasted manifest with a stale selector) is extremely common and easy to miss by eye without comparing both sides explicitly like this.

## Step 4: can you reach the pod directly, bypassing the Service?

```bash
kubectl get pod <target-pod> -n <target-namespace> -o jsonpath='{.status.podIP}'
kubectl exec -it <calling-pod> -n <namespace> -- curl -sv --max-time 5 http://<pod-ip>:<container-port>/<health-path>
```

This bypasses DNS and the Service entirely, talking straight to the pod's own IP. If this **succeeds**, DNS and the pod are both fine, the break is specifically in the Service/Endpoints/kube-proxy layer, re-check Step 3 carefully. If this **also fails or times out**, the problem is either the pod itself (wrong port, app not actually listening, crashed silently) or a `NetworkPolicy` blocking the connection at the network layer, continue to Step 5.

## Step 5: is a NetworkPolicy silently blocking the connection?

A `NetworkPolicy` restricts which pods may talk to which other pods, independent of DNS and independent of the Service object entirely. This produces a very specific, confusing signature: DNS resolves correctly, but the TCP connection itself times out or is refused.

```bash
kubectl get networkpolicy -n <target-namespace>
kubectl describe networkpolicy <policy-name> -n <target-namespace>
```

Check three things in the output:

- **`podSelector`**: which pods this policy applies to. An empty `{}` selector means "every pod in this namespace."
- **`policyTypes`**: `Ingress`, `Egress`, or both, a policy listing only `Ingress` does not restrict outbound traffic from selected pods, and vice versa.
- **The allow rules themselves**: once a pod is selected by any `NetworkPolicy` for a direction, all traffic in that direction is denied by default except what's explicitly listed. A second, narrower policy added later for an unrelated use case can silently cut off traffic a different, older policy used to implicitly allow.

To confirm a `NetworkPolicy` is really the cause (rather than guessing), launch an unrestricted pod in the same namespace and compare:

```bash
kubectl run netshoot --rm -it --image=nicolaka/netshoot -n <target-namespace> -- bash
# inside the shell:
curl -sv --max-time 5 http://<pod-ip>:<container-port>/<health-path>
```

If `netshoot` (which typically matches no restrictive policy's selector) can reach the target but your actual calling pod can't, you've confirmed the block is policy-based. The fix is adding an explicit `ingress`/`egress` rule permitting the calling pod's namespace or labels, not removing the policy outright, someone added it for a reason.

## Cause: the Service doesn't exist or the name is wrong

```bash
kubectl get svc -n <target-namespace>
```

Check for typos in the service name, and remember the full DNS form is `<service-name>.<namespace>.svc.cluster.local`, within the same namespace you can use just `<service-name>`, but across namespaces you need at least `<service-name>.<namespace>`. A very common mistake is dropping the namespace when calling cross-namespace, or getting the namespace itself wrong.

## Cause: CoreDNS itself is unhealthy

If Step 2's baseline `nslookup kubernetes.default` failed, cluster DNS is down cluster-wide, this affects every service, not just yours.

```bash
kubectl -n kube-system get pods -l k8s-app=kube-dns
kubectl -n kube-system logs -l k8s-app=kube-dns --tail=100
kubectl -n kube-system get svc kube-dns
```

If CoreDNS pods are crash-looping, pending, or logging errors, this is a cluster-infrastructure incident, not an application bug, escalate to whoever manages cluster infrastructure immediately since it affects everything running in the cluster, not just the service you started out investigating.

## Cause: stale cached DNS inside the calling application

This specifically affects long-running JVM processes (Java/Spring Boot applications). The JVM caches successful DNS lookups **forever** by default (`networkaddress.cache.ttl=-1`), which is harmless outside Kubernetes but dangerous inside it: pod IPs churn constantly from rolling deploys and rescheduling, and a JVM that resolved a hostname once at startup will keep sending traffic to a pod IP Kubernetes tore down long ago, producing intermittent `Connection refused` that looks like a Service problem but is actually a stale in-process cache.

```bash
# Check the effective setting inside a running JVM
kubectl exec -it <calling-pod> -n <namespace> -- jcmd 1 VM.system_properties | grep networkaddress
```

If unset or `-1`, this is very likely your cause if the failure is intermittent and correlates with recent rollouts of the *target* service. Fix by setting explicitly in JVM startup flags:

```
-Dnetworkaddress.cache.ttl=30 -Dnetworkaddress.cache.negative.ttl=5
```

`networkaddress.cache.ttl` bounds how long successful lookups are cached (seconds); `networkaddress.cache.negative.ttl` bounds how long *failed* lookups are cached, a high negative TTL means a one-time transient DNS blip during startup can make a JVM refuse to even retry resolving a hostname for an extended period afterward.

## Cause: an Ingress or load balancer is returning the 503, not your pod

If the `503`/`502`/`504` is showing up specifically at the edge (an Ingress controller, API gateway, or cloud load balancer) rather than from a direct in-cluster service call, the break may be one layer further out than anything above covers.

```bash
kubectl get ingress -n <namespace>
kubectl describe ingress <ingress-name> -n <namespace>

# Check the ingress controller's own logs for the specific request
kubectl -n <ingress-controller-namespace> logs -l <ingress-controller-label-selector> --tail=200
```

A `503` from an Ingress/gateway most often means it has no healthy backend Endpoints to route to (re-check Step 3 for the Service it points at), while a `504` usually means it did route the request but the backend took too long to respond, which shifts the investigation toward the target application's own latency, see [Slow Performance & Resource Throttling](/kubernetes/troubleshooting-slow-performance-and-resources).

## Quick reference: symptom to cause

| Symptom | Most likely layer |
|---|---|
| `nslookup kubernetes.default` itself fails | CoreDNS is unhealthy cluster-wide |
| `nslookup <service>` returns `NXDOMAIN` | Service doesn't exist, or wrong name/namespace |
| DNS resolves, but `curl` to the Service times out | Empty Endpoints (selector mismatch) or a `NetworkPolicy` block |
| DNS resolves, direct pod IP also fails | Problem is in the pod itself, or a `NetworkPolicy` |
| DNS resolves, direct pod IP works, Service DNS doesn't | Service selector doesn't match pod labels |
| Works sometimes, fails intermittently after target redeploys | Stale JVM DNS cache (`networkaddress.cache.ttl`) |
| 503 at the Ingress/load balancer specifically | No healthy Endpoints behind the Service it targets |
| 504 at the Ingress/load balancer specifically | Backend is reachable but too slow, see performance page |

## Checkpoint

- [ ] I confirmed the target pod's `READY` status before assuming a pure networking problem.
- [ ] I tested DNS with a known-good name (`kubernetes.default`) first to isolate cluster-wide DNS failures.
- [ ] I compared the Service's `selector` against actual pod labels side by side instead of eyeballing them separately.
- [ ] I tested the pod's IP directly to isolate whether the break is in the Service/Endpoints layer or the pod itself.
- [ ] I checked for a `NetworkPolicy` before concluding a connection failure is unexplained.

**Related:** [Operations & Troubleshooting: Start Here](/kubernetes/operations-troubleshooting-start-here) · [CrashLoopBackOff & Restarts](/kubernetes/troubleshooting-crashloop-and-restarts) · [Slow Performance & Resource Throttling](/kubernetes/troubleshooting-slow-performance-and-resources) · [Command Cheat Sheet](/kubernetes/command-cheat-sheet)
