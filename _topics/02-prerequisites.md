---
layout: web/course-reading
heading: "Prerequisites"
title: "Kubernetes Course Prerequisites for Java & Spring Boot Developers"
subtitles:
  - "What you need to know before starting this Kubernetes troubleshooting course: Docker basics, Java/Spring Boot development, and lab environment setup."
permalink: /kubernetes/prerequisites
seo:
  type: TechArticle
  date_published: false
description: "What you need to know before starting this Kubernetes troubleshooting course: Docker basics, Java/Spring Boot development, and lab environment setup."
---


## Knowledge prerequisites

| Area | Required level | Not required |
|---|---|---|
| Java | Comfortable writing and running a Java application | JVM internals: that's taught in this course |
| Spring Boot | Can build a REST controller, wire a `DataSource`, use `application.yml` | Spring Cloud, reactive stack: helpful but not required |
| Docker | Can write a `Dockerfile`, build and run a container image, understand what a container is | Docker Compose, Swarm |
| Terminal | Comfortable with a shell on macOS, Linux, or Windows (PowerShell or WSL): basic commands (`cd`, `grep`, `cat`) | Shell scripting |
| Kubernetes | **None**: this course starts from zero |: |

## Lab environment setup

You need a local Kubernetes cluster to run every lab in this course. Either **kind** or **minikube** works identically for all lessons through the Advanced level.

**All platforms:** Docker must be installed and running. On Windows, use [Docker Desktop](https://docs.docker.com/desktop/setup/install/windows-install/) with the WSL2 backend enabled.

{% include components/course-lab-option.html number="1" title="kind" tag="recommended" variant="primary" %}

Lightweight, fast to reset, and ideal for this course.

{% include components/course-callout.html
  type="note"
  title="Official documentation"
  content="The commands below are copied from the kind quick-start (v0.32.0 at time of writing). For the latest release, new platforms, or troubleshooting, always check the official docs."
%}

| Topic | Official docs |
|---|---|
| kind installation | [kind.sigs.k8s.io/docs/user/quick-start#installation](https://kind.sigs.k8s.io/docs/user/quick-start#installation) |
| kind releases (check current stable version) | [github.com/kubernetes-sigs/kind/releases](https://github.com/kubernetes-sigs/kind/releases) |
| kubectl installation (required companion tool) | [kubernetes.io/docs/tasks/tools/](https://kubernetes.io/docs/tasks/tools/) |

Install steps below follow the official quick-start (stable release **v0.32.0**). If a newer version is published, update the version in the download URLs or use the links above.

**macOS**

```bash
# kubectl
brew install kubectl

# kind — official release binaries
# For Intel Macs
[ $(uname -m) = x86_64 ] && curl -Lo ./kind https://kind.sigs.k8s.io/dl/v0.32.0/kind-darwin-amd64
# For Apple Silicon (M1/M2/M3)
[ $(uname -m) = arm64 ] && curl -Lo ./kind https://kind.sigs.k8s.io/dl/v0.32.0/kind-darwin-arm64
chmod +x ./kind
sudo mv ./kind /usr/local/bin/kind

kind create cluster --name course
```

```bash
# Or install with brew: 
brew install kind kubectl
```

**Linux**

```bash
# kubectl
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl

# kind — official release binaries
# For AMD64 / x86_64
[ $(uname -m) = x86_64 ] && curl -Lo ./kind https://kind.sigs.k8s.io/dl/v0.32.0/kind-linux-amd64
# For ARM64
[ $(uname -m) = aarch64 ] && curl -Lo ./kind https://kind.sigs.k8s.io/dl/v0.32.0/kind-linux-arm64
chmod +x ./kind
sudo mv ./kind /usr/local/bin/kind

kind create cluster --name course
```

**Windows (PowerShell)**

```powershell
# kubectl
winget install Kubernetes.kubectl

# kind — official release binary
curl.exe -Lo kind-windows-amd64.exe https://kind.sigs.k8s.io/dl/v0.32.0/kind-windows-amd64
Move-Item .\kind-windows-amd64.exe "$env:LOCALAPPDATA\Microsoft\WindowsApps\kind.exe"

kind create cluster --name course
```

Pick a directory already on your `PATH` if `WindowsApps` is not suitable on your machine.

```powershell
# Or: winget install Kubernetes.kind
```

{% include components/course-lab-option.html number="2" title="minikube" variant="primary" %}

**macOS**

```bash
brew install minikube kubectl
minikube start
```

**Linux**

```bash
curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64
sudo install minikube-linux-amd64 /usr/local/bin/minikube

curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl

minikube start
```

**Windows (PowerShell)**

```powershell
winget install Kubernetes.minikube Kubernetes.kubectl
# Or: choco install minikube kubernetes-cli
minikube start
```

### Verify your cluster

```bash
kubectl cluster-info
kubectl get nodes
```

You should see one `Ready` node. If you don't, stop here and fix your cluster setup before continuing, every later lesson assumes this works.

### Tools you'll install as you go

The course introduces tools progressively, at the level where they first become necessary, you don't need any of these on day one:

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

**Installation:** step-by-step setup for every tool in the table lives on [Tool Installation Guide](/kubernetes/tool-installation-guide). Return here for the "when you need it" overview.

## A note on the Expert level

Levels Beginner through Advanced are fully achievable on a local `kind`/`minikube` cluster. The Expert level's cloud-managed and multi-cluster lessons are easier with access to a real EKS/GKE/AKS cluster, if you don't have one, read those lessons for the mental model and command shapes even if you can't run every command against a live cloud account.

{% capture checkpoint_items %}
- [ ] I have a local Kubernetes cluster running (`kubectl get nodes` shows `Ready`).
- [ ] I know which tools I'll need later (see [Tool Installation Guide](/kubernetes/tool-installation-guide) when I'm ready to install them).
{% endcapture %}
{% include components/course-checkpoint.html body=checkpoint_items %}

