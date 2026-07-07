---
layout: web/course-reading
heading: "Tool Installation Guide"
title: "Kubernetes Course Tool Installation Guide for macOS, Linux, and Windows"
subtitles:
  - "Step-by-step setup for jq, k9s, async-profiler, Istio, JVM analysis tools, cloud CLIs, and chaos engineering platforms used in this course."
permalink: /kubernetes/tool-installation-guide
seo:
  type: TechArticle
  date_published: false
description: "Install jq, k9s, async-profiler, istioctl, Eclipse MAT, VisualVM, cloud CLIs, and Chaos Mesh or Litmus for the Kubernetes troubleshooting course on macOS, Linux, and Windows."
---

This page helps you decide **what** to install, **when**, and **why**, without repeating the full cluster setup on [Prerequisites](/kubernetes/prerequisites). Each tool below is introduced progressively in the course: you do not need any of them on day one.

Scan the summary table for your current level, open only the sections that apply to you, and install when you reach the linked lesson. Skip optional and Expert-only tools if you are reading for concepts only and will not run those labs hands-on. Cluster setup (Docker, kind or minikube, kubectl) stays on the Prerequisites page.

{% include components/course-callout.html
  type="note"
  title="Official documentation"
  content="Commands below are copied from each tool's official install docs at time of writing. For the latest release, new platforms, or troubleshooting, always check the official links in each section."
%}

---

## At a glance

| Tool | What it does | First needed | Required? |
|---|---|---|---|
| `jq` | Filter and format JSON from `kubectl` output | Intermediate | Yes, for those labs |
| `k9s` | Terminal UI for browsing pods, logs, and events | Beginner | No (optional) |
| `nicolaka/netshoot` | Networking toolbox pod for DNS/connectivity tests | Intermediate | Yes (pulled on demand, no host install) |
| Eclipse MAT | Deep heap dump analysis (leak suspects, dominator tree) | Advanced | Yes, for heap labs |
| VisualVM | Lighter JVM/heap browsing | Advanced | Optional (MAT is primary) |
| `async-profiler` | CPU flame graphs from a live JVM in a pod | Advanced | Yes, for profiling lab |
| `istioctl` | Istio control-plane and sidecar diagnostics | Advanced | Yes, for mesh lab hands-on |
| Cloud CLIs | Query EKS/GKE/AKS logs, IAM, and cluster config | Expert | Only if using that cloud |
| Chaos Mesh / Litmus | Controlled failure injection in the cluster | Expert | Pick one, if running chaos labs |

---

## Table of Contents

