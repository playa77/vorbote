# Vorbote — Document 3: Implementation Roadmap

| | |
|---|---|
| **Title** | Vorbote — Implementation Roadmap |
| **Version** | 1.0.0 |
| **Date** | 2026-10-26 |
| **Status** | Approved |
| **Baseline** | Doc 1 v2.0.0 (Approved) + Doc 2 v1.0.0 (Approved) — cross-references cite them as [Doc 1 §n], [Doc 2 §n] |
| **Consumption model** | A zero-context coding agent executes one work package (WP) per session, strictly in dependency order (§1.3) |

**Revision History**

| Version | Date | Status | Changes |
|---|---|---|---|
| 1.0.0 | 2026-10-26 | Approved | Initial issue from approved Doc 1 v2.0.0 + Doc 2 v1.0.0 |

---

## 1. Roadmap Overview

### 1.1 Implementation strategy

Vorbote is implemented **bottom-up along Doc 2's layering** ([Doc 1 §3.2]: dependencies point downward only; CLI/API the only entry points; one composition root). Phases follow the layer order exactly:

| Phase | Name | Builds | Ends at |
|---|---|---|---|
| P0 | Foundation | repo skeleton, shared types, persistence layer, config, init/seeds | **M1** |
| P1 | Source platform | ladder + politeness, adapter registry, Exa, all 6 adapter modules, reach bridge | — |
| P2 | Processing pipeline & post-sweep | change detector, LLM client + metering, extract/hypothesize stages, scheduler + orchestrator, retention | **M2** |
| P3 | Evaluation engine | funnel managers, scoring, economics, constraints, validation | — |
| P4 | Guidance engine | guidance engine, bounded writes, digests | — |
| P5 | API surface & ops | loopback API (all routes), dossier export, CLI completion, deploy artifacts | **M3** |
| P6 | Desktop client | shell/tunnel, all 8 widgets, i18n, export download | **M4** |
| P7 | Acceptance hardening | guard completion, coverage closure, §14 acceptance run | **M5** |

Rationale: every WP consumes only contracts that already exist (from prior WPs or Doc 2 §5/§8); no WP requires a forward reference. Verification-first (§2.4) makes each WP self-proving.

### 1.2 What a WP is

A WP is the unit of work one agent completes in **one session/context window**: it names exact files, ordered behavioral tasks, the contracts it consumes/produces, objective acceptance criteria, and the exact verification commands. WPs never contain implementation code — behavior is defined by the cited Doc 1/Doc 2 sections; this roadmap supplies only order, scope boundaries, and criteria.

### 1.3 Consumption model (binding on the executing agent)

1. **One WP per session.** A session opens a WP, completes its task list in order, verifies, commits, closes. Never carry two WPs in one session.
2. **Strict dependency order.** A WP may start only when every WP in its *Depends-on* list is merged (§3). Parallelizable WPs (marked ∥) may be executed in either order but never merged out of order.
3. **Verification-first.** At session start, run the WP's §Verification commands and observe the exact failures; implement until they pass; then run the whole suite (§2.5).
4. **Spec supremacy.** If implementation reveals a defect in Doc 1/Doc 2 (ambiguity, omission, contradiction), the agent does **not** improvise: it records the finding in the session log and resolves it by the narrowest reading of the governing document; if unresolvable, it stops that WP's affected task and flags it for operator decision.
5. **No scope creep.** A WP implements exactly its Scope list. Adjacent behavior belongs to another WP (see §6 matrix).

### 1.4 Shape of the delivered system

35 work packages (WP-00…WP-34), 8 phases, 5 integration checkpoints (M1–M5). Final state = Doc 2 §2 tree fully populated, hermetic suite green, guards green, §14's 15 acceptance criteria demonstrated.

---

## 2. Global Conventions

### 2.1 Repo bootstrap order (WP-00, once)

1. `git init` on `main`; create the full Doc 2 §2 tree (both `pyproject.toml` with §3 pinned deps; empty `__init__.py` files; `config/settings.example.toml` verbatim from Doc 2 §4).
2. Every module file that a later WP owns starts as a **placeholder**: a module that raises `NotImplementedError` on first use and exposes a module constant `PLACEHOLDER = "WP-xx"` naming its owning WP.
3. `tests/` skeleton: `tests/conftest.py`, `tests/unit/`, `tests/integration/`, `tests/guard/`, `tests/client/`; one guard file `tests/guard/test_placeholders.py` holding the **placeholder ledger** (§2.6).
4. Initial commit on `main`.

### 2.2 Branch/commit conventions

- Branch per WP: `chatbox/wp-<id>-<slug>` (e.g. `chatbox/wp-05-ladder-politeness`), cut from `main` after the dependency WPs are merged.
- Commit messages: `WP-<id>: <imperative summary>`; commits that include code changes carry the trailer `Co-authored-by: Chatbox <chatbox@chatboxai.com>`.
- Merge to `main` only when the WP's Definition of Done (§2.3) holds; `main` must be green (full suite) at every merge.
- Merge commits reference the WP id; no merge without its verification evidence in the session log.

### 2.3 Definition of Done (applies to every WP)

1. All tasks in the WP's ordered task list are complete; every Scope file exists; no `PLACEHOLDER` constant remains for any module in this WP's scope.
2. The WP's Verification commands pass with the expected results; the **entire** `pytest tests/` is green (no regression in any other area).
3. Any guard test named in [Doc 2 §12 level 4] that this WP owns is created/extended and green.
4. The test-construction rule holds: no test hand-builds state the code under test should produce; pre-existing state is established via the production path [Doc 2 §12].
5. Placeholder ledger updated in the same commit (§2.6).
6. Session log records: WP id, tasks done, verification output summary, any spec findings.

### 2.4 Verification-first

Each WP's Verification section is executed **before** implementation (expected: failing/missing) and **after** (expected: passing). A WP is never declared done from task-completion alone — only from green verification.

### 2.5 Test layout & commands (uniform across WPs)

```
pytest tests/ -q                      # full gate (every WP, every merge)
pytest tests/unit -q                  # area unit tests
pytest tests/integration -q
pytest tests/guard -q
pytest tests/client -q                # QT_QPA_PLATFORM=offscreen [Doc 2 §12 L3]
```

Client tests always run with `QT_QPA_PLATFORM=offscreen`. No test ever touches live network [NFR-11]; all externals are conftest fixture servers [Doc 2 §12].

### 2.6 Placeholder ledger (progress-tracing mechanism)

`tests/guard/test_placeholders.py` holds `EXPECTED_PLACEHOLDERS = <count>` and asserts: (a) the set of modules still carrying `PLACEHOLDER` exactly equals the set whose owning WP is unmerged; (b) the count equals `EXPECTED_PLACEHOLDERS`. Every WP that lands a module decrements the count in the same commit. **Invariant: 0 remaining ⇔ WP-34 complete.** This makes "silent scope skip" impossible — an unimplemented module fails the suite.

### 2.7 Versioning of delivered artifacts

- Documents: semantic versions in each doc's header; this roadmap is v1.0.0 and changes only via revision-history entry.
- Code: git history is the version record; **tags `v0.1.0`…`v0.5.0`** at M1…M5 respectively. No other versioning scheme exists (no Docker, no build step [Doc 1 §8]).
- Migrations: forward-only numbered SQL, recorded in `schema_migrations` [Doc 2 §5.1].

### 2.8 Conventions inherited verbatim from Doc 2

Error envelope, UTC ISO-8601 datetimes, money conventions, module prefixes (`vorbote/`, `vorbote_client/`), type syntax [Doc 2 §1.3]; single composition root `db.py::build_container()` with the §15 construction order — **no other module instantiates infrastructure**.

---

## 3. Dependency Graph

Full index. ∥ = parallelizable with its phase peers (order free, dependencies respected). Direct depends-on lists (transitive deps implied). The graph is **acyclic by construction**: every listed dependency has a strictly smaller WP number; WP-00 is the unique root; WP-34 the unique sink.

| WP | Title | Phase | Depends-on | ∥ |
|---|---|---|---|---|
| WP-00 | Repo bootstrap & placeholder ledger | P0 | — | — |
| WP-01 | `schema.py` — shared typed catalog | P0 | WP-00 | — |
| WP-02 | `db.py` + `store/*` — persistence layer & migrations | P0 | WP-01 | — |
| WP-03 | `config.py` — TOML schema, defaults, secrets loader | P0 | WP-00, WP-01 | — |
| WP-04 | `cli.py init` + `sources_seed.json` (S-01…S-20) | P0 | WP-02, WP-03 | — |
| WP-05 | `ladder.py` + `politeness.py` | P1 | WP-02, WP-03 | — |
| WP-06 | `sources/base.py` + `exa_client.py` + AdapterRegistry + composition root | P1 | WP-01, WP-02, WP-03, WP-05 | — |
| WP-07 | `sources/feeds.py` (feed, change_feed, trends) | P1 | WP-06 | ∥ (with 08–11) |
| WP-08 | `sources/api.py` (rest_api family) | P1 | WP-06 | ∥ |
| WP-09 | `sources/listing.py` (html_diff + marketplace) | P1 | WP-06 | ∥ |
| WP-10 | `sources/datasets.py` (dataset family) | P1 | WP-06 | ∥ |
| WP-11 | `normalizer.py` + `snapshots.py` | P1 | WP-01, WP-02 | ∥ |
| WP-24 | `reach_bridge.py` + `sources/reach.py` (community) | P1 | WP-06 | ∥ (with 07–11) |
| WP-12 | `changedetect.py` — deterministic change detection | P2 | WP-01, WP-02 | ∥ (with 13) |
| WP-13 | `llm.py` + `metering.py` | P2 | WP-02, WP-03 | ∥ (with 12) |
| WP-14 | `extract.py` + `hypothesize.py` + extractor/hypothesis prompts | P2 | WP-12, WP-13 | — |
| WP-15 | `scheduler.py` + `orchestrator.py` | P2 | WP-07…WP-12, WP-14, WP-24 | — |
| WP-25 | `retention.py` | P2 | WP-02, WP-15 | — |
| WP-16 | Evaluation: observation / hypothesis / candidate managers | P3 | WP-02, WP-14 | — |
| WP-17 | Evaluation: scoring + economics + constraints (+ weights data) | P3 | WP-02, WP-16 | — |
| WP-18 | Evaluation: validation manager | P3 | WP-02, WP-16 | ∥ (with 17? no — after 16; ∥ with nothing) |
| WP-21 | Guidance engine + role prompts + context assembly | P4 | WP-02, WP-05, WP-13, WP-16 | ∥ (with P3 tail: independent of 17/18) |
| WP-22 | Guidance bounded writes + session digest | P4 | WP-16, WP-17, WP-21 | — |
| WP-19 | Web app factory + gates + System + Sources routes + cookie endpoint | P5 | WP-02, WP-03, WP-06, WP-15, WP-24 | — |
| WP-20 | Web routes: sweeps/runs, observations, candidates, scoring/economics/constraints, validation, evidence/pins, ledgers | P5 | WP-02, WP-11, WP-13, WP-15, WP-16, WP-17, WP-18, WP-19 | — |
| WP-23 | Web routes: guidance (sessions/messages/proposals/digest) | P5 | WP-19, WP-22 | ∥ (with 27, after 20) |
| WP-27 | `export/dossier.py` + candidate dossier/export routes | P5 | WP-17, WP-18, WP-20 | ∥ (with 23) |
| WP-26 | CLI completion (`run`/`serve`/`help`) + deploy artifacts + deployment guide | P5 | WP-15, WP-20, WP-23, WP-24, WP-25, WP-27 | — |
| WP-28 | Client shell: app, tunnel, api, i18n, workers | P6 | WP-19, WP-20 | — |
| WP-29 | Client widgets: triage + dossier (+ `export_download.py`) | P6 | WP-20, WP-28 | ∥ (with 30–32) |
| WP-30 | Client widgets: scoring/economics + constraints editor | P6 | WP-28 | ∥ |
| WP-31 | Client widgets: validation | P6 | WP-28 | ∥ |
| WP-32 | Client widgets: source manager + cookie-upload wizard | P6 | WP-19, WP-24, WP-28 | ∥ |
| WP-33 | Client widgets: guidance chat + settings/ledgers | P6 | WP-20, WP-23, WP-27, WP-28 | — (after ∥ group) |
| WP-34 | Acceptance hardening: guards completion, coverage closure, §14 acceptance run | P7 | WP-26, WP-29, WP-30, WP-31, WP-32, WP-33 | — |

*(Correction to the ∥ mark on WP-18: WP-18 depends on WP-16 and is sequential within P3 after WP-16; it is parallelizable with WP-21.)*

Acyclicity: every edge points from a higher number to a lower number in the Depends-on column; no cycles are possible. Sink order: WP-34 last; WP-26 is the server-side tail of P5.

---

## 4. Work Packages

