# Kubernetes Troubleshooting Course
### For Java / Spring Boot / Microservices / Cloud Developers

This is a static-site-ready online course, structured as a Jekyll collection. It is **not** a book — every lesson is a standalone page with rich front matter (title, SEO metadata, prerequisites, ordering, tags) so a Jekyll site generator can render sidebar navigation, breadcrumbs, and prerequisite links automatically from data instead of manual "prev/next" links in the markdown body.

## How this maps to a Jekyll site

```
k8s/
├── README.md                      ← this file
├── _course/                       ← rename/move to your Jekyll collection dir (e.g. `_course`) as-is
│   ├── 00-getting-started/
│   │   ├── 01-welcome-and-how-this-course-works.md
│   │   └── 02-prerequisites.md
│   ├── 01-beginner/                (8 lessons incl. capstone)
│   ├── 02-intermediate/            (9 lessons incl. capstone)
│   ├── 03-advanced/                (9 lessons incl. capstone)
│   └── 04-expert/                  (8 lessons incl. capstone)
└── reference/                     ← standalone lookup pages, not part of the level progression
    ├── command-cheat-sheet.md
    ├── incident-runbook-template.md
    ├── hands-on-lab-ideas.md
    └── assessment-rubric.md
```

`_course/` is designed to become a Jekyll collection (`collections: { course: { output: true } }` in `_config.yml`). The numeric directory/file prefixes (`01-`, `02-`, ...) are for human/filesystem ordering only — the site should sort and link pages using the `order` front-matter field, not the filename, since Jekyll collections don't guarantee filename-based sort order across subdirectories.

`reference/` is a second, much smaller collection (or a plain `pages` directory) for material a working developer looks up day-to-day once they already know the concepts — it deliberately sits outside the level progression.

## Front matter schema

Every lesson in `_course/**/*.md` (except the two Getting Started pages, which share the schema minus `level`/`module`) uses:

```yaml
---
layout: lesson
title: "Human-readable title"
seo_title: "Keyword-rich <title> tag override, ~60 chars"
description: "Meta description, 150-160 chars, action-oriented"
permalink: /course/<level>/<topic-slug>/
level: beginner | intermediate | advanced | expert
module: 1 | 2 | 3 | 4
lesson: <N>                 # position within this level, 1-indexed
order: <level*100 + lesson> # global sort key, e.g. 205 = intermediate lesson 5
prerequisites:
  - <slug-of-prior-lesson>   # same-level slugs, or full /course/<level>/<slug>/ path cross-level
keywords: [search, terms]
tags: [topic, tags]
estimated_minutes: 20
last_modified: 2026-07-06
---
```

Reference pages in `reference/*.md` use a lighter schema (`layout: reference`) with no `level`/`module`/`lesson`/`order`/`prerequisites` fields, since they aren't part of a learning sequence:

```yaml
---
layout: reference
title: "..."
seo_title: "..."
description: "..."
permalink: /reference/<slug>/
keywords: [...]
tags: [reference, ...]
last_modified: 2026-07-06
---
```

Use these fields to drive site chrome:
- **Sidebar / table of contents** — group by `level`, sort by `order`.
- **Prerequisite callouts** — resolve each `prerequisites` entry's `permalink` to render "You should already know: [linked lesson title]" above the fold.
- **SEO** — `seo_title` → `<title>`, `description` → `<meta name="description">`, `keywords`/`tags` → structured data or category pages.
- **Reading time** — `estimated_minutes` for a "~20 min" badge per lesson.

## Course structure

The course is level-first: each level is a sequence of lessons that mixes concept, commands, and lab in one page — there's no separate "playbook vs. curriculum" split. Deep technical topics (JVM diagnostics, networking, service mesh, cloud specifics) are intentionally **split by depth across levels** rather than taught once — e.g. JVM troubleshooting appears as basic `jcmd`/heap-awareness in Intermediate, then thread/heap dumps and GC tuning in Advanced, so a learner only sees the depth appropriate to where they are.

| Level | Lessons | Focus |
|---|---|---|
| [Getting Started](_course/00-getting-started/01-welcome-and-how-this-course-works.md) | 2 | Course overview, prerequisites, lab setup |
| [Beginner](_course/01-beginner/01-kubernetes-architecture-fundamentals.md) | 8 | Core K8s objects, first Spring Boot deploy, reading pod status, basic restarts |
| [Intermediate](_course/02-intermediate/01-liveness-readiness-and-startup-probes.md) | 9 | Probes, exit codes, DNS, config propagation, storage, RBAC, JVM-in-container basics |
| [Advanced](_course/03-advanced/01-thread-dumps-and-deadlock-analysis.md) | 9 | Thread/heap dumps, GC tuning, profiling, service mesh, observability, autoscaling |
| [Expert](_course/04-expert/01-node-and-control-plane-internals.md) | 8 | Cluster internals, low-level networking, cloud IAM, GitOps, chaos engineering, incident command |

Every lesson body includes, in order: a **Prerequisites** callout, core concepts (with tables and Mermaid diagrams where they clarify structure/flow), full copy-pasteable commands, a **Lab** section for hands-on practice, and a **Checkpoint** self-assessment at the bottom. Site-level navigation (prev/next, breadcrumbs, sidebar) is left entirely to the framework via front matter — lesson bodies contain no manual navigation links.

## Reference section

Standalone, non-sequential lookup material:

| Page | Use it when... |
|---|---|
| [Command Cheat Sheet](reference/command-cheat-sheet.md) | You already know the concepts, just need copy-paste commands |
| [Incident Runbook Template](reference/incident-runbook-template.md) | You need to document a live incident |
| [Hands-on Lab Ideas](reference/hands-on-lab-ideas.md) | You want extra fault-injection reps beyond the per-lesson labs |
| [Assessment Rubric](reference/assessment-rubric.md) | You want to self-score readiness, or run a certification/game-day review |

## Total content

40 pages: 2 Getting Started + 34 level lessons (8+9+9+8) + 4 Reference pages.
