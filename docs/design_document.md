# Vorbote — Document 1: Design Document

| | |
|---|---|
| **Title** | Vorbote — Design Document |
| **Version** | 2.0.0 |
| **Date** | 2026-10-26 |
| **Status** | Approved |
| **Governs** | All later documents (Doc 2 Technical Spec, Doc 3 Roadmap) |
| **Basis** | Change-driven opportunity discovery — fully defined in §1.2 (self-contained; no external document assumed) |

**Revision History**

| Version | Date | Status | Changes |
|---|---|---|---|
| 1.0.0 | 2026-10-26 | Rejected | Initial issue |
| 2.0.0 | 2026-10-26 | Draft | Complete rework after operator rejection: (1) document made fully self-contained — the discovery method is now defined inline in §1.2; all external-document references removed; (2) §12 Initial Source Register added with exact endpoint URLs, verification status, and URL-change handling — exact initial provisioning is this document's committed task, not an open question; (3) dossier export fixed to Markdown/HTML only (confirmed); (4) session-digest artifact committed as a requirement (FR-33) and component responsibility; (5) open questions reduced to three implementation-time verifications, none architecture-gating; (6) §10 extended (D17, D18); (7) all opportunity-record and constraint terminology made self-contained (no "stage" jargon from any external source) |

---

## 1. Overview & Goals

### 1.1 What Vorbote is

Vorbote is a **personal, single-user AI intelligence system** that continuously watches the German and EU economic landscape for **change** — regulatory, technological, demographic, industry-structural, behavioral, and platform changes — and converts detected changes into **evaluated, evidence-backed small-business opportunities**, then **guides the operator** through investigating and validating them.

The system does **not** generate business ideas. It detects observable market signals, links every finding to stored evidence, proposes economic explanations (always with alternatives preserved), and drives a disciplined funnel:

**Observations → Hypotheses → Serious investigations → Commercial experiments**

The name is the design philosophy: a *Vorbote* (harbinger) is the detectable signal that precedes the change — Vorbote studies transitions, not static markets.

### 1.2 The discovery method Vorbote implements (normative, self-contained)

Vorbote implements **change-driven opportunity discovery**: looking for economic changes that create new requirements faster than existing suppliers can adapt. It is one of several systematic discovery approaches (demand–supply imbalance detection, dissatisfaction mining, arbitrage detection, change-driven discovery, following demonstrated spending); the operator selected change-driven discovery as Vorbote's engine, with machinery shared across those approaches — a candidate database, weighted scoring, backward economics, and a validation procedure — built in as Vorbote's evaluation layer.

**Change drivers watched (the six):**

| Change driver | What creates the opportunity |
|---|---|
| Technology | New tools/platforms change what's economically feasible |
| Regulation | New rules create compliance needs and deadlines |
| Demographics | Population shifts change demand structure |
| Industry restructuring | Closures, mergers, insolvencies free customers and staff |
| Consumer behavior | Longitudinal shifts in what people search and buy |
| Platform changes | Marketplace policies, API changes, advertising rules redistribute access |

**The signal chain (every candidate is traced through it):**

**Change → affected population → new problem → existing response → unresolved demand → commercial feasibility**

For a detected change, Vorbote's records must answer, in order: which organizations/people are affected; what they must now do differently; what deadlines exist; how they currently respond (existing alternatives); whether demand remains unresolved; whether an economically attractive gap remains for a small operator.

**Funnel discipline:** many observations → fewer hypotheses → few serious investigations → one commercial experiment at a time. Progressive increase of evidence quality while reducing candidate count. No auto-promotion anywhere: every funnel transition is an operator decision, optionally guided.

**Evidence rules:** every finding links to original stored evidence, carries a proposed economic explanation **plus alternative explanations** (a price spike may be shortage, seasonality, spec change, or sample bias), and records what remains unknown. The LLM's role is structured information extraction — never business ideation, never invented market-size numbers. **Negative findings are preserved with equal standing** (why an apparent inefficiency is economically rational), preventing repeated rediscovery of false opportunities.

**The incumbent question (central to every candidate):** *why hasn't competition eliminated this inefficiency?* Possible answers — information hard to obtain; transactions too small for large operators; geographic/linguistic separation; organizational overhead; supplier loyalty; coordination cost; regulatory barriers; capital constraints — distinguish a temporary anomaly from a durable, small-operator-exploitable advantage.

**Evaluation model (deterministic, operator-owned):**
- *Operator constraints* (capital, hours/week, target monthly profit, horizon, acceptable operational burdens, sales comfort, automation tolerance, unacceptable risks) drive **hard exclusions** — a high score never compensates a fatal problem.
- *Weighted scoring*, criteria scored 1–5: evidence of actual spending 25%, accessibility of customers 25%, potential contribution margin 20%, operational compatibility 15%, sustainable competitive advantage 10%, speed/affordability of validation 5%. These weights are research-prioritization settings, not empirical success predictors; they are config, not constants. Customer accessibility is evaluated separately from demand — a market can be strong and commercially inaccessible to this operator.
- *Backward economics*: contribution per transaction (revenue ex-VAT − variable costs: procurement, delivery, payment processing, commissions, returns, acquisition, variable labor); monthly operating profit (transactions × contribution − fixed costs); **effective hourly earnings** (profit ÷ total owner-hours incl. sourcing, admin, marketing, support); and the **reverse calculation** — from required profit and realistic capacity to the number of paying customers needed, then "can I realistically acquire and serve that many?" Unit-economics inputs are operator estimates or evidence-extracted values with citation — never LLM-invented.
- *Validation procedure*: an evidence hierarchy from weakest (LLM suggests a market) through moderate (customers demonstrably pay existing providers) to strongest (repeat purchases with profitable unit economics); customer interviews about **actual past purchasing** (specific experiences, workarounds, problem frequency, real budgets — never speculative interest); 5–10 focused initial conversations; experiments that test the **riskiest assumption** (concrete paid offer; priced offer; small acquisition campaign; several manually fulfilled transactions; repeat-purchase test; effort measurement); **failure criteria fixed in advance and immutable per experiment** (max research/validation spend, minimum contribution margin, max fulfillment time, max acquisition cost, required willingness-to-pay evidence); falsification-first — for every candidate record "what evidence would convince me this is not viable," then seek it; distinguish a failed proposition from a failed test (unrepresentative traffic says little about the market).

### 1.3 Goals

1. **Continuous discovery** — one scheduled daily sweep (plus per-source-group schedules, with catch-up) across a multi-family source platform; not episodic "what business should I start?" sessions.
2. **Evidence discipline** — every observation, hypothesis, and candidate carries evidence links, proposed + alternative explanations, and unknowns; negative findings preserved.
3. **Deterministic-first** — collection, diffing, arithmetic, scoring, thresholds, scheduling, metering are deterministic software; the LLM does classification, extraction, synthesis, hypothesis generation, research planning — never numbers, never judgments arithmetic can make.
4. **Guided evaluation** — conversational guidance engine plus structured triage, dossier, scoring, and validation surfaces.
5. **Economically literate validation** — per §1.2's evaluation model.
6. **Bounded, observable cost** — per-call metering, warn-only ceilings, monthly ledgers.