Format per WP: header fields → Objective/Rationale → Scope (files; out-of-scope) → Spec refs → Tasks (ordered, behavioral) → Contracts (consumed/produced) → Acceptance criteria → Verification → Sizing.

---

### WP-00 — Repo bootstrap & placeholder ledger

- **Phase/Depends-on:** P0 / —
- **Objective:** Establish the exact Doc 2 §2 repository skeleton with placeholder modules and a progress-tracing guard, so every later WP has a fixed home and silent scope-skips are impossible.
- **Rationale:** The layer contract and file map are normative [Doc 2 §2]; freezing them first removes all structural decisions from later sessions.
- **Scope (create):** `pyproject.toml` (server, §3.1 pins: httpx ~=0.28, feedparser ~=6.0, trafilatura ~=2.0, pypdf ~=6.0, selectolax ~=0.3, flask ~=3.1, python-dateutil ~=2.9; extras `dev` = pytest ~=8.3; **no pillow**); `client/pyproject.toml` (§3.2: PySide6 ~=6.8, httpx ~=0.28); full `src/vorbote/**` and `src/vorbote_client/**` tree per §2 as placeholder modules (`PLACEHOLDER = "WP-xx"`); `config/settings.example.toml` verbatim from Doc 2 §4; `deploy/`, `docs/` stubs; `tests/` skeleton + `tests/guard/test_placeholders.py`; `.gitignore`; `README.md` stub. **Out of scope:** any behavior, any test beyond the placeholder guard.
- **Spec refs:** [Doc 2 §2, §3, §4, §12].
- **Tasks:**
  1. `git init` on `main`; create the tree exactly as §2 prints it (including the `sources/datasets.py` vs `listing.py` family split comment).
  2. Write both `pyproject.toml` files with the pinned dependency sets; no other dependencies anywhere.
  3. Emit every §2 module file as a placeholder with its owning WP id constant (owning WP = the one whose Scope lists it below).
  4. Copy `settings.example.toml` from Doc 2 §4 verbatim (comments included).
  5. Create `tests/guard/test_placeholders.py` with `EXPECTED_PLACEHOLDERS` set to the full count; assert set equality of remaining placeholders vs. unmerged owning WPs.
  6. Install both distributions editable; run the suite; commit.
- **Contracts:** produced — the module-path namespace every later WP fills; the placeholder-ledger guard contract.
- **Acceptance:** (1) `pip install -e . && pip install -e ./client` succeeds on Python 3.12+; (2) tree matches §2 file-for-file; (3) `pytest tests/ -q` green with only the placeholder guard present; (4) `git log` shows one bootstrap commit.
- **Verification:** `python -m pytest tests/guard/test_placeholders.py -q` → 1 passed; `git status --porcelain` empty.
- **Sizing:** S — well under one session.

---

### WP-01 — `schema.py` shared typed catalog

- **Phase/Depends-on:** P0 / WP-00
- **Objective:** Implement every typed dataclass and JSON-blob shape named anywhere in Doc 2 as the single source of truth, so all later modules import types from one place.
- **Rationale:** [Doc 2 §8 preamble]: "Types live in `schema.py` (single source of truth)"; pinning all shapes first makes every store/adapter contract compile-time-checkable.
- **Scope (create):** `src/vorbote/schema.py` in full. **Out of scope:** any logic beyond (de)serialization + validation; DB code.
- **Spec refs:** [Doc 2 §8.0, §8.6, §8.9, §8.13, §5.2 blob columns, §5.3].
- **Tasks:**
  1. Implement the §8.0 excerpt catalog completely: `CanonicalItem`, `EndpointConfig` (family-tagged per §8.1–§8.10 endpoint schemas), `ErrorInfo`, `EvidenceRef`, `EconomicsInputs`, `EconomicsOutputs`, `ProposalPayload` variants (§8.9 closed set), `DigestSpec`.
  2. Add every type named in §5.3/§8 signatures: `SourceRow`, `FetchPlan`, `FetchResult`, `LadderRequest`, `LadderResponse`, `ExaFetchResult`/`ExaSearchResult`, `ReachResult`, `LLMResult`, `ChangeEventDraft`, `ObservationDraft`, `ConstraintsRow`, `ScoreCriteria`, `WeightsRow`, `InvestigationPlan`/`InvestigationFinding`, `InterviewGuide`, `ExperimentRecord`, `GuidanceMessage`, `SessionDigest`, `WriteAuditRecord`, `LLMUsageRow`, `ExaUsageRow`, `CeilingState`, `PinRecord`, `SecretsSpec` (presence/freshness booleans only — never values).
  3. Implement `to_json`/`from_json` for every blob type; blobs carry an internal `schema_version` [Doc 1 §5]; unknown version rejected on read.
  4. Datetime helpers: normalize to UTC ISO-8601 `…Z` strings [Doc 2 §1.3]; money conventions per §1.3.
  5. Validation: closed enum sets everywhere (parse_tier, families, statuses, action strings); malformed input → typed validation error.
- **Contracts:** consumed — none (leaf). Produced — the entire type namespace used by every later module.
- **Acceptance:** (1) round-trip serialize/deserialize equality for every type; (2) malformed-input rejection for every enum field; (3) every type named in §5.3 and §8 exists and is importable.
- **Verification:** `pytest tests/unit/test_schema.py -q` → all green (author this file: one round-trip + one rejection case per type).
- **Sizing:** M.

---

### WP-02 — `db.py` + `store/*` persistence layer

- **Phase/Depends-on:** P0 / WP-01
- **Objective:** Implement the one SQLite database: connection discipline, forward-only migrations with the complete §5.2 DDL, and every §5.3 store function.
- **Rationale:** C-07 (exactly one DB) and [Doc 2 §5] make this the foundation every component consumes; building it before any behavior means no later WP touches SQL directly.
- **Scope (create):** `src/vorbote/db.py` (container skeleton — full completion in WP-06), `src/vorbote/store/__init__.py`, all 10 store modules per §2 (`sources.py`, `runs.py`, `items.py`, `changes.py`, `funnel.py`, `scoring.py`, `validation.py`, `guidance.py`, `metering.py`, `migrations.py`). **Out of scope:** seeds content (WP-04), composition-root wiring of infra clients (WP-06).
- **Spec refs:** [Doc 2 §5.1, §5.2, §5.3, §8 (store function behaviors), §11 DB-busy row].
- **Tasks:**
  1. Connection discipline per §5.1: `$VORBOTE_HOME/vorbote.db`, WAL, `foreign_keys=ON`, `check_same_thread=False` + internal write mutex; all writes through store modules only.
  2. Migrations engine (`store/migrations.py`): numbered SQL files embedded; `schema_migrations` apply/record; forward-only; re-apply no-op.
  3. Migration 001 = the complete §5.2 DDL **verbatim** (all tables incl. `scoring_weights`, `ceiling_states`, `write_audit`, `pins`, `schema_migrations`).
  4. Implement every §5.3 function with typed signatures (one line each in §5.3 — behavior per §8 module specs): sources (incl. `mark_credential_configured` for seam S6b), runs, items (upsert/dedupe on `UNIQUE(source, external_key)`; prune-candidate listing), changes (unextracted queue + mark-extracted), funnel, scoring, validation, guidance, metering.
  5. DB-busy handling: retry 3× (100 ms, 200 ms, then raise) → error code `DB_BUSY` [Doc 2 §11].
  6. JSON-blob version check on write (reject unknown `schema_version`).
- **Contracts:** consumed — `schema.py` types (WP-01). Produced — the §5.3 function surface (every later module's persistence contract); migration 001.
- **Acceptance:** (1) migrations apply on a temp `VORBOTE_HOME`; re-run no-op; (2) every §5.3 function round-trips on a temp DB; (3) busy-retry path exercised with an injected lock; (4) dedupe upsert honored.
- **Verification:** `pytest tests/unit/test_store*.py -q` → green (author: per-module store tests).
- **Sizing:** L — full session; if the store module count strains context, the session may stop after `guidance.py` and finish `metering.py`+tests in the same WP's follow-up session (same WP id, recorded in session log).

---

### WP-03 — `config.py` configuration

- **Phase/Depends-on:** P0 / WP-00, WP-01
- **Objective:** Implement TOML settings loading/schema/defaults and the secrets-file presence loader with mutation lock, exactly per Doc 2 §4.
- **Rationale:** Every operational parameter (politeness, ceilings, retention, reach, guidance caps) is config [Doc 1 §1.3 goal 6]; it must exist before the components that consume it.
- **Scope (create):** `src/vorbote/config.py`. **Out of scope:** seed data (WP-04), any endpoint writing settings (mutation is file-level, §4.3).
- **Spec refs:** [Doc 2 §4, §4.1, §4.3, §4.4 (client keys documented only)].
- **Tasks:**
  1. Load `config/settings.toml` over defaults for every §4 section: server, sweep, politeness (table incl. per-source overrides), llm (+ceiling), exa (+ceiling), retention, reach, guidance.
  2. Secrets loader per §4.1: read the `.env`-style file; expose **presence/freshness booleans only** per key (values never leave the secrets module; NFR-05).
  3. `VORBOTE_HOME` resolution: required for state-touching processes; missing → startup error.
  4. Validation with precise messages; unknown keys → warning.
  5. Mutation discipline per §4.3: tmp-file + `os.replace` under a mutation lock; settings re-read per scheduler tick.
- **Contracts:** consumed — `SecretsSpec` (WP-01). Produced — a typed `Settings` object + secrets-presence API used by all later modules.
- **Acceptance:** (1) defaults load with no file; (2) every §4 override parses; (3) secrets presence matrix (blank `.env` → all-false, deterministic-only posture); (4) missing `VORBOTE_HOME` → error; (5) atomic write under concurrent reader.
- **Verification:** `pytest tests/unit/test_config.py -q` → green.
- **Sizing:** M.

---

### WP-04 — `cli.py init` + Initial Source Register seeds

- **Phase/Depends-on:** P0 / WP-02, WP-03
- **Objective:** Author the 20-source seed file from Doc 1 §12 and implement `vorbote init` so a clean host becomes a provisioned system in one command.
- **Rationale:** FR-35 makes exact provisioning a committed deliverable; seeds are **data**, not config [Doc 2 §4.2].
- **Scope (create):** `config/sources_seed.json` (20 entries + groups), `vorbote init` in `src/vorbote/cli.py` (serve/run/help remain placeholders — WP-26). **Out of scope:** any sweep behavior; endpoint fetch logic.
- **Spec refs:** [Doc 1 §12 (all rows + notes), Doc 2 §4.2, §8.19 (init), §13 (init sequence)].
- **Tasks:**
  1. Author `sources_seed.json`: one entry per S-01…S-20 with `name` slug, family, `endpoint_config` JSON per the family's §8 schema (TED POST template incl. body; DIP entity set + api-key auth; DESTATIS table + GAST guest + redirect-follow note; Eurostat dataset codes + `geo=DE`; GovData CKAN query; feeds' URLs; BAnz/insolvenz portal params; Amazon category paths + `exa_required` posture + top-N; policy-pages empty curated list; FDB form query + Exa-assist; HN query; Trends `geo=DE`; reach platform reddit/twitter), language, jurisdiction, `requires_credential`, `enabled` (false **only** for S-18/S-19), group.
  2. Slug names must equal the politeness override keys (`insolvenzbekanntmachungen`, `amazon-bestellers`, `bundesanzeiger-at`) — normative linkage [Doc 2 §8.4 note].
  3. Seed default groups with `schedule_spec` values per §5.2 format (`daily@HH:MM`, `weekly:DOW@HH:MM`).
  4. Implement `vorbote init` per §8.19/§13: create state dir; apply migrations; insert seeds (no-op on existing sources by `name`); write settings skeleton + `.env` skeleton (blank keys → deterministic-only out of the box); copy deploy files.
- **Contracts:** consumed — store loaders (WP-02), config (WP-03). Produced — initialized state dir; seeded `sources`/`source_groups` rows (input fixture data for every pipeline test).
- **Acceptance:** (1) init on clean temp home → 20 source rows, S-18/S-19 `enabled=0`, groups present; (2) re-init → no duplicate rows; (3) blank `.env` present; (4) exit 0.
- **Verification:** `pytest tests/integration/test_init.py -q` → green (author).
- **Sizing:** M.

---

### WP-05 — `ladder.py` + `politeness.py`

