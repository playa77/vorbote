# Vorbote

> A personal, single-user AI intelligence system that watches the German and EU economic landscape for **change** — and turns detected changes into evaluated, evidence-backed small-business opportunities.

Vorbote continuously scans a curated platform of official, free, and community sources for regulatory, technological, demographic, industry-structural, behavioral, and platform changes. It converts detected changes into **evaluated, evidence-backed opportunities** and guides you — the single operator — through investigating and validating them.

Vorbote does **not** generate business ideas. It detects observable market signals, links every finding to stored evidence, proposes economic explanations (always preserving alternatives), and enforces a disciplined funnel:

```text
Observations → Hypotheses → Serious investigations → Commercial experiments
```

The name is the philosophy: a *Vorbote* (harbinger) is the detectable signal that precedes the change. Vorbote studies transitions, not static markets.

---

## The discovery method

Vorbote implements **change-driven opportunity discovery**: looking for economic changes that create new requirements faster than existing suppliers can adapt.

**Six change drivers** are watched:

| Driver | What creates the opportunity |
|---|---|
| Technology | New tools/platforms change what's economically feasible |
| Regulation | New rules create compliance needs and deadlines |
| Demographics | Population shifts change demand structure |
| Industry restructuring | Closures, mergers, insolvencies free customers and staff |
| Consumer behavior | Longitudinal shifts in what people search and buy |
| Platform changes | Marketplace policies, API changes, advertising rules redistribute access |

Every candidate is traced through the **signal chain**:

> Change → affected population → new problem → existing response → unresolved demand → commercial feasibility

**Evidence discipline.** Every finding links to original stored evidence, carries a proposed economic explanation *plus alternative explanations*, and records what remains unknown. Negative findings — why an apparent inefficiency is economically rational — are preserved with equal standing, preventing repeated rediscovery of false opportunities. The LLM's role is structured information extraction: never business ideation, never invented market-size numbers.

**The incumbent question.** Every candidate must answer: *why hasn't competition eliminated this inefficiency?* The answer distinguishes a temporary anomaly from a durable, small-operator-exploitable advantage.

**Funnel discipline.** Many observations → fewer hypotheses → few serious investigations → one commercial experiment at a time. Nothing is auto-promoted: every funnel transition is your decision, optionally guided.

**Evaluation, deterministic and operator-owned.** Your constraints (capital, hours/week, target monthly profit, horizon, operational burdens, sales comfort, automation tolerance, unacceptable risks) drive *hard exclusions* — a high score never compensates a fatal problem. Weighted scoring (1–5 across six criteria: spending evidence 25%, customer accessibility 25%, contribution margin 20%, operational compatibility 15%, competitive advantage 10%, validation speed 5%) prioritizes research — it is config, not dogma. Backward economics computes contribution per transaction, monthly operating profit, effective hourly earnings, and the reverse calculation from target profit to the number of paying customers you'd need. Every unit-economics input is your estimate or evidence-extracted with citation — never LLM-invented.

## What Vorbote is not

- No multi-user/SaaS ambitions — personal software, exactly one operator
- No "50 ideas every morning" idea-noise generation
- Never spends money or executes transactions — commercial experiments are manual; the system tracks, guides, records
- No real-time/streaming ingestion — a daily cadence
- No email notifications — alerting is passive, via the triage queue
- Read-only netizen: never posts, comments, likes, messages, or writes to any external platform

## System at a glance

Two processes, one trust boundary:

- **Server** (`vorbote serve`) — headless Linux daemon: scheduler, source platform with eight adapter families, deterministic change detection, LLM extraction/hypothesis stages, evaluation engine (scoring, economics, validation), conversational guidance engine, retention, cost metering, and a loopback-only JSON API.
- **Desktop client** (`vorbote-client`) — PySide6 GUI for Linux: triage queue, candidate dossiers, scoring/economics, validation tracker, constraints editor, source manager with setup wizards, guidance chat, and cost ledgers. It manages its own SSH local forward to the server.

```text
SOURCES (feeds · APIs · listings · datasets · community · trends · marketplace)
   │  retrieval ladder: deterministic+politeness → Exa fallback → LLM-driven (deep investigation only)
   ▼
CHANGE DETECTION (deterministic diffs, rank movements, thresholds, snapshot comparison)
   ▼
LLM EXTRACTION & HYPOTHESES → OBSERVATIONS (evidence-linked, triage queue)
   ▼
YOU: promote → score → economics → investigate → experiment
```