### 1.4 System in one picture

```text
┌─────────────────────────── SERVER (headless, Linux) ───────────────────────────┐
│                                                                                │
│  SOURCE PLATFORM (§12)              PROCESSING                 EVALUATION       │
│  ┌───────────────────┐   ┌──────────────────────────────┐   ┌───────────────┐  │
│  │ Adapter families  │   │ Normalization (canonical     │   │ Observations  │  │
│  │  · feeds (RSS)    │   │  item store + snapshots)     │──▶│  (evidence-   │  │
│  │  · REST APIs      │──▶│ Change detection             │   │   linked)     │  │
│  │  · HTML diff      │   │  (deterministic diffs, ranks, │   │      │        │  │
│  │  · datasets       │   │   new-item detection,         │   │      ▼        │  │
│  │  · change feeds   │   │   snapshot comparison)        │   │ Hypotheses    │  │
│  │  · community      │   │ LLM extraction/classification │──▶│ (economics + │  │
│  │    (agent-reach)  │   │ LLM hypothesis generation    │   │  alternatives)│  │
│  │  · marketplace    │   └──────────────────────────────┘   │      │        │  │
│  │    HTML diff      │            ▲                         │   ▼ Candidates │  │
│  └───────────────────┘   ┌───────┴────────┐                │  (scored,     │  │
│         ▲                │ LLM (single    │                │   triaged)    │  │
│  Retrieval ladder:       │  path, one     │                │      │        │  │
│  deterministic+politeness│  model, per-   │                │      ▼        │  │
│  → backoff → Exa → LLM   │  stage metering│                │ Investigations│  │
│  └───────────────────────┘                │                │ + experiments │  │
│                                           │                │ (validation   │  │
│  ONE SQLite DB + snapshot store (disk)    │                │  tracker)     │  │
│  Scheduler (daily sweep, groups, catch-up)│                └───────┬───────┘  │
│  Retention (12 mo, FIFO, pinned)          │                        │          │
│  Metering ledgers (LLM + Exa, warn-only)  │                        │          │
│  Loopback JSON API ◀──────────────────────┼────────────────────────┘          │
└───────────────────────────────┬────────────────────────────────────────────-┘
                                │ SSH local forward (trust model)
┌───────────────────────┬───────┴──────────┐
│  DESKTOP CLIENT (PySide6)                 │
│  Triage · Dossier · Scoring/economics ·   │
│  Validation tracker · Constraints editor ·│
│  Source manager · Guidance chat ──────────┼──▶ Conversational guidance engine
│  (bilingual UI, EN default)               │     (server-side, bounded writes,
└──────────────────────────────────────────-┘      session digest per session)
```

### 1.5 Operator

Exactly one operator (the owner): configures sources, performs guided setups, triages observations, confirms scores and evidence, conducts interviews and commercial experiments manually, chats with the guidance engine. No other users; no accounts, roles, or sharing.

---

## 2. Requirements Summary

Every requirement below governs every later section of every document. Traceability tags: **OR** operator requirement, **CF** confirmed interview answer, **AS** stated assumption, **PD** proposed design decision. All OR/CF items were captured in the four interview rounds; §1.2 makes their method content self-contained.

### 2.1 Functional requirements

| ID | Requirement | Tag |
|---|---|---|
| FR-01 | Implement change-driven opportunity discovery (§1.2): detect changes across the six change drivers; resolve each through the signal chain Change → affected population → new problem → existing response → unresolved demand → commercial feasibility | OR |
| FR-02 | Operate a continuous discovery process (not episodic): scheduled collection, change detection, evidence extraction, hypothesis generation, opportunity database, prioritization | OR |
| FR-03 | Multi-family source platform per the §12 Initial Source Register: RSS/Atom feeds; JSON REST APIs; HTML listing-page diff; dataset snapshot + change detection; structured change feeds; community sources (Reddit, X/Twitter, Hacker News); Google Trends; Amazon marketplace pages; optional credentialed sources (Google Keyword Planner; Amazon SP-API) | OR/CF |
| FR-04 | Jurisdictional scope: Germany + EU; languages: German and English; open scan across all sectors | CF |
| FR-05 | Free/official sources only; credentialed optional sources documented with guided setup and **disabled by default** until set up | CF |
| FR-06 | eBay Product Research and Etsy Marketplace Insights are **not** sources (dropped by operator) | CF |
| FR-07 | Community sources are first-class, high-priority; X/Twitter accepted as experimental-class | CF |
| FR-08 | PDF extraction in v1 (gazettes, regulator bulletins) — text layer only; no OCR | CF |
| FR-09 | Retrieval ladder, strictly ordered: (1) deterministic fetch with netizen discipline — never multiple concurrent queries to one source, sensible inter-request delays per source, exponential backoff; (2) Exa.ai fetch/search on failure; (3) LLM-driven retrieval for deep investigation riding the same ladder — the LLM never fetches directly | CF |
| FR-10 | Change detection is deterministic (diffs, new-item detection, rank movements, snapshot comparison); the LLM never "detects" a change | PD (from CF) |
| FR-11 | Observations carry original evidence links, a proposed economic explanation, **alternative explanations**, and what remains unknown | OR |
| FR-12 | Negative findings preserved (why an apparent inefficiency is economically rational) with the same record structure as observations | OR |
| FR-13 | Opportunity database records, per candidate: market/segment, observed inefficiency, discovery source, evidence, existing alternatives, estimated customer expenditure, competition, potential advantage, customer access, estimated unit economics, operational complexity, confidence level, next experiment (full schema §5) | OR |
| FR-14 | Funnel discipline per §1.2, with operator-decided transitions; no auto-promotion | OR |
| FR-15 | First-run, optionally guided **constraints editor** (capital, hours/week, target monthly profit, horizon, operational burdens, sales comfort, automation tolerance, unacceptable risks); editable anytime; drives hard exclusions + operational-compatibility scoring | CF |
| FR-16 | Weighted scoring 1–5 across six criteria per §1.2 (25/25/20/15/10/5); weights are config; hard exclusions applied before weighted score | OR |
| FR-17 | Scoring arithmetic deterministic; LLM may draft criterion rationale, operator confirms values; a high weighted score never compensates a fatal exclusion | OR/PD |
| FR-18 | Backward economics per candidate per §1.2: contribution per transaction, monthly operating profit, effective hourly earnings, reverse calculation; inputs tagged operator-estimate or evidence-ref; C-08 forbids LLM-invented numbers | OR |
| FR-19 | Validation guidance per §1.2: evidence hierarchy, past-purchasing interview guides, riskiest-assumption experiments, immutable failure criteria, proposition-vs-test distinction | OR |
| FR-20 | Falsification-first: record "what evidence would convince me this is not viable" per candidate before further investment | OR |
| FR-21 | The incumbent question recorded and answered per candidate: why hasn't the market eliminated this inefficiency? | OR |
| FR-22 | Interactive surfaces in a self-contained PySide6 desktop client: triage queue, candidate dossier, scoring/economics view, validation tracker, constraints editor, source manager with setup wizards, settings | CF |
| FR-23 | Built-in **conversational guidance engine**: LLM session reading the database, interviewing the operator, drafting scores/evidence/experiments, updating records under **bounded direct writes** — notes and disposition status written directly; scores, evidence records, and deletions only after explicit in-chat confirmation | CF |
| FR-24 | Guidance sessions persisted as audit trail (messages, every write with proposal → confirm/decline record) | PD (from CF) |
| FR-25 | **Session digest artifact per guidance session**: a deterministic summary generated at session end — topics covered, proposals raised/confirmed/declined, writes made, open questions — stored as a first-class record and viewable in the client | CF |
| FR-26 | Dossier export formats: **Markdown and HTML only** | CF |
| FR-27 | Bilingual UI, switchable, English default; source excerpts always preserved verbatim in original language | CF |
| FR-28 | One daily sweep; per-source-group schedules; catch-up of missed runs | CF |
| FR-29 | Alerting on high-priority candidates is **passive only** — triage queue; no email/notifications | CF |
| FR-30 | Retention: raw snapshots/full-text/PDF captures of non-selected items kept 12 months (configurable); anything referenced by an observation/hypothesis/candidate/investigation/experiment pinned indefinitely; FIFO eviction with warning; configurable storage ceiling | CF |
| FR-31 | LLM via OpenRouter, OpenCode Go, DeepSeek, or a custom OpenAI-compatible endpoint; **one model for all roles for now** (glm-5.3-flash default); roles differ by prompt, not model | CF |
| FR-32 | LLM cost ceiling $50/month (configurable), **warn-only**; metered per stage per run at configurable blended prices (default $0.04/M input, $0.60/M output); monthly ledger persisted | CF |
| FR-33 | Exa: own configurable monthly ceiling (warn-only), own ledger, price-per-call in config; **no key → Exa-dependent behavior degrades to deterministic-only**; Exa-required sources quarantine visibly | CF |
| FR-34 | Everything that can reasonably/sensibly be done deterministically **will** be done deterministically | CF |
| FR-35 | Initial source provisioning with exact endpoints is committed by this document (§12); URL drift is an expected, handled condition (URL-change notice in §12.3) | CF |