- **Phase/Depends-on:** P1 / WP-02, WP-03
- **Objective:** Implement the retrieval-ladder executor and politeness governor: the single outbound HTTP path with per-source serialization, delays, backoff, Exa fallback, and full `fetch_attempts` accounting.
- **Rationale:** FR-09/D3: every outbound retrieval rides the ladder; adapters never use httpx directly [Doc 2 §8.1].
- **Scope (create):** `src/vorbote/ladder.py`, `src/vorbote/politeness.py`; guard files `tests/guard/test_ladder.py`, `tests/guard/test_politeness.py`. **Out of scope:** adapter fetchers (family WPs), Exa client itself (WP-06 — ladder calls it via the `FetchPlan` seam with a stub until then).
- **Spec refs:** [Doc 1 §3.4, Doc 2 §4 (politeness values), §8.4, §11 (fetch-failure rows), T8/T8b].
- **Tasks:**
  1. `PolitenessGovernor`: per-source asyncio single-flight lock; inter-request delay (default 4.0 s; per-source overrides 6.0 insolvenz / 10.0 amazon / 6.0 BAnz; class defaults per §4); `wait_for_slot(source)` the sole authority (seam S2).
  2. Backoff: `backoff_base_s × 2^retry`, honoring `Retry-After` capped at `retry_after_cap_s` (120 s), max 4 retries.
  3. `Ladder.fetch`: step 1 deterministic fetch via the family adapter's native fetcher (injected callable); on failure after bounded retries → step 2 Exa (via injected client interface); LLM never fetches directly (by construction — no LLM import).
  4. Every attempt logged as `fetch_attempts` (ladder_step 1/2, outcome incl. `ok_exa`, `exa_disabled`, `skipped_login`; http_status; retry_count; delay_applied_ms; exa_usage ref).
  5. No-key Exa → `exa_disabled` outcome; metering untouched [Doc 2 §8.5].
- **Contracts:** consumed — store/runs (WP-02), config (WP-03). Produced — `Ladder.fetch(source, LadderRequest) → LadderResponse` (seam S1); the politeness seam S2.
- **Acceptance:** (1) backoff schedule exact per §4 values; (2) two concurrent same-source calls serialize with `delay_applied_ms > 0` (T8b); (3) Retry-After honored and capped; (4) retry budget 4 then step 2; (5) `fetch_attempts` rows complete for the T8 trace.
- **Verification:** `pytest tests/unit/test_politeness.py tests/unit/test_ladder.py tests/guard/test_ladder.py tests/guard/test_politeness.py -q` → green.
- **Sizing:** M.

---

### WP-06 — `sources/base.py` + `exa_client.py` + registry + composition root

- **Phase/Depends-on:** P1 / WP-01, WP-02, WP-03, WP-05
- **Objective:** Define the adapter contract, implement the Exa client, and complete the composition root that constructs all infrastructure in the §15 order.
- **Rationale:** [Doc 2 §15]: one composition root, no other module instantiates infrastructure; the adapter contract is the seam every family module implements.
- **Scope (create):** `src/vorbote/sources/base.py`, `src/vorbote/sources/__init__.py` (AdapterRegistry), `src/vorbote/exa_client.py`; complete `db.py::build_container()`. **Out of scope:** any concrete family adapter.
- **Spec refs:** [Doc 2 §8.1, §8.5, §8.3.1 (family mapping), §15 (construction order), §2 (registry split)].
- **Tasks:**
  1. `base.py`: `Adapter` protocol (`family`, `fetch(source, plan) → FetchResult`); `FetchPlan{ladder, politeness, exa|None}`; adapters never import httpx.
  2. Registry: map families (incl. trends→feeds, change_feed→feeds, marketplace→listing, community→reach) to adapter instances [Doc 2 §2 comment, §8.3.1].
  3. `ExaClient` per §8.5: constructed with key or `None`; `fetch`/`search`; `ExaDisabledError` when keyless; per-call metering hook; ceiling check after each call (warn-only).
  4. `build_container()` per §15 order: SQLite + migrations → PolitenessGovernor → LLMClient (placeholder until WP-13; constructed as interface) → ExaClient → ReachBridge (placeholder until WP-24) → Ladder → AdapterRegistry → Orchestrator/Scheduler (placeholders until WP-15) → GuidanceEngine (until WP-21) → RetentionPruner (until WP-25) → Metering (until WP-13) → Flask app (until WP-19). Placeholders here are interface stubs owned by their WPs; ledger counts them.
- **Contracts:** consumed — ladder (WP-05), store, config, schema. Produced — `Adapter`/`FetchPlan`/`FetchResult` contract; `ExaClient`; wired container (seams S1–S4 partially live).
- **Acceptance:** (1) registry maps all 8 families (incl. aliases); (2) keyless Exa → `ExaDisabledError`, no metering row; (3) container constructs on a temp home with blank secrets; (4) no module outside `db.py` instantiates infra (grep-verifiable — author a test that imports every module and asserts no direct httpx client construction outside the root).
- **Verification:** `pytest tests/unit/test_registry.py tests/unit/test_exa_client.py tests/unit/test_container.py -q` → green.
- **Sizing:** M.

---

### WP-07 — `sources/feeds.py` (feed, change_feed, trends)

- **Phase/Depends-on:** P1 / WP-06 · ∥ with 08–11
- **Objective:** Implement the feed adapter family: RSS/Atom parsing, dedupe by external_key, amendment semantics for change feeds, and Trends approx-traffic parsing.
- **Rationale:** Three of the eight families (feed, change_feed, trends) are served by this module [Doc 2 §2, §8.2].
- **Scope (create):** `src/vorbote/sources/feeds.py`; conftest fixture feed server (RSS 2.0, Atom, trends samples). **Out of scope:** change detection itself (WP-12), snapshot writing of linked content (ladder handles; store persists).
- **Spec refs:** [Doc 2 §8.2, §8.12 (feed rows), Doc 1 §12 S-01/S-09/S-10/S-11/S-12].
- **Tasks:**
  1. Parse RSS/Atom via feedparser through the ladder; `external_key` = guid with `<link>` fallback.
  2. Dedupe upsert semantics: same key + changed content hash ⇒ `value_delta` **draft** handed to the detector (WP-12 consumes; adapter emits draft in FetchResult metadata).
  3. Trends variant (S-12): parse `<ht:approx_traffic>` into `payload.traffic_hint`; language `de` for `geo=DE`.
  4. Change-feed variant (S-01): amendment metadata parsed from title/summary into payload.
  5. Politeness class default 4 s; no Exa-assist posture.
- **Contracts:** consumed — `Adapter`/`FetchPlan` (WP-06), ladder. Produced — `CanonicalItem`s for feed/change_feed/trends families; fixture feed server for all later integration tests.
- **Acceptance:** (1) parse→CanonicalItem mapping (parse_tier summary/full_text) correct for all three fixture feed shapes; (2) external_key stable across refetch; (3) trends traffic_hint parsed; (4) amendment metadata captured.
- **Verification:** `pytest tests/unit/test_feeds.py -q` → green (fixture server, no live network).
- **Sizing:** M.

---

### WP-08 — `sources/api.py` (rest_api family)

- **Phase/Depends-on:** P1 / WP-06 · ∥
- **Objective:** Implement the REST-API adapter for exactly {S-02 TED, S-05 DIP, S-07 Eurostat, S-15 HN, S-18 Keyword Planner, S-19 SP-API}, including disabled-until-credential behavior.
- **Rationale:** §8.3.1 fixes the family membership normatively.
- **Scope (create):** `src/vorbote/sources/api.py`; fixture API server (JSON endpoints incl. POST-only TED, paged windows, auth-required variant). **Out of scope:** dataset family (WP-10), TED XML parsing into text (normalizer/snapshots WP-11; adapter requests via ladder).
- **Spec refs:** [Doc 2 §8.3, §8.3.1, Doc 1 §12 S-02/S-05/S-07/S-15/S-18/S-19].
- **Tasks:**
  1. Endpoint template engine per `EndpointConfig` (`url_template`, method, `body_template`, paging `none|offset|date-window`, auth `none|api_key_header|oauth2_client`).
  2. TED: POST expert query `PD>=YYYYMMDD`; GET must not be attempted (405 semantics); date-window paging; notice XML links fetched via ladder for later snapshotting.
  3. DIP: key from secrets in header; 6 s politeness; entity endpoints `vorgang`/`drucksache`/`plenarprotokoll`.
  4. Eurostat: JSON-stat `geo=DE`; series values handed to detector as value_delta drafts.
  5. HN: Algolia `search_by_date`, `tags=story`, language `en`.
  6. Keyword Planner / SP-API: disabled path — absent credentials ⇒ planned-skip, never error [Doc 2 §8.3, §4.2]; Keyword-Planner volumes stored verbatim as ranges (never averaged; C-08); SP-API values land as evidence only.
- **Contracts:** consumed — ladder, politeness, secrets-presence (WP-03). Produced — canonical items for the rest_api family; the endpoint-template engine reused by no one else (family-local).
- **Acceptance:** (1) each source class maps fixture responses to canonical items; (2) date-window paging iterates; (3) optional-credential skip works with blank secrets; (4) TED XML link fetch goes through the ladder (fetch_attempts row exists).
- **Verification:** `pytest tests/unit/test_api_sources.py -q` → green.
- **Sizing:** M.

---

### WP-09 — `sources/listing.py` (html_diff + marketplace)

