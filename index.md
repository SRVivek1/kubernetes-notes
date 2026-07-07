---
layout: web/course-reading
heading: Kubernetes Troubleshooting for Java & Spring Boot Developers
title: "Kubernetes Troubleshooting for Java and Spring Boot Developers"
subtitles:
  - A hands-on Kubernetes troubleshooting course for Java, Spring Boot, and microservices developers, from first pod to full incident command, beginner to expert.
permalink: /kubernetes/
seo:
  type: TechArticle
  date_published: false
description: Free Kubernetes troubleshooting course for Java and Spring Boot developers. Learn cluster architecture, pod diagnostics, probes, JVM-in-container debugging, service mesh, observability, cloud operations, and full game-day incident response.
---

## Overview

This course takes Java and **Spring Boot** microservice developers from zero Kubernetes operational experience to confident on-call troubleshooting. You already know Java, Spring Boot, and Docker. Here you add the cluster mental model, kubectl diagnostics, probe configuration, JVM debugging inside containers, networking and storage triage, service mesh observability, and the incident-command skills to run a production game day.

{% capture callout_body %}
Every lesson ships with copy-pasteable commands, a hands-on lab, and a checkpoint self-assessment. Deep topics like JVM diagnostics and networking are split by level so you only see the depth appropriate to where you are. Capstones at the end of each level simulate real on-call scenarios under time pressure.
{% endcapture %}
{% include components/course-callout.html type="note" title="Our promise" body=callout_body %}

---

## Table of Contents

- [Overview](#overview)
- [Table of Contents](#table-of-contents)
- [Course Sections](#course-sections)
- [Who This Is For](#who-this-is-for)

---

## Course Sections

The course is organized into seven progressive sections. Getting Started covers setup, Beginner through Expert build troubleshooting depth in stages, Operations & Troubleshooting is a standalone, symptom-first incident guide usable without reading the rest of the course, and Reference holds standalone lookup material for day-to-day operations.

1. **Getting Started:** course overview, prerequisites, local lab setup with kind or minikube, and a tool installation guide for jq, k9s, Istio, JVM tools, and more.
2. **Beginner:** core K8s objects, first Spring Boot deploy, reading pod status, resource limits, and a broken-deployment capstone.
3. **Intermediate:** probes, exit codes, DNS, config propagation, storage, RBAC, JVM-in-container basics, and an on-call simulation capstone.
4. **Advanced:** thread and heap dumps, GC tuning, CPU profiling, service mesh troubleshooting, observability, and a production incident capstone.
5. **Expert:** cluster internals, admission webhooks, packet capture, cloud-managed clusters (EKS/GKE/AKS), GitOps, chaos engineering, incident command, and a full game-day capstone.
6. **Operations & Troubleshooting:** a self-contained, end-to-end triage guide, start with a 5-minute health check, then follow a symptom router straight to the fix for pods that won't start, crash loops, networking/503s, slow performance, or storage and config issues, no prior lessons required.
7. **Reference:** command cheat sheet, incident runbook template, hands-on lab ideas, and assessment rubric.

---

## Who This Is For

- **Developers** building Spring Boot microservices who get paged when pods crash-loop at 2 a.m.
- **Application-support engineers** who need kubectl fluency and JVM diagnostic skills inside containers.
- **Architects** evaluating how Java workloads behave on Kubernetes under failure and load.

You should be comfortable with Java, Spring Boot, and basic Docker. No prior Kubernetes operational experience is assumed. The Getting Started section covers lab setup; each later section assumes the ones before it.

---

{% include components/course-start-cta.html
  kicker="Start with the course overview"
  title="Welcome: Kubernetes Troubleshooting for Java & Spring Boot Developers"
  description="Learn how this course is structured, what you need before you start, and how the four levels build from first pod to full incident command."
  href="kubernetes/welcome"
  button_label="Start Kubernetes Course"
%}

---
