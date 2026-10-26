# Vorbote — Document 2: Technical Specification

| | |
|---|---|
| **Title** | Vorbote — Technical Specification |
| **Version** | 1.0.0 |
| **Date** | 2026-10-26 |
| **Status** | Approved |
| **Baseline** | Vorbote Design Document v2.0.0 (Approved 2026-10-26) — cross-references cite it as [Doc 1 §n] |
| **Companion** | Document 3 (Implementation Roadmap) issued separately on request |

**Revision History**

| Version | Date | Status | Changes |
|---|---|---|---|
| 1.0.0 | 2026-10-26 | Approved | Initial issue from approved Doc 1 v2.0.0 |

---

## 1. Implementation Scope

### 1.1 In scope (v1.0.0)

1. Server distribution `vorbote` (src layout): CLI entry, scheduler, orchestrator, source platform (8 adapter classes), retrieval ladder + politeness governor, Exa client, reach bridge, normalizer, snapshot store, change detector, extractor + hypothesis generator (LLM), evaluation engine (observation/hypothesis/candidate managers, scoring, economics, constraints, validation), guidance engine (sessions, proposals, bounded writes, digests), metering, retention, persistence (one SQLite DB), loopback JSON API [Doc 1 §3].
2. Client distribution `vorbote-client` (src layout): shell/tunnel, triage, dossier (+ Markdown/HTML export), scoring/economics, validation, constraints editor, source manager (incl. reach cookie-upload wizard), guidance chat, settings/ledgers [Doc 1 §4].
3. Initial Source Register provisioning: S-01…S-20 as editable source-config entries [Doc 1 §12].
4. Bilingual UI (EN default, DE) with source-language preservation [FR-27].
5. Hermetic test suite [NFR-11].
6. Ops tooling: systemd unit, backup script, deployment guide [Doc 1 §8].

### 1.2 Out of scope (v1.0.0)

- OCR; email notifications; export formats beyond Markdown/HTML; per-role LLM model routing (config schema present, one model active per C-03); eBay/Etsy sources (FR-06); multi-user anything; live migration from any other system.

### 1.3 Spec conventions

- Types as `name: Type`; lists `list[T]`; optionals `T | None`; JSON blob fields are objects whose internal shape is pinned in the owning module spec.
- Error envelope everywhere: `{"error": {"code": <str>, "message": <str>, "detail": <obj|null>}}`.
- All datetimes: UTC ISO-8601 strings `YYYY-MM-DDTHH:MM:SSZ` at rest.
- Money: float USD in ledgers; economics inputs decimal-typed in UI, stored as documented JSON numbers.
- Module path prefix `vorbote/` = server; `vorbote_client/` = client.

---

## 2. Project Structure

```text
vorbote/                        # repo root
├── pyproject.toml              # server distribution
├── client/pyproject.toml       # client distribution (separate setuptools)
├── config/
│   └── settings.example.toml
├── deploy/
│   ├── vorbote.service         # systemd unit
│   └── backup.sh
├── docs/
│   └── deployment.md
├── src/vorbote/
│   ├── __init__.py
│   ├── cli.py                  # entry: run/serve/init/backup helpers
│   ├── config.py               # TOML load, schema, defaults, mutation lock
│   ├── schema.py               # ALL typed dataclasses + JSON blob shapes
│   ├── db.py                   # composition root (see §15) + migrations
│   ├── store/                  # §5 data layer
│   │   ├── __init__.py
│   │   ├── sources.py          # sources, source_groups, source_credentials_status
│   │   ├── runs.py             # sweep_runs, source_runs, fetch_attempts
│   │   ├── items.py            # raw_items, snapshots, pins
│   │   ├── changes.py          # change_events
│   │   ├── funnel.py           # observations, hypotheses, candidates, evidence_records, exclusions
│   │   ├── scoring.py          # constraints, scores, economics_records
│   │   ├── validation.py       # investigations, interview_guides, experiments, falsification_notes
│   │   ├── guidance.py         # guidance_sessions, messages, proposals, digests, write_audit
│   │   ├── metering.py         # llm_usage, exa_usage, ceiling_states
│   │   └── migrations.py       # schema_migrations apply/record
│   ├── scheduler.py
│   ├── orchestrator.py
│   ├── sources/
│   │   ├── __init__.py         # AdapterRegistry
│   │   ├── base.py             # Adapter, FetchPlan, CanonicalItem types
│   │   ├── feeds.py            # feed + change_feed + trends (RSS/Atom)
│   │   ├── api.py              # rest_api (TED, DIP, Eurostat, HN, SP-API, Keyword Planner)
│   │   ├── listing.py          # html_diff + marketplace (BAnz, insolvenz, Amazon, policy pages, FDB)
│   │   ├── datasets.py         # dataset (DESTATIS, Eurostat-snapshots, GovData)
│   │   │                       #     (listing.py covers html_diff family; datasets.py covers dataset family)
│   │   └── reach.py            # community via reach bridge
│   ├── ladder.py               # retrieval ladder executor
│   ├── politeness.py           # per-source single-flight, delay, backoff
│   ├── exa_client.py
│   ├── reach_bridge.py         # headless browser + cookie injection
│   ├── normalizer.py
│   ├── snapshots.py            # snapshot store (content-addressed)
│   ├── changedetect.py         # change detector
│   ├── llm.py                 # single-path LLM client + metering hook
│   ├── prompts/               # role prompts: extractor, hypothesis, guidance-*
│   │   └── *.md                # one file per role (EN), i18n-neutral
│   ├── extract.py             # extractor stage wrapper
│   ├── hypothesize.py          # hypothesis-generator stage wrapper
│   ├── evaluation/
│   │   ├── __init__.py
│   │   ├── observations.py
│   │   ├── hypotheses.py
│   │   ├── candidates.py
│   │   ├── scoring.py          # deterministic weighted score + exclusions
│   │   ├── economics.py        # deterministic calculators
│   │   ├── constraints.py
│   │   └── validation.py
│   ├── guidance/
│   │   ├── __init__.py
│   │   ├── engine.py           # sessions, context assembly, roles
│   │   ├── writes.py           # bounded-write controller (proposals)
│   │   └── digest.py           # deterministic session digest
│   ├── metering.py             # ledgers, ceilings (warn-only)
│   ├── retention.py
│   ├── web/
│   │   ├── __init__.py         # app factory
│   │   └── api.py              # all routes, Host/Origin gates
│   └── export/
│       └── dossier.py          # Markdown/HTML dossier renderer
└── src/vorbote_client/
    ├── __init__.py
    ├── app.py                  # main window, tunnel, language switch
    ├── tunnel.py                # SSH local-forward management
    ├── api.py                  # HTTP client for /api/v1
    ├── i18n.py                 # EN/DE catalogs
    ├── workers.py               # QThread wrappers for api.py
    ├── widgets/
    │   ├── triage.py
    │   ├── dossier.py
    │   ├── scoring.py           # includes economics panel
    │   ├── validation.py
    │   ├── constraints.py
    │   ├── sources.py           # source manager + wizards (incl. cookie upload)
    │   ├── guidance.py          # chat + proposal cards + digest view
    │   └── settings.py          # settings + ledger views
    └── export_download.py       # save dossier exports locally
```

Two setuptools distributions, one repo. The `sources/datasets.py` adapter handles the dataset family; `sources/listing.py` the html_diff and marketplace families (they share the listing engine; register entries differ by detector kinds) [Doc 1 §12.2].

---

## 3. Dependencies

### 3.1 Server (`pyproject.toml`)

| Package | Version | Purpose |
|---|---|---|
| `httpx` | `~=0.28` | All outbound HTTP: ladder, Exa, LLM |
| `feedparser` | `~=6.0` | RSS/Atom parsing (S-01, S-10, S-11, S-12) |
| `trafilatura` | `~=2.0` | HTML text extraction |
| `pypdf` | `~=6.0` | PDF text layer (S-02 XML/PDF links, S-03, gazettes) |
| `selectolax` | `~=0.3` | Fast DOM queries for listing adapters |
| `flask` | `~=3.1` | Loopback API |
| `python-dateutil` | `~=2.9` | Feed/gazette date parsing |
| `pillow` | — | **None** — removed; no image processing anywhere |

Optional extras: `reach` = nothing on PyPI; the reach adapter is a submodule path (pinned in-repo, §8.10) — its Node runtime is a host prerequisite, not a Python dep. `dev` = `pytest~=8.3`, `pytest-flask` not required.

### 3.2 Client (`client/pyproject.toml`)

| Package | Version | Purpose |
|---|---|---|
| `PySide6` | `~=6.8` | GUI |
| `httpx` | `~=0.28` | API client |

### 3.3 Host prerequisites (server)

Python 3.12+; Node ≥ 18 (reach runtime); headless Chromium or Chrome binary (`CHROME_BIN` env or auto-detect, §8.10); rsync (backups); SQLite 3.4x (Python stdlib `sqlite3`).

No Docker, no build step [Doc 1 §8].

---

## 4. Configuration

Two files, strictly separated [Doc 1 §7]: `config/settings.toml` (no secrets) and `.env`-style secrets file (server only, never API-visible). State dir relocatable via `VORBOTE_HOME` env (must be set for every state-touching process).