Data lives in exactly **one SQLite database** plus a content-addressed snapshot store on disk. Scope: Germany + EU, German and English sources, open scan across all sectors. The UI is bilingual (English default, German switchable) while source excerpts are always preserved verbatim in their original language.

## Installation

### Server (Linux host, always-on — VPS assumed)

Prerequisites: Python 3.12+, SQLite (stdlib), rsync (backups); Node ≥ 18 and a headless Chromium/Chrome binary only if you enable the Reddit/X community bridge.

```bash
git clone <repo-url> && cd vorbote
python3.12 -m venv .venv
source .venv/bin/activate
pip install -e .

# Create the state dir, database + migrations, 20 seeded sources, config skeleton
vorbote init
```

`init` writes a blank `.env`-style secrets skeleton into the state dir. Edit it over SSH and add at least an LLM key:

```ini
VORBOTE_LLM_API_KEY=…        # OpenRouter / OpenCode Go / DeepSeek / custom OpenAI-compatible endpoint
VORBOTE_EXA_KEY=…            # optional — without it Vorbote runs deterministic-only
```

Secrets live *only* in this file — never in the database, config, logs, or API responses. There is no secret-setting API endpoint; the client shows presence/freshness only.

Start serving (loopback-only JSON API, default port 8471):

```bash
vorbote serve              # foreground
# or install deploy/vorbote.service as a systemd unit (Restart=on-failure, journal logging)
```

| Command | Purpose |
|---|---|
| `vorbote init` | Create state dir, DB + migrations, source seeds, config skeleton (no-op on an existing DB) |
| `vorbote serve` | Run the API + scheduler |
| `vorbote run --all` | One-shot sweep (also `--group <name>`) |
| `vorbote migrate` | Apply pending migrations (upgrade path) |

### Desktop client (Linux desktop)

```bash
pip install -e ./client
vorbote-client
```

First launch: enter the server host/SSH user/port, test the tunnel (the client spawns `ssh -N -L` itself), pick the UI language, and complete the first-run constraints editor — optionally guided.

## Configuration

Copy `config/settings.example.toml` → `config/settings.toml`. Everything has sane defaults:

| Section | What it controls |
|---|---|
| `[server]` | Loopback API port (default 8471) |
| `[sweep]` | Daily sweep time (default 05:00), catch-up of missed windows |
| `[politeness]` | Per-source inter-request delays, backoff base, retry caps, `Retry-After` cap |
| `[llm]` | Provider, model (one model for all roles; default `glm-5.3-flash`), timeout, monthly ceiling (default $50, warn-only) |
| `[exa]` | Fallback fetch/search provider with its own ledger and monthly ceiling |
| `[retention]` | Retention window (12 months), storage ceiling (10 GB, warn-only) |
| `[reach]` | Headless-browser bridge for Reddit/X: profile dir, call gaps, timeouts |
| `[guidance]` | Context assembly cap, direct-write vs confirm-required action sets |

Sources are **data, not config**: the 20 seeded sources are edited in the client's source manager (URL updates, schedules, quarantine/release) — no code changes required when a site moves.

## Daily operation

1. **Overnight sweep** (05:00 default): every due source is fetched politely (single-flight, delays, exponential backoff), changes are detected deterministically, and the LLM classifies events and drafts evidence-linked observations into the triage queue.
2. **Triage pass**: review the queue; promote to hypothesis, decline, or mark as a negative finding.
3. **Dossier work**: confirm criterion scores (the LLM may draft rationale; you confirm values), enter backward economics, and answer the incumbent question.
4. **Validation**: run investigations and interviews focused on actual past purchasing; design experiments that test the riskiest assumption with failure criteria fixed in advance and immutable once set; record results and falsification notes.
5. **Guidance chats** (optional): a conversational LLM session that reads your database, interviews you, drafts scores/evidence/experiments — under bounded writes: notes and dispositions written directly, scores/evidence/deletions only after in-chat confirmation. Every write lands in the audit trail, and each session closes with a deterministic digest.

Failing sources are quarantined automatically (visible in the source manager; one-click release after fixing). URL drift is handled as normal operation: edit the endpoint, test-fetch, re-enable.

## Seeded sources (v1)