### 2.2 Non-functional requirements

| ID | Requirement | Tag |
|---|---|---|
| NFR-01 | Single user, personal software; no multi-user concerns anywhere | CF |
| NFR-02 | Standalone from first principles: own repo, own process, **exactly one SQLite database** (plus unstructured snapshot files on disk); no other codebase used, imported, integrated, or referenced — all patterns are Vorbote's own design decisions | CF |
| NFR-03 | Server: Linux host, always-on (VPS assumed); client: operator's Linux desktop | AS |
| NFR-04 | Read-only netizen: Vorbote never posts, comments, likes, messages, or writes to any external platform | CF/PD |
| NFR-05 | Secrets (API keys, OAuth client secrets) live only in the server's secret file: never in the database, never in config, never in logs, never in API responses; no secret-setting API endpoint; client shows presence/freshness only | PD |
| NFR-06 | Network-facing surface: loopback-only JSON API; the SSH local forward is the trust model (no TLS, no tokens); localhost Host/Origin gates as defense-in-depth | PD/AS |
| NFR-07 | Cost observability: both ledgers inspectable via client, per-group and per-stage breakdown | FR-32/33 consequence |
| NFR-08 | Graceful degradation: failing source quarantines without failing the sweep; missing optional credential disables that source; LLM unavailability leaves collection and change detection fully functional; extraction retries idempotently next sweep | PD |
| NFR-09 | Python 3.12+ (server and client); SQLite; PySide6 client | AS/PD |
| NFR-10 | Timestamps UTC ISO-8601 in storage; UI renders local timezone | PD |
| NFR-11 | Test suite hermetic: no live network in tests; fixture HTTP servers; pytest the only automated gate | PD |
| NFR-12 | Self-contained documentation: no concept, method, or artifact referenced by Vorbote's documents requires any external document to understand | CF |

### 2.3 Constraints (hard boundaries)

| ID | Constraint | Tag |
|---|---|---|
| C-01 | No paid research subscriptions; free/official sources only (Keyword Planner, SP-API are free-tier credentialed optional resources) | CF |
| C-02 | $50/month LLM ceiling, warn-only — the system must make approaching/exceeding it visible | CF |
| C-03 | One model for all LLM roles (glm-5.3-flash) — architecture keeps model-per-role switchable later via config only | CF |
| C-04 | Daily cadence, not real-time | CF |
| C-05 | The system never spends money and never executes transactions; commercial experiments are manual operator actions — the system tracks, guides, records | OR/CF |
| C-06 | No email delivery; passive triage only | CF |
| C-07 | Exactly one SQLite database | CF |
| C-08 | The system does not invent numbers: market sizes, prices, volumes, prevalence are evidence-extracted with citation or marked unknown — never LLM-estimated | OR |
| C-09 | Document set self-contained per NFR-12; no external-document references | CF |

### 2.4 Explicit non-goals

No multi-user/product/SaaS ambitions · no idea-noise generation ("50 ideas every morning") · no transaction execution, purchasing, bidding, or posting · no real-time/streaming ingestion · no OCR (scanned PDFs stored and flagged, not parsed) · no email notifications · no dossier export formats beyond Markdown/HTML · no other codebase used, imported, or referenced · nothing external to Vorbote (any chat tool or third-party service beyond the source platforms and LLM/Exa providers themselves) participates in its operation.

---

## 3. System Architecture

### 3.1 Process topology

Two processes, one trust boundary:

| Process | Runs on | Role |
|---|---|---|
| **Vorbote server** (`vorbote serve`) | Linux host | Headless daemon: scheduler, source platform, processing pipeline, evaluation engine, guidance engine, retention, metering, loopback JSON API |
| **Vorbote client** | Linux desktop | PySide6 GUI: all interactive surfaces; HTTP to the server's loopback API through an SSH local forward it manages itself |

The client spawns/manages the SSH local forward (`ssh -N -L <port>:127.0.0.1:<api-port> <user>@<host>`) and only ever talks to its own loopback. The tunnel **is** the security model (§7).

### 3.2 Server-internal architecture (layered, one direction of dependency)