```toml
# config/settings.toml — Vorbote server configuration (NO SECRETS here)

[server]
port = 8471                        # loopback bind 127.0.0.1 [Doc 1 AS-02]

[sweep]
daily_at = "05:00"                # server-local time [Doc 1 AS-02]
catchup = true                    # one catch-up per missed window, bounded

[politeness]
default_delay_s = 4.0             # inter-request delay per source class default
per_source_overrides = [          # source name → delay seconds
  { name = "insolvenzbekanntmachungen", delay_s = 6.0 },
  { name = "amazon-bestellers", delay_s = 10.0 },
  { name = "bundesanzeiger-at", delay_s = 6.0 },
]
backoff_base_s = 2.0              # exponential backoff on retryable failures
backoff_max_retries = 4           # bounded retries per ladder step 1
retry_after_cap_s = 120.0         # cap on honored Retry-After

[llm]
provider = "openrouter"          # openrouter | opencode_go | deepseek | custom
base_url = "https://openrouter.ai/api/v1"   # default per provider table §8.2
model = "glm-5.3-flash"           # ONE model all roles [C-03]
timeout_s = 120.0
max_tokens_default = 2000
reasoning_effort = ""             # empty = not sent; else forwarded as reasoning.effort

[llm.ceiling]                     # FR-32: warn-only, monthly
budget_usd = 50.0
input_price_per_mtok = 0.04       # blended, configurable
output_price_per_mtok = 0.60      # blended, configurable

[exa]
base_url = "https://api.exa.ai"   # fixed unless custom
timeout_s = 30.0

[exa.ceiling]                     # FR-33: own ledger, warn-only
budget_usd = 15.0                 # default; operator-configurable
price_per_call = 0.005            # default; operator-configurable

[retention]
months = 12                       # non-selected raw snapshots/fulltext/PDF [Doc 1 FR-30]
storage_ceiling_gb = 10.0         # warn-only ceiling
pin_referenced = true             # referenced ⇒ pinned indefinitely (always true)

[reach]
chromium_bin = ""                 # empty = auto-detect; else absolute path
profile_dir = ""                  # empty = <state>/reach-profile
node_bin = ""                     # empty = PATH lookup
bridge_timeout_s = 120.0
call_gap_within_source_s = 10.0   # two calls on same platform ≥10 s apart (netizen rule)

[guidance]
max_context_items = 40            # deterministic context assembly cap
confirm_required_actions = [      # FR-23 boundary; the closed set of confirm-required
  "score.write", "evidence.write", "delete"
]
direct_write_actions = ["note.write", "disposition.write"]

[sources.initial]                 # §12 register entries are SEED DATA, not config
                                 # (see §4.2): source rows created at init; edited via API/UI
```

### 4.1 Secrets file (`.env` class, server only)

```
VORBOTE_LLM_API_KEY=…             # per provider (OpenRouter/DeepSeek/OpenCode Go/custom)
VORBOTE_EXA_KEY=…                 # absent ⇒ exa disabled → deterministic-only degradation
VORBOTE_DIP_KEY=…                 # optional; S-05
VORBOTE_GOOGLE_ADS_OAUTH_CLIENT_ID=…      # optional; S-18
VORBOTE_GOOGLE_ADS_OAUTH_CLIENT_SECRET=…  # optional; S-18
VORBOTE_GOOGLE_ADS_DEV_TOKEN=…            # optional; S-18
VORBOTE_SPAPI_LWA_CLIENT_ID=…             # optional; S-19
VORBOTE_SPAPI_LWA_CLIENT_SECRET=…         # optional; S-19
VORBOTE_SPAPI_REFRESH_TOKEN=…              # optional; S-19
VORBOTE_REDDIT_COOKIES=…          # optional; raw cookie header string(s) (secret §8.10)
VORBOTE_TWITTER_COOKIES=…        # optional; raw cookie header string(s)
```

Note: reach cookies are uploaded via the client wizard over the tunnel → stored only in the reach profile on disk; the `.env` entries are an alternative provisioning path for bootstrap. Both are secret material [Doc 1 §7/NFR-05]: never in DB, logs, or API responses.

### 4.2 Seed data vs config

At `vorbote init`, the 20 register sources (§12 of Doc 1 → seeds in `config/` shipped as a JSON file `config/sources_seed.json`) are inserted as `sources` rows (S-18/S-19 `enabled=false`). Source edits (URL, group, schedule) go through the API/UI — sources are **data**, not config [Doc 1 §12.3]. The seed file is versioned; re-running init on an existing DB is a no-op for existing sources (by `name`).

---

## 4.3 Config mutation

- Settings re-read per scheduler tick (edits apply without restart).
- Config writes: tmp-file + `os.replace` under a mutation lock (same discipline as the state files).
- No endpoint writes settings.toml paths for secrets; secrets only via file edit over SSH or the cookie wizard (which writes only to reach profile + `.env`-style markers, never toml).

---

## 4.4 Client settings (local QSettings)

`target_host`, `ssh_user`, `ssh_port`, `local_tunnel_port`, `ui_language` (`en` default), `dossier_export_dir`. Nothing else persists client-side.

---

## 5. Data Layer

### 5.1 File-set & connection discipline

- Exactly one DB: `$VORBOTE_HOME/vorbote.db` (WAL mode, `foreign_keys=ON`, `check_same_thread=False` + an internal write mutex; all writes go through store modules).
- Snapshot store: `$VORBOTE_HOME/snapshots/` content-addressed by SHA-256: `snapshots/ab/abcd1234…def.pdf`; DB row in `snapshots` holds `store_path`, `content_hash`, `bytes`, `mime`, `captured_at`, `pinned` [Doc 1 §5].
- Migrations: numbered SQL files embedded in `store/migrations.py`; `schema_migrations` records applied ids; forward-only.

### 5.2 DDL (normative, complete)