| # | Source | Family |
|---|---|---|
| S-01 | Gesetze im Internet — federal law amendments | change feed |
| S-02 | TED — EU procurement notices | REST API |
| S-03 | Bundesanzeiger — Amtlicher Teil | HTML diff |
| S-04 | Insolvenzbekanntmachungen — insolvency announcements | HTML diff |
| S-05 | Bundestag DIP — bills & parliamentary materials | REST API |
| S-06 | DESTATIS GENESIS-Online — federal statistics | dataset |
| S-07 | Eurostat — EU statistics | REST API |
| S-08 | GovData — German open-data catalog | dataset |
| S-09 | heise online | feed |
| S-10 | tagesschau | feed |
| S-11 | Ministry/regulator press rooms (BMF, BMWK, BMJV, …) | feed (auto-discovery) |
| S-12 | Google Trends — daily trending searches (DE) | trends |
| S-13 | Reddit (r/de + curated subreddits) | community (reach bridge) |
| S-14 | X/Twitter (experimental class) | community (reach bridge) |
| S-15 | Hacker News (Algolia API) | REST API |
| S-16 | Amazon.de Best Sellers / New Releases / Movers & Shakers | marketplace |
| S-17 | Platform policy/changelog pages (Google, Meta, Amazon, app stores) | HTML diff |
| S-18 | Google Keyword Planner *(optional, credentialed, off by default)* | REST API |
| S-19 | Amazon SP-API *(optional, credentialed, off by default)* | REST API |
| S-20 | Förderdatenbank — federal funding programs | HTML diff |

Free/official sources only; the two optional credentialed sources stay disabled until set up through their guided wizards. Community logins (Reddit/X) use read-only session cookies uploaded through the client wizard — no interactive login ever happens on the server.

## Cost control

- **LLM ledger** — metered per stage per run at configurable blended prices; monthly ledger with per-stage/run/session breakdown; ceiling default $50/month.
- **Exa ledger** — per-call metering with its own monthly ceiling and breakdown.
- Both ceilings are **warn-only**: the system makes approaching/exceeding them visible and keeps operating — the warning is your decision point. Without an Exa key, Vorbote degrades cleanly to deterministic-only collection (Exa-required sources quarantine visibly).

## Security model

- **Loopback-only API.** The JSON API binds `127.0.0.1` only; the SSH local forward *is* the security model. No TLS, no tokens, no application auth — deliberate, for single-operator personal software — with localhost `Host`/`Origin` checks as defense in depth.
- **Secrets discipline.** API keys, OAuth secrets, and browser-cookie strings live only in the server's secrets file / reach browser profile (`0600`). Never in the database, config, logs, or API responses; backups exclude the reach profile.
- **Read-only netizen.** Vorbote never writes to any external platform, and retrieval is polite by construction: per-source single-flight, sensible inter-request delays, bounded exponential backoff honoring `Retry-After`, honest User-Agent, robots-aware behavior.

## Data, retention & backups

- Exactly one SQLite DB (`vorbote.db`, WAL) + a content-addressed snapshot store; the state dir is relocatable via the `VORBOTE_HOME` environment variable.
- Non-selected raw captures are kept 12 months (configurable); anything referenced by an observation, hypothesis, candidate, investigation, or experiment is **pinned indefinitely**; FIFO eviction with a visible warning (count + bytes); storage-ceiling warnings.
- `deploy/backup.sh`: nightly `sqlite3 … VACUUM INTO` + snapshot-store rsync (excludes the reach profile), scheduled by the operator; restore = copy back + re-apply migrations (idempotent).

## Development

```bash
pip install -e '.[dev]'
pytest tests/ -q    # fully hermetic — no live network anywhere in tests
```

- Two setuptools distributions, one repo: the server (`src/vorbote/`) and the PySide6 client (`src/vorbote_client/`).
- Layered architecture, dependencies point downward only; a single composition root (`db.py::build_container()`) constructs all infrastructure; every cross-module handoff is a named seam.
- Client tests run offscreen: `QT_QPA_PLATFORM=offscreen pytest tests/client -q`.
- Implementation follows the roadmap: 35 work packages in 8 phases, with milestones tagged `v0.1.0`–`v0.5.0` and 15 acceptance criteria as the definition of done.

### Documentation

| Document | Contents |
|---|---|
| [Design Document](docs/design_document.md) | Product definition, discovery method, requirements, architecture, security, deployment, initial source register |
| [Technical Specification](docs/technical_specification.md) | Project structure, dependencies, config, database DDL, API spec, module specs, testing strategy |
| [Implementation Roadmap](docs/roadmap.md) | Work packages, dependency graph, milestones, acceptance criteria |

## License

[MIT](LICENSE)
