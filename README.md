<p align="center">
  <img src="assets/Kitsune-OS v3.1a.webp" width="250" alt="Kitsune-OS Logo">
</p>

# 🦊 Kitsune-OS v6.1

**Agentic RevOps Workflow Framework — Google Workspace, $0 infrastructure, production-grade.**

[![Version](https://img.shields.io/badge/version-6.1_stable-blue)](https://piotr-stachurski.github.io/Kitsune-OS/)
[![Stack](https://img.shields.io/badge/stack-Google_Apps_Script-orange)](https://developers.google.com/apps-script)
[![AI](https://img.shields.io/badge/AI-Gemini_3_Flash_%7C_2.5_cascade-green)](https://ai.google.dev)
[![Infra](https://img.shields.io/badge/infrastructure-%240-brightgreen)](https://workspace.google.com)
[![License](https://img.shields.io/badge/source-private_IP-red)](#)

---

## What it is

Kitsune-OS is a production agentic workflow system that automates end-to-end pipeline management for high-volume recruitment operations. Designed and built solo, during an active job search, to solve its own use case — a live demonstration of systems thinking applied to operational problems.

**The problem:** After sending 200+ applications, manual tracking became the bottleneck. Tried Google Sheets, then scripted Gemini prompts, then a custom Gemini GEM — which proved unreliable (hallucinations, no duplicate detection, lost session context). Custom architecture turned out to be the lower-effort path to reliability.

**The model:** AI as scalable workforce. Human as SSoT Architect.

---

## Architecture

```
┌──────────────────────────────────────────────────────────────────────┐
│  ROUTER LAYER (doGet)                                                │
│  /exec              → Main App  (index.html)                         │
│  /exec?view=dash    → Dashboard (dashboard.html)                     │
└────────────────────┬──────────────────┬──────────────────────────────┘
                     │                  │
                     ▼                  ▼
┌────────────────────────────┐  ┌─────────────────────────────────────┐
│  MAIN APP (index.html)     │  │  DASHBOARD V6.0+ (dashboard.html)   │
│  Dark-mode terminal UI     │  │  Standalone HTML web app            │
│  Stage 1/2 analysis flow   │  │  Real-time KPI grid · 8-col table   │
│  Session ID guard          │  │  Client-side filtering · Status/SLA │
│  Payload sanitizer (15k)   │  │  Toast notifications · Inline scan  │
│  UI lock · Fail-open dup   │  │  Off GAS execution time limits      │
└────────────┬───────────────┘  └────────────┬────────────────────────┘
             │ google.script.run             │ getRawDataPayload()
             ▼                                │ scanGmailForResponses()
┌─────────────────────────────────────────────▼───────────────────────┐
│  BACKEND (Code.gs)                                                  │
│                                                                     │
│  ┌──────────┐    ┌─────────────┐    ┌───────────────────┐           │
│  │ Stage 1  │───▶│  DUP CHECK  │───▶│     Stage 2       │           │
│  │ Triage   │    │ fail-open   │    │  One-Shot Deep    │           │
│  │ temp 0.1 │    │ SSoT query  │    │  Dive  temp 0.6   │           │
│  └──────────┘    └─────────────┘    └─────────┬─────────┘           │
│                                               │                     │
│  ┌────────────────────────────────────────────▼──────────────────┐  │
│  │  CASCADE LLM ROUTER                                           │  │
│  │  Flash 3 → Flash 2.5 → Pro 2.5 (cost-optimized order)         │  │
│  │  Fast-track switch on 503/429/quota · Backoff on JSON errors  │  │
│  │  Kitsune-Shield: 4-layer JSON fallback parser                 │  │
│  └───────────────────────────────────────────────────────────────┘  │
│                                                                     │
│  ┌─────────────────────────────────┐  ┌────────────────────────┐    │
│  │  GMAIL SCANNER V6.1             │  │  PERSISTENCE LAYER     │    │
│  │  ▸ Inverted Discovery           │  │  Sheets: SSoT Master   │    │
│  │  ▸ ATS sender detection         │  │  Docs: Analysis archive│    │
│  │  ▸ Lenient domain matching      │  │  Drive: Auto-archive   │    │
│  │  ▸ Token match + stop words     │  │  safeMoveFile() backoff│    │
│  │  ▸ Cost cap (3 PLN per scan)    │  │                        │    │
│  │  ▸ Orphan label · Inbox Zero    │  │                        │    │
│  └─────────────────────────────────┘  └────────────────────────┘    │
└─────────────────────────────────────────────────────────────────────┘
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

### Stage 2 — Cognitive Deep Dive (One-Shot)
Single unified LLM call producing 6 analytical sections.

> **Why one-shot instead of 3 parallel nodes (V1 architecture)?**
> V1 used 3 parallel nodes for fault isolation. After measuring production behavior, Gemini 3 Flash demonstrated stable JSON adherence under a unified prompt at temperature 0.6. Refactor result: **80% cost reduction (20 gr → 4 gr PLN per full analysis cycle)**. Single-point-of-failure risk accepted, mitigated by Kitsune-Shield 4-layer fallback parser.

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
  ▼                                  [Gemini 2.5 Flash]
exponential backoff                      │ 503/429 ──▶ fast-track
4s → 8s → 16s → 32s → 64s              ▼
  │                                  [Gemini 2.5 Pro]
  │                                      │
  └──────────────────────────────────────┘
                   │
                   ▼
            Kitsune-Shield
         4-layer JSON parser
```

**Cost-optimized cascade order:** Flash 3 (~4 gr) → Flash 2.5 (~6 gr) → Pro 2.5 (~35 gr). Cheapest first, most expensive only as last resort. Pro fallback rare in production (typically ~5% of calls under sustained Flash availability).

**Why fast-track on infrastructure errors, not backoff?**
Backoff on a throttled endpoint = retrying against a wall. The problem is at Google's infrastructure, not the model. Switching immediately finds an available endpoint. Backoff is correct for cognitive errors (corrupt JSON) where the problem is transient model behavior.

---

## Gmail Scanner V6.1 — Inverted Discovery

V5.4 used per-row search: iterate SSoT → query Gmail by company name → match. Production data showed selective failures (matched 33-50% per scan, threads ranking non-deterministic, single-thread-per-company gates).

V6.1 inverts the control flow.

### Architecture

```
STAGE A — Discovery (no LLM)
  Pull all unprocessed Gmail threads (4-7 day window)
  Filter: -label:Kitsune-Processed -label:Kitsune-Unrelated
          (in:inbox OR category:updates OR category:promotions)

STAGE B — Triage (matching cascade)
  ATS sender detection ▸ hardcoded regex (Workday, LinkedIn, Lever, etc.)
  ↓
  Domain match (lenient) ▸ first-word tokenization, prefix support
                            "deloittece.com" matches "Deloitte"
  ↓
  Subject match ▸ full-name OR token match (≥4 chars, stop-words filtered)
  ↓
  Body match ▸ same cascade
  ↓
  Orphan detection ▸ LLM classify, label "Kitsune-Recruitment-Orphan"

STAGE C — Action (deterministic decision matrix)
  REJECTION         → Recruitment + archive + SSoT status update
  HUMAN_CONTACT     → Recruitment + archive + SSoT "In Progress"
  INTERVIEW_INVITE  → Recruitment + archive + SSoT urgency flag
  APPLICATION_CONF  → Recruitment + archive + no SSoT mutation
  ORPHAN            → Yellow label, no archive (user signal)
  UNRELATED         → Gray label, no archive (visibility retained)
```

### Defense-in-depth match cascade

Each layer has independent failure modes; no single point of failure.

| Layer | Strength | Edge case handled |
|-------|----------|-------------------|
| **ATS sender regex** | Catches platform-hosted mails (Workday, LinkedIn, Lever) | `Roche-Workday-DoNotReply@roche.com` |
| **Lenient domain match** | First-word tokenization + prefix matching | "Deloitte CE" sender → "Deloitte" SSoT row |
| **Stop words filter** | Excludes generic tokens ("group", "insurance", "solutions") | Prevents false-positive matches on noise tokens |
| **Min token length 4** | No 3-char generic matching | Avoids "and", "the", "ltd" false positives |
| **Full-string match before token** | Short company names handled | "TRG", "SGS" matched as whole strings |
| **Orphan handling** | Recruitment-like mails without SSoT match → yellow label | Calendar invites from interview platforms |

### Cost model — production data

Measured over 19-thread scan with 11 active SSoT rows:

```
Discovery:           1 Gmail.search() call          0 gr
ATS detection:       inline regex                   0 gr
Domain/Subject/Body match: deterministic            0 gr
LLM classification:  ~1 call per matched thread     ~4 gr each
Orphan classification: 1 call per non-matched      ~4 gr each
─────────────────────────────────────────────────────────────
Total observed:      ~0.7 PLN per scan (19 threads)
Cost cap (hard):     3 PLN per scan, then defer remaining
```

**Cost cap rationale:** Defense against LLM cascade runaway. If Flash 3 + Flash 2.5 both fail and scan falls back to Pro 2.5 repeatedly, system halts at 3 PLN, labels remaining threads as `Kitsune-Cost-Deferred`, retries on next scan. Production-grade cost guardrail, not a feature.

---

## Dashboard V6.0 — Standalone HTML Web App

V5.4 dashboard lived as a tab inside Google Sheets — convenient but constrained by GAS execution time limits and Sheets rendering quirks.

V6.0 spins it out as a standalone HTML web app, deployed under the same `doGet()` router via `?view=dash` parameter.

### Pipeline

```
Browser → /exec?view=dash → doGet(e) router → dashboard.html
                                ↓
                        google.script.run.getRawDataPayload()
                                ↓
                        Sheets RAW DATA → JSON normalized → Browser
                                ↓
                        Client-side: SLA logic · KPI computation · Filtering
```

**Decoupling:** Dashboard reads RAW DATA directly. Status/SLA logic ported 1:1 from `refreshDashboard()` to client-side JS. Single source of truth for data; logic computation distributed where it belongs (presentation layer = browser, data layer = Sheets).

### Features

- **8-column live table:** Date · Company · Position · Status/SLA · Source · Meeting · Cognitive Verdict · Archive
- **Status/SLA computed client-side:** ⚪ Sent (Xd) → ⚠️ Ghosting (>10d) → ⚫ Archived (>21d)
- **4 live KPIs:** Total Applications · Killed by System · In Progress · Conversion Rate
- **Instant client-side filtering:** Search by Company/Position, no round-trip to GAS
- **Inline Gmail Scan:** Trigger full V6.1 scanner from Dashboard, toast notification with results
- **Cross-view navigation:** "Analytics" link in Main App sidebar → opens Dashboard in new tab via `getWebAppUrl()` bridge (sandboxed iframe URL handling)
- **Mobile-responsive:** KPI grid collapses, table scrollable horizontally

---

## Inbox Operations Module

Autonomous Gmail scanner running on a time-based trigger. Classifies threads by semantic intent: rejection vs. human-initiated contact vs. interview invite vs. application confirmation.

- **Silent Kill Mode:** Processes rejections without any UI noise. Direct SSoT injection (column E status update) and Drive archive, zero interruption.
- **Autopilot toggle:** Hardware-style kill-switch persisted in `PropertiesService`. Binary ON/OFF, visible in UI at all times. Not a nice-to-have — it's a governance requirement for any agentic system. When Autopilot is OFF, system enters human-in-the-loop mode.
- **Multi-source coverage:** Scans `in:inbox`, `category:updates`, `category:promotions` — Gmail tabs are not visibility barriers, they're routing hints.

---

## Key engineering decisions

### Defense-in-depth payload sanitization
Both frontend and backend sanitize independently. If frontend is bypassed (direct API call, future mobile client, alternative interface), the backend layer still prevents binary noise from reaching the LLM context window. Each layer is independently effective — no single point of failure.

### Inverted control flow (V6.1 scanner)
V5.4 iterated SSoT to find Gmail. V6.1 iterates Gmail to find SSoT. Same logical operation, opposite control flow — eliminates selective matching failures from non-deterministic Gmail thread ranking. Reduces LLM calls from `1 per active row` to `1 per matched thread` (~30% cost reduction in addition to V5.4 baseline).

### Cost cap as architecture, not feature
LLM cascade runaway scenarios (multi-model outage forcing repeated Pro 2.5 fallbacks) are operationally rare but financially catastrophic. Hard 3 PLN cap per scan with `Kitsune-Cost-Deferred` label preserves work-in-progress for next scan while bounding worst-case spend. Visible audit trail; no silent failures.

### Session ID guard (frontend)
GAS callbacks are asynchronous and non-cancellable. A user resetting mid-analysis would trigger two concurrent result sets rendering simultaneously — ghost data in the DOM. Timestamped session IDs invalidate callbacks from superseded sessions before they render.

### `setText(" ")` instead of `setText("")`
Google Docs API throws `Cannot insert an empty text element` when calling `setText("")` on a `ListItem` or the last element in a document body. A single space is accepted as valid content, is visually invisible in rendered output, and preserves DOM integrity. Simple fix, non-obvious root cause.

### PropertiesService for all credentials
Zero hardcoded secrets. API keys live in `PropertiesService.getScriptProperties()`. Source code is safe to share without credential exposure.

### Router pattern for multi-view deployment
Single `doGet(e)` reads `?view=` parameter, routes to appropriate HTML file. Adding new views (Auto-Drafts, Genesis UI) requires one `if` branch, zero deployment overhead. URL stays stable; users bookmark once, work forever.

---

## Cost model

| | |
|---|---|
| Infrastructure | **$0** — native Google Workspace, no servers, no SaaS |
| Per full analysis (Stage 1 + Stage 2 + archive) | **≈4 gr PLN (≈$0.01)** — measured in production, Gemini 3 Flash PAYG |
| Per Gmail scan (~19 threads) | **≈0.7 PLN (≈$0.18)** — V6.1 inverted discovery |
| Cost cap per scan (hard ceiling) | **3 PLN (≈$0.75)** — runaway protection |
| Previous architecture (V5.4 per-row scan) | ≈1.5 PLN per scan (variable, selective) |
| 200 analyses/month + daily scans | **≈15 PLN (≈$3.75)** |
| Comparable SaaS (Teal, Jobscan, Huntr) | 30–150 PLN/month (≈$7–$38) |

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
                               + auto-creates Inbox Operations trigger (1h)
4. Deploy as Web App         → Execute as: Me · Who has access: Only myself
                               URL becomes both Main App (default) and
                               Dashboard (?view=dash)
5. Manual Gmail label colors → after first production scan:
                               Kitsune-Recruitment-Orphan → yellow
                               Kitsune-Unrelated → gray
                               Kitsune-Cost-Deferred → red
```

> **Note:** `setupKitsuneEnvironment()` takes 10–15 seconds (creates Drive structures + Doc template + trigger). The UI locks with a visual progress overlay during this time. If it appears frozen, it is working.

> **Triggers:** The Inbox Operations time-based trigger is created automatically during `setupKitsuneEnvironment()`. Existing triggers for the same handler are deleted first, so the function is safe to run multiple times without duplicating triggers. No manual trigger configuration required.

> **Re-deployment:** GAS Web Apps require explicit version bumps. After code changes: `Manage deployments → Edit (✏️) → Version: New version → Deploy`. URL stays stable across versions; bookmarks survive.

---

## Roadmap

| Version | Status | Scope |
|---------|--------|-------|
| **V5.4** | ✅ Stable (legacy) | Core SSoT · Dual-Stage Analysis · Inbox Operations V1 · Defense-in-Depth · One-Shot Stage 2 |
| **V6.0** | ✅ Stable | Standalone HTML Web Dashboard · Router pattern · Cross-view navigation |
| **V6.1** | ✅ Stable (current) | Inverted Discovery Scanner · ATS detection · Lenient domain match · Stop-words filtering · Cost cap · Dashboard logo + inline scan |
| **V6.2** | 📋 Planned | Auto-Drafts Engine — autonomous recruiter response drafts based on SSoT context and SLA triggers |
| **V6.3** | 🔬 Backlog | Smart Stage 1 fallback (no "Not Specified" placeholders) · UNRELATED noise vs true unrelated distinction · Notification badge in Main App |
| **Kitsune Genesis** | 🔬 Separate R&D | Multi-modal Candidate DNA profiling — CV + interview synthesis → Synarch Profile. Separate project, feeds into Kitsune-OS as injection layer in future integration. |

---

## Repository structure

```
kitsune-os/                  ← this repo (public, portfolio showcase)
  ├── README.md              ← architecture documentation (this file)
  ├── kitsune-portfolio.html ← live portfolio / case study
  └── assets/                ← logos, screenshots

kitsune-os-source/           ← private repo (IP protected)
  ├── Code.gs                ← backend (~1,500 lines after V6.1 refactor)
  ├── index.html             ← Main App frontend (~1,000 lines)
  └── dashboard.html         ← V6.0+ Dashboard frontend (~600 lines)
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