```sql
-- sources & groups
CREATE TABLE sources (
  id INTEGER PRIMARY KEY,
  name TEXT NOT NULL UNIQUE,              -- stable slug e.g. 'gesetze-amendments'
  family TEXT NOT NULL CHECK (family IN ('feed','rest_api','html_diff','dataset',
        'change_feed','community','trends','marketplace')),
  group_id INTEGER REFERENCES source_groups(id),
  endpoint_config TEXT NOT NULL,          -- JSON EndpointConfig (§8 adapter schemas)
  language TEXT NOT NULL DEFAULT 'de',    -- 'de' | 'en' | 'de,en'
  jurisdiction TEXT NOT NULL DEFAULT 'DE',-- 'DE' | 'EU'
  enabled INTEGER NOT NULL DEFAULT 1,
  requires_credential INTEGER NOT NULL DEFAULT 0,
  health TEXT NOT NULL DEFAULT 'ok' CHECK (health IN
        ('ok','quarantined','disabled')),
  quarantine_reason TEXT,
  last_success_at TEXT,
  created_at TEXT NOT NULL,
  updated_at TEXT NOT NULL
);
CREATE TABLE source_groups (
  id INTEGER PRIMARY KEY,
  name TEXT NOT NULL UNIQUE,
  schedule_spec TEXT NOT NULL,           -- 'daily@05:00' | 'daily@HH:MM' | 'weekly:DOW@HH:MM'
  notes TEXT
);
CREATE TABLE source_credentials_status (
  source INTEGER NOT NULL REFERENCES sources(id) ON DELETE CASCADE,
  kind TEXT NOT NULL,                    -- 'api_key' | 'oauth_client' | 'browser_cookies'
  configured INTEGER NOT NULL DEFAULT 0,
  last_verified_at TEXT,
  PRIMARY KEY (source, kind)
);

-- runs & retrieval
CREATE TABLE sweep_runs (
  id INTEGER PRIMARY KEY,
  started_at TEXT NOT NULL,
  finished_at TEXT,
  status TEXT NOT NULL CHECK (status IN ('running','completed','failed','interrupted')),
  trigger_kind TEXT NOT NULL CHECK (trigger_kind IN ('scheduled','catchup','manual')),
  stats TEXT NOT NULL DEFAULT '{}'       -- JSON: items, changes, errors, llm_cost_usd, exa_cost_usd
);
CREATE TABLE source_runs (
  id INTEGER PRIMARY KEY,
  sweep_run INTEGER NOT NULL REFERENCES sweep_runs(id),
  source INTEGER NOT NULL REFERENCES sources(id),
  status TEXT NOT NULL CHECK (status IN ('ok','error','skipped')),
  items_fetched INTEGER NOT NULL DEFAULT 0,
  errors TEXT NOT NULL DEFAULT '[]',     -- JSON list[ErrorInfo]
  ladder_summary TEXT NOT NULL DEFAULT '{}' -- JSON: step1/step2 counts
);
CREATE TABLE fetch_attempts (
  id INTEGER PRIMARY KEY,
  source_run INTEGER NOT NULL REFERENCES source_runs(id),
  ladder_step INTEGER NOT NULL CHECK (ladder_step IN (1,2)),
  started_at TEXT NOT NULL,
  outcome TEXT NOT NULL CHECK (outcome IN
    ('ok','ok_exa','retryable_error','fatal_error','exa_disabled','skipped_login')),
  http_status INTEGER,
  retry_count INTEGER NOT NULL DEFAULT 0,
  delay_applied_ms INTEGER NOT NULL DEFAULT 0,
  exa_usage INTEGER REFERENCES exa_usage(id)
);

-- canonical content
CREATE TABLE raw_items (
  id INTEGER PRIMARY KEY,
  source INTEGER NOT NULL REFERENCES sources(id),
  external_key TEXT NOT NULL,            -- dedupe key per source
  title TEXT NOT NULL,
  url TEXT,
  author TEXT,
  published_at TEXT,
  fetched_at TEXT NOT NULL,
  language TEXT NOT NULL,
  jurisdiction TEXT NOT NULL,
  content_hash TEXT NOT NULL,
  snapshot INTEGER REFERENCES snapshots(id),
  parse_tier TEXT NOT NULL CHECK (parse_tier IN
    ('full_text','summary','listing_entry','pdf_text','pdf_flagged_no_text')),
  payload TEXT NOT NULL,                -- JSON CanonicalItem payload
  UNIQUE (source, external_key)
);
CREATE TABLE snapshots (
  id INTEGER PRIMARY KEY,
  store_path TEXT NOT NULL,
  content_hash TEXT NOT NULL,
  bytes INTEGER NOT NULL,
  mime TEXT NOT NULL,
  captured_at TEXT NOT NULL,
  pinned INTEGER NOT NULL DEFAULT 0
);
CREATE TABLE pins (
  target_kind TEXT NOT NULL CHECK (target_kind IN ('snapshot','raw_item')),
  target_id INTEGER NOT NULL,
  reason TEXT NOT NULL,
  created_at TEXT NOT NULL,
  PRIMARY KEY (target_kind, target_id)
);

-- change detection
CREATE TABLE change_events (
  id INTEGER PRIMARY KEY,
  source INTEGER NOT NULL REFERENCES sources(id),
  detected_at TEXT NOT NULL,
  kind TEXT NOT NULL CHECK (kind IN
    ('new_item','list_diff','rank_movement','value_delta','snapshot_diff')),
  before_state TEXT,                     -- JSON
  after_state TEXT,                      -- JSON
  magnitude REAL,
  raw_item_ids TEXT NOT NULL DEFAULT '[]',  -- JSON list[int]
  extracted INTEGER NOT NULL DEFAULT 0    -- extractor consumed flag (idempotent retry)
);

-- funnel: evidence & records
CREATE TABLE observations (
  id INTEGER PRIMARY KEY,
  change_event_ids TEXT NOT NULL DEFAULT '[]',  -- JSON list[int]
  created_at TEXT NOT NULL,
  title TEXT NOT NULL,
  description TEXT NOT NULL,             -- what changed
  affected_population TEXT,              -- draft
  driver_tags TEXT NOT NULL DEFAULT '[]',-- JSON list[str]
  proposed_explanation TEXT NOT NULL,
  alternative_explanations TEXT NOT NULL DEFAULT '[]',  -- JSON list[str]
  unknowns TEXT NOT NULL DEFAULT '[]',   -- JSON list[str]
  status TEXT NOT NULL DEFAULT 'triage' CHECK (status IN
    ('triage','promoted','declined','negative_finding')),
  language TEXT NOT NULL,
  evidence TEXT NOT NULL DEFAULT '[]',   -- JSON list[EvidenceRef]
  guidance_session_refs TEXT NOT NULL DEFAULT '[]'
);
CREATE TABLE hypotheses (
  id INTEGER PRIMARY KEY,
  observation INTEGER NOT NULL REFERENCES observations(id),
  statement TEXT NOT NULL,
  basis TEXT NOT NULL DEFAULT '[]',     -- JSON list[EvidenceRef]
  incumbent_response TEXT,              -- analysis
  unresolved_demand TEXT,
  status TEXT NOT NULL DEFAULT 'draft' CHECK (status IN
    ('draft','active','merged','rejected'))
);
CREATE TABLE candidates (
  id INTEGER PRIMARY KEY,
  hypothesis INTEGER REFERENCES hypotheses(id),
  created_at TEXT NOT NULL,
  market_segment TEXT NOT NULL,
  observed_inefficiency TEXT NOT NULL,
  discovery_source TEXT NOT NULL,       -- sources.name
  existing_alternatives TEXT,
  est_customer_expenditure TEXT,        -- JSON {value, confidence, source}
  competition TEXT,
  potential_advantage TEXT,
  customer_access TEXT,
  unit_economics TEXT,                  -- JSON EconomicsInputs (§8.6)
  operational_complexity TEXT,           -- operator/LLM-drafted note
  confidence_level TEXT NOT NULL DEFAULT 'unknown' CHECK (confidence_level IN
    ('unknown','low','medium','high')),
  next_experiment TEXT,
  status TEXT NOT NULL DEFAULT 'active' CHECK (status IN
    ('active','investigating','experimenting','shelved','abandoned')),
  pinned INTEGER NOT NULL DEFAULT 0
);
CREATE TABLE evidence_records (
  id INTEGER PRIMARY KEY,
  owner_type TEXT NOT NULL CHECK (owner_type IN
    ('observation','hypothesis','candidate','investigation','experiment')),
  owner_id INTEGER NOT NULL,
  raw_item INTEGER REFERENCES raw_items(id),
  snapshot INTEGER REFERENCES snapshots(id),
  url TEXT,
  excerpt TEXT,
  note TEXT,
  created_by TEXT NOT NULL CHECK (created_by IN ('operator','guidance')),
  confirmed INTEGER NOT NULL DEFAULT 1,
  created_at TEXT NOT NULL
);
CREATE TABLE exclusions (
  id INTEGER PRIMARY KEY,
  candidate INTEGER NOT NULL REFERENCES candidates(id) ON DELETE CASCADE,
  constraint_version INTEGER NOT NULL REFERENCES constraints(id),
  reason TEXT NOT NULL
);

-- scoring & economics
CREATE TABLE constraints (
  id INTEGER PRIMARY KEY,
  version INTEGER NOT NULL,
  capital TEXT NOT NULL,                 -- JSON {amount, currency}
  hours_per_week REAL NOT NULL,
  target_monthly_profit TEXT NOT NULL,  -- JSON {amount, currency}
  horizon_months INTEGER NOT NULL,
  operational_burdens TEXT NOT NULL DEFAULT '[]',  -- JSON list[str] chosen set
  sales_comfort TEXT NOT NULL CHECK (sales_comfort IN
    ('direct','inbound','both','none')),
  automation_tolerance TEXT NOT NULL CHECK (automation_tolerance IN
    ('low','medium','high')),
  unacceptable_risks TEXT NOT NULL DEFAULT '[]',   -- JSON list[str]
  created_at TEXT NOT NULL,
  active INTEGER NOT NULL DEFAULT 1      -- exactly one active row
);
CREATE TABLE scores (
  id INTEGER PRIMARY KEY,
  candidate INTEGER NOT NULL REFERENCES candidates(id) ON DELETE CASCADE,
  constraint_version INTEGER NOT NULL,
  c_spending INTEGER NOT NULL CHECK (c_spending BETWEEN 1 AND 5),
  c_accessibility INTEGER NOT NULL CHECK (c_accessibility BETWEEN 1 AND 5),
  c_margin INTEGER NOT NULL CHECK (c_margin BETWEEN 1 AND 5),
  c_compatibility INTEGER NOT NULL CHECK (c_compatibility BETWEEN 1 AND 5),
  c_advantage INTEGER NOT NULL CHECK (c_advantage BETWEEN 1 AND 5),
  c_validation INTEGER NOT NULL CHECK (c_validation BETWEEN 1 AND 5),
  rationale TEXT NOT NULL DEFAULT '{}', -- JSON {criterion: {text, author}}
  weighted_total REAL NOT NULL,
  computed_at TEXT NOT NULL,
  author TEXT NOT NULL CHECK (author IN ('operator','guidance_confirmed'))
);
CREATE TABLE economics_records (
  id INTEGER PRIMARY KEY,
  candidate INTEGER NOT NULL REFERENCES candidates(id) ON DELETE CASCADE,
  inputs TEXT NOT NULL,                 -- JSON EconomicsInputs
  outputs TEXT NOT NULL,                -- JSON EconomicsOutputs
  computed_at TEXT NOT NULL
);
-- weights are DATA rows (editable), one active set:
CREATE TABLE scoring_weights (
  id INTEGER PRIMARY KEY,
  c_spending REAL NOT NULL, c_accessibility REAL NOT NULL, c_margin REAL NOT NULL,
  c_compatibility REAL NOT NULL, c_advantage REAL NOT NULL, c_validation REAL NOT NULL,
  active INTEGER NOT NULL DEFAULT 0, created_at TEXT NOT NULL
);

-- validation
CREATE TABLE investigations (
  id INTEGER PRIMARY KEY,
  candidate INTEGER NOT NULL REFERENCES candidates(id) ON DELETE CASCADE,
  plan TEXT NOT NULL,
  unknowns_targeted TEXT NOT NULL DEFAULT '[]',
  status TEXT NOT NULL DEFAULT 'open' CHECK (status IN ('open','in_progress','done')),
  findings TEXT NOT NULL DEFAULT '[]'   -- JSON list[InvestigationFinding]
);
CREATE TABLE interview_guides (
  id INTEGER PRIMARY KEY,
  investigation INTEGER NOT NULL REFERENCES investigations(id) ON DELETE CASCADE,
  questions TEXT NOT NULL,              -- JSON list[str] (past-purchasing focus)
  notes TEXT
);
CREATE TABLE experiments (
  id INTEGER PRIMARY KEY,
  candidate INTEGER NOT NULL REFERENCES candidates(id) ON DELETE CASCADE,
  riskiest_assumption TEXT NOT NULL,
  design TEXT NOT NULL,
  failure_criteria TEXT,                -- JSON; write-once: NULL→set, then immutable
  budget_cap TEXT,                      -- JSON {research_usd?, validation_usd?}
  time_cap_hours REAL,
  status TEXT NOT NULL DEFAULT 'planned' CHECK (status IN
    ('planned','running','complete','cancelled')),
  result TEXT,
  proposition_vs_test TEXT CHECK (proposition_vs_test IN
    ('proposition_failed','test_failed','inconclusive','unclear')),
  created_at TEXT NOT NULL
);
CREATE TABLE falsification_notes (
  id INTEGER PRIMARY KEY,
  candidate INTEGER NOT NULL REFERENCES candidates(id) ON DELETE CASCADE,
  statement TEXT NOT NULL,              -- "what would convince me this is not viable"
  created_at TEXT NOT NULL,
  resolved_at TEXT,
  resolution TEXT
);

-- guidance
CREATE TABLE guidance_sessions (
  id INTEGER PRIMARY KEY,
  started_at TEXT NOT NULL,
  ended_at TEXT,
  focus_type TEXT CHECK (focus_type IN ('observation','candidate', NULL)),
  focus_id INTEGER,
  token_summary TEXT NOT NULL DEFAULT '{}',  -- JSON {prompt_tokens, completion_tokens}
);
CREATE TABLE guidance_messages (
  id INTEGER PRIMARY KEY,
  session INTEGER NOT NULL REFERENCES guidance_sessions(id) ON DELETE CASCADE,
  role TEXT NOT NULL CHECK (role IN ('operator','assistant')),
  content TEXT NOT NULL,
  created_at TEXT NOT NULL
);
CREATE TABLE guidance_proposals (
  id INTEGER PRIMARY KEY,
  session INTEGER NOT NULL REFERENCES guidance_sessions(id) ON DELETE CASCADE,
  action TEXT NOT NULL,                 -- 'note.write' | 'disposition.write' |
                                        -- 'score.write' | 'evidence.write' | 'delete'
  payload TEXT NOT NULL,                -- JSON per action (§8.9)
  state TEXT NOT NULL DEFAULT 'proposed' CHECK (state IN
    ('proposed','confirmed','declined'))
  decided_at TEXT
);
CREATE TABLE guidance_digests (
  id INTEGER PRIMARY KEY,
  session INTEGER NOT NULL REFERENCES guidance_sessions(id) ON DELETE CASCADE,
  topics TEXT NOT NULL DEFAULT '[]',    -- JSON list[str]
  proposals_summary TEXT NOT NULL DEFAULT '{}', -- {raised, confirmed, declined}
  writes TEXT NOT NULL DEFAULT '[]',    -- JSON list[{action, target}]
  open_questions TEXT NOT NULL DEFAULT '[]',   -- JSON list[str]
  created_at TEXT NOT NULL
);
CREATE TABLE write_audit (
  id INTEGER PRIMARY KEY,
  session INTEGER REFERENCES guidance_sessions(id),
  actor TEXT NOT NULL CHECK (actor IN ('operator','guidance')),
  action TEXT NOT NULL,
  target_table TEXT NOT NULL,
  target_id INTEGER NOT NULL,
  before_state TEXT,
  after_state TEXT,
  at TEXT NOT NULL
);

-- metering & ops
CREATE TABLE llm_usage (
  id INTEGER PRIMARY KEY,
  at TEXT NOT NULL,
  stage TEXT NOT NULL,                  -- 'extract' | 'hypothesize' | 'guidance'
  run INTEGER REFERENCES sweep_runs(id),
  session INTEGER REFERENCES guidance_sessions(id),
  model TEXT NOT NULL,
  prompt_tokens INTEGER NOT NULL,
  completion_tokens INTEGER NOT NULL,
  cost_usd REAL NOT NULL
);
CREATE TABLE exa_usage (
  id INTEGER PRIMARY KEY,
  at TEXT NOT NULL,
  source INTEGER REFERENCES sources(id),
  run INTEGER REFERENCES sweep_runs.id,
  kind TEXT NOT NULL CHECK (kind IN ('fetch','search')),
  cost_usd REAL NOT NULL,
  success INTEGER NOT NULL
);
CREATE TABLE ceiling_states (
  ledger TEXT NOT NULL CHECK (ledger IN ('llm','exa')),
  month TEXT NOT NULL,                  -- 'YYYY-MM'
  budget_usd REAL NOT NULL,
  spent_usd REAL NOT NULL DEFAULT 0,
  warned_at TEXT,
  PRIMARY KEY (ledger, month)
);
CREATE TABLE schema_migrations (
  id INTEGER PRIMARY KEY,
  applied_at TEXT NOT NULL
);
```