```text
CLI / serve entry  ──▶  Scheduler  ──▶  Orchestrator (run engine)
                                                    │
        ┌───────────────────────────────────────────┼────────────────────────────────┐
        ▼                                           ▼                                ▼
  Source Platform                            Processing Pipeline              Evaluation Engine
  (adapter registry, groups,                  (normalizer, change detector,     (observation manager,
   retrieval ladder, politeness,              extractor [LLM], hypothesis       hypothesis manager,
   Exa client, agent-reach bridge)             generator [LLM], evidence         candidate manager,
                                              store bridge)                     scoring, economics,
                                                                                validation manager)
        │                                           │                                │
        └─────────────────────▶ Persistence (the ONE SQLite DB + snapshot store) ◀────┘
                                                    │
                                    Retention (pruner)  ·  Metering (LLM + Exa ledgers)
                                                    │
                                    Guidance Engine (sessions, role prompts,
                                     bounded-write controller, audit trail,
                                     session digests)
                                                    │
                                    Loopback JSON API (127.0.0.1)
```

Rules: dependencies point downward only; CLI and API are the only entry points; every cross-layer handoff is a named seam (registered in Doc 2 §15); one composition root constructs all infrastructure clients (DB handle, LLM client, Exa client, agent-reach bridge).

### 3.3 The daily sweep (orchestration)

1. Scheduler fires the daily sweep (default 05:00 server time [AS-02]) and per-source-group schedules; missed windows are caught up at startup (bounded to one catch-up per missed window).
2. Orchestrator plans the sweep: due sources, health state, quarantines, disabled-by-missing-credential sources.
3. Each source's adapter executes its **fetch plan** through the retrieval ladder (§3.4) under per-source serialization, inter-request delay, exponential backoff.
4. Fetched content normalizes to **canonical items**; full-text/PDF captures persist to the **snapshot store** (files on disk; path + content hash in the DB).
5. **Change detection** (deterministic) compares against prior state: new items, listing diffs, rank movements, value deltas vs thresholds, snapshot diffs.
6. Change events → LLM **extractor** (classification: driver taxonomy, affected-population hints, jurisdiction, sector, language) → LLM **hypothesis generator** → **observations** with proposed + alternative explanations and unknowns, each hard-linked to evidence.
7. Observations land in the **triage queue**. No auto-promotion (FR-14); promotion is an operator decision, optionally guided.
8. Metering accumulates per stage; warnings surface in the client.

### 3.4 Retrieval ladder (normative order)

For every outbound retrieval attempt:

1. **Deterministic fetch** — HTTP GET / API call / feed parse / HTML fetch + extraction / PDF text-layer extraction. Governed by: per-source single-flight, inter-request delay (configurable per source; sensible defaults per source class [AS-02]), bounded exponential backoff honoring `Retry-After`, honest browser-like User-Agent, robots-aware behavior.
2. **Exa fallback** — on deterministic failure after bounded retries (or for sources configured Exa-required): Exa fetch/search, metered in the Exa ledger; no key → deterministic-only, Exa-required sources quarantined with visible status.
3. **LLM-driven retrieval** — during deep investigation only (on-demand, not in the daily sweep): the guidance engine formulates queries; those queries execute through steps 1–2. The LLM never fetches directly.

### 3.5 Evaluation engine (deterministic core + LLM assist)

