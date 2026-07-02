# Command reference

All five commands are prompt files in [`commands/`](../../commands/), registered in [`extension.yml`](../../extension.yml). Each resolves `HARNESS_DIR` the same way: config file → `SPECKIT_HARNESS_*` env vars → arguments; feature directory from the current git branch (e.g. branch `003-user-auth` → `specs/003-user-auth/`), else the most recently modified directory under `specs/`, else the configured fallback (default `.specify/harness/global/`).

## `/speckit.harness.init`

- **Purpose**: create the externalized working memory — the six state files — for the active feature.
- **Arguments**: optional free text = the research **mission**; optional `key=value` budget overrides (e.g. `searches=50 inspections=60`). Both may appear together.
- **Reads**: `.specify/extensions/harness/harness-config.yml` (if present). Deliberately does **not** read `spec.md`/`plan.md`.
- **Writes**: creates `HARNESS_DIR/` with `budget.md`, `candidates.md`, `curated.md`, `evidence.md`, `verification.md`, `observations.md`, each written exactly once with resolved config values.
- **Invariants**: **idempotent** — if `budget.md` already exists it overwrites nothing, reports the harness as initialized, and behaves like `/speckit.harness.status`; a new mission in the arguments is *appended* as an additional numbered mission in `budget.md`. Never deletes or truncates an existing harness; never copies spec/plan content into state files (links only).

## `/speckit.harness.explore`

- **Purpose**: budget-aware exploration loop with strict policy/bookkeeping separation: each iteration is one policy decision (`SEARCH` / `INSPECT` / `CURATE` / `STOP`) followed by mandatory bookkeeping.
- **Arguments**: the research question for the session; if empty, uses the mission from `budget.md`; if both empty, asks the user and stops.
- **Reads**: slices only — budget table + last 5 action-log rows, top `curated_slice` entries, up to `candidates_slice` open candidates, last `observations_slice` observations.
- **Writes**: all six state files — appends observations, adds/updates candidate rows, promotes findings into `curated.md` + pointer entries in `evidence.md`, and accounts every budgeted action in `budget.md` (`SEARCH` → searches, `INSPECT` → inspections; `CURATE` costs no budget).
- **Invariants**: one action per iteration (no batching to dodge the ledger); evidence excerpts ≤ 25 words; stops on budget exhaustion, marginal gain (default 3 no-yield actions), mission answered, or user interrupt; never edits `spec.md`/`plan.md`/`tasks.md`; if state files are missing or corrupt, stops and directs the user to `init` rather than recreating state mid-loop.

## `/speckit.harness.verify`

- **Purpose**: adversarially verify load-bearing factual claims — attempt to *refute* each claim against the primary source, never the curated summary.
- **Arguments**: optional target artifacts (`plan.md`, `spec.md`, `curated`) and/or specific claims. Default targets: the feature's `spec.md` and `plan.md` plus all unverified `critical` curated entries.
- **Reads**: target artifacts, `budget.md` (verification budget), existing `verification.md`, primary sources.
- **Writes**: `verification.md` (one row per checked claim), `evidence.md` (pointer add/update), `budget.md` (decrement + action log), and `curated.md` only to mark refuted entries `refuted (see V-xxx)` — never deleting them.
- **Invariants**: requires an initialized harness (else stop and point to `init`); one primary-source check per budget unit; never `verified` at `low` confidence; does **not** edit `spec.md`/`plan.md` — refutations are reported as suggested edits.

## `/speckit.harness.status`

- **Purpose**: render a compact, budget-aware snapshot of the harness state plus exactly one recommended next action; the session-resume entry point.
- **Arguments**: optional — `full` renders 3× the configured slice sizes; any other string filters curated/candidate rows by topic.
- **Reads**: slices of `budget.md`, `curated.md`, `candidates.md`, `verification.md` (all `refuted` rows + counts + unverified criticals), `observations.md`. Does not read `evidence.md` bodies or spec/plan files.
- **Writes**: nothing. **Read-only** — no file writes, no budget changes, not even timestamp updates; consumes no budget.
- **Invariants**: slices, never full files (counts stand in for what is not shown); output stays within the `context_tokens` cap, truncating lowest-importance material first; if a harness does not exist it points to `init` and creates nothing; the recommendation must follow from the rendered state alone.

## `/speckit.harness.report`

- **Purpose**: synthesize the curated set, evidence links, and verification records into the feature's `research.md`, with a requirement-coverage table (`covered-verified` / `covered-unverified` / `contradicted` / `uncovered` and a coverage score such as "7/10 requirements verified").
- **Arguments**: optional scope filter (only matching topics) or an alternative output path. Default output: `FEATURE_DIR/research.md`.
- **Reads**: the **full** state files (the only command that does) plus `spec.md` if present.
- **Writes**: `research.md` only — and only the block between `<!-- harness:begin -->` and `<!-- harness:end -->` (markers appended if absent), preserving hand-written sections outside the markers.
- **Invariants**: never invents evidence — every coverage row cites real `evidence.md` IDs, and unsupported requirements are listed as `uncovered`, not omitted; suggested corrections to `spec.md`/`plan.md` are reported, not applied; never edits the harness state files themselves.