### 5.3 Store API (module → function signatures)

Each store module exposes typed functions over its tables (full signatures in §8 store specs; the complete list, one line each): `sources.py`: `create_source`, `get_source`, `update_source`, `list_sources`, `set_source_health`; `runs.py`: `create_sweep_run`, `finish_sweep_run`, `create_source_run`, `finish_source_run`, `record_fetch_attempt`, `list_sweep_runs`, `get_sweep_run`; `items.py`: `upsert_raw_item`, `get_raw_item`, `raw_item_exists`, `get_raw_items_for_source`, `create_snapshot`, `get_snapshot`, `pin_target`, `unpin_target`, `list_prune_candidates`; `changes.py`: `create_change_event`, `get_unextracted_change_events`, `mark_change_events_extracted`; `funnel.py`: `create_observation`, `get_observation`, `list_observations`, `update_observation_status`, `create_hypothesis`, `get_hypothesis`, `create_candidate`, `get_candidate`, `update_candidate`, `create_evidence_record`, `list_evidence_for_owner`, `create_exclusion`; `scoring.py`: `create_constraints`, `get_active_constraints`, `create_score`, `get_latest_score`, `create_economics_record`, `get_latest_economics`, `get_active_weights`, `set_active_weights`; `validation.py`: `create_investigation`, `update_investigation`, `create_interview_guide`, `create_experiment`, `set_failure_criteria`, `record_experiment_result`, `create_falsification_note`, `resolve_falsification_note`; `guidance.py`: `create_session`, `append_message`, `list_messages`, `create_proposal`, `decide_proposal`, `create_digest`, `get_digest`, `audit_write`; `metering.py`: `record_llm_usage`, `record_exa_usage`, `get_month_totals`, `get_ceiling`, `upsert_ceiling`, `warn_if_needed`.

---

## 6. API Specification

All routes under `/api/v1`, loopback-only Flask app. Request/response bodies are JSON. Host/Origin localhost gates on every request (§8.14). Error envelope per §1.3. Abbreviated schemas — full field lists follow §5 DDL; nested objects as specified in §8 module specs.

### 6.1 System

- `GET /status` → `{service:'vorbote', uptime_s, scheduler:{running:bool,next_sweep_at}, db_ok:bool, quarantined:[{source,reason}], ledgers:{llm:{month,spent_usd,budget_usd},exa:{…}}}`
- `GET /health` → `204` or error envelope.

### 6.2 Sources

- `GET /sources` → `[{id,name,family,group,enabled,requires_credential,health,quarantine_reason,last_success_at,language,jurisdiction}]`
- `POST /sources` body `{name,family,endpoint_config,language?,jurisdiction?,group?}` → `201 {id}`
- `GET /sources/<id>` → source + `endpoint_config` JSON + `groups` membership
- `PUT /sources/<id>` — edit `endpoint_config` (URL drift [Doc 1 §12.3]), group, language, enabled → `200`
- `DELETE /sources/<id>` → `204`
- `GET /sources/<id>/test` → runs **one polite fetch** (ladder step 1 only) → `{outcome, http_status, items?, sample_titles?, error?}` (no quarantine side-effects)
- `GET /sources/<id>/setup-requirements` → `{kind:'none'|'api_key'|'oauth_client'|'browser_cookies', steps:[…], credential_configured:bool}`
- `POST /sources/<id>/quarantine` body `{reason}` → `200`; `POST /sources/<id>/release` → `200`
- `POST /sources/<id>/cookies` body `{platform:'reddit'|'twitter', cookie_string}` → `200` — **secret-tunnel endpoint**: value injected into reach profile server-side; never stored in DB; never echoed; response contains no cookie material; audit: `source_credentials_status` row `configured=1` (kind `browser_cookies`). (This is the wizard upload path [Doc 1 §7].)

### 6.3 Sweeps & runs

- `POST /sweeps/trigger` body `{group?:str, all?:bool}` → `202 {sweep_run_id}` (manual trigger)
- `GET /runs` → list `[{id,started_at,finished_at,status,trigger_kind,stats}]`
- `GET /runs/<id>` → run + `source_runs:[{source,status,items_fetched,errors,ladder_summary}]`

### 6.4 Observations (triage)

- `GET /observations?status=&limit=&offset=` → list with evidence counts
- `GET /observations/<id>` → full: observation + evidence refs + linked change_events summary
- `POST /observations/<id>/promote` → creates `hypotheses` row (draft) → `{hypothesis_id}`
- `POST /observations/<id>/decline` → `200` (status → declined)
- `POST /observations/<id>/negative` body `{rationale}` → status → `negative_finding`, rationale stored in description addendum [FR-12]

### 6.5 Candidates

- `GET /candidates?status=` / `POST /candidates` body `{hypothesis_id, market_segment, observed_inefficiency, …}` → `201 {id}`
- `GET /candidates/<id>` → record
- `PUT /candidates/<id>` → `200`; `POST /candidates/<id>/pin` / `unpin` → `200`
- `GET /candidates/<id>/dossier` → assembled dossier: candidate + evidence + latest score + latest economics + validation state + linked hypothesis/observation lineage [Doc 1 §4]
- `GET /candidates/<id>/export?format=markdown|html` → rendered export (UTF-8 body; client saves via dialog) [FR-26]

### 6.6 Scoring & economics

- `GET /candidates/<id>/scores` → history `[{…,weighted_total,computed_at}]`; `POST` same path body `{c_spending,…,c_validation, rationale}` → computes `weighted_total` deterministically (active weights) → `201` (rejects if any exclusion active for candidate — exclusions gate scoring per [Doc 1 FR-16])
- `POST /candidates/<id>/economics` body `EconomicsInputs` (§8.6) → `201 EconomicsOutputs`
- `GET /constraints` → active + history; `POST /constraints` body `Constraints` → `201`, deactivates previous (version+1)

### 6.7 Validation

- `GET/POST /investigations` body `{candidate_id, plan, unknowns_targeted?}`; `PUT /investigations/<id>` (findings appended)
- `POST /investigations/<id>/guides` body `{questions}` → interview guide
- `GET/POST /experiments` body `{candidate_id, riskiest_assumption, design, budget_cap?, time_cap_hours?}`
- `PUT /experiments/<id>/failure-criteria` body `{criteria:[…]}` → write-once: `409 CONFLICT` if already set [FR-19 immutability]
- `POST /experiments/<id>/result` body `{result, proposition_vs_test}` → `200`

### 6.8 Guidance

- `POST /guidance/sessions` body `{focus_type?, focus_id?}` → `201 {session_id}`
- `GET /guidance/sessions/<id>` → session + digest if closed
- `GET /guidance/sessions/<id>/messages` → `[{role,content,created_at}]`
- `POST /guidance/sessions/<id>/messages` body `{content}` → `200 {reply, proposals:[…]}` — the **only** assistant-reply path (server-side LLM call, metered)
- `GET /guidance/sessions/<id>/proposals` → `[{id,action,payload,state}]`
- `POST /guidance/proposals/<id>/confirm` → applies the write through `guidance/writes.py` → `200` (+ write_audit row)
- `POST /guidance/proposals/<id>/decline` → `200`
- `POST /guidance/sessions/<id>/close` → generates digest (deterministic) → `200 {digest_id}`
- `GET /guidance/digests/<id>` → digest record

### 6.9 Evidence & pins

- `GET /snapshots/<id>` → file content (correct mime)
- `POST /pins` body `{target_kind, target_id, reason}` → `201`
- `GET /pins?target_kind=&target_id=` → list

### 6.10 Ledgers

- `GET /ledgers/llm?month=YYYY-MM` → `{stage_totals, run_totals, session_totals, spent_usd, budget_usd, warned:bool}`
- `GET /ledgers/exa?month=YYYY-MM` → `{source_totals, spent_usd, budget_usd, warned:bool}`

### 6.11 Concurrency contract

- One sweep at a time (orchestrator lock); manual trigger during sweep → `409`.
- API reads during sweep are always safe (WAL + short transactions).
- Guidance confirm applies writes inside one transaction per proposal.

---

## 6.12 Client→server route map (tunnel)

All client features map onto the above routes 1:1 (triage ← 6.4; dossier ← 6.5; scoring ← 6.6; validation ← 6.7; guidance chat ← 6.8; sources/cookies ← 6.2; ledgers ← 6.10). The tunnel is transport-only — no additional server routes exist beyond `/api/v1`.

---

## 7. AuthN/AuthZ

- **No application-level auth.** The SSH local forward is the trust boundary; API binds `127.0.0.1` only [Doc 1 §7].
- Defense-in-depth: every request must pass Host/Origin localhost gates (§8.14); non-local Origin → `403`.
- Secrets live only in the server secrets file + reach profile (§4.1, §8.10); no secret-setting API except the cookie-injection endpoint (6.2), which accepts and forwards but never stores/echoes secret material.
- Cookie upload path: client → tunnel → `POST /sources/<id>/cookies` → `reach_bridge.inject_cookies(platform, cookie_string)` → written into profile dir with `0600` perms; `source_credentials_status.configured=1`; nothing logged except the audit fact (no values).
- Client stores no secrets (QSettings §4.4 has none).

