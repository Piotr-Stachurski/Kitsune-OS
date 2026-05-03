<p align="center">
  <img src="assets/Kitsune-OS v3.1a.webp" width="250" alt="Kitsune-OS Logo">
</p>

# 🦊 Kitsune-OS v5.4

**Agentic RevOps Workflow Framework — Google Workspace, $0 infrastructure, production-grade.**

[![Version](https://img.shields.io/badge/version-5.4_stable-blue)](https://piotr-stachurski.github.io/Kitsune-OS/)
[![Stack](https://img.shields.io/badge/stack-Google_Apps_Script-orange)](https://developers.google.com/apps-script)
[![AI](https://img.shields.io/badge/AI-Gemini_3_Flash_%7C_2.5_Pro_fallback-green)](https://ai.google.dev)
[![Infra](https://img.shields.io/badge/infrastructure-%240-brightgreen)](https://workspace.google.com)
[![License](https://img.shields.io/badge/source-private_IP-red)](#)

---

## What it is

Kitsune-OS is a production agentic workflow system that automates end-to-end pipeline management for high-volume recruitment operations. It was designed and built solo, during an active job search, to solve its own use case — a live demonstration of the same systems thinking applied to enterprise clients.

**The problem:** After sending 200+ applications, manual tracking became the bottleneck. Tried Google Sheets, then scripted Gemini prompts, then a custom Gemini GEM — which proved unreliable (hallucinations, no duplicate detection, lost session context). Custom architecture turned out to be the lower-effort path to reliability.

**The model:** AI as scalable workforce. Human as SSoT Architect.

---

## Architecture

```
┌─────────────────────────────────────────────────────────────┐
│  FRONTEND (index.html)                                      │
│  Dark-mode terminal UI · Session ID guard · Payload         │
│  sanitizer (15k cap) · UI lock · Fail-open duplicate gate   │
└────────────────────┬────────────────────────────────────────┘
                     │ google.script.run (async bridge)
┌────────────────────▼────────────────────────────────────────┐
│  BACKEND (Code.gs)                                          │
│                                                             │
│  ┌──────────┐    ┌─────────────┐    ┌───────────────────┐  │
│  │ Stage 1  │───▶│  DUP CHECK  │───▶│     Stage 2       │  │
│  │ Triage   │    │ fail-open   │    │  One-Shot Deep    │  │
│  │ temp 0.1 │    │ SSoT query  │    │  Dive  temp 0.6   │  │
│  └──────────┘    └─────────────┘    └─────────┬─────────┘  │
│                                               │             │
│  ┌────────────────────────────────────────────▼──────────┐  │
│  │  CASCADE LLM ROUTER                                   │  │
│  │  Flash → Pro fallback · Exponential backoff 4s→64s    │  │
│  │  Fast-track switch on 503/429/quota                   │  │
│  │  Kitsune-Shield: 4-layer JSON fallback parser         │  │
│  └───────────────────────────────────────────────────────┘  │
│                                                             │
│  ┌───────────────────────┐  ┌────────────────────────────┐  │
│  │  INBOX OPERATIONS     │  │  PERSISTENCE LAYER         │  │
│  │  Gmail scanner        │  │  Sheets: SSoT Master Log   │  │
│  │  Rejection classifier │  │  Docs: Analysis reports    │  │
│  │  Silent Kill Mode     │  │  Drive: Auto-archive       │  │
│  │  Label management     │  │  safeMoveFile() + backoff  │  │
│  └───────────────────────┘  └────────────────────────────┘  │
└─────────────────────────────────────────────────────────────┘
```

---

## Pipeline walkthrough

### Stage 1 — Operational Triage
Single LLM call. Extracts structured JSON from raw job offer text.

**5 embedded logic guardrails:**
- Seniority calibration (no over-qualification flag suppression)
- Geographic radius enforcement
- Role-type filtering (excludes developer roles, etc.)
- GIGO prevention (full company name, not abbreviations)
- Irrelevant offer rejection

**Output:** `{companyName, jobTitle, recommendation, justification}`  
**Temp:** 0.1 — deterministic, no creative drift.

---

### Duplicate Check — Sequential Gate
After Stage 1 extracts company + title, system queries SSoT before rendering results. Blocks render until user decision.

**Fail-open behavior:** If the check fails (Sheets API timeout, connectivity), user workflow continues uninterrupted. Duplicate check is a convenience feature, not a security gate. Cost of false positive (user manually ignores a duplicate) is lower than cost of false negative (user blocked from working).

---

### Stage 2 — Cognitive Deep Dive (One-Shot, V5.4)
Single unified LLM call producing 6 analytical sections.

> **Why one-shot instead of 3 parallel nodes (V1 architecture)?**  
> V1 used 3 parallel nodes for fault isolation — each failing independently without affecting others. After measuring production behavior, Gemini 3 Flash demonstrated stable JSON adherence under a unified prompt at temperature 0.6. Refactor result: **80% cost reduction (20 gr → 4 gr PLN per full analysis cycle)**. Single-point-of-failure risk accepted, mitigated by Kitsune-Shield 4-layer fallback parser.

| Section | Content |
|---------|---------|
| `bsBingo` | Corporate-speak decoded to operational chaos signals |
| `gapAnalysis` | Requirements vs. candidate DNA, per-requirement delta |
| `redFlags` | Hidden operational debt, org dysfunction, trap clauses |
| `interviewThreats` | 5 executive power questions targeting process maturity |
| `cognitiveNote` | First-person elevator pitch ready for cover letter use |
| `finalVerdict` | APPLY / REJECT / EXPAND with justification |

---

### Persistence — SSoT Write-Back
Stage 2 output piped through `htmlToDocText()` converter (handles `<ul>/<li>`, `<strong>`, HTML entities, bold preservation), then injected into Google Doc template. Master Log row updated. Dashboard KPIs auto-refresh.

---

## Cascade LLM Router

```
Request
  │
  ▼
[Gemini 3 Flash] ── 503/429/quota ──▶ fast-track switch
  │                                      │
  │ corrupt JSON                         ▼
  ▼                                  [Gemini 2.5 Pro]
exponential backoff                      │
4s → 8s → 16s → 32s → 64s              │
  │                                      │
  └──────────────────────────────────────┘
                   │
                   ▼
            Kitsune-Shield
         4-layer JSON parser
```

**Why fast-track on infrastructure errors, not backoff?**  
Backoff on a throttled endpoint = retrying against a wall. The problem is at Google's infrastructure, not the model. Switching immediately finds an available endpoint. Backoff is correct for cognitive errors (corrupt JSON) where the problem is transient model behavior.

---

## Inbox Operations Module

Autonomous Gmail scanner running on a time-based trigger. Classifies threads by semantic intent: rejection vs. human-initiated contact.

- **Silent Kill Mode:** Processes rejections without any UI noise. Direct SSoT injection (column E status update) and Drive archive, zero interruption.
- **Autopilot toggle:** Hardware-style kill-switch persisted in `PropertiesService`. Binary ON/OFF, visible in UI at all times. Not a nice-to-have — it's a governance requirement for any agentic system. When Autopilot is OFF, system enters human-in-the-loop mode.

---

## Key engineering decisions

### Defense-in-depth payload sanitization
Both frontend and backend sanitize independently. If frontend is bypassed (direct API call, future mobile client, alternative interface), the backend layer still prevents binary noise from reaching the LLM context window. Each layer is independently effective — no single point of failure.

### Session ID guard (frontend)
GAS callbacks are asynchronous and non-cancellable. A user resetting mid-analysis would trigger two concurrent result sets rendering simultaneously — ghost data in the DOM. Timestamped session IDs invalidate callbacks from superseded sessions before they render.

### `setText(" ")` instead of `setText("")`
Google Docs API throws `Cannot insert an empty text element` when calling `setText("")` on a `ListItem` or the last element in a document body. A single space is accepted as valid content, is visually invisible in rendered output, and preserves DOM integrity. Simple fix, non-obvious root cause.

### PropertiesService for all credentials
Zero hardcoded secrets. API keys live in `PropertiesService.getScriptProperties()`. Source code is safe to share without credential exposure.

---

## Cost model

| | |
|---|---|
| Infrastructure | **$0** — native Google Workspace, no servers, no SaaS |
| Per full analysis (Stage 1 + Stage 2 + archive) | **~4 gr PLN (~$0.01)** — measured in production, Gemini 3 Flash PAYG |
| Previous architecture (3-node Stage 2) | ~20 gr PLN (~$0.05) |
| 200 analyses/month | **~10 PLN (~$2.50)** |
| Comparable SaaS (Teal, Jobscan, Huntr) | 30–150 PLN/month (~$7–$38) |

---

## Setup

After `HARD_RESET_Triggers()`, execute in this order:

```
1. HARD_RESET_Triggers()     → clears PropertiesService + all triggers
2. setApiKey()               → injects Gemini API key
3. setupKitsuneEnvironment() → creates Drive folder, Sheet (11-col schema +
                               validation), Docs template, binds 5 properties:
                               KITSUNE_ROOT_URL, KITSUNE_SS_ID, KITSUNE_SS_URL,
                               KITSUNE_ARCHIVE_ID, KITSUNE_TEMPLATE_ID
                               + auto-creates Inbox Operations trigger (30 min)
4. Redeploy Web App          → only required when code changes
```

> **Note:** `setupKitsuneEnvironment()` takes 10–15 seconds (creates Drive structures + Doc template + trigger). The UI locks with a visual progress overlay during this time. If it appears frozen, it is working.

> **Triggers:** The Inbox Operations time-based trigger (every 30 minutes) is created automatically during `setupKitsuneEnvironment()`. Existing triggers for the same handler are deleted first, so the function is safe to run multiple times without duplicating triggers. No manual trigger configuration required.

---

## Roadmap

| Version | Status | Scope |
|---------|--------|-------|
| **V5.4** | ✅ Stable | Core SSoT · Dual-Stage Analysis · Inbox Operations · Defense-in-Depth · Duplicate Guard · One-Shot Stage 2 |
| **V6.0** | 🔨 Active development | Standalone HTML Web Dashboard — visual KPIs, pipeline drill-down, off GAS execution limits |
| **V6.1** | 📋 Planned | Auto-Drafts Engine — autonomous recruiter response drafts based on SSoT context |
| **Kitsune Genesis** | 🔬 Separate R&D | Multi-modal Candidate DNA profiling — CV + interview synthesis → Synarch Profile. Separate project, feeds into Kitsune-OS as injection layer in future integration. |

---

## Repository structure

```
kitsune-os/                  ← this repo (public, portfolio showcase)
  └── kitsune-portfolio.html ← live portfolio / case study

kitsune-os-source/           ← private repo (IP protected)
  ├── Code.gs                ← backend (~1,255 lines)
  └── index.html             ← frontend (~992 lines)
```

Source code is maintained in a private repository for IP protection. This public repo serves as an architectural showcase and portfolio. If you're a recruiter or hiring manager reviewing this as part of an application — the source is available for review on request after an initial conversation.

---

## Principal Architect

**Piotr Stachurski** — Business Operations Manager · RevOps Architect  
Cracow, Poland · Available for EMEA remote / hybrid roles

12+ years across Cisco (PS CXC EMEA), TE Connectivity, Shell. Core expertise: strategic operations, AI & digital transformation, SSoT architecture, pipeline management, process engineering. Philosophy graduate turned operational systems architect.

> *Kitsune-OS was designed and built solo during an active job search — zero R&D budget, 100% native Google Workspace tooling, production-grade output. It solves a real operational problem I had. That's the only reason it exists.*

[![LinkedIn](https://img.shields.io/badge/LinkedIn-stachurskipiotr-blue)](https://www.linkedin.com/in/stachurskipiotr)
[![Portfolio](https://img.shields.io/badge/Portfolio-Project_Showcase-orange)](https://piotrstachurski.github.io/kitsune-portfolio/)
[![Email](https://img.shields.io/badge/Email-stachurski.piotr%40gmail.com-green)](mailto:stachurski.piotr@gmail.com)