- **Observation manager** — evidence-linked records from change events; negative findings with identical structure.
- **Hypothesis manager** — proposed explanation, ≥1 alternative, unknowns, incumbent-question candidates (§1.2's explanation taxonomy as starter set).
- **Candidate manager** — opportunity records (FR-13 fields, §5 schema); dossier assembly.
- **Scoring engine** — deterministic weighted arithmetic; exclusions before score; criterion values operator-confirmed; LLM drafts rationale only.
- **Economics engine** — deterministic calculators per §1.2; inputs tagged operator-estimate or evidence-ref (C-08).
- **Constraints manager** — versioned operator constraints; exclusion + compatibility inputs.
- **Validation manager** — investigations, past-purchasing interview guides, experiments with riskiest assumption, immutable failure criteria, result recording, proposition-vs-test verdicts, falsification notes.

### 3.6 Guidance engine (conversational)

An on-demand LLM session hosted server-side, exposed through the client's chat surface:

- **Context assembly:** deterministic — the session assembles relevant DB context (dossier, observations, evidence, constraints, scores); the LLM never queries the DB ad hoc.
- **Roles:** interviewer (targets unknowns), analyst (drafts explanations/alternatives), research planner (proposes queries — executed via the ladder), scribe (drafts records).
- **Write boundary (bounded direct writes):** notes and disposition status written directly; scores, evidence records, deletions emitted as **proposals** confirmed/declined in-chat. Every write (direct or confirmed) lands in the audit trail.
- **Session digest (FR-25):** generated deterministically at session end — topics, proposals raised/confirmed/declined, writes made, open questions; stored as a first-class record; viewable in the client.
- **Model:** the single configured model (glm-5.3-flash), role-specific system prompts.

### 3.7 Retention

Daily prune pass after the sweep: non-selected raw items and snapshots older than 12 months (configurable) → FIFO eviction with a warning report (count + bytes); **pins block eviction** — anything referenced by an observation, hypothesis, candidate, investigation, or experiment is retained indefinitely. Storage ceiling (default 10 GB [AS-02]) configurable; exceeding it triggers the warning path (warn-only).

---

## 4. Component Responsibilities

| Component | Responsibility |
|---|---|
| **CLI / serve entry** | Parse commands; construct composition root; start API + scheduler |
| **Scheduler** | Fire daily sweep + group schedules; catch-up; startup sweep of interrupted runs |
| **Orchestrator** | Plan/execute sweep runs; group sequencing; failure isolation; run state transitions |
| **Source registry** | Source definitions, group membership, health/quarantine state, credential-requirement flags |
| **Adapter families** | Turn a source definition + fetch plan into canonical items + snapshots; adapter-local parsing only |
| **Retrieval ladder executor** | Enforce ladder order, per-source serialization, delays, backoff, retry budgets |
| **Politeness governor** | Per-source rate state; sole authority on "when may I hit this source next" |
| **Exa client** | Fetch/search via Exa API; metered; absent key → disabled state |
| **agent-reach bridge** | Execute read-only channel actions (Reddit, X) via the adapted reach CLI; interpret its JSON result/error contract; map error codes to source states (login needed, rate-limited, blocked → quarantine) |
| **Normalizer** | Canonical item schema; language detection; jurisdiction tagging helpers |
| **Snapshot store** | Content-addressed file store (full text, PDFs); hash + metadata in DB |
| **Change detector** | Deterministic diff/rank/threshold/snapshot comparison → change events |
| **Extractor (LLM)** | Structured classification/extraction from evidence text; strict output contract; degrade-to-raw on contract violation |
| **Hypothesis generator (LLM)** | Observations with proposed + alternative explanations + unknowns; never numbers (C-08) |
| **Observation manager** | Triage queue lifecycle; negative findings |
| **Hypothesis manager** | Hypothesis records; promotion flows |
| **Candidate manager** | Opportunity records (FR-13); dossier assembly |
| **Scoring engine** | Deterministic weighted score + exclusions |
| **Economics engine** | Deterministic calculators (FR-18) |
| **Constraints manager** | Versioned operator constraints |
| **Validation manager** | Investigations, interview guides, experiments, failure criteria, results |
| **Guidance engine** | Sessions, context assembly, role prompts, proposal/confirm protocol, bounded writes, audit trail, **session digests** |
| **LLM client** | The single path for chat completions; per-stage metering; never logs secrets |
| **Metering** | Two ledgers (LLM, Exa); monthly rollups; warn thresholds |
| **Retention** | Prune pass; pin enforcement; storage ceiling warnings |
| **Persistence** | The one SQLite DB: schema, migrations, transactions |
| **Loopback JSON API** | Client surface: all CRUD/read endpoints; localhost Host/Origin defense-in-depth |
| **Client: shell/tunnel** | Window, tunnel management, API client, language switch |
| **Client: triage view** | Observation queue, filters, promote/decline/negative |
| **Client: dossier view** | Candidate record + evidence browser; Markdown/HTML export |
| **Client: scoring/economics view** | Criteria entry + rationale, deterministic score display, economics calculators incl. reverse calculation |
| **Client: validation view** | Investigations, experiments, failure criteria, results |
| **Client: constraints editor** | Optionally-guided first-run; editable anytime |
| **Client: source manager** | Source/group management, health view, guided setup wizards (incl. agent-reach login flow), credential status (never secret values) |
| **Client: guidance chat** | Session UI; proposal cards confirm/decline; session digest + audit browsing |
| **Client: settings & ledgers** | Server settings mirror, ceilings, usage views |

---

## 5. Data Model

One SQLite database (`vorbote.db`). Canonical tables (attribute-level; DDL in Doc 2 §5):

**Source platform**
- `sources` — id, name, family (feed/rest_api/html_diff/dataset/change_feed/community/trends/marketplace), group_id, endpoint/config blob (JSON; no secrets), language(s), jurisdiction, cadence override, enabled, requires_credential, health (ok/quarantined/disabled), quarantine_reason, last_success_at.
- `source_groups` — id, name, schedule window spec, notes.
- `source_credentials_status` — source_id, credential_kind, configured (bool), last_verified_at. **No secret material** (NFR-05).

**Runs & retrieval**
- `sweep_runs` — id, started_at, finished_at, status (running/completed/failed/interrupted), trigger (scheduled/catch-up/manual), stats JSON.
- `source_runs` — id, sweep_run_id, source_id, status, items_fetched, errors JSON, ladder usage summary.
- `fetch_attempts` — id, source_run_id, ladder_step, started_at, outcome, http_status, retry_count, delay_applied_ms, exa_cost_event_id (nullable).

**Canonical content**
- `raw_items` — id, source_id, external_key (dedupe), title, url, author, published_at, fetched_at, language, jurisdiction, content_hash, snapshot_id (nullable), parse_tier (full_text/summary/listing_entry/pdf_text/pdf_flagged_no_text), payload pointer.
- `snapshots` — id, store_path, content_hash, bytes, mime, captured_at, pinned.

**Change detection**
- `change_events` — id, source_id, detected_at, kind (new_item/list_diff/rank_movement/value_delta/snapshot_diff), before_state JSON, after_state JSON, magnitude, raw_item_ids JSON.

**Evidence & funnel**
- `observations` — id, change_event_ids, created_at, title, description, affected_population (draft), driver tags, proposed_explanation, alternative_explanations JSON, unknowns JSON, status (triage/promoted/declined/negative_finding), language, evidence JSON, guidance_session_refs.
- `hypotheses` — id, observation_id, statement, basis (evidence refs), incumbent-response analysis, unresolved-demand assessment, status.
- `candidates` — id, hypothesis_id, created_at, market_segment, observed_inefficiency, discovery_source, evidence refs, existing_alternatives, estimated_customer_expenditure (value + confidence + source), competition, potential_advantage, customer_access, unit_economics (operator/evidence inputs), operational_complexity, confidence level, next_experiment, status (active/investigating/experimenting/shelved/abandoned), pin state.
- `evidence_records` — id, owner_type, owner_id, raw_item_id / snapshot_id / url, excerpt, note, created_by (operator/guidance), confirmed.
- `exclusions` — id, candidate_id, constraint_id, reason.

**Scoring & economics**
- `constraints` — id, version, capital, hours_per_week, target_monthly_profit, horizon_months, operational burdens, sales comfort, automation tolerance, unacceptable risks (structured), created_at, active.
- `scores` — id, candidate_id, constraint_version, six criterion values (1–5) each with rationale + author (operator/guidance-confirmed), weighted_total, computed_at.
- `economics_records` — id, candidate_id, inputs JSON (each tagged operator-estimate or evidence-ref), outputs JSON (contribution, monthly profit, hourly earnings, required transactions), computed_at.

**Validation**
- `investigations` — id, candidate_id, plan, unknowns targeted, status, findings JSON.
- `interview_guides` — id, investigation_id, questions (past-purchasing focused), recording notes.
- `experiments` — id, candidate_id, riskiest_assumption, design, failure_criteria JSON (immutable once set), budget/time caps, status, result, proposition_vs_test verdict.
- `falsification_notes` — id, candidate_id, not-viable evidence statement, created_at, resolved_at, resolution.

**Guidance**
- `guidance_sessions` — id, started_at, ended_at, focus ref, token accounting refs, digest_id (nullable until close).
- `guidance_messages` — id, session_id, role (operator/assistant), content, created_at.
- `guidance_proposals` — id, session_id, action, payload, state (proposed/confirmed/declined), decided_at.
- `guidance_digests` — id, session_id, topics JSON, proposals summary JSON, writes JSON, open questions JSON, created_at. (FR-25)
- `write_audit` — id, session_id nullable, actor (operator/guidance), action, table + record, before/after JSON, at. **Every guidance write lands here.**

**Metering & ops**
- `llm_usage` — id, at, stage, run_id/session_id, model, prompt_tokens, completion_tokens, cost estimate.
- `exa_usage` — id, at, source_id/run_id, kind (fetch/search), cost, success.
- `ceiling_states` — ledger kind, month, budget, spent, warned_at.
- `pins` — snapshot_id/raw_item_id, reason, created_at.
- `schema_migrations` — applied history.

All JSON blobs internally schema-versioned; C-07 honored (only DB; snapshot bytes on disk under the store).

---

## 6. API & Interface Design

Loopback JSON API (default port 8471 [AS-02]), single `/api/v1` namespace, JSON in/out, machine-readable error envelope. No auth layer (tunnel is the model; localhost Host/Origin gates as defense-in-depth).

Endpoint inventory (representative; full schemas Doc 2 §6):

- **System**: `GET /api/v1/status`; `GET /api/v1/health`
- **Sources**: CRUD `/api/v1/sources`, `/api/v1/source-groups`; `POST /api/v1/sources/<id>/test` (one polite fetch); `GET /api/v1/sources/<id>/setup-requirements`; `POST /api/v1/sources/<id>/quarantine|/release`
- **Sweeps**: `GET /api/v1/runs`, `GET /api/v1/runs/<id>`; `POST /api/v1/sweeps/trigger`
- **Triage**: `GET /api/v1/observations?status=…`; `GET /api/v1/observations/<id>`; `POST …/promote|/decline|/negative`
- **Candidates**: CRUD `/api/v1/candidates`; `GET /api/v1/candidates/<id>/dossier`; export Markdown/HTML (FR-26)
- **Scoring/economics**: `GET/POST /api/v1/candidates/<id>/scores`; `POST /api/v1/candidates/<id>/economics`
- **Constraints**: `GET/POST /api/v1/constraints`
- **Validation**: CRUD `/api/v1/investigations`, `/api/v1/experiments`; `POST /api/v1/experiments/<id>/result`
- **Guidance**: `POST /api/v1/guidance/sessions`; `GET …/messages`; `POST …/messages`; `GET …/proposals`; `POST /api/v1/guidance/proposals/<id>/confirm|/decline`; `GET /api/v1/guidance/sessions/<id>/digest`
- **Evidence**: `GET /api/v1/snapshots/<id>`; `POST /api/v1/pins`
- **Ledgers**: `GET /api/v1/ledgers/llm?month=…`, `GET /api/v1/ledgers/exa?month=…`

External interface contracts the design consumes: the adapted reach CLI's JSON contract (`items[]` with `title/url/author/published_at/text`; error codes `NEED_SETUP/NEED_LOGIN/RATE_LIMITED/PLATFORM_BLOCKED/TIMEOUT/BUSY/…` with stop-don't-retry semantics), Exa's public API, and OpenAI-compatible chat completions for the LLM.