- [At a glance](#at-a-glance)
- [jq](#jq)
- [k9s (optional)](#k9s-optional)
- [nicolaka/netshoot (container image)](#nicolakanetshoot-container-image)
- [Eclipse MAT](#eclipse-mat)
- [VisualVM](#visualvm)
- [async-profiler](#async-profiler)
- [istioctl](#istioctl)
- [Cloud CLIs (Expert, optional)](#cloud-clis-expert-optional)
- [Chaos Mesh or Litmus (Expert)](#chaos-mesh-or-litmus-expert)

---

## jq

| | |
|---|---|
| **First needed at** | Intermediate |
| **First lesson** | [Restart Troubleshooting](/kubernetes/restart-troubleshooting-across-a-deployment) |

**What it does:** Parses `kubectl ... -o json` output so you can filter restart counts, decode secret fields, and inspect pod state without scrolling through raw JSON.

**Do you need it?** Install before Intermediate. Every restart-troubleshooting example and cheat-sheet `jq` one-liner assumes it is on your PATH.

| Topic | Official docs |
|---|---|
| jq downloads | [jqlang.org/download](https://jqlang.org/download/) |
| jq manual | [jqlang.org/manual](https://jqlang.org/manual/) |

**macOS**

```bash
brew install jq
```

**Linux**

```bash
# Debian / Ubuntu
sudo apt-get update && sudo apt-get install -y jq

# Fedora / RHEL
sudo dnf install -y jq
```

**Windows (PowerShell)**

```powershell
winget install jqlang.jq
# Or: choco install jq
```

**Verify**

```bash
jq --version
```

---

## k9s (optional)

| | |
|---|---|
| **First needed at** | Beginner (optional TUI) |
| **First lesson** | [Reading Pod Status & Logs](/kubernetes/reading-pod-status-and-logs) |

**What it does:** Keyboard-driven cluster browser for pods, logs, describe, and port-forwards in one terminal UI.

**Do you need it?** Optional from Beginner onward. Skip if you are comfortable with `kubectl`; every lab in this course works without it.

| Topic | Official docs |
|---|---|
| k9s installation | [k9scli.io/topics/install](https://k9scli.io/topics/install/) |
| k9s releases | [github.com/derailed/k9s/releases](https://github.com/derailed/k9s/releases) |

**macOS**

```bash
brew install derailed/k9s/k9s
```

**Linux**

```bash
# Binary install (amd64 example; check releases for arm64)
curl -sS https://webinstall.dev/k9s | bash
# Or download from GitHub releases and move to /usr/local/bin/k9s
```

**Windows (PowerShell)**

```powershell
winget install derailed.k9s
# Or: choco install k9s
```

**Verify**

```bash
k9s version
```

---

## nicolaka/netshoot (container image)

| | |
|---|---|
| **First needed at** | Intermediate |
| **First lesson** | [DNS & Service Discovery](/kubernetes/dns-and-service-discovery-deep-dive) |

**What it does:** Temporary debug pod with `dig`, `curl`, `tcpdump`, and other networking utilities to test DNS and connectivity from inside the cluster.

**Do you need it?** Required for Intermediate networking labs. No workstation install: Kubernetes pulls the image when you run it.

| Topic | Official docs |
|---|---|
| netshoot image | [github.com/nicolaka/netshoot](https://github.com/nicolaka/netshoot) |

**Run a one-off debug shell**

```bash
kubectl run netshoot --rm -it --image=nicolaka/netshoot -- bash
```

**Run in a namespace (matches course labs)**

```bash
kubectl run netshoot --rm -it --image=nicolaka/netshoot -n <ns> -- bash
```

**Verify** (inside the pod)

```bash
curl --version
dig -v
```

---

## Eclipse MAT

| | |
|---|---|
| **First needed at** | Advanced |
| **First lesson** | [Heap Dumps & Memory Leaks](/kubernetes/heap-dumps-and-memory-leaks) |

**What it does:** Opens `.hprof` heap dumps offline. Leak Suspects and the dominator tree pin down which objects are retaining memory and why they were not garbage collected.

**Do you need it?** Required for Advanced heap-dump labs. Runs on your laptop, not inside the cluster.

| Topic | Official docs |
|---|---|
| MAT downloads | [eclipse.org/mat/downloads.php](https://www.eclipse.org/mat/downloads.php) |

**All platforms**

1. Download the MAT archive for your OS from the link above (standalone RCP package).
2. Unpack the archive.
3. Run `MemoryAnalyzer` (macOS/Linux) or `MemoryAnalyzer.exe` (Windows).

MAT requires a JDK. If the MAT launcher cannot find one, set `JAVA_HOME` to a JDK 17+ install before starting.

**Verify**

Open MAT and confirm the splash screen loads. No CLI version check is required.

---

## VisualVM

| | |
|---|---|
| **First needed at** | Advanced |
| **First lesson** | [Heap Dumps & Memory Leaks](/kubernetes/heap-dumps-and-memory-leaks) |

**What it does:** Quick JVM heap histograms and lighter profiling without MAT's full Leak Suspects and dominator-tree workflow.

**Do you need it?** Optional. Install if you want a simpler first look at heap data; MAT covers the capstone analysis path in the course.

| Topic | Official docs |
|---|---|
| VisualVM downloads | [visualvm.github.io/download.html](https://visualvm.github.io/download.html) |

**macOS**

```bash
brew install --cask visualvm
```

**Linux**

Download the `.zip` from the official site, unpack, and run `bin/visualvm`.

**Windows**

Download the `.exe` installer from the official site and run it.

**Verify**

Start VisualVM and confirm the main window opens.

---

## async-profiler

| | |
|---|---|
| **First needed at** | Advanced |
| **First lesson** | [CPU Profiling with async-profiler](/kubernetes/cpu-profiling-with-async-profiler) |

**What it does:** Samples the live JVM and produces CPU flame graphs to find which methods are burning time under load.

**Do you need it?** Required for the Advanced CPU profiling lesson. Download once on your workstation, then `kubectl cp` the tarball into the pod.

| Topic | Official docs |
|---|---|
| async-profiler releases | [github.com/async-profiler/async-profiler/releases](https://github.com/async-profiler/async-profiler/releases) |

**macOS / Linux (download on workstation)**

```bash
# Example: linux x64 (most course lab containers)
curl -sL -o async-profiler.tar.gz \
  https://github.com/async-profiler/async-profiler/releases/download/v3.0/async-profiler-3.0-linux-x64.tar.gz
tar xzf async-profiler.tar.gz
ls async-profiler-*/profiler.sh
```

Match the archive to the JVM's OS and architecture inside the container. Check before copying:

```bash
kubectl exec -it <pod> -n <ns> -- uname -m
```

**Windows (PowerShell)**

Download the matching release `.zip` or `.tar.gz` from GitHub releases in a browser, then use `kubectl cp` the same way as on macOS/Linux.

**Verify** (after copying into a pod)

```bash
kubectl exec -it <pod> -n <ns> -- /tmp/async-profiler/profiler.sh --version
```

---

## istioctl

| | |
|---|---|
| **First needed at** | Advanced |
| **First lesson** | [Service Mesh Troubleshooting (Istio)](/kubernetes/service-mesh-troubleshooting-istio) |

**What it does:** Inspects Istio sidecar sync state, Envoy listener/route/cluster config, mTLS, and routing when mesh traffic misbehaves.

**Do you need it?** Required for hands-on service mesh labs. Skip the install if you will read that lesson for command shapes only.

| Topic | Official docs |
|---|---|
| istioctl install | [istio.io/latest/docs/setup/getting-started](https://istio.io/latest/docs/setup/getting-started/) |
| istioctl download | [istio.io/latest/docs/ops/diagnostic-tools/istioctl](https://istio.io/latest/docs/ops/diagnostic-tools/istioctl/) |

The `demo` profile below works on a local `kind` or `minikube` cluster.

**macOS**

```bash
brew install istioctl
istioctl install --set profile=demo -y
```

**Linux**

```bash
curl -L https://istio.io/downloadIstio | sh -
cd istio-*
export PATH="$PWD/bin:$PATH"
istioctl install --set profile=demo -y
```

**Windows (PowerShell)**

```powershell
# Download from https://github.com/istio/istio/releases, then unpack and add bin to PATH
istioctl install --set profile=demo -y
```

**Verify**

```bash
istioctl version
kubectl get pods -n istio-system
```

---

## Cloud CLIs (Expert, optional)

| | |
|---|---|
| **First needed at** | Expert (only if you use a managed cluster for that module) |
| **First lesson** | [Cloud Managed Clusters (EKS/GKE/AKS)](/kubernetes/cloud-managed-clusters-eks-gke-aks) |

**What it does:** Talks to your cloud account to describe clusters, tail control-plane and workload logs, and verify IRSA, Workload Identity, and managed-identity wiring from the shell.

**Do you need it?** Expert-only. Install only the CLI for the provider you use (`aws`, `gcloud`, or `az`). Skip entirely if you stay on local `kind`/`minikube` through Advanced.

### AWS CLI (`aws`)

| Topic | Official docs |
|---|---|
| AWS CLI install | [docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html](https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html) |

**macOS**

```bash
brew install awscli
```

**Linux**

```bash
curl -s "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o awscliv2.zip
unzip -q awscliv2.zip && sudo ./aws/install
```

**Windows (PowerShell)**

```powershell
winget install Amazon.AWSCLI
```

**Verify**

```bash
aws --version
aws sts get-caller-identity
```

### Google Cloud CLI (`gcloud`)

| Topic | Official docs |
|---|---|
| gcloud install | [cloud.google.com/sdk/docs/install](https://cloud.google.com/sdk/docs/install) |

**macOS**

```bash
brew install --cask google-cloud-sdk
```

**Linux**

```bash
curl -O https://dl.google.com/dl/cloudsdk/channels/rapid/downloads/google-cloud-cli-linux-x86_64.tar.gz
tar -xf google-cloud-cli-linux-x86_64.tar.gz
./google-cloud-sdk/install.sh
```

**Windows (PowerShell)**

```powershell
winget install Google.CloudSDK
```

**Verify**

```bash
gcloud --version
gcloud auth list
```

### Azure CLI (`az`)

| Topic | Official docs |
|---|---|
| Azure CLI install | [learn.microsoft.com/en-us/cli/azure/install-azure-cli](https://learn.microsoft.com/en-us/cli/azure/install-azure-cli) |

**macOS**

```bash
brew install azure-cli
```

**Linux**

```bash
curl -sL https://aka.ms/InstallAzureCLIDeb | sudo bash
```

**Windows (PowerShell)**

```powershell
winget install Microsoft.AzureCLI
```

**Verify**

```bash
az --version
az account show
```

---

## Chaos Mesh or Litmus (Expert)

| | |
|---|---|
| **First needed at** | Expert |
| **First lesson** | [Chaos Engineering](/kubernetes/chaos-engineering-and-failure-injection) |

**What it does:** Injects controlled failures (pod kills, network latency, CPU/memory stress) so you can validate probes, alerts, and runbooks before a real outage.

**Do you need it?** Expert-only. Install one platform if you plan to run chaos labs hands-on. Skip if you will read that lesson for experiment design only.

{% capture chaos_callout %}
Install **one** chaos platform, not both, unless your organization already standardizes on a specific tool. Both run fine on local `kind` or `minikube`. Helm 3 and a working cluster are required.
{% endcapture %}
{% include components/course-callout.html type="important" title="Pick one" body=chaos_callout %}

### Option A: Chaos Mesh

| Topic | Official docs |
|---|---|
| Chaos Mesh install | [chaos-mesh.org/docs/next/production-installation-using-helm](https://chaos-mesh.org/docs/next/production-installation-using-helm) |

**macOS / Linux / Windows (with Helm)**

```bash
helm repo add chaos-mesh https://charts.chaos-mesh.org
helm repo update
kubectl create ns chaos-mesh --dry-run=client -o yaml | kubectl apply -f -
helm install chaos-mesh chaos-mesh/chaos-mesh -n chaos-mesh --set chaosDaemon.runtime=containerd --set chaosDaemon.socketPath=/run/containerd/containerd.sock
```

On Docker Desktop or some `kind` setups, you may need `docker` as the runtime instead of `containerd`. See the official docs link above for your environment.

**Verify**

```bash
kubectl get pods -n chaos-mesh
```

### Option B: Litmus

| Topic | Official docs |
|---|---|
| Litmus install | [docs.litmuschaos.io/docs/getting-started/installation](https://docs.litmuschaos.io/docs/getting-started/installation) |

**macOS / Linux / Windows (with Helm)**

```bash
kubectl create ns litmus --dry-run=client -o yaml | kubectl apply -f -
helm repo add litmuschaos https://litmuschaos.github.io/litmus-helm/
helm repo update
helm install chaos litmuschaos/litmus -n litmus
```

**Verify**

```bash
kubectl get pods -n litmus
```

---

{% capture checkpoint_items %}
- [ ] I read the at-a-glance table and know which tools apply to my current level.
- [ ] I know which tools I need for my current course level (see the summary table on [Prerequisites](/kubernetes/prerequisites)).
- [ ] I installed `jq` (or know I will before Intermediate) and verified with `jq --version`.
- [ ] For Expert: I know whether I need a cloud CLI, and I picked Chaos Mesh **or** Litmus if I plan to run chaos labs hands-on.
{% endcapture %}
{% include components/course-checkpoint.html body=checkpoint_items %}
