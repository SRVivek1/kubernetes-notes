---
layout: lesson
title: "Prerequisites"
seo_title: "Kubernetes Course Prerequisites for Java & Spring Boot Developers"
description: "What you need to know before starting this Kubernetes troubleshooting course: Docker basics, Java/Spring Boot development, and lab environment setup."
permalink: /course/getting-started/prerequisites/
level: getting-started
module: 0
lesson: 2
order: 2
prerequisites:
  - welcome
keywords: [kubernetes prerequisites, kind, minikube, kubectl setup, java kubernetes lab]
tags: [getting-started, setup]
estimated_minutes: 10
last_modified: 2026-07-06
---

## Knowledge prerequisites

| Area | Required level | Not required |
|---|---|---|
| Java | Comfortable writing and running a Java application | JVM internals — that's taught in this course |
| Spring Boot | Can build a REST controller, wire a `DataSource`, use `application.yml` | Spring Cloud, reactive stack — helpful but not required |
| Docker | Can write a `Dockerfile`, build and run a container image, understand what a container is | Docker Compose, Swarm |
| Linux shell | Comfortable with a terminal, basic commands (`cd`, `grep`, `cat`) | Shell scripting |
| Kubernetes | **None** — this course starts from zero | — |

## Lab environment setup

You need a local Kubernetes cluster to run every lab in this course. Either of these works identically for all lessons through the Advanced level:

```bash
# Option 1: kind (Kubernetes in Docker) — recommended, lightweight, fast to reset
brew install kind kubectl        # macOS; see kind.sigs.k8s.io for other platforms
kind create cluster --name course

# Option 2: minikube
brew install minikube kubectl
minikube start
```

Verify it works:

```bash
kubectl cluster-info
kubectl get nodes
```

You should see one `Ready` node. If you don't, stop here and fix your cluster setup before continuing — every later lesson assumes this works.

### Tools you'll install as you go

The course introduces tools progressively, at the level where they first become necessary — you don't need any of these on day one:

| Tool | First needed at |
|---|---|
| `jq` | Intermediate |
| `k9s` (optional TUI) | Beginner, optional |
| `nicolaka/netshoot` (container image, no install) | Intermediate |
| Eclipse MAT / VisualVM | Advanced |
| `async-profiler` | Advanced |
| `istioctl` | Advanced |
| Cloud CLI (`aws`/`gcloud`/`az`) | Expert, only if you use a managed cluster for that module |
| Chaos Mesh or Litmus | Expert |

## A note on the Expert level

Levels Beginner through Advanced are fully achievable on a local `kind`/`minikube` cluster. The Expert level's cloud-managed and multi-cluster lessons are easier with access to a real EKS/GKE/AKS cluster — if you don't have one, read those lessons for the mental model and command shapes even if you can't run every command against a live cloud account.

## Checkpoint

- [ ] I have a local Kubernetes cluster running (`kubectl get nodes` shows `Ready`).
- [ ] I know which tools I'll need later, so nothing in a later lesson surprises me.

**Next:** [Level 1 — Kubernetes Architecture Fundamentals →](../01-beginner/01-kubernetes-architecture-fundamentals.md)