---

## 7. Security Architecture

- **Trust model:** loopback-only API bind; SSH local forward is the entire security boundary; no TLS, no tokens (deliberate: personal software, one operator); localhost Host/Origin checks as defense-in-depth.
- **Secrets discipline (NFR-05):** server-side secret file is the only residence for LLM provider keys, Exa key, OAuth client secrets; never in DB/config/logs/API responses; no secret-setting endpoint; secrets placed by the operator over SSH; client shows presence/freshness only.
- **Read-only netizen (NFR-04):** no write actions on any platform; the reach adaptation is read-only by its own design; Vorbote adds no writes.
- **Source credentials & cookies:** the reach bridge runs against a headless Chromium-class browser on the server; no interactive login ever occurs there. Session state for credentialed community sources (Reddit, X) is provisioned by cookie transfer from the operator's desktop browser: the operator exports the session cookies of their existing logged-in accounts (e.g. via a cookie-export browser extension) and uploads the cookie strings through the PyQt client's setup wizard, which transmits them over the SSH tunnel; the server injects them into the reach bridge's browser profile. Cookie strings are secret material under the secrets discipline (NFR-05): stored only server-side in the bridge's browser profile, never in the database, never logged, never echoed back by any API response. When a platform reports the session invalid (bridge error class "login needed"), the source is quarantined with status login state stale and the wizard prompts for a fresh export and upload.
- **Content handling:** LLM prompts assembled from DB content contain no secrets; evidence text is untrusted input — mitigated by strict output contracts and deterministic post-validation.
- **Backups:** nightly `sqlite3 …VACUUM INTO` + snapshot-store rsync [PD], operator-scheduled.

---

## 8. Infrastructure & Deployment

- **Server host:** one Linux box (VPS assumed [AS-01]), systemd unit for `vorbote serve` (restart-on-failure, journal logging). Layout: checkout (or installed package) + state dir (env-var-relocatable) containing `vorbote.db`, snapshot store, config TOML, secret file.
- **Client host:** Linux desktop, PySide6; separate distribution from the same repo; SSH access to the server host.
- **Runtime dependencies (server):** Python 3.12+, HTTP client, feed parser, HTML text extraction, PDF text-layer extraction, SQLite, the reach adaptation (prerequisites: Node, a Chromium-class browser — documented in guided setup), Exa client, LLM over HTTP. Pinned set in Doc 2 §3.
- **Client dependencies:** PySide6, HTTP client. [PD]
- **No Docker, no containers, no build step.** [PD/AS]
- **Provisioning guide:** first-run init; source setup wizards; constraints editor; first sweep.
- **Scale envelope:** dozens of sources, thousands of raw items/day, single-digit GB store — SQLite and one process are correct; growth beyond personal scale is out of scope (C-04).

---

## 9. Operational Model

- **Daily rhythm:** sweep (default 05:00 [AS-02]) → change detection → LLM extraction/hypothesis → triage queue → passive operator review (FR-29). Retention prune post-sweep; ledger rollups at month boundary.
- **Operator sessions:** triage pass; dossier work (scoring with rationale confirmation, economics entry); validation work (interview guides → findings; experiment results); guidance chats (digest per session).
- **Source lifecycle:** add → test (one polite fetch) → enable → monitor; quarantine on repeated failure/rate-limit/login-expiry; wizard-driven re-setup; optional credentialed sources disabled until setup completes.
- **Failure & recovery:** source failures isolated (error recorded per source_run; quarantine after threshold); sweep failure ≠ source failure; interrupted runs swept at startup; LLM unavailability → collection/change detection continue, extraction retried idempotently next sweep.
- **Cost operations:** monthly ledger review; ceilings editable; warn-only means operations continue — the warning is the operator's decision point.
- **Observability:** structured logs per component; run stats; source health timeline; ladder usage counters; guidance write audit; all inspectable via API/client.

---

## 10. Key Design Decisions

