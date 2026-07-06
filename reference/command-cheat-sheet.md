---
layout: reference
title: "Kubernetes Command Cheat Sheet"
seo_title: "Kubernetes kubectl Command Cheat Sheet for Troubleshooting"
description: "Copy-paste kubectl commands for diagnosing pods, JVM issues, networking, storage, and rollouts — organized by troubleshooting topic."
permalink: /reference/command-cheat-sheet/
keywords: [kubectl cheat sheet, kubernetes commands, kubectl troubleshooting, jvm diagnostics commands, kubernetes networking commands, kubectl reference]
tags: [reference, kubectl, commands, cheat-sheet]
last_modified: 2026-07-06
---

No explanations here by design — this page is for people who already understand the concepts and just need copy-paste commands fast, mid-incident or mid-debug. If a command doesn't make sense, work through the matching course lesson first; each section header links to where that topic is taught.

## [Context & health](/course/beginner/reading-pod-status-and-logs/)
```bash
kubectl config get-contexts
kubectl cluster-info
kubectl get nodes -o wide
kubectl get events -A --sort-by='.lastTimestamp'
```

## [Pods](/course/beginner/reading-pod-status-and-logs/)
```bash
kubectl get pods -n <ns> -o wide
kubectl describe pod <pod> -n <ns>
kubectl logs <pod> -n <ns> [--previous] [-c <container>] [-f] [--since=10m]
kubectl exec -it <pod> -n <ns> -- sh
kubectl debug -it <pod> -n <ns> --image=busybox --target=<container>
kubectl top pod <pod> -n <ns> --containers
kubectl delete pod <pod> -n <ns>                     # force reschedule
```

## [Restart counts & reasons](/course/beginner/reading-pod-status-and-logs/)
```bash
kubectl get pods -n <ns>                                                        # RESTARTS column
kubectl get pods -n <ns> --sort-by='.status.containerStatuses[0].restartCount'  # worst offenders last
kubectl get pods -A --sort-by='.status.containerStatuses[0].restartCount' | tail -20   # cluster-wide
kubectl get pods -n <ns> -l app=<app-label>                                     # all replicas of one Deployment
kubectl describe pod <pod> -n <ns>                                              # Last State + Restart Count + Events
kubectl get pod <pod> -n <ns> -o jsonpath='{.status.containerStatuses[*].restartCount}'
kubectl get pod <pod> -n <ns> -o json | jq '.status.containerStatuses[] | {name, restartCount, lastState, state}'
kubectl get events -n <ns> --field-selector reason=BackOff --sort-by='.lastTimestamp'
kubectl get events -n <ns> --field-selector involvedObject.name=<pod> --sort-by='.lastTimestamp'
```

## [JVM inside pod](/course/advanced/heap-dumps-and-memory-leaks/)
```bash
jcmd 1 VM.flags
jcmd 1 Thread.print
jcmd 1 GC.heap_dump /tmp/heap.hprof
jcmd 1 GC.class_histogram
jstat -gcutil 1 1000 10
kubectl cp <ns>/<pod>:/tmp/heap.hprof ./heap.hprof
```

## [Networking](/course/intermediate/dns-and-service-discovery-deep-dive/)
```bash
kubectl get svc,endpoints -n <ns>
kubectl exec -it <pod> -n <ns> -- nslookup <svc>.<ns>.svc.cluster.local
kubectl run netshoot --rm -it --image=nicolaka/netshoot -- bash
kubectl port-forward svc/<svc> -n <ns> 8080:80
```

## [Storage](/course/intermediate/persistent-storage-for-stateful-workloads/)
```bash
kubectl get pv,pvc -n <ns>
kubectl describe pvc <pvc> -n <ns>
kubectl exec -it <pod> -n <ns> -- df -h
```

## [Config/Secrets](/course/intermediate/configmap-secret-propagation/)
```bash
kubectl exec -it <pod> -n <ns> -- env
kubectl get configmap <cm> -n <ns> -o yaml
kubectl get secret <secret> -n <ns> -o jsonpath='{.data}' | jq 'map_values(@base64d)'
```

## [Scaling/Resources](/course/advanced/observability-metrics-logs-traces/)
```bash
kubectl get hpa -n <ns>
kubectl top pods -n <ns>
kubectl describe resourcequota -n <ns>
```

## [Rollouts](/course/expert/gitops-progressive-delivery-and-rollback/)
```bash
kubectl rollout status deployment/<d> -n <ns>
kubectl rollout undo deployment/<d> -n <ns>
kubectl rollout history deployment/<d> -n <ns>
```

## [RBAC](/course/intermediate/namespaces-rbac-and-multi-tenancy/)
```bash
kubectl auth can-i <verb> <resource> -n <ns> --as=<user-or-sa>
```

## [Node/cluster](/course/expert/node-and-control-plane-internals/)
```bash
kubectl describe node <node>
kubectl drain <node> --ignore-daemonsets --delete-emptydir-data
kubectl cordon/uncordon <node>
```

**Related:** [Incident Runbook Template](/reference/incident-runbook-template/) · [Hands-on Lab Ideas](/reference/hands-on-lab-ideas/) · [Assessment Rubric](/reference/assessment-rubric/)
