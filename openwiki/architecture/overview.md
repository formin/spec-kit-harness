# Architecture overview

Research Harness contains no executable code. It is a set of declarative files that Spec Kit installs into a user's project; all behavior is carried out by the user's own coding agent following the prompt files.

## Extension manifest (`extension.yml`)

The manifest uses `schema_version: "1.0"` and has five top-level sections:

- `extension` — identity: `id: harness`, `name: "Research Harness"`, `version: "1.0.0"`, `category: "process"`, `effect: "read-write"`, author/repository/license/homepage.
- `requires` — `speckit_version: ">=0.2.0"`.
- `provides.commands` — the five command registrations, each mapping a command name (e.g. `speckit.harness.init`) to a prompt file under `commands/` plus a one-line description.
- `provides.config` — one optional config file, `harness-config.yml`, generated from `config-template.yml`.
- `hooks`, `tags`, `defaults` — see below.

## Command mechanism: prompt files, not code

Each entry in `commands/` is a markdown file with a YAML frontmatter `description`. When the user runs `/speckit.harness.<name>`, their coding agent executes the file's instructions as a prompt: a `$ARGUMENTS` placeholder receives user input, and sections such as Steps and Guardrails direct the agent's behavior. Nothing in this repository runs on its own — the agent is the interpreter. `docs/concepts.md` calls this out as the honest caveat versus the paper: bookkeeping is enforced by instruction rather than by construction, mitigated by making the rules mechanical and drift visible in file diffs (ledger rows, append-only IDs, `dup-of` markers).

## The six state files

`/speckit.harness.init` creates six markdown files in `HARNESS_DIR` (default `specs/<feature>/harness/`, fallback `.specify/harness/global/` when no feature directory exists):

- `budget.md` — mission list, budget ledger table (searches / inspections / verifications, Budget/Spent/Remaining), stop conditions, and an append-only action log.
- `candidates.md` — the candidate pool: everything discovered, deduplicated by source + topic, append-only IDs (`C001`…), status lifecycle `new → inspected → curated:<E-id> | discarded(<reason>)`.
- `curated.md` — the importance-tagged curated set: hard cap (default 25), importance `critical | high | medium | low`, findings ≤ 2 sentences, eviction `lowest-importance-first`, refuted entries marked rather than deleted.
- `evidence.md` — compact evidence links: pointers only (source, locator, ≤ 25-word excerpt, what it supports); IDs (`E001`…) match `curated.md`.
- `verification.md` — verification records: one row per checked claim with method, verdict (`verified | refuted | unverifiable`), confidence, evidence ID, date.
- `observations.md` — compressed, deduplicated observation log: append-only, ≤ 3 lines per entry, `dup-of O-xxx` marking, never raw tool output.

`research.md` is not a state file: it is written later by `/speckit.harness.report` into the feature directory as the bridge to `/speckit.plan`.

## The Harness-1 policy/bookkeeping split

The design separates two jobs (see `docs/concepts.md`):

- **Policy (the agent's reasoning)** makes only semantic decisions. In `/speckit.harness.explore` this is narrowed to four verbs: `SEARCH <query>`, `INSPECT <candidate-id>`, `CURATE`, `STOP <reason>`.
- **Harness (state files + mechanical bookkeeping rules)** owns everything else: deduplication, observation compression, curated-set eviction, and budget accounting, performed every iteration exactly as specified.

Stop rules combine budget exhaustion, a marginal-gain window (default 3 consecutive budgeted actions with no new curated evidence), and mission coverage with all `critical` claims verified. Every command loads *slices* of state, never full files, within a `context_tokens` render cap (default 4000) — `/speckit.harness.status` exposes this budget-aware rendering as a command and doubles as the session-resume entry point. The one exception is `/speckit.harness.report`, which reads the full state files.

Two integration rules keep the harness safe inside Spec Kit: it never edits `spec.md`, `plan.md`, or `tasks.md` (corrections flow back as suggested edits), and its only write to a core artifact is `research.md`, between `<!-- harness:begin -->` / `<!-- harness:end -->` markers.

## Hooks

Declared in `extension.yml`, both optional (`optional: true` with a user-facing `prompt`):

- `after_specify` → runs `speckit.harness.init` ("Set up a state-externalized research harness for this feature?").
- `after_plan` → runs `speckit.harness.verify` ("Verify plan claims against primary sources and the harness evidence base?").

There are no hook scripts — hooks are manifest entries that trigger the named prompt-file commands.

## Configuration precedence

Lowest to highest:

1. Extension defaults — the `defaults` block in `extension.yml` (budgets 30/40/20, `context_tokens: 4000`, `max_curated: 25`, slices 10/15/8).
2. Config file — `.specify/extensions/harness/harness-config.yml`, copied from `config-template.yml`; every key optional.
3. Environment variables — `SPECKIT_HARNESS_*` prefix (e.g. `SPECKIT_HARNESS_BUDGET_SEARCHES=50`).
4. Per-invocation `key=value` arguments to `/speckit.harness.init` (e.g. `searches=50 inspections=60`).

The config file additionally controls `state.directory` / `state.fallback_directory` and `stop_conditions` (`marginal_gain_window`, `require_critical_verified`).
