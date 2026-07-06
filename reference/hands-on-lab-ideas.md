---
layout: reference
title: "Hands-on Lab Ideas: Kubernetes Fault Injection Recipes"
seo_title: "Kubernetes Fault Injection Lab Ideas for Chaos Testing"
description: "Reusable Kubernetes fault-injection recipes for building a chaos lab — inject real failures across pods, JVM, networking, storage, and RBAC."
permalink: /reference/hands-on-lab-ideas/
keywords: [kubernetes chaos engineering, fault injection, kubernetes lab ideas, chaos lab, kubernetes practice labs, jvm fault injection]
tags: [reference, labs, chaos-engineering, practice]
last_modified: 2026-07-06
---

Reach for this page once you've finished the per-module labs and want more reps, or when you're designing your own capstone or game-day scenario. Each row is a reusable fault-injection recipe you can build into a private "chaos lab" repo — inject the fault, then diagnose it using nothing but the tools taught in the linked course lesson.

## Fault-injection recipes

| # | Fault to inject | Layer | Expected learner action | Course lesson |
|---|---|---|---|---|
| 1 | Wrong image tag in Deployment | Pod | Diagnose `ErrImagePull` via `describe`/events | [Reading Pod Status and Logs](/course/beginner/reading-pod-status-and-logs/) |
| 2 | `-Xmx` set above container memory limit | JVM | Correlate `OOMKilled` with `jcmd VM.flags` | [CrashLoopBackOff and Exit Code Deep Dive](/course/intermediate/crashloopbackoff-and-exit-codes/), [Heap Dumps and Memory Leak Hunting](/course/advanced/heap-dumps-and-memory-leaks/) |
| 3 | Readiness probe path typo (`/actuator/helth`) | Probe | Diagnose via `describe pod` Unhealthy events | [CrashLoopBackOff and Exit Code Deep Dive: liveness/readiness probe-induced restarts](/course/intermediate/crashloopbackoff-and-exit-codes/) |
| 4 | Service selector label mismatch | Networking | Diagnose via empty `kubectl get endpoints` | [DNS and Service Discovery Deep Dive](/course/intermediate/dns-and-service-discovery-deep-dive/) |
| 5 | NetworkPolicy default-deny without explicit allow | Networking | Diagnose via `netshoot` connectivity test | [DNS and Service Discovery Deep Dive: NetworkPolicy blocking traffic](/course/intermediate/dns-and-service-discovery-deep-dive/) |
| 6 | ConfigMap key renamed without updating Deployment env mapping | Config | Diagnose via `env` inside pod vs ConfigMap content | [ConfigMap and Secret Propagation](/course/intermediate/configmap-secret-propagation/) |
| 7 | HikariCP pool size > DB max connections | JVM/DB | Diagnose via thread dump showing waiting threads | [Thread Dumps and Deadlock Analysis](/course/advanced/thread-dumps-and-deadlock-analysis/) |
| 8 | Infinite loop / unbounded cache causing heap growth | JVM | Capture and analyze heap dump, find leak suspect | [Heap Dumps and Memory Leak Hunting](/course/advanced/heap-dumps-and-memory-leaks/) |
| 9 | CPU limit too low under load | Resource | Diagnose via `cpu.stat` `nr_throttled` vs GC logs | [GC Tuning and CPU Throttling in Containers](/course/advanced/gc-tuning-and-cpu-throttling/) |
| 10 | Istio DestinationRule circuit breaker with low `maxRequestsPerConnection` | Mesh | Diagnose via Envoy access log flags | [Service Mesh Troubleshooting with Istio](/course/advanced/service-mesh-troubleshooting-istio/) |
| 11 | PVC storage class typo | Storage | Diagnose via `Pending` PVC + `describe` events | [Persistent Storage for Stateful Workloads](/course/intermediate/persistent-storage-for-stateful-workloads/) |
| 12 | IRSA role ARN missing a trust relationship | Cloud/IAM | Diagnose via AWS SDK `AccessDenied` + SA annotation check | [Cloud-Managed Clusters: EKS, GKE, and AKS](/course/expert/cloud-managed-clusters-eks-gke-aks/) |
| 13 | PodDisruptionBudget blocking node drain | Cluster ops | Diagnose stuck `kubectl drain` | [Observability: Metrics, Logs, Traces, and Autoscaling](/course/advanced/observability-metrics-logs-traces/) (PDB), [Node and Control Plane Internals](/course/expert/node-and-control-plane-internals/) (drain mechanics) |
| 14 | Admission webhook backend pod down | Cluster ops | Diagnose cluster-wide resource creation failures | [Namespaces, RBAC, and Multi-Tenancy](/course/intermediate/namespaces-rbac-and-multi-tenancy/) |
| 15 | SIGTERM not handled gracefully, causing dropped requests on rollout | App lifecycle | Diagnose via error spike during `rollout restart`, fix with `preStop` + graceful shutdown | [Actuator Runbooks and Ephemeral Debug Containers](/course/advanced/actuator-and-ephemeral-debug-containers/) |

**Related:** [Command Cheat Sheet](/reference/command-cheat-sheet/) · [Assessment Rubric](/reference/assessment-rubric/) · [Incident Runbook Template](/reference/incident-runbook-template/)