---

## 7.1 Threat notes

- Untrusted evidence text (web content) enters prompts: extractor/hypothesis roles use strict JSON output contracts + deterministic post-validation; guidance roles never receive raw evidence text outside assembled contexts [Doc 1 §7].
- Prompt-injection resilience: LLM outputs never execute; proposals are validated against closed action sets (§8.9) before confirmation.

---

## 7.2 Secrets ledger (what lives where)

| Secret | Residence |
|---|---|
| LLM API key | server `.env` only |
| Exa key | server `.env` only |
| DIP key / Google Ads / SP-API creds | server `.env` only (optional sources) |
| Reddit/X cookie strings | reach profile dir on server (0600) — uploaded via 6.2 |
| Everything in the DB | no secret material anywhere |

---

## 7.3 Test-time secrets

Tests use fixture keys ("test-key-…"); no real credentials in repo; hermetic servers replace all outbound calls [NFR-11].

---

## 8. Module Specifications

Types live in `schema.py` (single source of truth); every type named in any signature below is defined there or in §5 DDL. JSON blob shapes are pinned in `schema.py` as frozen dataclasses with `to_json/from_json`.

### 8.0 `schema.py` — shared types (excerpt; complete catalog)

`CanonicalItem{source_id:int, external_key:str, title:str, url:str|None, author:str|None, published_at:str|None, fetched_at:str, language:str, jurisdiction:str, parse_tier:str, content:str|None, content_hash:str, snapshot_id:int|None}` · `EndpointConfig` = family-JSON per §8.1–8.8 · `ErrorInfo{code:str, message:str, detail:obj|None}` · `EvidenceRef{kind:'raw_item'|'snapshot'|'url', id:int|None, url:str|None, excerpt:str|None}` · `EconomicsInputs{price:{amount,currency}, vat_rate:float, volume_assumptions:{monthly_transactions:int}, variable_costs:list[{name,amount,unit:'per_transaction'|'monthly_fixed'}], owner_hours_per_month:float, fixed_costs:list[{name,amount}], required_profit:{amount,currency}}` · `EconomicsOutputs{contribution_per_transaction:float, monthly_operating_profit:float, effective_hourly:float, required_transactions:int, notes:list[str]}` · `ProposalPayload` variants per §8.9 · `DigestSpec` fields per DDL `guidance_digests`.

### 8.1 `sources/base.py` — adapter contract

```python
class Adapter(Protocol):
    family: str
    def fetch(self, source: SourceRow, plan: FetchPlan) -> FetchResult
class FetchPlan{ladder:Ladder, politeness:PolitenessGovernor, exa:ExaClient|None}
class FetchResult{items:list[CanonicalItem], errors:list[ErrorInfo], ladder_summary:dict}
```
`Ladder` (§8.4) is the only outbound path; adapters never use httpx directly.

### 8.2 `sources/feeds.py` — feed, change_feed, trends families

- Parses RSS/Atom via feedparser; `external_key` = stable entry id (guid/`<link>` fallback).
- Trends adapter (S-12): parses `<ht:approx_traffic>` into `payload.traffic_hint`; language `de` for `geo=DE`.
- Change-feed semantics (S-01): new entry ⇒ change event `new_item` with amendment metadata parsed from title/summary.
- Politeness class default 4 s; override per politeness table.
- Dedupe: `UNIQUE(source, external_key)`; content-hash change on same key ⇒ `value_delta` event (summary changed).
- Exa assist posture: none (feeds are direct).

### 8.3 `sources/api.py` — rest_api family (S-02, S-05, S-07, S-15, S-18, S-19)

- Endpoint template per source in `EndpointConfig`: `{url_template:str, method:'GET'|'POST', body_template:dict|None, paging:'none'|'offset'|'date-window', auth:'none'|'api_key_header'|'oauth2_client'}`.
- TED (S-02): `POST` expert query `PD>=YYYYMMDD`; **GET rejected (405) — POST-only**; notice XML links fetched via ladder for snapshotting; paging: date-window iteration.
- DIP (S-05): key in header; rate-capped; `vorgang`/`drucksache`/`plenarprotokoll` entity endpoints; guided setup stores key presence only (value in `.env`).
- Eurostat (S-07): GET JSON-stat; `geo=DE` filter; value-delta detection on selected series.
- HN (S-15): GET Algolia `search_by_date`; `tags=story`; language `en`.
- Keyword Planner (S-18): OAuth2 client credentials → generateKeywordIdeas; **disabled until creds present**; volumes are ranges → stored verbatim with range marker; never averaged into point estimates by LLM (C-08).
- SP-API (S-19): LWA token refresh; **disabled until creds present**; supplements S-16 with offer counts/prices; values land as evidence, not detection inputs, until operator pins them.
- Politeness: 4 s default; TED/DIP GET-paging 6 s (gentle with regulators).

### 8.3.1 Family mapping (normative)

The rest_api family is exactly {S-02, S-05, S-07, S-15, S-18, S-19}, all served by this module. The dataset family is {S-06 DESTATIS, S-08 GovData} — plus Eurostat snapshot tables if the operator adds them — served by `sources/datasets.py` (§2 inline comment; [Doc 1 §12.2]). S-18 and S-19 ship `enabled=false` until credentials are configured (§4.1, §4.2).

### 8.4 `ladder.py` + `politeness.py` — retrieval ladder

- `class Ladder{async def fetch(source, request: LadderRequest) -> LadderResponse}` — the single outbound HTTP path for all adapters (httpx client held by composition root).
- Step 1: deterministic fetch via the family adapter's native fetcher, governed by `PolitenessGovernor`: per-source asyncio lock (single-flight), inter-request delay (per politeness table), exponential backoff `backoff_base_s * 2^retry` honoring `Retry-After` (capped), max retries 4, then step 2.
- Step 2: `ExaClient.fetch|search` (§8.5); `exa_disabled` outcome if no key → source marked `skipped_exa_needed` in source_run (not a failure; quarantine only if source is `exa_required`).
- LLM-driven retrieval: guidance engine only; goes through `Ladder` steps 1–2 — never direct.
- All attempts logged as `fetch_attempts` rows (ladder_step 1 or 2).

### 4→8 renumber note (correcting §4 line references)

§4's politeness table lists overrides keyed by source name (e.g. `insolvenzbekanntmachungen`); those names must equal the `sources.name` slugs of seed data (§4.2). This is the normative linkage.

### 8.5 `exa_client.py`

- `class ExaClient{def __init__(base_url, api_key:str|None), async def fetch(url)->ExaFetchResult, async def search(query, num)->list[ExaSearchResult]}`
- No key ⇒ client constructed with `api_key=None` → methods return `ExaDisabledError` → ladder records `exa_disabled`, metering not touched.
- Each call metered: `record_exa_usage(kind, cost=price_per_call)`; ceiling checked after each call (warn-only: `ceiling_states.warned_at` set once per month per ledger).

### 8.6 `evaluation/economics.py` — deterministic calculators

Pure functions; no LLM anywhere; all inputs from `EconomicsInputs`:

```
contribution_per_transaction = price.amount / (1 + vat_rate) - Σ(variable_costs where unit='per_transaction')
monthly_operating_profit    = monthly_transactions × contribution - Σ(fixed_costs) - Σ(variable_costs where unit='monthly_fixed')
effective_hourly           = monthly_operating_profit / owner_hours_per_month
required_transactions      = ceil( (required_profit + Σ fixed + Σ monthly_fixed) / contribution )
```
- Output `notes` list flags: negative contribution; hourly below a `min_hourly` (config, default 0 → flag only); required transactions vs `monthly_transactions` mismatch.
- Every input carries provenance tag in UI (`operator-estimate` / `evidence-ref`) [C-08]; the module never invents defaults for missing inputs — missing → validation error.

### 8.7 `evaluation/scoring.py`

- `compute_score(criteria: dict[str,int], weights: dict[str,float]) -> float` — deterministic weighted sum; criteria must be 1–5 ints; weights normalized (sum to 1) before weighting.
- `apply_exclusions(candidate, constraints, exclusions) -> bool` — exclusion check before scoring [Doc 1 FR-16].
- Reused by guidance proposals for `score.write` (values still operator-confirmed).

### 8.8 `guidance/engine.py` + `writes.py` + `digest.py`