- **Phase/Depends-on:** P1 / WP-06 · ∥
- **Objective:** Implement the listing engine for html_diff and marketplace families: cookie-jar stateful portals, structure-aware row extraction, Amazon rank capture, policy-page snapshots, FDB Exa-assisted form query.
- **Rationale:** §2 normative split: listing.py covers both families; register entries differ by detector kinds [Doc 2 §2, §8.12].
- **Scope (create):** `src/vorbote/sources/listing.py`; fixture listing server (stateful JSF-ish HTML, Amazon-style rows, plain policy pages, FDB form). **Out of scope:** diff computation (WP-12).
- **Spec refs:** [Doc 2 §8 (listing family), §4 politeness overrides, §8.12 html_diff/marketplace rows, Doc 1 §12 S-03/S-04/S-16/S-17/S-20, OQ-2 posture].
- **Tasks:**
  1. Cookie jar + stateful row-link following (Wicket/JSF: session cookie, follow listing structure) for BAnz AT and insolvenz portal; bounded per-day window for insolvenz (high volume).
  2. Structure-aware list extraction with selectolax; rows → listing_entry canonical items with external keys derived from row identity.
  3. Amazon: top-N rank extraction per category path; movers-and-shakers posture conservative (slow cadence, honest UA, `exa_required` config flag) per OQ-2.
  4. Policy pages: whole-page snapshot per run (page-level hash basis for WP-12's snapshot_diff); operator-curated URL list from source config.
  5. FDB: form-query fetch; Exa-assist posture declared in endpoint_config (`exa_required`).
  6. Politeness 6 s (BAnz/insolvenz), 10 s (amazon) via config overrides.
- **Contracts:** consumed — ladder, politeness, registry. Produced — canonical items + page snapshots for html_diff/marketplace families; rank data in payloads for the detector.
- **Acceptance:** (1) stateful portal fetch reuses cookie jar across fixture requests; (2) row-set extraction stable across two fixture versions; (3) Amazon ranks captured for top-N; (4) FDB with step-1 failure + `exa_required` routes to step 2 with mocked Exa (fetch_attempts `ok_exa`).
- **Verification:** `pytest tests/unit/test_listing.py -q` → green.
- **Sizing:** M.

---

### WP-10 — `sources/datasets.py` (dataset family)

- **Phase/Depends-on:** P1 / WP-06 · ∥
- **Objective:** Implement the dataset adapter: DESTATIS CSV tables (redirect-following), Eurostat snapshot tables, GovData CKAN new-dataset detection.
- **Rationale:** §8.3.1 fixes dataset family membership {S-06, S-08 (+ operator-added Eurostat snapshots)}.
- **Scope (create):** `src/vorbote/sources/datasets.py`; fixture dataset server (302/307 redirects, CSV, CKAN JSON). **Out of scope:** snapshot compare logic (WP-12).
- **Spec refs:** [Doc 2 §8.3.1, §8.12 dataset row, Doc 1 §12 S-06/S-07/S-08].
- **Tasks:**
  1. DESTATIS: `GAST`/`GAST` guest access; **follow 302/307 redirects** (fixture asserts client follows); table CSV → snapshot per run; table list from endpoint_config (curated codes).
  2. Eurostat snapshot tables: dataset code → JSON-stat snapshot (S-07 rest_api handles live series; dataset-family rows handle operator-added snapshot tables per §8.3.1).
  3. GovData: CKAN `package_search`; package id as external_key; new package ⇒ new_item draft.
  4. Table snapshots persisted via snapshot store; change drafts emitted with before/after pointers for WP-12.
- **Contracts:** consumed — ladder, politeness, snapshot store (WP-11 interface — adapter emits bytes, store persists; if WP-11 unmerged, store seam is its placeholder and this WP's tests inject the finished signature per Doc 2 §5.3). Produced — dataset-family canonical items + table snapshots.
- **Acceptance:** (1) redirects followed (fixture issues 307 then content); (2) CSV table snapshot written and hashed; (3) new CKAN package detected on second fixture state; (4) no fabricated values — raw table data only.
- **Verification:** `pytest tests/unit/test_datasets.py -q` → green.
- **Sizing:** S–M.
- *Note:* if WP-11 has not landed, this WP's tests stub the snapshot-store seam with the Doc 2 §5.3 signature (allowed: the contract exists in Doc 2; the seam's placeholder implements the signature raising until WP-11).

---

### WP-11 — `normalizer.py` + `snapshots.py`

- **Phase/Depends-on:** P1 / WP-01, WP-02 · ∥
- **Objective:** Implement language/jurisdiction/parse-tier normalization and the content-addressed snapshot store including the PDF text-layer pipeline with scanned-PDF flagging.
- **Rationale:** Canonical content discipline (FR-27 language preservation; AS-05 PDF policy) is central to evidence quality.
- **Scope (create):** `src/vorbote/normalizer.py`, `src/vorbote/snapshots.py`; PDF fixtures (text-layer + scanned). **Out of scope:** extraction consumption of snapshots (WP-14).
- **Spec refs:** [Doc 2 §8.11, §8.11.1, §5.1 snapshot layout, Doc 1 AS-05, FR-08].
- **Tasks:**
  1. Normalizer: small in-house language heuristic (title+snippet; no extra dependency); jurisdiction from source row; parse_tier mapping per family.
  2. Snapshots: content-addressed write — SHA-256 → `snapshots/ab/<hash>.<ext>`; DB row per §5.2; idempotent (same hash → same row).
  3. PDF pipeline: pypdf text extraction; **flagged when extracted text < 100 chars over ≥ 4 pages** → `pdf_flagged_no_text`; stored (evidence-grade), excluded from extractor input list.
  4. Full-text snapshots for listing/policy pages (page bytes → snapshot).
- **Contracts:** consumed — schema types, store/items. Produced — `normalize()` and snapshot-write/read functions used by orchestrator (T1) and dossier evidence browser.
- **Acceptance:** (1) language heuristic correct on DE/EN fixture pairs; (2) hash-addressed write idempotent; (3) text-layer PDF → `pdf_text` item; scanned → flagged tier, stored, excluded; (4) parse_tier mapping per family matches §8.12 inputs.
- **Verification:** `pytest tests/unit/test_normalizer.py tests/unit/test_snapshots.py -q` → green.
- **Sizing:** M.

---

### WP-24 — `reach_bridge.py` + `sources/reach.py` (community)

- **Phase/Depends-on:** P1 / WP-06 · ∥ with 07–11
- **Objective:** Implement the reach bridge (pinned in-repo Node adaptation driving headless Chromium; read-only channel actions; cookie injection; error mapping) and the community adapter over it.
- **Rationale:** D4/AS-04: Reddit/X go exclusively through the bridge; cookie-transfer login is the committed mechanism [Doc 1 §7, AS-04].
- **Scope (create):** `src/vorbote/reach_bridge.py`, `src/vorbote/sources/reach.py`, the pinned in-repo reach adaptation under `vendor/reach/` (path pinned by this roadmap; content = the read-only channel CLI adaptation), test stub `tests/fixtures/reach_cli_stub` emitting the channel JSON contract; guard `tests/guard/test_reach_bridge.py`. **Out of scope:** cookie-upload API route (WP-19), wizard UI (WP-32), any interactive login (forbidden by AS-04).
- **Spec refs:** [Doc 2 §8.10, §10.1, §11 reach rows, T7/T7b, Doc 1 §7 (cookie paragraph), AS-04, §12 S-13/S-14].
- **Tasks:**
  1. `ReachBridge.call_channel(platform, 'search'|'read', args)` via the pinned Node adaptation; items[] JSON contract (`title/url/author/published_at/text`); **whole bridge sequential**, one call at a time per platform, `call_gap_within_source_s` ≥ 10 s between same-platform calls.
  2. Error mapping per §8.10: `NEED_SETUP`→quarantine `setup incomplete`; `NEED_LOGIN`→quarantine `login state stale`; `RATE_LIMITED`/`TIMEOUT`/`BUSY`→ backoff + retry next sweep (respect `retry_after_seconds`); `UPSTREAM_BROKEN`/`NOT_FOUND`→ per-item error, no quarantine.
  3. `inject_cookies(platform, cookie_string)`: validate non-empty; write platform cookie file inside profile dir (0600); set `source_credentials_status(browser_cookies, configured=1)`; **never log values; echo nothing**.
  4. `login_state(platform)`: `NEED_LOGIN`/`PLATFORM_BLOCKED` → `stale`.
  5. Community adapter: fetch → bridge → canonical items → new_item drafts; read-only actions only.
  6. Netizen: no write actions exposed by the bridge API (NFR-04).
- **Contracts:** consumed — bridge JSON contract (external interface per Doc 1 §6), store sources (set_source_health, credentials). Produced — `ReachBridge` API (seams S5, S6, S6b consumer), community-family FetchResult.
- **Acceptance:** (1) T7 trace: reach fetch in sweep → items → new_item events; `NEED_LOGIN` → quarantine with reason `login state stale`; (2) error-mapping table fully covered; (3) ≥10 s gap enforced between same-platform fixture calls; (4) cookie injection writes 0600 file, sets credentials row, no value in any log/response (guard).
- **Verification:** `pytest tests/unit/test_reach_bridge.py tests/unit/test_reach_adapter.py tests/guard/test_reach_bridge.py -q` → green (stub CLI; no browser, no network — hermetic per §12).
- **Sizing:** M.

---

### WP-12 — `changedetect.py`

- **Phase/Depends-on:** P2 / WP-01, WP-02 · ∥ with WP-13
- **Objective:** Implement deterministic per-family change detection producing `change_events` rows with before/after state and raw-item links.
- **Rationale:** FR-10/D14: the LLM never detects a change; this module is the sole origin of change_events.
- **Scope (create):** `src/vorbote/changedetect.py`. **Out of scope:** consuming events (WP-14), emitting observations.
- **Spec refs:** [Doc 2 §8.12, §5.2 change_events, Doc 1 FR-10, §12.2 detection-kind table].
- **Tasks:**
  1. Per-family detection exactly per §8.12: feed/change_feed/trends → new_item (+ trends value_delta on traffic_hint change ≥ 2×); rest_api → new_item + value_delta; html_diff/marketplace → list_diff, rank_movement (threshold ±3 default), snapshot_diff, new_item; dataset → value_delta + snapshot_diff.
  2. Prior-state loading: last run's items/snapshots per source from the store.
  3. Drafts → `create_change_event` (seam S8b): before/after JSON, magnitude, raw_item_ids; `extracted=0`.
  4. Thresholds from config defaults; deterministic only — no LLM import.
- **Contracts:** consumed — store reads (prior state), adapters' drafts (WP-07–10, 24). Produced — `detect(source, items, prior) → list[ChangeEventDraft]` (seam S8) + persisted events.
- **Acceptance:** (1) every family × kind combination produces correct events on fixture transitions; (2) rank threshold and 2× traffic threshold honored; (3) event rows carry before/after and raw_item_ids; (4) re-run with unchanged state → no events.
- **Verification:** `pytest tests/unit/test_changedetect.py -q` → green (prior state built by running the production fetch path twice — test-construction rule).
- **Sizing:** M.

---

### WP-13 — `llm.py` + `metering.py`

- **Phase/Depends-on:** P2 / WP-02, WP-03 · ∥ with WP-12
- **Objective:** Implement the single-path LLM client (provider table, one model, conditional reasoning param) and both metering ledgers with warn-only ceilings.
- **Rationale:** D6/C-03 single path; FR-32/33 cost observability; every later LLM consumer (extract, hypothesize, guidance) rides exactly this client.
- **Scope (create):** `src/vorbote/llm.py`, `src/vorbote/metering.py`; conftest LLM mock `/v1/chat/completions` (contract-valid + contract-violating variants). **Out of scope:** stage wrappers (WP-14), guidance engine (WP-21).
- **Spec refs:** [Doc 2 §8.13, §8.18, §4 llm/exa sections, §6.10, §11 LLM rows, T13, Doc 1 FR-31/32/33, D16].
- **Tasks:**
  1. `LLMClient.chat_completion(messages, model|None, temperature, max_tokens)` — the only chat-completions path; provider table openrouter/opencode_go/deepseek/custom with base_urls; model string from config (`glm-5.3-flash` default); `reasoning_effort` forwarded only when set; secrets never logged.
  2. Every call metered into `llm_usage` (stage tag extract/hypothesize/guidance; run/session refs) and ceiling-checked after call.
  3. Cost = tokens × blended prices from config; `ceiling_states` upsert; `warned_at` set once per month per ledger; **warn-only — operations continue**.
  4. Exa metering functions per §8.5/§8.18 (`record_exa_usage`, `get_month_totals`, rollup shapes per §6.10).
- **Contracts:** consumed — store/metering (WP-02), config prices. Produced — `LLMClient` (seam S20-class), metering functions (seams S4, S11, S32), LLM mock fixture for the whole suite.
- **Acceptance:** (1) metering math exact per §4 prices; (2) warn set-once per month (T13); (3) reasoning param absent when empty; (4) fixture LLM mock serves both variants for later stages.
- **Verification:** `pytest tests/unit/test_llm.py tests/unit/test_metering.py -q` → green.
- **Sizing:** M.

---

### WP-14 — `extract.py` + `hypothesize.py` + stage prompts

- **Phase/Depends-on:** P2 / WP-12, WP-13
- **Objective:** Implement the two LLM pipeline stages with strict JSON output contracts, closed-tag validation, idempotent retry, and degrade-to-raw on contract violation, landing observations in triage.
- **Rationale:** FR-11/C-08 discipline lives here: extraction is structured; hypothesis records carry ≥1 alternative + ≥1 unknown or are rejected.
- **Scope (create):** `src/vorbote/extract.py`, `src/vorbote/hypothesize.py`, `src/vorbote/prompts/extractor.md`, `src/vorbote/prompts/hypothesis.md`. **Out of scope:** orchestration of stage order (WP-15), guidance prompts (WP-21).
- **Spec refs:** [Doc 2 §8.13.1, §8.21, §11 (LLM failure/violation rows), T12, Doc 1 FR-11, C-08, §1.2 evidence rules].
- **Tasks:**
  1. `extractor.md`: EN prompt, pinned JSON output contract `{driver_tag, affected_population_hint, sector_hint, language}` per event; closed driver-tag set.
  2. `extract.run(sweep_run)`: consume unextracted `change_events`, batch ≤ 20; evidence context from linked raw items (title + excerpt ≤ 500 chars each); validate closed tag sets; failure → event stays `extracted=0` (idempotent retry next sweep); success → mark extracted, pass to hypothesize.
  3. `hypothesis.md`: EN prompt, pinned JSON `{title, description, proposed_explanation, alternative_explanations ≥1, unknowns ≥1, evidence_refs}`; **no-numbers rule baked in** [C-08].
  4. `hypothesize.run(sweep_run)`: batch ≤ 10; enforce ≥1 alternative and ≥1 unknown; violation → degrade-to-raw observation (raw text as description; run-stats flag `llm_contract_violation`); success → create `observations` rows (status triage) + evidence link rows.
- **Contracts:** consumed — change_events (WP-12), LLMClient + mock (WP-13), store/funnel. Produced — triage observations with evidence (seams S9/S9b/S10/S10b).
- **Acceptance:** (1) valid contract → observation rows with alternatives, unknowns, evidence links; (2) invalid JSON at extract → event stays unextracted, retried idempotently; (3) violation at hypothesize → degrade-to-raw observation + stats flag (T12); (4) batch caps 20/10 honored; (5) no invented numbers in any output (C-08 guard assertion on fixtures).
- **Verification:** `pytest tests/unit/test_extract.py tests/unit/test_hypothesize.py tests/integration/test_pipeline_stages.py -q` → green.
- **Sizing:** M.

---

### WP-15 — `scheduler.py` + `orchestrator.py`

- **Phase/Depends-on:** P2 / WP-07, WP-08, WP-09, WP-10, WP-11, WP-12, WP-14, WP-24
- **Objective:** Implement the run engine: due-source planning, sweep execution with per-source isolation and lock, post-sweep stage chain, catch-up/startup semantics, quarantine thresholds — the T1 trace end-to-end.
- **Rationale:** FR-02/FR-28; every §9 background job except retention/backup originates here.
- **Scope (create):** `src/vorbote/scheduler.py`, `src/vorbote/orchestrator.py`. **Out of scope:** retention hook body (no-op until WP-25 — hook slot present), API trigger routes (WP-20).
- **Spec refs:** [Doc 2 §8.16, §9, §9.1, §6.11, T1/T2/T11, Doc 1 §3.3, §9].
- **Tasks:**
  1. Scheduler: daily window per group `schedule_spec` + catch-up (one per missed window, bounded) + startup sweep of interrupted runs (`running` → `interrupted`; no auto re-run) + month-boundary ledger-rollup tick (T13) + post-sweep retention hook slot (S24).
  2. Orchestrator planning: enabled ∧ health ok (or releasable) ∧ credential checks for optional sources; quarantined skipped.
  3. Execution: per source — adapter.fetch via ladder under politeness; normalizer; snapshot write; upsert raw items; change detect; per-source failure isolation (source failure ≠ sweep failure); 3 consecutive failures → quarantine `repeated failure`.
  4. Sweep lock: one at a time (seam S26); manual-during-sweep → `409 SWEEP_IN_PROGRESS` (consumed by WP-20).
  5. Post-sweep chain: extract → hypothesize → triage; stats into `sweep_run.stats` (items, changes, errors, llm_cost_usd, exa_cost_usd); retention hook last (no-op placeholder until WP-25).
  6. LLM unavailability → collection/detection continue; events remain `extracted=0` (NFR-08).
- **Contracts:** consumed — every family adapter, ladder, stores, stages. Produced — `run_sweep(trigger)` (seam S25), `try_lock` (S26), sweep_run/source_run/fetch_attempts state.
- **Acceptance:** (1) T1 happy path on fixture sources: sweep completes, raw items + change events + triage observations committed, stats populated; (2) T2: concurrent trigger → 409 envelope, no state change; (3) T11 catch-up: missed window → one catch-up run, trigger_kind `catchup`; (4) boot with `running` row → swept to `interrupted`; (5) fixture-source failure → source_run error, sweep completes; 3 consecutive → quarantine `repeated failure`; (6) LLM mock down → sweep completes, events unextracted, retried next run.
- **Verification:** `pytest tests/integration/test_sweep.py -q` → green (the T1/T2/T11 scenarios as tests).
- **Sizing:** L.

---

### WP-25 — `retention.py`

- **Phase/Depends-on:** P2 / WP-02, WP-15
- **Objective:** Implement the daily prune pass: FIFO eviction of unreferenced, unpinned, aged raw items/snapshots with warning stats, and the storage-ceiling warn path.
- **Rationale:** FR-30; pins block eviction — anything referenced by the funnel survives indefinitely.
- **Scope (create):** `src/vorbote/retention.py`; guard `tests/guard/test_retention.py`; wire the S24 hook (replacing WP-15's no-op). **Out of scope:** backup script (WP-26).
- **Spec refs:** [Doc 2 §8.17, §9 (retention row), T9, Doc 1 FR-30, §3.7].
- **Tasks:**
  1. Prune candidates = snapshots + raw items not referenced by any evidence-bearing row (`evidence_records`, `change_events.raw_item_ids`, `observations.evidence`) AND `fetched_at < now − months` AND not pinned — via SQL joins (deterministic).
  2. FIFO delete (snapshot file + row); count+bytes → warning surfaced via run stats → client.
  3. Storage ceiling: warn-only; surfaced via status endpoint fields.
  4. Wire post-sweep hook (S24) into scheduler.
- **Contracts:** consumed — store items (`list_prune_candidates`), orchestrator hook. Produced — `prune()` with warning stats.
- **Acceptance:** (1) T9: prune removes only unreferenced ∧ unpinned ∧ aged; referenced/pinned survive; (2) FIFO order honored; (3) warning stats carry count + bytes; (4) ceiling warn-only (no deletion from ceiling).
- **Verification:** `pytest tests/unit/test_retention.py tests/guard/test_retention.py -q` → green.
- **Sizing:** S.

---

### WP-16 — Evaluation: observation / hypothesis / candidate managers

- **Phase/Depends-on:** P3 / WP-02, WP-14
- **Objective:** Implement the three funnel managers: triage lifecycle incl. negative findings, hypothesis lifecycle, candidate records with dossier-assembly lineage and pinning.
- **Rationale:** FR-12/13/14: no auto-promotion; negative findings first-class; candidates carry the full FR-13 field set.
- **Scope (create):** `src/vorbote/evaluation/__init__.py`, `observations.py`, `hypotheses.py`, `candidates.py`. **Out of scope:** scoring/economics (WP-17), validation (WP-18), API routes (WP-20).
- **Spec refs:** [Doc 2 §5.2 (observations/hypotheses/candidates DDL), §5.3 funnel functions, T3/T3b, Doc 1 §3.5, FR-12/13/14].
- **Tasks:**
  1. Observation manager: triage queue listing (filters, evidence counts); promote (creates hypothesis draft row, status transition, audit-fact via write path actor operator); decline; negative (status → `negative_finding` + rationale addendum in description — identical record structure, FR-12).
  2. Hypothesis manager: draft → active → merged/rejected lifecycle; basis evidence refs; incumbent-response analysis and unresolved-demand assessment fields.
  3. Candidate manager: CRUD over the full FR-13 field set; pin/unpin (pins table + `candidates.pinned`); dossier-assembly inputs: lineage observation → hypothesis → candidate + evidence + validation state pointers.
- **Contracts:** consumed — store/funnel (WP-02), observations from pipeline (WP-14). Produced — manager functions used by API routes (S12–S14) and guidance context assembly (S19).
- **Acceptance:** (1) T3: promote → hypothesis draft + observation promoted; (2) T3b: negative → status + rationale addendum, same schema; (3) candidate CRUD round-trips all FR-13 fields; (4) lineage query returns full chain; (5) all test state built via production path (sweep → promote → candidate).
- **Verification:** `pytest tests/unit/test_funnel_managers.py tests/integration/test_triage_flow.py -q` → green.
- **Sizing:** M.

---

### WP-17 — Evaluation: scoring + economics + constraints (+ weights data)

- **Phase/Depends-on:** P3 / WP-02, WP-16
- **Objective:** Implement the deterministic evaluation core: weighted scoring with exclusion gate, the four backward-economics calculators with provenance-tagged inputs, and versioned constraints.
- **Rationale:** FR-15/16/17/18; D2/D8 keep the LLM out of judgment arithmetic; C-08 forbids invented numbers.
- **Scope (create):** `src/vorbote/evaluation/scoring.py`, `economics.py`, `constraints.py`; guards `tests/guard/test_scoring.py`, `tests/guard/test_economics.py`; default `scoring_weights` row insertion (25/25/20/15/10/5, one active). **Out of scope:** API routes, client panels.
- **Spec refs:** [Doc 2 §8.6, §8.7, §5.2 (constraints/scores/economics_records/scoring_weights), §11 (EXCLUSION_ACTIVE), T6, Doc 1 §1.2 evaluation model, FR-15/16/17/18].
- **Tasks:**
  1. `compute_score`: 1–5 int criteria; weights normalized (sum to 1) before weighting; deterministic weighted sum.
  2. `apply_exclusions`: exclusion check before scoring; active exclusion → scoring rejected (API layer surfaces `422 EXCLUSION_ACTIVE`).
  3. Economics per §8.6 formulas **exactly** (contribution, monthly profit, effective hourly, required transactions via ceil); notes flags: negative contribution; hourly below `min_hourly`; required-vs-assumed mismatch; missing inputs → validation error, **never defaults**; provenance tags operator-estimate / evidence-ref on every input.
  4. Constraints: versioned rows, exactly one active; POST deactivates previous, version+1.
  5. Weights as data rows: default set active; `get_active_weights`/`set_active_weights`.
- **Contracts:** consumed — store/scoring (WP-02), candidates (WP-16). Produced — `compute_score` (S15), `apply_exclusions` (S16), `compute(EconomicsInputs)` (S17).
- **Acceptance:** (1) formula outputs match §8.6 to the cent on the spec's own shapes; (2) edge cases: negative contribution, zero hours → flags not errors; (3) exclusion gate blocks scoring; (4) weights normalization; (5) constraint versioning; (6) no input without provenance is accepted (C-08).
- **Verification:** `pytest tests/unit/test_scoring.py tests/unit/test_economics.py tests/guard/test_scoring.py tests/guard/test_economics.py -q` → green.
- **Sizing:** M.

---

### WP-18 — Evaluation: validation manager

- **Phase/Depends-on:** P3 / WP-02, WP-16
- **Objective:** Implement investigations, past-purchasing interview guides, experiments with riskiest assumption + write-once immutable failure criteria + proposition-vs-test verdicts, and falsification notes.
- **Rationale:** FR-19/20; failure criteria immutability is a spec invariant no test may weaken.
- **Scope (create):** `src/vorbote/evaluation/validation.py`. **Out of scope:** API routes (WP-20), client widget (WP-31).
- **Spec refs:** [Doc 2 §5.2 (validation DDL), §5.3 validation functions, §6.7, §11 (409 CONFLICT), T-row failure criteria, Doc 1 FR-19/20, §1.2 validation procedure].
- **Tasks:**
  1. Investigations: create (plan, unknowns_targeted), update (findings append), status open/in_progress/done.
  2. Interview guides: create with questions list (past-purchasing focus per Doc 1 §1.2), notes.
  3. Experiments: create (riskiest_assumption, design, budget_cap, time_cap_hours); `set_failure_criteria` write-once (NULL → set; second → `409 CONFLICT`-class error); `record_experiment_result` (result + proposition_vs_test verdict from the closed set).
  4. Falsification notes: create ("what would convince me this is not viable"), resolve (resolution + resolved_at).
- **Contracts:** consumed — store/validation (WP-02), candidates. Produced — validation manager functions (seam S18) consumed by API + dossier.
- **Acceptance:** (1) write-once: second criteria write → 409-class error; (2) findings append-only; (3) verdict closed set enforced; (4) falsification note lifecycle complete.
- **Verification:** `pytest tests/unit/test_validation.py -q` → green.
- **Sizing:** S–M.

---

### WP-21 — Guidance engine + role prompts + context assembly

- **Phase/Depends-on:** P4 / WP-02, WP-05, WP-13, WP-16 · ∥ with WP-17/18 (independent of them)
- **Objective:** Implement the guidance engine: session lifecycle, deterministic context assembly, four roles via system prompts, reply+proposal emission with closed-action validation, and research-planner queries executed through the ladder.
- **Rationale:** FR-23; D7; the LLM sees assembled context only — never the DB (S19); never fetches directly (S31).
- **Scope (create):** `src/vorbote/guidance/__init__.py`, `engine.py`, `prompts/guidance-interviewer.md`, `guidance-analyst.md`, `guidance-planner.md`, `guidance-scribe.md`. **Out of scope:** writes controller + digest (WP-22), routes (WP-23).
- **Spec refs:** [Doc 2 §8.8, §8.21, §4 guidance section, T14, Doc 1 §3.6, FR-23/25].
- **Tasks:**
  1. `start_session(focus)` / `send_operator_message(session_id, content) → {reply, proposals}` / `close_session(session_id)` signatures per §8.8.
  2. Context assembly: deterministic store reads (dossier/observations/evidence/constraints/scores for focus); cap `max_context_items` = 40; LLM sees assembled context only.
  3. Roles via the four system prompts (interviewer targets unknowns; analyst drafts explanations/alternatives; research planner proposes queries; scribe drafts records).
  4. Proposals: validated against the §8.9 closed action set **before persistence** — unknown action string discarded pre-persistence; only confirm-required actions may be emitted as proposals.
  5. Planner queries: engine formulates → executes via `Ladder.fetch` steps 1–2 (S31) → results appended to session context next turn (S31b); metered.
- **Contracts:** consumed — LLMClient (S20), ladder (S1), store reads (S19), guidance store functions (WP-02). Produced — `GuidanceEngine` API consumed by routes (WP-23) and writes (WP-22).
- **Acceptance:** (1) context assembled deterministically from fixture DB, capped at 40; (2) reply + proposals persisted `proposed`; (3) unknown action discarded — never surfaces as a proposal; (4) T14: planner query rides ladder, results in next-turn context, usage rows committed; (5) every LLM call metered with stage `guidance`.
- **Verification:** `pytest tests/unit/test_guidance_engine.py -q` → green (LLM mock).
- **Sizing:** M.

---

### WP-22 — Guidance bounded writes + session digest

- **Phase/Depends-on:** P4 / WP-16, WP-17, WP-21
- **Objective:** Implement the bounded-write controller (direct + confirm-required actions, one transaction per proposal, every write audited) and the deterministic session digest.
- **Rationale:** FR-23/24/25; the write boundary is the accountability core — no DB write without audit row [Doc 2 §14 item 5].
- **Scope (create):** `src/vorbote/guidance/writes.py`, `guidance/digest.py`; guard `tests/guard/test_guidance_writes.py`. **Out of scope:** routes (WP-23), client cards (WP-33).
- **Spec refs:** [Doc 2 §8.8 (writes/digest paragraphs), §8.9, §6.8, T5, Doc 1 FR-23/24/25, D7/D18].
- **Tasks:**
  1. `apply_direct(action, payload)`: `note.write`, `disposition.write` only; inserts `write_audit` row itself.
  2. `apply_confirmed(proposal)`: validate payload against the closed §8.9 schema per action; apply inside one transaction; for `score.write` compute via `evaluation/scoring.compute_score` (values operator-confirmed); audit row with before/after.
  3. `decide_proposal`: confirm/decline state transitions with `decided_at`.
  4. Digest: deterministic from session records — topics (assistant message headings), proposals summary {raised, confirmed, declined}, writes (from write_audit for that session), open questions (last assistant message's open-questions block); generated at close; stored `guidance_digests`.
- **Contracts:** consumed — engine proposals (WP-21), scoring compute (WP-17), store guidance functions. Produced — writes controller (S21/S21b/S22), digest builder (S23).
- **Acceptance:** (1) T5: confirm score.write → score row + audit row + proposal confirmed, one transaction; (2) bounded set: direct actions limited to the two; confirm-required limited to the three + delete; (3) every write lands in write_audit — no write without audit row; (4) digest deterministic: same session records → identical digest; (5) decline path marks declined, no write.
- **Verification:** `pytest tests/unit/test_guidance_writes.py tests/unit/test_digest.py tests/guard/test_guidance_writes.py -q` → green.
- **Sizing:** M.

---

### WP-19 — Web app factory + gates + System + Sources routes + cookie endpoint

- **Phase/Depends-on:** P5 / WP-02, WP-03, WP-06, WP-15, WP-24
- **Objective:** Implement the Flask app factory with Host/Origin gates, system status/health, the full sources route group including one-polite-fetch test and the no-echo cookie-injection endpoint.
- **Rationale:** NFR-06 defense-in-depth; the cookie endpoint is the wizard's server-side landing (S6/S6b) and the most secrets-sensitive route.
- **Scope (create):** `src/vorbote/web/__init__.py` (app factory), `src/vorbote/web/api.py` (this WP's routes only); start guard `tests/guard/test_secrets.py` (cookie no-echo cases; extended in WP-24/WP-34). **Out of scope:** all other route groups (WP-20/23/27).
- **Spec refs:** [Doc 2 §6.1, §6.2, §6.11, §7, §7.1, §8.14, T4, §11 (BAD_COOKIE_STRING), Doc 1 §6, NFR-05/06].
- **Tasks:**
  1. App factory; before-request gate: `Host` ∈ {127.0.0.1[:port], localhost[:port]}; `Origin` (if present) localhost-scheme; else `403` — every request.
  2. `GET /status` (service, uptime, scheduler next_sweep_at, db_ok, quarantined list, ledger summaries) and `GET /health` (204/envelope) per §6.1.
  3. Sources CRUD per §6.2 (`endpoint_config` JSON in/out; PUT for URL drift; DELETE); `GET /sources/<id>/test` — one polite fetch, ladder step 1 only, no quarantine side-effects; `setup-requirements` (kind + steps + credential_configured); quarantine/release.
  4. `POST /sources/<id>/cookies`: parse body; forward to `ReachBridge.inject_cookies`; respond `{ok:true}` — **no cookie material in any response or log**; audit fact row (`cookies.injected`, no values); malformed → `400 BAD_COOKIE_STRING`; credentials_status row set via S6b.
  5. Bind 127.0.0.1 only; error envelope everywhere.
- **Contracts:** consumed — stores, orchestrator state, reach bridge (WP-24), ladder/adapters for test fetch. Produced — app factory (all later route WPs register onto it), seams S6/S6b, the sources route surface.
- **Acceptance:** (1) non-local Host/Origin → 403 (both gate variants); (2) status shape per §6.1; (3) test-fetch returns outcome/http_status/sample_titles without quarantine side-effects; (4) T4 trace: upload → credentials configured, bridge profile written, response `{ok:true}`, **no value in logs/responses/DB** (guard); (5) empty cookie string → 400.
- **Verification:** `pytest tests/unit/test_web_sources.py tests/guard/test_secrets.py -q` → green (Flask test client per §12).
- **Sizing:** L — deliberately split from WP-20 (route-group split keeps each session inside one context window).

---

### WP-20 — Web routes: sweeps/runs, observations, candidates, scoring/economics/constraints, validation, evidence/pins, ledgers

- **Phase/Depends-on:** P5 / WP-02, WP-11, WP-13, WP-15, WP-16, WP-17, WP-18, WP-19
- **Objective:** Implement all remaining non-guidance, non-export API routes with their exact §6 schemas and error semantics.
- **Rationale:** These routes are the machine surface of the entire evaluation engine; schemas are pinned in §6.
- **Scope (create):** route handlers in `src/vorbote/web/api.py`; guard `tests/guard/test_api_gates.py`. **Out of scope:** guidance routes (WP-23), export routes (WP-27), client (P6).
- **Spec refs:** [Doc 2 §6.3–§6.10, §6.11, §11 (409/422 rows), T2/T3/T3b/T6, §14 items 3–4].
- **Tasks:**
  1. Sweeps/runs per §6.3: trigger (`202 {sweep_run_id}`; during active sweep → `409 SWEEP_IN_PROGRESS` via S26), run lists/detail with source_runs.
  2. Observations per §6.4: list (status/limit/offset, evidence counts), detail (+ linked change_events summary), promote/decline/negative.
  3. Candidates per §6.5: CRUD, pin/unpin, dossier assembly (candidate + evidence + latest score + latest economics + validation state + lineage). (Export route is WP-27.)
  4. Scoring/economics/constraints per §6.6: score history + POST (deterministic compute, active weights; active exclusion → `422 EXCLUSION_ACTIVE`); economics POST → 201 outputs; constraints GET/POST (version+1, deactivate previous).
  5. Validation per §6.7: investigations CRUD + guides; experiments CRUD; failure-criteria write-once (`409 CONFLICT`); result recording.
  6. Evidence/pins per §6.9: snapshot file content (correct mime); pin create/list.
  7. Ledgers per §6.10: llm (stage/run/session totals + warned) and exa (source totals + warned).
- **Contracts:** consumed — managers (WP-16/17/18), orchestrator lock, metering, snapshot store. Produced — the full §6.3–§6.10 route surface (client maps 1:1 per §6.12).
- **Acceptance:** (1) every route's request/response shape matches §6; (2) T2 409; (3) exclusion gate 422; (4) failure-criteria second write 409; (5) ledger breakdowns per §6.10; (6) snapshot served with correct mime.
- **Verification:** `pytest tests/integration/test_api_routes.py tests/guard/test_api_gates.py -q` → green.
- **Sizing:** L — route groups are thin delegation over existing managers, but the schema breadth fills a session; if context strains, the session boundary may fall after task 4 (rest in the WP's follow-up session, same WP id).

---

### WP-23 — Web routes: guidance

- **Phase/Depends-on:** P5 / WP-19, WP-22 · ∥ with WP-27
- **Objective:** Implement the guidance route group: session create/get/close, messages (the only assistant-reply path), proposals list/confirm/decline, digest fetch.
- **Rationale:** §6.8 + T5; confirm applies through the writes controller inside one transaction.
- **Scope (create):** guidance route handlers in `src/vorbote/web/api.py`. **Out of scope:** engine/writes logic (done), client chat UI.
- **Spec refs:** [Doc 2 §6.8, §6.11 (confirm transaction), T5, §14 item 5].
- **Tasks:**
  1. `POST /guidance/sessions` (focus optional) → 201; `GET …/sessions/<id>` (+ digest if closed); messages GET/POST (server-side LLM call, metered; reply + proposals).
  2. `GET …/proposals`; `POST …/proposals/<id>/confirm` → `apply_confirmed` (one transaction, audit) → 200; `/decline` → 200.
  3. `POST …/sessions/<id>/close` → deterministic digest → `{digest_id}`; `GET /guidance/digests/<id>`.
- **Contracts:** consumed — GuidanceEngine (WP-21), writes controller (WP-22), app factory. Produced — the guidance route surface (§6.12 client map).
- **Acceptance:** (1) T5 end-to-end via API: session → message → proposal → confirm → score + audit rows → close → digest; (2) message POST is the only reply path; (3) confirm is transactional (failure mid-apply → no partial write).
- **Verification:** `pytest tests/integration/test_guidance_api.py -q` → green.
- **Sizing:** S–M.

---

### WP-27 — `export/dossier.py` + candidate dossier/export routes

- **Phase/Depends-on:** P5 / WP-17, WP-18, WP-20 · ∥ with WP-23
- **Objective:** Implement Markdown/HTML dossier rendering and the `GET /candidates/<id>/export` route.
- **Rationale:** FR-26 (Markdown and HTML only); dossier assembly inputs already exist from WP-16/20.
- **Scope (create):** `src/vorbote/export/dossier.py`; export route in `web/api.py`. **Out of scope:** client save dialog (WP-29).
- **Spec refs:** [Doc 2 §8.15, §6.5 (export), T10, Doc 1 FR-26].
- **Tasks:**
  1. Dossier assembly from store APIs (candidate + evidence + latest score + latest economics + validation timeline + lineage).
  2. Markdown renderer: headings, evidence blocks with excerpts, score table, economics table, validation timeline.
  3. HTML renderer: same structure, CSS inline.
  4. Route: `?format=markdown|html` → UTF-8 body; other formats → error envelope (out-of-scope formats rejected).
- **Contracts:** consumed — manager/store reads. Produced — `render_markdown/html` (seam S27), export route.
- **Acceptance:** (1) both formats render the full dossier on a fixture candidate built via production path; (2) evidence excerpts verbatim (original language); (3) unknown format → envelope error.
- **Verification:** `pytest tests/unit/test_dossier_export.py -q` → green.
- **Sizing:** S.

---

### WP-26 — CLI completion + deploy artifacts + deployment guide

- **Phase/Depends-on:** P5 / WP-15, WP-20, WP-23, WP-24, WP-25, WP-27
- **Objective:** Complete `vorbote run`/`serve`/`help`, the systemd unit, backup script, and the deployment guide — the server distribution reaches its §13 form.
- **Rationale:** §13 build/run/deploy is in-scope deliverable #6; M3 needs `vorbote serve` real.
- **Scope (create):** `cli.py` completion; `deploy/vorbote.service`; `deploy/backup.sh`; `docs/deployment.md`. **Out of scope:** client packaging (P6).
- **Spec refs:** [Doc 2 §8.19, §9 (backup row), §13, §10.1, Doc 1 §8].
- **Tasks:**
  1. `vorbote run [--group|--all]`: one-shot sweep; exit 0/1.
  2. `vorbote serve`: API + scheduler thread; binds 127.0.0.1 only; config re-read per scheduler tick.
  3. `vorbote help`; `vorbote migrate` (applies pending migrations; init no-op on existing) per §13 upgrade row.
  4. systemd unit: `Environment=VORBOTE_HOME=/var/lib/vorbote`, `Restart=on-failure`, journal logging.
  5. `backup.sh`: `sqlite3 …VACUUM INTO` + `rsync -a --exclude reach-profile` snapshots dir; restore = copy back + idempotent migrations.
  6. `docs/deployment.md`: prerequisites (Node ≥ 18, headless Chromium/`CHROME_BIN`, rsync), init sequence, systemd enable, first-sweep checklist, cookie-upload runbook, quarantine release, URL-drift runbook (test fetch → edit endpoint → re-enable, T15), backup/restore, ledgers review cadence, logrotate.
- **Contracts:** consumed — everything server-side. Produced — operator-runnable server distribution; the M3 smoke procedure.
- **Acceptance:** (1) `vorbote run --all` on fixture home exits 0 and produces a completed sweep_run; (2) `vorbote serve` serves `/status` on loopback; (3) backup script on temp home produces DB copy + snapshot sync without reach profile; (4) guide covers every §10.1 runbook pointer.
- **Verification:** `pytest tests/integration/test_cli.py -q` → green; manual smoke per M3 procedure (below).
- **Sizing:** M.

---

### WP-28 — Client shell: app, tunnel, api, i18n, workers

- **Phase/Depends-on:** P6 / WP-19, WP-20
- **Objective:** Implement the client foundation: main window + left nav, SSH local-forward management, thin async API client, EN/DE i18n with runtime switch, QThread workers.
- **Rationale:** §6.12: every client feature maps 1:1 onto `/api/v1`; the tunnel is transport-only and **is** the trust model [Doc 1 §3.1/§7].
- **Scope (create):** `src/vorbote_client/app.py`, `tunnel.py`, `api.py`, `i18n.py`, `workers.py`; guard `tests/guard/test_i18n.py` (completeness). **Out of scope:** all widgets (WP-29–33).
- **Spec refs:** [Doc 2 §8.20 (app/tunnel/api/i18n/workers rows), §4.4 (QSettings), §6.12, §12 L3, Doc 1 FR-22/27, NFR-09].
- **Tasks:**
  1. `app.py`: QMainWindow; left nav with the eight views (placeholders until their WPs); first-run settings (host/user/port, language choice); client stores only §4.4 keys — **no secrets client-side**.
  2. `tunnel.py`: spawn/manage `ssh -N -L <port>:127.0.0.1:<api-port> <user>@<host>`; ensure/restart; connection status surfaced.
  3. `api.py`: thin client for the full §6 route inventory (get/post/put); error-envelope unpacking.
  4. `i18n.py`: EN (default) + DE catalogs; runtime switch; **source excerpts never translated** (verbatim rendering contract).
  5. `workers.py`: QThread wrappers so no API call blocks the UI thread.
- **Contracts:** consumed — server API (WP-19/20, offscreen tests against fixture server/Flask test client). Produced — widget-ready shell + api/i18n/worker infrastructure (seam S28/S29).
- **Acceptance:** (1) offscreen launch shows all nav entries; (2) tunnel spawn command constructed exactly per §8.20 (unit test on command construction, no real ssh); (3) language switch runtime-effective; (4) i18n catalogs complete for all client strings (guard); (5) QSettings limited to §4.4 keys.
- **Verification:** `QT_QPA_PLATFORM=offscreen pytest tests/client/test_shell.py tests/guard/test_i18n.py -q` → green.
- **Sizing:** M.

---

### WP-29 — Client widgets: triage + dossier (+ export download)

- **Phase/Depends-on:** P6 / WP-20, WP-28 · ∥ with 30–32
- **Objective:** Implement the triage queue view and the dossier view with evidence browser and Markdown/HTML export saving.
- **Rationale:** The daily operator rhythm starts here (triage pass → dossier work) [Doc 1 §9].
- **Scope (create):** `src/vorbote_client/widgets/triage.py`, `widgets/dossier.py`, `export_download.py`. **Out of scope:** scoring panel (WP-30).
- **Spec refs:** [Doc 2 §8.20 (triage/dossier/export_download rows), §6.4/§6.5, T3/T3b/T10, Doc 1 FR-22/26].
- **Tasks:**
  1. Triage: queue table with status filters, evidence-count column; actions promote/decline/negative (rationale dialog) mapping to §6.4.
  2. Dossier: candidate record + evidence browser (snapshot preview for pdf_text via text dump); lineage display.
  3. Export buttons (markdown/html) → fetch §6.5 export body → `export_download.py` saves via file dialog to `dossier_export_dir`.
- **Contracts:** consumed — api client (WP-28), §6.4/§6.5 routes. Produced — the two primary operator views.
- **Acceptance:** (1) offscreen: queue renders fixture observations with evidence counts; promote/decline/negative flow through API; (2) dossier renders full record; export writes file with correct content; (3) excerpts verbatim.
- **Verification:** `QT_QPA_PLATFORM=offscreen pytest tests/client/test_triage.py tests/client/test_dossier.py -q` → green.
- **Sizing:** M.

---

### WP-30 — Client widgets: scoring/economics + constraints editor

- **Phase/Depends-on:** P6 / WP-28 · ∥
- **Objective:** Implement the criteria-entry + deterministic score preview + economics panel (provenance-tagged inputs, reverse calculation) and the optionally-guided constraints editor.
- **Rationale:** FR-15/16/17/18 surfaces; the guided editor is client-side scripted questions (i18n), **not LLM** [Doc 2 §8.20].
- **Scope (create):** `src/vorbote_client/widgets/scoring.py`, `widgets/constraints.py`. **Out of scope:** validation widget.
- **Spec refs:** [Doc 2 §8.20 (scoring/constraints rows), §6.6, §8.6/§8.7 formulas, T6, Doc 1 FR-15/16/17/18].
- **Tasks:**
  1. Scoring: 1–5 spinboxes + rationale per criterion; **deterministic preview computed client-side with the same formula as §8.7** (identical result, no LLM).
  2. Economics: inputs form, every field provenance-tagged (operator-estimate / evidence-ref); outputs panel incl. reverse calculation display; missing inputs blocked client-side too.
  3. Constraints: optionally-guided first-run sequence (assistant-drafted questions from client i18n); editable anytime; versioning display.
- **Contracts:** consumed — api client, §6.6 routes, formula contract from §8.6/§8.7. Produced — the scoring/economics/constraints surfaces.
- **Acceptance:** (1) client-side preview equals server `weighted_total` on identical inputs (fixture test); (2) provenance tag mandatory per input; (3) guided editor completes and stores constraints via API with version+1; (4) exclusion-active state blocks score submission with visible reason.
- **Verification:** `QT_QPA_PLATFORM=offscreen pytest tests/client/test_scoring.py tests/client/test_constraints.py -q` → green.
- **Sizing:** M.

---

### WP-31 — Client widgets: validation

- **Phase/Depends-on:** P6 / WP-28 · ∥
- **Objective:** Implement the validation view: investigations, interview guides, experiments with write-once failure-criteria editor and result recording incl. proposition-vs-test.
- **Rationale:** FR-19 surfaces; immutability UX must make the write-once rule explicit (confirm dialog).
- **Scope (create):** `src/vorbote_client/widgets/validation.py`. **Out of scope:** guidance chat.
- **Spec refs:** [Doc 2 §8.20 (validation row), §6.7, §11 (409), Doc 1 FR-19].
- **Tasks:**
  1. Investigations list + plan editor + findings appender.
  2. Interview guides: question list editor (past-purchasing focus).
  3. Experiments: riskiest assumption, design, budget/time caps; failure-criteria editor with **confirm dialog and write-once warning**; result recording with proposition_vs_test selector.
- **Contracts:** consumed — api client, §6.7 routes. Produced — validation tracker surface.
- **Acceptance:** (1) full experiment lifecycle offscreen incl. 409 surfacing on second criteria write; (2) findings append; (3) guides editable.
- **Verification:** `QT_QPA_PLATFORM=offscreen pytest tests/client/test_validation.py -q` → green.
- **Sizing:** S–M.

---

### WP-32 — Client widgets: source manager + cookie-upload wizard

- **Phase/Depends-on:** P6 / WP-19, WP-24, WP-28 · ∥
- **Objective:** Implement the source manager: table with health/quarantine/credential status, per-source wizard (add/edit URL, test fetch, setup requirements, cookie upload, quarantine/release).
- **Rationale:** FR-22/35 + AS-04: the wizard is the operator's only cookie-upload path; URL drift is handled here (T15).
- **Scope (create):** `src/vorbote_client/widgets/sources.py`. **Out of scope:** bridge server-side behavior (WP-24).
- **Spec refs:** [Doc 2 §8.20 (sources row), §6.2, T4/T7b/T15, Doc 1 §7, AS-04, §12.3].
- **Tasks:**
  1. Source table: health, quarantine reason, credential status (presence/freshness only — never values), enabled state.
  2. Wizard steps: (1) add/edit incl. URL update (`PUT /sources/<id>`); (2) test fetch; (3) setup-requirements display; (4) **cookie upload** for reddit/twitter — textarea for the exported cookie string, upload to §6.2 cookies endpoint over the tunnel; (5) quarantine/release.
  3. URL-drift runbook surfaced: failing source → edit endpoint → test → release (T15).
  4. `login state stale` quarantine → wizard prompts re-upload (T7b).
- **Contracts:** consumed — api client, §6.2 routes, bridge semantics. Produced — the source-management surface + the cookie-upload path completion (S6c).
- **Acceptance:** (1) offscreen: full T4 flow — upload → credentials configured → no cookie value anywhere in client state/logs; (2) T15 URL-edit flow; (3) quarantine/release; (4) test-fetch result rendering.
- **Verification:** `QT_QPA_PLATFORM=offscreen pytest tests/client/test_sources_widget.py -q` → green.
- **Sizing:** M.

---

### WP-33 — Client widgets: guidance chat + settings/ledgers

- **Phase/Depends-on:** P6 / WP-20, WP-23, WP-27, WP-28
- **Objective:** Implement the guidance chat surface (sessions, proposal cards, digest view) and the settings/ledgers views (server settings mirror, ceilings, usage).
- **Rationale:** FR-23/25/32/33 operator surfaces; NFR-07 cost observability.
- **Scope (create):** `src/vorbote_client/widgets/guidance.py`, `widgets/settings.py`. **Out of scope:** export download (WP-29).
- **Spec refs:** [Doc 2 §8.20 (guidance/settings rows), §6.8/§6.10, T5, Doc 1 FR-23/25, D16].
- **Tasks:**
  1. Guidance: session list, chat pane (via workers), proposal cards with confirm/decline, digest view post-close.
  2. Settings: server settings mirror (read + applicable edits via API), ceiling displays, warn flags.
  3. Ledgers: llm (per-stage/run/session) + exa (per-source) breakdowns with month selector.
- **Contracts:** consumed — api client, §6.8/§6.10 routes. Produced — guidance chat + settings/ledger surfaces.
- **Acceptance:** (1) offscreen: session → message → reply → proposal card → confirm → audit visible in digest view → close → digest rendered; (2) ledgers render per-stage/per-source breakdowns with month selector and warn flags; (3) settings mirror shows server settings without ever displaying secret values; (4) all chat API calls ride workers (no UI freeze under fixture latency).
- **Verification:** `QT_QPA_PLATFORM=offscreen pytest tests/client/test_guidance_widget.py tests/client/test_settings_widget.py -q` → green.
- **Sizing:** M.

---

### WP-34 — Acceptance hardening: guards completion, coverage closure, §14 acceptance run

- **Phase/Depends-on:** P7 / WP-26, WP-29, WP-30, WP-31, WP-32, WP-33
- **Objective:** Close every [Doc 2 §12 level 4] guard to final form, reach the ≥ 90% line-coverage target on all server modules, and execute the full §14 fifteen-item acceptance run against the completed system.
- **Rationale:** NFR-11: pytest is the only gate; §14 is the project definition of done. This WP turns "all WPs merged" into "system proven."
- **Scope (modify/complete):** `tests/guard/test_secrets.py` (final no-echo/no-store sweep under forced paths: DB dump scan, all log files, every API response class); `tests/guard/test_i18n.py` (final); any guard with residual cases from earlier WPs; coverage closure across `src/vorbote/**`. **Out of scope:** any new feature behavior (a §14 item failing here routes back to the owning WP as a defect fix — same WP id, defect branch `chatbox/wp-<id>-defect`).
- **Spec refs:** [Doc 2 §12, §14, §16; Doc 1 NFR-11].
- **Tasks:**
  1. Placeholder ledger invariant: `EXPECTED_PLACEHOLDERS = 0` and the assertion holds — every §2 module landed (this is the structural 100%-coverage proof).
  2. Guard completion: run each guard file against its full §12 case list; extend where earlier WPs covered only their slice.
  3. Coverage closure: `pytest tests/ --cov=src/vorbote -q`; fill gaps in any server module below 90% lines — via tests over the production path only (test-construction rule).
  4. §14 acceptance run: execute each of the fifteen items as an integration/offscreen test or scripted runbook procedure; record results in the session log (this doubles as the M5 smoke evidence).
  5. Final regression: full suite + client suite, all green, single `main`.
- **Contracts:** consumed — the entire system. Produced — the acceptance evidence; tag `v0.5.0`.
- **Acceptance:** (1) all eleven guard files green and complete; (2) server modules ≥ 90% lines; (3) all fifteen §14 items pass, each traceable to a test or runbook step; (4) `pytest tests/ -q` fully green (server + client collected).
- **Verification:** `pytest tests/ -q` → all green; `pytest tests/ --cov=src/vorbote -q` → ≥ 90% every module; placeholder guard → 0 remaining.
- **Sizing:** L.

---

## 5. Milestones & Integration Checkpoints

At each checkpoint the system **must build and run end-to-end** on `main` (all merged WPs green). Each has a smoke procedure executable by the agent (fixture-based, hermetic); M1 and M3 additionally have an operator variant for the real host (runbook, not agent-gating). Tags `v0.1.0`…`v0.5.0` are cut at M1…M5 (§2.7).

### M1 — Provisioned system on loopback (after WP-04)

**System state:** init → DB → seeds → config → blank-secrets deterministic-only posture.

**Smoke procedure (agent, hermetic):**
1. Fresh temp `VORBOTE_HOME`; `vorbote init` → exit 0.
2. Assert: `vorbote.db` exists with 20 source rows (S-18/S-19 disabled), groups present; `.env` skeleton blank; `settings.toml` skeleton present.
3. `vorbote init` again → exit 0, no duplicate sources (no-op by `name`).
4. `pytest tests/ -q` green (WP-00…04 tests only exist at this point).

**Operator variant:** real host, same commands; `sqlite3 $VORBOTE_HOME/vorbote.db "SELECT count(*) FROM sources"` → 20.

### M2 — Full pipeline end-to-end on fixtures (after WP-25)

**System state:** sweep → ladder/politeness → all families → snapshots → change events → extract/hypothesize (LLM mock) → triage observations → retention hook.

**Smoke procedure:**
1. Container from `db.py::build_container()` with fixture LLM/Exa/reach overrides (seam S30).
2. `orchestrator.run_sweep('manual')` against fixture sources of every family — expect: completed `sweep_run` with stats; raw items + snapshots on disk; `change_events` per family kinds; observations in triage with ≥1 alternative + ≥1 unknown + evidence.
3. Force one fixture source to fail → sweep still completes; 3 consecutive → quarantine `repeated failure`.
4. With Exa key absent: ladder records `exa_disabled`; `exa_required` fixture source → `skipped_exa_needed` + quarantine `setup incomplete`; deterministic sources complete (T8).
5. Retention: age one fixture item past the window → post-sweep prune removes only it; referenced/pinned survive (T9).
6. `pytest tests/ -q` fully green through WP-25.

### M3 — Operator-runnable server (after WP-26)

**System state:** `vorbote serve` real (API + scheduler thread), CLI complete, deploy artifacts + guide.

**Smoke procedure (agent, hermetic):**
1. Temp home; `vorbote serve` in a thread; poll `GET /health` → 204.
2. `GET /status` → shape per §6.1: scheduler next_sweep_at present, quarantined list, ledger summaries.
3. `GET /sources` → 20 rows; `GET /sources/<feed-id>/test` → outcome + sample_titles from the fixture server.
4. `POST /sweeps/trigger {"all":true}` → 202; poll `GET /runs/<id>` → completed with source_runs; second trigger during sweep → 409 (T2).
5. Promote one observation through the API (`POST /observations/<id>/promote` → hypothesis_id) (T3); `GET /candidates/<id>/dossier` after building a candidate via production path.
6. Cookie flow with reach stub: `POST /sources/<reddit-id>/cookies` → `{ok:true}`; credentials_status configured; **no cookie value in any response or log file** (grep the temp home).
7. `vorbote run --all` → exit 0. Backup script on temp home → DB copy + snapshots synced, reach profile excluded.

**Operator variant (real host):** per `docs/deployment.md` first-sweep checklist — init, systemd enable, journal tail, `vorbote run --all`, ledgers review.

### M4 — Client complete against fixture server (after WP-33)

**System state:** full desktop client operating the fixture-backed server.

**Smoke procedure (offscreen):**
1. `QT_QPA_PLATFORM=offscreen` launch; first-run settings + constraints editor (guided path) → constraints stored via API (version 1).
2. Triage view: queue with evidence counts; promote/decline/negative flows.
3. Dossier view: full record; export both formats → files saved, content structurally correct.
4. Scoring: criteria entry → client preview equals server `weighted_total` (identical inputs); exclusion-active candidate → submission blocked with visible reason.
5. Economics: full inputs → outputs match §8.6 exactly (compare against server compute).
6. Validation: experiment lifecycle; second failure-criteria write → 409 surfaced.
7. Sources: wizard test-fetch; cookie upload end-to-end (fixture server + reach stub) → configured status, no value in client state.
8. Guidance: session → message → reply → proposal card → confirm → digest view.
9. Language switch EN↔DE live; evidence excerpts stay verbatim German (fixture).
10. Ledgers view with month selector.
11. `pytest tests/ -q` fully green including all client tests.

### M5 — Acceptance (after WP-34)

**System state:** the complete system; §14 proven.

**Smoke procedure:** the WP-34 §14 fifteen-item run (all items pass) + full-suite green + placeholder ledger at zero. This checkpoint is the project's completion gate (§7).

**Checkpoint discipline:** a checkpoint failing blocks all further WPs (its downstream phase cannot start); the failure routes to the owning WP as a defect. All checkpoints (M1–M5) are agent-executable hermetically; M1/M3 additionally have operator variants for the real host — runbook only, not gating the agent (tests gate [NFR-11]).

---

## 6. Coverage & Traceability Matrix

Union of all WPs = 100% of Doc 2 §1.1 scope; nothing beyond it (§1.2 exclusions appear nowhere as scope).

### 6.1 Doc 2 §1.1 scope items → WPs

| §1.1 item | WPs |
|---|---|
| 1 — server distribution (all components) | WP-02…WP-27 (per §6.2 module map below) |
| 2 — client distribution (all views) | WP-28…WP-33 |
| 3 — Initial Source Register S-01…S-20 | WP-04 (seeds), WP-07…WP-10, WP-24 (adapters), WP-32 (manager UI) |
| 4 — bilingual UI + source-language preservation | WP-28 (i18n engine), WP-29…WP-33 (views), WP-34 (i18n guard) |
| 5 — hermetic test suite | every WP's tests + WP-34 (guards/coverage) |
| 6 — ops tooling (systemd, backup, deployment guide) | WP-26 |

### 6.2 Doc 2 §2 module tree → WPs (every file, server + client)

| Module | WP | Module | WP |
|---|---|---|---|
| `cli.py` | 04, 26 | `widgets/triage.py` | 29 |
| `config.py` | 03 | `widgets/dossier.py` | 29 |
| `schema.py` | 01 | `widgets/scoring.py` | 30 |
| `db.py` | 02, 06 | `widgets/validation.py` | 31 |
| `store/sources.py` | 02 | `widgets/constraints.py` | 30 |
| `store/runs.py` | 02 | `widgets/sources.py` | 32 |
| `store/items.py` | 02 | `widgets/guidance.py` | 33 |
| `store/changes.py` | 02 | `widgets/settings.py` | 33 |
| `store/funnel.py` | 02 | `export_download.py` | 29 |
| `store/scoring.py` | 02 | `scheduler.py` | 15 |
| `store/validation.py` | 02 | `orchestrator.py` | 15 |
| `store/guidance.py` | 02 | `sources/__init__.py` (registry) | 06 |
| `store/metering.py` | 02 | `sources/base.py` | 06 |
| `store/migrations.py` | 02 | `sources/feeds.py` | 07 |
| `ladder.py` | 05 | `sources/api.py` | 08 |
| `politeness.py` | 05 | `sources/listing.py` | 09 |
| `exa_client.py` | 06 | `sources/datasets.py` | 10 |
| `reach_bridge.py` | 24 | `sources/reach.py` | 24 |
| `normalizer.py` | 11 | `extract.py` | 14 |
| `snapshots.py` | 11 | `hypothesize.py` | 14 |
| `changedetect.py` | 12 | `evaluation/observations.py`, `hypotheses.py`, `candidates.py` | 16 |
| `llm.py` | 13 | `evaluation/scoring.py`, `economics.py`, `constraints.py` | 17 |
| `prompts/extractor.md`, `hypothesis.md` | 14 | `evaluation/validation.py` | 18 |
| `prompts/guidance-*.md` (4) | 21 | `guidance/engine.py` | 21 |
| `guidance/writes.py`, `digest.py` | 22 | `metering.py` | 13 |
| `retention.py` | 25 | `web/__init__.py`, `web/api.py` | 19, 20, 23, 27 |
| `export/dossier.py` | 27 | `client/app.py`, `tunnel.py`, `api.py`, `i18n.py`, `workers.py` | 28 |
| `config/sources_seed.json` | 04 | `config/settings.example.toml` | 00 |
| `deploy/*`, `docs/deployment.md` | 26 | both `pyproject.toml` | 00 |

Every §2 file is owned by ≥ 1 WP; the placeholder ledger enforces the landing of every one (0 remaining at WP-34).

### 6.3 Doc 2 §8 module specs + §5/§6/§9–§13 → WPs

| Spec section | WP(s) | Spec section | WP(s) |
|---|---|---|---|
| §5 DDL + §5.3 store API | 02 | §8.14 web gates | 19 |
| §4 config/seeds | 03, 04 | §8.15 dossier export | 27 |
| §6.1 system routes | 19 | §8.16 scheduler/orchestrator | 15 |
| §6.2 sources + cookies | 19 (route), 32 (wizard) | §8.17 retention | 25 |
| §6.3 sweeps/runs | 20 | §8.18 metering | 13 |
| §6.4 observations | 20, 16 | §8.19 CLI | 04, 26 |
| §6.5 candidates + dossier/export | 20, 16, 27 | §8.20 client modules | 28–33 |
| §6.6 scoring/economics/constraints | 20, 17 | §8.21 prompts | 14, 21 |
| §6.7 validation | 20, 18 | §9 background jobs | 15, 25, 26 |
| §6.8 guidance | 23, 21, 22 | §10 observability | 15, 19, 26, 33 |
| §6.9 evidence/pins | 20, 11 | §11 error handling | 05, 13, 14, 15, 19, 20, 18, 24 (per-row as cited in each WP) |
| §6.10 ledgers | 20, 13 | §12 testing strategy | all WPs' verification; 34 |
| §7/§7.1/§7.2 auth + secrets | 19, 24, 28 (no client secrets), guards 34 | §13 build/run/deploy | 26 |
| §8.0 schema | 01 | §14 acceptance | 34 (M5) |
| §8.1 adapter contract | 06 | §15 composition root + seams | 06 (root); seam-by-seam in each WP's Contracts |
| §8.2 feeds | 07 | §16 consistency checklist | 34 (final) |
| §8.3(+8.3.1) api family | 08 | T1–T15 call traces | smoke procedures M2/M3/M4 + WP verifications (per-WP as cited) |
| §8.4 ladder/politeness | 05 | FR-01…FR-35, NFR-01…12, C-01…09, D1–D18 | mapped via Doc 2 §16 → the §6.3 row of this matrix (each WP cites its FRs directly) |
| §8.5 exa | 06 | | |
| §8.6 economics | 17 | | |
| §8.7 scoring | 17 | | |
| §8.8 guidance engine/writes/digest | 21, 22 | | |
| §8.9 proposal schemas | 21, 22 | | |
| §8.10 reach bridge | 24 | | |
| §8.11(+.1) normalizer/snapshots/PDF | 11 | | |
| §8.12 changedetect | 12 | | |
| §8.13(+.1) llm + stages | 13, 14 | | |

### 6.4 Nothing beyond scope

No WP implements anything in [Doc 2 §1.2]: no OCR (flagged PDFs stored, excluded — WP-11); no email; no export formats beyond Markdown/HTML (WP-27 rejects others); no per-role model routing (WP-13 single model); no eBay/Etsy (not in seeds — WP-04); no multi-user (loopback + single operator actor everywhere — WP-19 gates); no live migration (migrations forward-only from empty DB — WP-02). The reach adaptation is pinned in-repo per §8.10 (host Node prerequisite), not a PyPI dependency — within §3.1.

---

## 7. Completion Criteria

Project-level definition of done — mirrors [Doc 2 §14]; a criterion passes only via its cited verification, never by assertion:

1. **Structure:** placeholder ledger at zero — every §2 module landed (WP-34 invariant).
2. **Bring-up:** `vorbote init && vorbote serve` on a clean host brings up API + scheduler; `GET /status` shows 20 seeded sources [§14-1] — M3.
3. **Sweep:** daily sweep / `run --all` executes all enabled sources via the ladder with politeness observed in `fetch_attempts`; per-source isolation holds [§14-2] — M2 + WP-15.
4. **Pipeline:** change events per family kinds; extract + hypothesize run; observations with ≥1 alternative and ≥1 unknown land in triage [§14-3] — M2 + WP-14.
5. **Funnel:** triage actions end-to-end; promote → hypothesis → candidate; scoring gated by exclusions; economics match §8.6 exactly [§14-4] — M2/M3 + WP-16/17.
6. **Guidance:** context assembly, roles, proposal confirm/decline with write_audit rows; bounded write set enforced; digest on close; no DB write without audit row [§14-5] — M3/M4 + WP-21/22/23.
7. **Cookies:** upload → credentials configured; stale → quarantine `login state stale` → re-upload works; no cookie value in any log/response/DB [§14-6] — M3/M4 + WP-19/24/32 + guards.
8. **Exa degradation:** no key → `exa_disabled` recorded; `exa_required` sources quarantine visibly; deterministic-only sweep completes [§14-7] — M2 + WP-05/06.
9. **Ceilings:** exceeding LLM/Exa budgets in fixtures sets warned flags; operations continue; ledgers show breakdown [§14-8] — M2 + WP-13.
10. **Retention:** prune removes only unreferenced, unpinned, aged items; referenced/pinned survive; FIFO warning surfaced [§14-9] — M2 + WP-25.
11. **Client:** all views operate against fixtures; EN↔DE switch; exports render full dossiers; offscreen client tests green [§14-10] — M4.
12. **Immutability:** failure-criteria second write → 409 [§14-11] — WP-18/20/31.
13. **Hermetic suite:** `pytest tests/` fully green (server + client); no live network; guard files green; test-construction rule held [§14-12] — every merge + WP-34.
14. **PDF pipeline:** text-layer PDFs extract; scanned flagged, stored, excluded [§14-13] — WP-11.
15. **Secrets:** no secret material in DB, logs, or API responses under forced paths [§14-14] — guards.
16. **Degrade-to-raw + no invented numbers:** contract violations degrade; run stats flag; economics/scoring inputs provenance-tagged or rejected [§14-15] — WP-14/17 + guards.

**Project complete ⇔** all sixteen hold (structural criterion 1 + Doc 2's fifteen), `main` green, tag `v0.5.0` cut. Any later change follows §2 conventions as defect WPs — never weakening a spec invariant (§12 rule).

---
