---
layout: reference
title: "Assessment Rubric & Certification Checklist"
seo_title: "Kubernetes Troubleshooting Assessment Rubric & Checklist"
description: "Score Kubernetes troubleshooting competency from 0-3 across diagnostics, JVM, networking, mesh, and incident process — with a certification bar."
permalink: /reference/assessment-rubric/
keywords: [kubernetes assessment rubric, kubernetes certification checklist, on-call readiness, kubernetes competency scoring, incident commander checklist]
tags: [reference, assessment, rubric, certification]
last_modified: 2026-07-06
---

Reach for this page when you want an honest read on your own (or a teammate's) troubleshooting competency — as a self-check after finishing a level, or as a structured checklist for evaluating someone before they go on-call. Score each competency 0–3 (0 = cannot perform, 3 = performs unassisted under time pressure).

## Capstone summary

Each course level ends in a capstone exercise. Here they are in one place:

**Level 1 Capstone (Beginner)** — Given a broken Spring Boot Deployment manifest (bad image, missing ConfigMap key, no resource limits), fix it using only `kubectl` diagnostics — no source code changes allowed.

**Level 2 Capstone (Intermediate)** — On-call simulation: given 3 simultaneously broken microservices (networking, config, and JVM memory issues), triage and resolve all three within a time box, producing a written root-cause summary for each.

**Level 3 Capstone (Advanced)** — Load-test a Spring Boot microservice to induce a real production-like incident (pick 2 of: memory leak, thread pool exhaustion, GC pause spike, mesh circuit-breaker trip) and produce a full root-cause report using only the tools from the Advanced level's modules.

**Level 4 Capstone (Expert)** — Full game-day: an unannounced multi-layered failure (e.g., NetworkPolicy + IRSA + node pressure combined) is injected into a staging cluster; the trainee leads incident response solo, including stakeholder communication drafts and a postmortem, evaluated against this rubric.

## Diagnostic speed & method
- [ ] Runs the triage funnel without prompting
- [ ] Uses `describe` → `events` → `logs` in correct order
- [ ] Distinguishes cluster-level vs workload-level issues within 2 minutes

## Pod & JVM diagnostics
- [ ] Correctly interprets all exit codes from memory
- [ ] Captures and reads a thread dump to find a deadlock/pool exhaustion
- [ ] Captures and reads a heap dump to find a leak suspect
- [ ] Explains cgroup-vs-heap relationship and fixes an OOM via JVM flags

## Networking
- [ ] Diagnoses a Service/Endpoint mismatch
- [ ] Diagnoses a NetworkPolicy block using a debug pod
- [ ] Explains JVM DNS caching implications

## Mesh & Observability
- [ ] Reads Envoy access log flags correctly
- [ ] Writes a PromQL query for a given JVM/HTTP metric unaided
- [ ] Follows a trace ID across 3+ services to isolate latency source

## Cluster & infra
- [ ] Diagnoses a node pressure condition and its blast radius
- [ ] Explains admission webhook failure blast radius
- [ ] Performs a safe node drain/cordon/uncordon cycle

## Incident process
- [ ] Produces a complete [runbook doc](/reference/incident-runbook-template/) under time pressure
- [ ] Communicates root cause in one clear paragraph, no jargon soup
- [ ] Proposes a concrete follow-up (alert, probe, doc) after every incident

## Certification bar

Level 3 complete = "production-ready developer on-call." Level 4 complete + capstone game-day = "incident commander ready."

**Related:** [Incident Runbook Template](/reference/incident-runbook-template/) · [Hands-on Lab Ideas](/reference/hands-on-lab-ideas/) · [Expert level](/course/expert/)