- `class GuidanceEngine{async def start_session(focus)->Session, async def send_operator_message(session_id, content)->ReplyWithProposals, async def close_session(session_id)->Digest}`
- Context assembly: deterministic — pulls dossier/observations/evidence/constraints/scores for focus; cap `max_context_items` (40) [Doc 1 §3.6]; the LLM sees assembled context only, never DB access.
- Roles via system prompts (`prompts/guidance-*.md`): interviewer, analyst, research-planner (proposes queries → executed via Ladder by server, results appended to context on next turn), scribe.
- **Bounded writes** (FR-23): `writes.py` closed action set — direct: `note.write`, `disposition.write`; confirm-required: `score.write`, `evidence.write`, `delete`. The engine may ONLY emit proposals from the confirm-required set plus direct writes via `writes.py.apply_direct(action, payload)` which itself inserts `write_audit` rows.
- Proposals: `guidance_proposals` rows `state=proposed`; confirm endpoint applies via `writes.py.apply_confirmed(proposal)`; decline marks declined. Payload validation against closed schemas per action (§8.9).
- Digest (FR-25): deterministic from session records — topics (assistant message headings), proposals summary, writes (from write_audit for that session), open questions (last assistant message's open-questions block); generated at `close_session`; stored `guidance_digests`.

### 8.9 Proposal payload schemas (closed set)

| Action | Payload schema |
|---|---|
| `note.write` | `{target_type, target_id, note}` |
| `disposition.write` | `{target_type, target_id, status}` (status ∈ allowed set for target type) |
| `score.write` | `{candidate_id, c_spending:int 1–5, …, rationale}` |
| `evidence.write` | `{owner_type, owner_id, EvidenceRef}` |
| `delete` | `{target_type, target_id}` |

Any other action string in an LLM proposal → discarded before persistence (never surfaces as a proposal).

### 8.10 `reach_bridge.py` — headless browser + cookie injection

- `class ReachBridge{def __init__(chromium_bin, profile_dir, node_bin, timeout_s, call_gap_s), def call_channel(platform:'reddit'|'twitter', action:'search'|'read', args:dict) -> ReachResult, def inject_cookies(platform, cookie_string:str) -> None, def login_state(platform) -> 'ok'|'stale'}`
- Runtime: pinned in-repo adaptation of the reach channel CLI (Node runtime; Node ≥18 host prereq) driving the **headless Chromium-class browser**; `chromium_bin` auto-detects or uses config; profile dir on server disk (0600) — **no interactive login ever occurs on the server** [Doc 1 §7/AS-04].
- `inject_cookies` (§6.2): validates string non-empty; writes platform cookie file inside profile; sets `source_credentials_status(browser_cookies, configured=1)`; never logs values; response echoes nothing.
- `login_state`: bridge status call maps `NEED_LOGIN`/`PLATFORM_BLOCKED` → `stale` → source quarantine reason `login state stale`; wizard prompts re-upload [Doc 1 §7].
- Error mapping (from channel JSON contract): `NEED_SETUP`→quarantine `setup incomplete`; `NEED_LOGIN`→`login state stale`; `RATE_LIMITED`/`TIMEOUT`/`BUSY`→ backoff + retry next sweep (never hammering; `call_gap_within_source_s` enforced ≥10 s between same-platform calls); `UPSTREAM_BROKEN`/`NOT_FOUND`→ per-item error, no quarantine.
- Netizen: read-only actions only (search/read); no write actions exposed by the bridge API.
- Concurrency: one bridge call at a time per platform; queue with the gap; whole bridge is sequential (serialized calls) — single-threaded command execution against the headless browser.

### 10.1 (moved from §8.10 note) secrets in reach profile

Cookie files inside `profile_dir` are secret (0600); excluded from any log; `backup.sh` **excludes** the reach profile (§13); DB never contains cookie material [§7.2].

### 8.11 `normalizer.py` + `snapshots.py`

- Normalizer: language detect (fast heuristic on title+snippet; `langdetect`-class optional — implemented as small in-house classifier to avoid extra dep), jurisdiction from source row, `parse_tier` mapping per family.
- Snapshots: content-addressed write (hash → path); `pypdf` text layer; scanned-PDF detection (no extractable text) → `pdf_flagged_no_text` + snapshot retained (evidence-grade, excluded from extractor input).

### 8.11.1 PDF pipeline note

`pypdf` extracts text; if extracted text < 100 chars over ≥ 4 pages → flagged tier. Flagged PDFs are stored but produce no `full_text` items — extraction stage skips them [AS-05].

### 8.12 `changedetect.py`

Per family (deterministic only [FR-10]):
- feed/change_feed/trends: new_item (unseen external_key); trends adds value_delta on traffic_hint change ≥ threshold (default 2×).
- rest_api: new_item + value_delta (Eurostat series; DIP procedure updates via content_hash).
- html_diff/marketplace: list_diff (row set change), rank_movement (Amazon top-N ranks; threshold default ±3 positions), snapshot_diff (page-level hash when structureless), new_item.
- dataset: value_delta + snapshot_diff (table snapshot compare; DESTATIS/GovData).
- change_events row per event; `extracted=0` until consumed.

### 8.13 `llm.py` — single-path LLM client

- `class LLMClient{async def chat_completion(messages, model:str|None, temperature=0.5, max_tokens=2000) -> LLMResult{content:str, prompt_tokens:int, completion_tokens:int}}` — the **only** chat-completions code path (mirrors Doc 1 D6); provider table in config: openrouter/opencode_go/deepseek/custom with base_urls; model string `glm-5.3-flash` [C-03]; `reasoning_effort` forwarded only when set.
- Every call metered into `llm_usage` (stage tag) and ceiling-checked after call (warn-only).
- Strict output contracts for extract/hypothesize stages: JSON schema validation; violation → degrade-to-raw (observation gets raw text as description; no fabricated structure) [Doc 1 D2, C-08].

### 8.13.1 `extract.py` + `hypothesize.py` stage wrappers

- `extract.py`: consumes unextracted `change_events` (batch ≤ 20); builds evidence context from linked raw items (title + excerpt ≤ 500 chars each); extractor prompt → JSON `{driver_tag, affected_population_hint, sector_hint, language}` per event; validation: closed tag sets; on failure → event stays `extracted=0`, retried next sweep; on success → marks extracted, passes to hypothesize.
- `hypothesize.py`: batch ≤ 10 extracted events; hypothesis prompt → JSON observations draft `{title, description, proposed_explanation, alternative_explanations:list[str]≥1, unknowns:list[str]≥1, evidence_refs}`; validation: ≥1 alternative and ≥1 unknown enforced (else reject → degrade-to-raw observation with raw text, flag `llm_contract_violation` in run stats); creates `observations` rows (status triage) + evidence link rows.

### 8.14 `web/api.py` — routes + Host/Origin gates

- Flask app factory; all §6 routes; before-request gate: `Host` must be `127.0.0.1[:port]` or `localhost[:port]`; `Origin` (if present) must be localhost-scheme; else `403`.
- Sweep lock: one orchestrator sweep at a time (thread lock) — `POST /sweeps/trigger` during active sweep → `409`.
- `POST /sources/<id>/cookies` handler: no-echo discipline (request body parsed, value forwarded, response contains only `{ok:true}`); audit row via `write_audit`-style log (action `cookies.injected`, target sources row; **no values**).

### 8.15 `export/dossier.py`

- Dossier assembly from store APIs; Markdown renderer (headings, evidence blocks with excerpts, score table, economics table, validation timeline) and HTML renderer (same structure, CSS inline).
- Export body returned via API (§6.5); client saves via file dialog [FR-26].

### 8.16 `scheduler.py` + `orchestrator.py`

- Scheduler: daily window per group schedule_spec + catch-up (one per missed window, bounded) + startup sweep of interrupted runs (status running→interrupted at boot) [Doc 1 §9].
- Orchestrator: plans due sources (enabled, health ok or release-able, credential checks for optional sources), runs families through adapters via Ladder, records source_runs/fetch_attempts, isolation per source (source failure ≠ sweep failure), post-sweep: change detection → extract → hypothesize → triage queue; stats into sweep_run.stats; retention prune last.

### 8.17 `retention.py`

- Daily post-sweep prune: candidates = snapshots + raw items not referenced by any evidence-bearing row (evidence_records, change_events.raw_item_ids, observations.evidence) AND `fetched_at < now - months` AND not pinned → FIFO delete (snapshot file + row), count+bytes → warning surfaced in client [Doc 1 §3.7].
- Referenced detection = pins table + live FK-style scan queries (deterministic; SQL joins over evidence_records, change_events, observations).
- Storage ceiling: warn-only (stats endpoint).

### 8.18 `metering.py`

- `record_llm_usage` (stage, run/session refs, tokens) → cost = tokens × blended prices → `llm_usage` + ceiling upsert (warn-only). Monthly rollup view per §6.10.
- Exa per §8.5. `get_month_totals` for ledgers.

### 8.19 `cli.py`

- `vorbote init` (create state dir, DB, seeds, config skeleton, systemd unit copy), `vorbote run [--group|--all]` (one-shot sweep; exit 0/1), `vorbote serve` (API + scheduler thread), `vorbote help` (vorbote help).

### 8.20 `vorbote_client/*` (client)

- `app.py`: QMainWindow; left nav (Triage/Dossier/Scoring+Economics/Validation/Constraints/Sources/Guidance/Settings); `tunnel.py` spawns `ssh -N -L …`; `api.py` thin async client; `workers.py` QThread wrappers; `i18n.py` EN/DE catalogs, runtime switch.
- `widgets/triage.py`: queue table (status filters), evidence-count column, actions: promote/decline/negative (with rationale dialog).
- `widgets/dossier.py`: record + evidence browser (snapshot preview for pdf_text via text dump), export buttons (markdown/html) → `export_download.py`.
- `widgets/scoring.py`: criteria entry (1–5 spinboxes) + rationale per criterion; deterministic score preview (computed client-side identically to server formula §8.7 — same formula, no LLM); economics panel: inputs form (all fields provenance-tagged), outputs panel; reverse calculation display.
- `widgets/validation.py`: investigations list, interview guides (question list editor), experiments (riskiest assumption, design, failure criteria write-once editor with confirm dialog, result recording incl. proposition_vs_test).
- `widgets/constraints.py`: optionally-guided editor (guided mode = assistant-drafted questions sequence stored in client i18n, not LLM), versioning display.
- `widgets/sources.py`: source table (health/quarantine/credential status), per-source wizard: (1) add/edit incl. URL update; (2) test fetch; (3) setup-requirements display; (4) **cookie upload** for reddit/twitter (textarea for cookie string, upload → 6.2 endpoint); (5) quarantine/release.
- `widgets/guidance.py`: session list, chat pane, proposal cards (confirm/decline), digest view post-close.
- `widgets/settings.py`: server settings mirror (read + port/schedule edits via API where applicable), ledgers view.
- `app.py` languages: EN default; full DE catalog; source excerpts rendered verbatim (no translation) [FR-27].

### 8.21 `prompts/` — role prompts

`extractor.md`, `hypothesis.md`, `guidance-interviewer.md`, `guidance-analyst.md`, `guidance-planner.md`, `guidance-scribe.md` — EN text, pinned output contracts (JSON shapes per §8.13.1/§8.9); no numbers rule baked into hypothesis prompt [C-08].

---

## 9. Background Jobs

| Job | Trigger | Behavior |
|---|---|---|
| Daily sweep | scheduler window (05:00 default) | §8.16 full pipeline; one at a time (lock) |
| Group sweeps | per-group schedule_spec | subset sweep of group's sources |
| Extraction retry | next sweep | events with `extracted=0` re-batched (idempotent) |
| Retention prune | post-sweep | §8.17 |
| Ledger rollup | month boundary (scheduler tick) | `ceiling_states` month row create; warned flags reset |
| Startup sweep | serve boot | interrupted `running` rows → `interrupted`; no auto re-run (next window or manual) |
| Backup | operator cron (documented) | `sqlite3 …VACUUM INTO` + snapshot rsync (excludes reach profile) |

No other background jobs exist. Guidance sessions are synchronous request-scoped (operator-driven).

## 9.1 Job failure semantics

- Sweep: per-source isolation; sweep fails only if orchestrator itself errors; failed source → source_run errors + quarantine threshold (3 consecutive failures → quarantine `repeated failure`) — release via wizard.
- Quarantined sources: skipped at plan time; wizard can release (health→ok) → next window.
- LLM unavailable: collection/detection continue; extract/hypothesize events remain `extracted=0` → retried idempotently next sweep; no data loss (raw items + change_events persist) [NFR-08].

## 10. Observability

- Structured logs (stdlib logging, JSON lines): component tag, run id, source id; secrets never logged (G1-class discipline [Doc 1 NFR-05]).
- `GET /status` and `/health` per §6.1; run stats; `fetch_attempts` inspection via runs endpoints; guidance `write_audit` browsable via client.
- LLM/Exa ledgers per §6.10 with stage/source breakdown; ceiling warn flags surfaced in client settings.
- Log retention: logrotate config documented in deployment guide (§13).

## 10.1 Ops runbook pointers

Deployment guide (§13 / `docs/deployment.md`) covers: init sequence, systemd enable, cookie upload runbook, quarantine release, backup/restore, URL-drift runbook (§12.3 of Doc 1: test fetch → edit endpoint → re-enable).

## 11. Error Handling

| Situation | Behavior |
|---|---|
| Deterministic fetch fails (retryable) | Backoff within ladder step 1 (≤4 retries, Retry-After honored, capped); then Exa step 2; then `fetch_attempts.outcome=fatal_error` → source_run error, next sweep retries |
| Exa disabled (no key) | `exa_disabled` outcome; source skipped if `exa_required` (quarantine `setup incomplete`); otherwise deterministic-only continues |
| Reach: NEED_LOGIN / cookies stale | Quarantine `login state stale`; wizard re-upload; no auto-retry loop |
| Reach: RATE_LIMITED / TIMEOUT / BUSY | Respect `retry_after_seconds` / gap ≥ 10 s; retry next sweep; never hammer |
| LLM call fails | Stage retry ≤ 2 (in-client); events stay unextracted; sweep continues; warn in stats |
| LLM contract violation | Degrade-to-raw (§8.13.1); run stats flag `llm_contract_violation` |
| PDF without text layer | `pdf_flagged_no_text`; stored, excluded from extractor |
| DB busy (locked) | Store modules retry 3× (100 ms, 200 ms); then error envelope `500 DB_BUSY` |
| Invalid proposal action | Discarded pre-persistence (§8.9) |
| Cookie upload malformed | `400 BAD_COOKIE_STRING` (non-empty string required); no partial writes |
| Manual sweep during sweep | `409 SWEEP_IN_PROGRESS` |
| Failure criteria already set | `409 CONFLICT` (write-once [FR-19]) |
| Exclusion active + score POST | `422 EXCLUSION_ACTIVE` (scores gated by exclusions [Doc 1 FR-16]) |
| Ceiling exceeded (LLM/Exa) | **Warn-only**: ceiling_states.warned_at set; status + ledgers surface warning; operation continues [D16] |

## 12. Testing Strategy

Hermetic; pytest only gate [NFR-11]. Fixture infra (in `tests/conftest.py`): loopback Flask test client; seeded in-memory/temp-dir DB (real migrations applied); fixture HTTP server (aiohttp/`http.server` thread) playing every external endpoint class (feed XML, API JSON, listing HTML, PDF, Exa mock, LLM mock `/v1/chat/completions`, reach-CLI mock emitting the channel JSON contract); temp snapshot store.

**Test-construction rule (binding)**: a test may never hand-build the state the code under test is supposed to produce. Pre-existing state is established **by calling the production path** (e.g. run a sweep against fixtures, promote an observation through the API) — never by direct fixture writes to `observations`/`change_events`/`scores` etc. Fixtures write only what production code consumes as **input** (source rows, seed config, fixture responses).

Levels:
1. **Unit**: economics (§8.6 exact formulas, edge cases: negative contribution, zero hours), scoring (weights normalization, exclusions gate), politeness (backoff schedule, delay enforcement, single-flight), retention references/pins, proposal payload validation, digest determinism, i18n completeness, Host/Origin gate.
2. **Integration (via API)**: full sweep on fixture sources → change_events → extract (LLM mock with contract-valid + contract-violating variants) → hypotheses → triage; promote → hypothesis → candidate; score with exclusion gate; economics; guidance session with proposal confirm/decline → write_audit; cookie upload → credentials_status + no echo; failure-criteria write-once; catch-up windows; retention prune with pin.
3. **Client (offscreen)**: `QT_QPA_PLATFORM=offscreen` widget tests per view (triage actions, scoring entry, economics form, cookie wizard flow, guidance proposal cards, i18n switch), tunnel spawn command construction, export download.
4. **Guard files** (named, per behavior): `test_secrets` (no secret in logs/DB/API under forced paths), `test_ladder` (order, retries, exa fallback, disabled), `test_politeness` (delays, single-flight), `test_reach_bridge` (error mapping, cookie injection, no-echo), `test_guidance_writes` (bounded set, audit rows, confirm flow), `test_economics`, `test_scoring`, `test_retention`, `test_api_gates` (Host/Origin, 409s, write-once), `test_i18n`.

Coverage target: server modules ≥ 90% line coverage (pytest-cov not a gate — internal target); guard files must stay green; no test weakens a spec invariant (write-once criteria, bounded writes, warn-only ceilings, no-echo cookies, degrade-to-raw).

## 13. Build/Run/Deploy

- Install: `python3.12 -m venv .venv && pip install -e .` (+ `pip install -e ./client` on desktop).
- Init: `vorbote init` → state dir, DB + migrations, source seeds (20 rows), config skeleton, `.env` skeleton (keys blank; exa blank → deterministic-only mode out of the box), systemd unit copied to `deploy/`.
- Serve: `vorbote serve` (binds 127.0. excludes public). systemd unit `deploy/vorbote.service`: `Environment=VORBOTE_HOME=/var/lib/vorbote`, `Restart=on-failure`, journal logging.
- Run once: `vorbote run --all` (exit 0/1).
- Backup: `deploy/backup.sh` — `sqlite3 $VORBOTE_HOME/vorbote.db "VACUUM INTO …"` + `rsync -a --exclude reach-profile` snapshots dir; nightly operator cron; restore = copy back + re-apply migrations (idempotent).
- Client: `vorbote-client` entry point; first-run: settings (host/user/port), tunnel test, language choice; constraints editor on first launch (optionally guided).
- Upgrade: `pip install -e .` + `vorbote migrate` (applies pending migrations; init is no-op on existing).
- Deployment guide `docs/deployment.md` covers: prerequisites (Node ≥ 18, Chromium headless, `CHROME_BIN`), first sweep checklist, cookie upload runbook, URL-drift runbook, quarantine release, ledgers review cadence.

## 14. Acceptance Criteria

Project-level definition of done (mirrors Doc 1 §2). A check item is verified by the tests of §12 level 2/3 and the runbook of §13:

1. `vorbote init && vorbote serve` on a clean host (with prerequisites) brings up API + scheduler; `GET /status` returns live status with 20 seeded sources.
2. Daily sweep (or `run --all`) executes all enabled sources via the ladder with politeness (delays, single-flight, backoff observed in fetch_attempts); per-source isolation holds (fixture-source failure doesn't fail sweep).
3. Change events appear per family detection kinds; extractor + hypothesize stages run (LLM mock in tests; live LLM in runbook verification); observations with ≥1 alternative and ≥1 unknown land in triage.
4. Triage actions work end-to-end; promote → hypothesis → candidate; scoring gated by exclusions; economics outputs match §8.6 formulas exactly.
5. Guidance session: context assembly, roles, proposal confirm/decline with write_audit rows; bounded write set enforced (unknown action discarded); digest generated on close; no DB write without audit row.
6. Cookie upload wizard: upload → credentials_status configured; bridge `login_state` ok; stale → quarantine `login state stale` → re-upload works; no cookie value in any log/response/DB.
7. Exa degradation: no key → ladder records `exa_disabled`; `exa_required` sources quarantine visibly; deterministic-only sweep completes.
8. Warn-only ceilings: exceeding LLM/Exa budget in fixtures sets warned flags; operations continue; ledgers show breakdown.
9. Retention: prune removes only unreferenced, unpinned, older-than-N items; referenced/pinned survive; FIFO warning surfaced.
10. Client: all views operate against fixtures; i18n switch EN↔DE; exports (markdown/html) render full dossiers; offscreen client tests green.
11. Failure criteria immutable: second write attempt → 409.
12. Hermetic suite green (`pytest tests/` + offscreen client tests); no live network anywhere in tests; guard files green; test-construction rule held (no hand-built funnel state in any test).
13. PDF pipeline: text-layer PDFs extract; scanned PDF flagged, stored, excluded.
14. Secrets: no secret material in DB, logs, or API responses under forced paths (guard test).
15. Degrade-to-raw on LLM contract violation; run stats flag it; no fabricated numbers anywhere (economics/scoring inputs provenance-tagged or rejected).

## 15. Seam Register & Call Traces (executability record)

**Composition root**: `src/vorbote/db.py::build_container()` — constructs, in order: SQLite handle + migrations; `PolitenessGovernor`; `LLMClient`; `ExaClient` (or `None`-keyed disabled); `ReachBridge`; `Ladder`; `AdapterRegistry` (family → adapter instance, wired with Ladder + politeness); `Orchestrator`; `Scheduler`; `GuidanceEngine` (wired with LLMClient + writes + digest); `RetentionPruner`; `Metering`; Flask app. **No other module instantiates infrastructure** — every consumer receives its dependency via constructor from `db.py` (or a factory it owns for pure-domain objects). Client composition: `vorbote_client/app.py::main()` builds tunnel → api client → widgets.

**Seam register (cross-module handoffs, each with named constructor/invoker)**:

| # | Seam | Caller → Callee (constructor/invoker) | Types crossing |
|---|---|---|---|
| S1 | Ladder fetch | `sources/*.fetch` → `ladder.py::Ladder.fetch` | `LadderRequest→LadderResponse` |
| S2 | Politeness gate | `ladder.py` → `politeness.py::PolitenessGovernor.wait_for_slot(source)` | `SourceRow` |
| S3 | Exa fallback | `ladder.py` → `exa_client.py::ExaClient.fetch/search` | `url/query → ExaResult` |
| S4 | Exa metering | `exa_client.py` → `metering.py::record_exa_usage` | kind, cost |
| S5 | Reach call | `sources/reach.py` → `reach_bridge.py::ReachBridge.call_channel` | `platform, action, args → ReachResult` |
| S6 | Cookie inject | `web/api.py` (route) → `reach_bridge.py::ReachBridge.inject_cookies` | `platform, cookie_string` |
| S6b | Cookie route → credentials | `web/api.py` → `store/sources.py::mark_credential_configured` | source id, kind |
| S6c | Wizard upload | `widgets/sources.py` → `api.py::post_cookies` → `web/api.py` | cookie string (secret; no echo) |
| S7 | Raw item persist | `orchestrator.py` → `store/items.py::upsert_raw_item` | `CanonicalItem` |
| S7b | Snapshot persist | `normalizer/snapshots` → `store/items.py::create_snapshot` | bytes→hash/path |
| S8 | Change detect | `orchestrator.py` → `changedetect.py::detect(source, items, prior)` | `→ list[ChangeEventDraft]` |
| S8b | Change persist | `changedetect.py` → `store/changes.py::create_change_event` | draft |
| S9 | Extract stage | `orchestrator.py` → `extract.py::run(sweep_run)` → `llm.py::chat_completion` | batch → JSON |
| S9b | Extract persist | `extract.py` → `store/funnel.py::create_observation` | observation draft |
| S10 | Hypothesize stage | `orchestrator.py` → `hypothesize.py::run(sweep_run)` → `llm.py` | batch → JSON |
| S10b | Hypothesis persist | `hypothesize.py` → `store/funnel.py::create_observation` (+ evidence) | draft + refs |
| S11 | LLM metering | `llm.py` → `metering.py::record_llm_usage` | stage, tokens |
| S12 | Triage action | `web/api.py` → `store/funnel.py::update_observation_status` / `evaluation/hypotheses.py` | status/id |
| S13 | Promote → hypothesis | `evaluation/observations.py` → `store/funnel.py::create_hypothesis` | obs id |
| S14 | Candidate CRUD | `web/api.py` → `store/funnel.py` / `evaluation/candidates.py` | fields |
| S15 | Scoring compute | `web/api.py` → `evaluation/scoring.py::compute_score` | criteria+weights→total |
| S16 | Exclusion gate | `evaluation/scoring.py` → `store/scoring.py::get_active_constraints` + `store/funnel.py::list_exclusions` | candidate |
| S17 | Economics compute | `web/api.py` → `evaluation/economics.py::compute` | `EconomicsInputs→Outputs` |
| S18 | Validation records | `web/api.py` → `store/validation.py` | fields |
| S19 | Guidance context | `guidance/engine.py` → store read functions (funnel/scoring/validation reads) | focus → context |
| S20 | Guidance LLM | `guidance/engine.py` → `llm.py::chat_completion` | messages |
| S21 | Proposal write | `guidance/writes.py::apply_confirmed` → store write functions | payload per §8.9 |
| S21b | Direct write | `guidance/writes.py::apply_direct` → store write functions | payload |
| S22 | Audit insert | `guidance/writes.py` → `store/guidance.py::audit_write` | action/target/before/after |
| S23 | Digest build | `guidance/digest.py::build(session)` → `store/guidance.py::create_digest` | digest fields |
| S24 | Retention prune | `scheduler.py` (post-sweep hook) → `retention.py::prune` | — |
| S24b | Pin check | `retention.py` → `store/items.py::list_prune_candidates` | ids |
| S25 | Scheduler → orchestrator | `scheduler.py::fire` → `orchestrator.py::run_sweep(trigger)` | trigger kind |
| S26 | Sweep lock | `scheduler.py`/`web/api.py` → `orchestrator.py::try_lock` | — |
| S27 | Dossier export | `web/api.py` → `export/dossier.py::render_markdown/html` | candidate id |
| S28 | Client call | every widget → `vorbote_client/api.py::get/post/put` | route+body |
| S29 | Tunnel spawn | `app.py` → `tunnel.py::ensure_tunnel` | settings |
| S30 | Test seam | `tests/conftest.py` → `db.py::build_container` (with fixture LLM/Exa/reach) | overrides |
| S31 | research-planner queries | `guidance/engine.py` → `ladder.py::Ladder.fetch` (S1) via server | query |
| S31b | planner results appended | `ladder.py` → (return) `guidance/engine.py` context | results |
| S32 | status/ledgers read | `web/api.py` → `metering.py::get_month_totals` | month |
| S32b | quarantine set | `web/api.py`/`orchestrator.py` → `store/sources.py::set_source_health` | reason |

**Call traces (primary runtime paths, entry → calls → committed state):**

**T1 — Scheduled daily sweep (happy path).**
`scheduler.fire(daily@05:00)` → [S25] `orchestrator.run_sweep('scheduled')` → lock [S26] → `store/runs.create_sweep_run` (running) → per source: adapter.fetch(source, plan) [S1] → `politeness.wait_for_slot` [S2] → ladder step 1 (httpx via Ladder) → on fail: `ExaClient.fetch` [S3] → metered [S4] → adapter returns `FetchResult` → `normalizer.normalize` → `snapshots.write` [S7b] → `upsert_raw_item` [S7] → `changedetect.detect` [S8] → `create_change_event` [S8b] → source loop next → post-sweep: `extract.run` [S9] → `llm.chat_completion` [S20-class] → metered [S11] → `create_observation` [S9b] → `hypothesize.run` [S10] → observations + evidence [S10b] → `finish_sweep_run(completed, stats)` → retention hook [S24] → `retention.prune` [S24b] → committed: sweep_run completed; raw_items, change_events, observations (triage) rows; snapshots on disk; llm/exa usage rows.
*State at process boundaries: SQLite WAL (all rows) + snapshot files persist across restarts; sweep_run 'running' rows swept to 'interrupted' at boot (§8.16).*

**T2 — Manual sweep during active sweep.** `web/api POST /sweeps/trigger` → [S26] `try_lock` fails → `409 SWEEP_IN_PROGRESS` envelope → no state change.

**T3 — Triage promote.** `web/api POST /observations/<id>/promote` → [S12] read observation → [S13] `create_hypothesis(draft)` → audit via write path (operator actor) [S22-class] → committed: hypotheses row; observation status promoted.

**T3b — Negative finding.** `POST /observations/<id>/negative` → [S12] status→negative_finding + rationale addendum → committed.

**T4 — Cookie upload (S-13/S-14 login state).** `widgets/sources.py` upload → [S6c] tunnel POST → [S6] `ReachBridge.inject_cookies` → profile write (0600) → [S6b] `mark_credential_configured(browser_cookies)` → response `{ok:true}` (no echo) → committed: credentials_status row; profile file; audit fact row (no values).

**T5 — Guidance session with confirmed score write.** `POST /guidance/sessions` (focus candidate X) → [S19] context assembly (store reads) → operator message → [S20] LLM call → metered [S11] → reply + proposals (`score.write` payload) persisted `proposed` → operator confirms → [S21] `apply_confirmed` → validate payload (§8.9) → `evaluation/scoring.compute_score` [S15] → `store/scoring.create_score` → [S22] `audit_write` → committed: score row + audit row; proposal state confirmed. `close_session` → [S23] digest → committed digest row.

**T6 — Economics compute.** `POST /candidates/<id>/economics` → [S17] `compute(EconomicsInputs)` → validate (no missing inputs) → `store/scoring.create_economics_record` → committed outputs row.

**T7 — Reach fetch in sweep.** `sources/reach.py` fetch → [S5] `ReachBridge.call_channel('reddit','search',{query})` → JSON contract → items → normalizer → [S7] upsert → [S8] new_item events → committed rows; `NEED_LOGIN` → quarantine [S32b] → committed source health.

**T7b — Reach login stale → wizard.** quarantine noticed in widgets/sources → upload flow T4 → release quarantine [S32b] → next window.

**T8 — Exa fallback + degradation.** ladder step-1 exhausts retries → [S3] `ExaClient.fetch` → no key → `exa_disabled` → fetch_attempts row (outcome) → source_run `skipped_exa_needed` (if exa_required) → committed rows; deterministic-only sources continue.

**T8b — Politeness violation attempt.** Two concurrent ladder calls same source → [S2] single-flight lock + delay → serialized; fetch_attempts show `delay_applied_ms` > 0.

**T9 — Retention prune.** post-sweep hook [S24] → `retention.prune` → [S24b] `list_prune_candidates` (SQL: unreferenced ∧ unpinned ∧ aged) → FIFO delete (file+row) → warning stats → committed; referenced/pinned untouched.

**T10 — Dossier export.** `GET /candidates/<id>/export?format=html` → [S27] `export/dossier.render` → store reads → HTML body → client save.

**T11 — Catch-up sweep.** serve boot → scheduler computes missed windows (bounded 1/window) → [S25] `run_sweep('catchup')` → T1 path.

**T12 — LLM contract violation.** `extract.run` → LLM mock returns invalid JSON → validation fails → event stays `extracted=0` (no observation) → run stats flag → next sweep retries; on persistent violation at hypothesize → degrade-to-raw observation (§8.13.1) → committed raw-tier observation.

**T13 — Scheduler → retention at month boundary.** tick → ledger rollup [S32] → ceiling_states new month row → warned reset → committed.

**T14 — research-planner query.** guidance session → planner proposes query → [S31] `Ladder.fetch` (steps 1–2) → results appended to session context [S31b] → next turn context includes results → committed usage rows; no direct LLM fetch.

**T15 — URL drift.** source fails transport-level after success history → quarantine `url_stale` [S32b] → wizard: edit endpoint_config via `PUT /sources/<id>` → test fetch [S6c-class route] → release → next window.

## 16. Final Consistency Checklist

- Every Doc 1 FR/NFR/C mapped: FR-01…FR-35 → §8 (modules), §5 (DDL), §6 (API); NFR-01…NFR-12 → §7/§10/§11/§12; C-01…C-09 → §4 (ceilings/scope), §8.6/8.13 (no invented numbers), §5 (one DB), §12 (tests), §1 (self-contained).
- One model all roles (C-03): `llm.py` single path; model string config; reasoning param conditional [§8.13].
- Cookie secrets discipline: §4.1, §6.2, §7.2, §8.10, §12 guard `test_secrets`, backup exclusion §13.
- Exa degradation path complete: §8.5, T8; source rows carry `exa_required` flag in endpoint_config.
- Bounded writes closed set: §8.9; unknown actions
