---
layout: reference
title: "Incident Response Runbook Template"
seo_title: "Kubernetes Incident Response Runbook Template (Copy-Paste)"
description: "A copy-paste incident response runbook template for Kubernetes production incidents, covering triage timeline, root cause, mitigation, and follow-up."
permalink: /reference/incident-runbook-template/
keywords: [kubernetes incident response, runbook template, incident postmortem, sev1 sev2 template, kubernetes on-call, triage timeline]
tags: [reference, incident-response, runbook, on-call]
last_modified: 2026-07-06
---

Reach for this the moment a real production incident starts — copy the block below straight into your incident doc or chat channel and fill it in as you go. It keeps triage, root cause, mitigation, and follow-up in one consistent shape so nothing gets skipped under pressure, and so postmortems don't need to be reconstructed from memory afterward.

## Incident Response Runbook Template

```markdown
## Incident: <short title>
- Detected at: <timestamp>
- Detected by: <alert name / user report>
- Severity: SEV-<1-4>
- Affected service(s): <deployment/namespace>

### Symptom
<what users/dashboards observed>

### Triage timeline
- [ ] `kubectl get events -A --sort-by='.lastTimestamp'` reviewed
- [ ] `kubectl get pods -n <ns>` status classified (see pod lifecycle & status)
- [ ] `kubectl describe pod` reviewed for root-cause event
- [ ] `kubectl logs --previous` reviewed for crash signature
- [ ] Recent deploy/config change correlated (rollout history / git log / ArgoCD sync)
- [ ] Dependency health checked (DB, cache, downstream services)
- [ ] Node/cluster health ruled out or confirmed as cause

### Root cause
<one paragraph, causal chain from trigger to symptom>

### Mitigation taken
<rollback / scale up / restart / config fix / traffic shift>

### Follow-up actions
- [ ] Permanent fix ticket filed
- [ ] Alerting/probe gap addressed
- [ ] Runbook/doc updated with new failure signature
- [ ] Postmortem scheduled (SEV-1/2)
```

To turn this into a practiced skill rather than a document you read once, work through the [course](/course/) — the Expert level capstone ends in a full game-day exercise that uses this exact template under time pressure.

**Related:** [Command Cheat Sheet](/reference/command-cheat-sheet/) · [Assessment Rubric](/reference/assessment-rubric/) · [Expert level](/course/expert/)