| # | Decision | Rationale / alternatives rejected |
|---|---|---|
| D1 | Standalone, one-DB, own-process design; every pattern is Vorbote's own design decision; no other codebase used or referenced | Operator requirement (CF); independence and much wider scope than any single existing tool |
| D2 | Deterministic-first division of labor: diffs/arithmetic/scoring/metering deterministic; LLM only classification/extraction/synthesis/hypotheses/planning | FR-34, C-08; kills fabricated numbers by construction; alternative "LLM judges everything" rejected |
| D3 | Retrieval ladder: per-source serialization, delays, exponential backoff; Exa fallback; LLM never fetches directly | Operator netizen requirement; cost and politeness stay deterministic; keyless Exa → deterministic-only degradation (FR-33) |
| D4 | Reach-CLI adaptation for Reddit/X via server-side bridge; X accepted experimental-class; operator-performed logins | Operator requirement (logged-in cookies needed for both); verified read-only skill with clean error contract |
| D5 | One SQLite DB + content-addressed snapshot files; pins enforce retention | C-07; personal scale; alternatives (server DB, multiple DBs) rejected |
| D6 | Single-path LLM client; one model all roles; role = prompt; per-stage metering | FR-31/32, C-03; model-per-role switchable later via config only |
| D7 | Guidance engine: propose/confirm protocol; bounded direct writes; full audit; **session digest per session** | FR-23/25; deterministic-first and single-operator accountability |
| D8 | Weighted scoring as deterministic arithmetic, operator-confirmed criterion values; exclusions before score | FR-16/17; keeps the LLM out of judgment arithmetic |
| D9 | Loopback API + SSH-forward trust; secrets only in server secret file | NFR-05/06; proven single-operator pattern; no exposed surface to harden |
| D10 | No auto-promotion; funnel transitions are operator decisions (optionally guided) | FR-14; prevents queue flooding (idea-noise anti-goal) |
| D11 | Passive alerting (triage queue only) | CF; email rejected in interview |
| D12 | Bilingual UI with i18n layer; evidence verbatim in source language | CF; German sources are first-class inputs |
| D13 | Python 3.12+/SQLite/PySide6 stack | NFR-09; ecosystem fit; operator environment [AS] |
| D14 | Change detection fully deterministic; LLM stages consume only detected events | FR-10; auditable provenance from evidence to hypothesis |
| D15 | Negative findings as first-class records, same schema as observations | FR-12; prevents repeated false-opportunity discovery |
| D16 | Warn-only ceilings (LLM + Exa) with visible ledgers | CF; the operator decides at warning; never silently stops, never silently overspends invisibly |
| D17 | Initial Source Register with exact, verification-status-tagged endpoints (§12); URL drift handled as an expected condition via per-source URL override + wizard re-verification | FR-35; exact provisioning is a committed deliverable, not an open question |
| D18 | Session digest as a deterministic, server-generated artifact per guidance session | FR-25; complements (not replaces) the audit trail |

---

## 11. Open Questions & Assumptions

**Assumptions (carried into Doc 2 unless corrected):**
- [AS-01] Server host is a VPS (or always-on Linux box) reachable via SSH from the desktop; host/clock/timezone set at deployment.
- [AS-02] Defaults, all config: sweep 05:00 server time; inter-request delay 2–10 s by source class; storage ceiling 10 GB; retention 12 months; API port 8471.
- [AS-03] LLM provider endpoint/key/model naming (glm-5.3-flash as literal model id per provider) verified at setup; provider quirks handled by the single client path.
- [AS-04] Reach-bridge browser and session-state provisioning. The VPS runs a headless Chromium-class browser for the reach bridge — there is no interactive login surface there, and no login is ever performed on the server. Session state for credentialed community sources (Reddit, X) is instead transferred from the operator's desktop browser: the operator exports the session cookies of their existing logged-in accounts (e.g. via a cookie-export browser extension) and uploads the cookie strings through the PyQt client's UI (source-manager setup wizard). The client transmits them over the SSH tunnel to the server, which injects them into the reach bridge's browser profile. Cookie strings are secret material under the secrets discipline (NFR-05): stored only server-side in the bridge's profile, never in the database, never logged, never echoed back by the API. When a platform reports the session invalid (bridge error class "login needed"), the source is quarantined with status login state stale and the wizard prompts for a fresh cookie export and upload.
- [AS-05] No OCR in v1 — scanned PDFs flagged `pdf_flagged_no_text`, stored, excluded from extraction.
- [AS-06] Keyword Planner OAuth (Google Ads developer token, free tier) feasible; if not, the optional source stays disabled — no dependency.

**Open questions (implementation-time verifications; none architecture-gating):**
- OQ-1 Ministry press feeds: exact feed URLs are curated at setup via feed auto-discovery on each ministry site (§12.3 pattern); the register commits the family and the discovery procedure, not per-ministry URLs that churn with site redesigns.
- OQ-2 Amazon bot-wall posture: bestsellers page served content but movers-and-shakers returned 503 during design probes; the html_diff adapter starts conservative (slow cadence, honest UA) with Exa assist, and posture is revisited after the first live week.
- OQ-3 EUR-Lex feed endpoints: URL pattern documented; EUR-Lex serves a 202 challenge to plain HTTP clients during design probes, so live verification is a guided-setup step with the ladder's Exa fallback available.

Constraint re-check (§2 vs §§3–10): every FR/NFR/C has an owning section; no section assumes a capability §2 excludes — C-08 structurally enforced by D2/D8; C-07 by §5's single-DB design; C-09 by §1.2 and this document's self-containment; C-05 by §3.6/§3.7 (guidance drafts, experiments manual); C-02 by §3.3 step 8 + D16.

---

## 12. Initial Source Register (exact provisioning, Vorbote v1)

This register is the committed initial source list. Legend for **Status**:
- ✅ **live-verified** — endpoint probed successfully via HTTP during design (2026-10-26)
- 📄 **officially documented** — endpoint pattern from official documentation; live probe inconclusive in design (challenge/redirect), verified during guided setup
- 🧩 **reach-adapted** — accessed via the agent-reach bridge (community family)

### 12.1 Register

| # | Family → adapter class | Source | Exact endpoint / access pattern | Status | Notes |
|---|---|---|---|---|---|
| S-01 | change_feed → feed | Gesetze im Internet — Änderungsdienst (federal law amendments) | `https://www.gesetze-im-internet.de/aktuDienst-rss-feed.xml` | ✅ | RSS 2.0, ~98 KB, full feed of amendment notices; DE |
| S-02 | rest_api → api | TED (EU procurement notices) | `POST https://api.ted.europa.eu/v3/notices/search` body `{"query":"PD>=YYYYMMDD","limit":N,"fields":["publication-number"]}` | ✅ | Anonymous; **POST only** (GET rejected 405); expert-query syntax — date field `PD`; field/value restrictions apply (e.g. `CY=DE` unsupported value); notice XML at returned `links.xml` URLs |
| S-03 | html_diff → listing | Bundesanzeiger — Amtlicher Teil (official gazette, AT) | `https://www.bundesanzeiger.de/pub/de/amtlicher-teil` (+ `?year=YYYY&edition=BAnz+AT+DD.MM.YYYY`) | ✅ | Wicket/JSF app: session cookie + stateful row links; adapter keeps a cookie jar and follows listing structure; entries often PDF (text layer); DE |
| S-04 | html_diff → listing | Insolvenzbekanntmachungen (insolvency announcements) | `https://neu.insolvenzbekanntmachungen.de/ap/index.jsf` | ✅ | JSF search portal (www → neu 301); form-driven date search; session-bound; DE. High volume — bounded per-day window |
| S-05 | rest_api → api | Bundestag DIP (parliamentary materials: bills, procedures) | `https://dip.bundestag.de/http-api/v1/<entity>?…&apikey=KEY` — entities: `vorgang` (with `vorgangstyp=Gesetz&f.zuordnung=BT`), `drucksache`, `plenarprotokoll` | 📄 | Key required; a documented public key is published in the official API docs (rate-capped); guided setup fetches/enters it; DE |
| S-06 | dataset → dataset | DESTATIS GENESIS-Online (federal statistics) | `https://www-genesis.destatis.de/genesisWS/rest/2020/data/table?username=GAST&password=GAST&name=<table>&area=all&format=csv` (also `helloworld/whoami`) | ✅ | **Redirects (302/307) are normal — client must follow**; guest access `GAST`/`GAST`; free registration for a personal key optional; DE. Table list curated in wizard (e.g. insolvencies 12411, business registrations 41311…) |
| S-07 | dataset → dataset | Eurostat (EU statistics) | `https://ec.europa.eu/eurostat/api/dissemination/statistics/1.0/data/<dataset-code>?format=JSON&lang=en&geo=DE` | ✅ | Keyless, live (example `demo_r_d3dens` verified); EN/DE |
| S-08 | dataset → dataset | GovData (German open-data catalog — new-dataset detection) | `https://www.govdata.de/ckan/api/3/action/package_search?q=<query>&rows=<n>` | ✅ | CKAN 3-action API, anonymous, live; EN/DE |
| S-09 | feed → feed | heise online (tech change driver) | `https://www.heise.de/rss/heise-atom.xml` | ✅ | Atom, live; DE |
| S-10 | feed → feed | tagesschau (news context) | `https://www.tagesschau.de/xml/rss2/` | 📄 | 308 redirect — client follows redirects; canonical published URL; DE |
| S-11 | feed → feed | Ministry/regulator press rooms (BMF, BMWK, BMJV, …) | Per-site **feed auto-discovery**: fetch ministry homepage, parse `<link rel="alternate" type="application/rss+xml">`; wizard curates the set | 📄 | bund.de-family sites reorganize hosts (301s observed) — discovery per site at setup, not hard-coded feed URLs; DE. Family committed; per-ministry feed URLs curated at setup (OQ-1) |
| S-12 | trends → api | Google Trends — daily trending searches (DE) | `https://trends.google.com/trending/rss?geo=DE` (also `?geo=US` etc.) | ✅ | Official RSS, live, ~20 KB/day incl. approx-traffic; undocumented keyword-trends JSON endpoint deliberately **not** used (flagged unstable; ladder/Exa covers ad-hoc) |
| S-13 | community → reach | Reddit (r/de and operator-curated subreddits) | Reach-CLI `reddit` channel: search/read actions, `--backend opencli` / public-web; `items[]` JSON contract | 🧩 | Plain `.json` endpoints return 403 to non-browser clients (verified) — the bridge is the mandatory route; logged-in cookies in the reach browser profile (operator-performed login); DE/EN |
| S-14 | community → reach | X/Twitter | Reach-CLI `twitter` channel (experimental): search/read actions | 🧩 | Experimental-class (operator-acknowledged); logged-in cookies required; `RATE_LIMITED`/`PLATFORM_BLOCKED` → stop attempt, quarantine |
| S-15 | rest_api → api | Hacker News (Algolia API) | `https://hn.algolia.com/api/v1/search_by_date?query=<q>&tags=story&hitsPerPage=<n>` | ✅ | Keyless, live; EN; technology + platform-change signal |
| S-16 | marketplace → html_diff | Amazon.de Best Sellers / New Releases | `https://www.amazon.de/gp/bestsellers` (+ per-category paths; `…/movers-and-shakers` for rank movement) | ✅ | HTML served to polite clients (bot-wall sensitivity observed — 503 on movers during probe; OQ-2 posture: slow cadence, honest UA, Exa assist); rank_movement detection |
| S-17 | html_diff → listing | Platform policy/changelog pages (Google/Meta/Amazon seller rules, app-store release notes) | Operator-curated URL list per platform (wizard); html_diff snapshot comparison | 📄 | URLs churn by design — §12.3 URL-change handling; family committed, initial set curated in wizard at setup |
| S-18 | rest_api → api (optional, credentialed, **off by default**) | Google Keyword Planner (search volumes, competition) | Google Ads API `KeywordPlanningService.generateKeywordIdeas` — OAuth2 client + developer token | 📄 | Free tier; guided setup; volumes as ranges without active spend; disabled until configured |
| S-19 | rest_api → api (optional, credentialed, **off by default**) | Amazon SP-API (offers/pricing detail) | SP-API Products/Pricing endpoints — Professional seller account + LWA credentials | 📄 | Free-tier credential; optional enhancement over S-16's public pages; disabled until configured |
| S-20 | html_diff → listing | Förderdatenbank (federal funding programs — new/changed programs) | `https://www.foerderdatenbank.de/` site search (form query) | 📄 | JS-heavy site — Exa-assisted html_diff; DE; grant/structural change signal |

Retrieval-class notes: every source above rides the §3.4 ladder; sources whose primary route needs a logged-in browser session (S-13, S-14) go exclusively through the reach bridge; sources with bot-walls or challenge behavior (S-16, S-02's XML retrieval, EUR-Lex-family S-21 below) declare Exa-assist posture in their source config.

### 12.2 Family → adapter mapping (normative)

| Family | Adapter class | Change detection kinds produced |
|---|---|---|
| feed | feed adapter (parse, dedupe by external_key) | new_item |
| rest_api | api adapter (endpoint template + paging) | new_item, value_delta |
| html_diff | listing adapter (cookie jar, structure-aware list extraction, snapshot diff) | new_item, list_diff, snapshot_diff |
| dataset | dataset adapter (snapshot table, compare) | value_delta, snapshot_diff |
| change_feed | feed adapter (amendment-specific semantics) | new_item |
| community | reach bridge (channel actions, JSON contract) | new_item |
| trends | trends adapter (RSS with approx-traffic parsing) | new_item, value_delta (traffic level) |
| marketplace | listing adapter (rank extraction) | rank_movement, list_diff |

### 12.3 URL-change handling (expected condition)

Exact URLs are current as of 2026-10-26 and were live-probed or officially documented at design time. **URLs can and will change**; Vorbote treats this as normal operation, not failure:
1. Every source stores its endpoint in the editable source config (not code) — the operator can update a URL without a code change.
2. A source that starts failing with transport-level errors (404/301-to-nowhere/DNS) after having succeeded is quarantined with status `url_stale` and surfaced in the source manager wizard, which re-proposes the endpoint (manual edit or Exa-assisted re-discovery).
3. Feed-family sites get auto-discovery-based URL refresh (S-11 pattern) in the wizard.
4. The register is data, not architecture: adding/removing sources never requires a document revision.

---
