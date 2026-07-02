# spec-kit-harness quickstart

This repository is **Research Harness**, a [Spec Kit](https://github.com/github/spec-kit) community extension (extension id `harness`, version 1.0.0, MIT, author formin). It distributes **5 prompt-file commands and 2 optional hooks — no executable code**: every command is a markdown prompt file under `commands/` that the user's coding agent (Claude Code, Copilot, Cursor, Gemini CLI, …) interprets when the corresponding `/speckit.harness.*` slash command is invoked. The extension adapts the state-externalizing harness design of Harness-1 (Jiang et al., [arXiv:2606.02373](https://arxiv.org/abs/2606.02373), reference code [pat-jj/harness-1](https://github.com/pat-jj/harness-1)) to spec-driven development.

## What this repository does

- Provides five slash commands: `/speckit.harness.init`, `/speckit.harness.explore`, `/speckit.harness.verify`, `/speckit.harness.status`, `/speckit.harness.report`.
- Externalizes research state into six markdown files per feature (`budget.md`, `candidates.md`, `curated.md`, `evidence.md`, `verification.md`, `observations.md`) under `specs/<feature>/harness/`, so research survives context compaction, session restarts, and agent handoffs.
- Enforces budgeted exploration (searches / inspections / verifications), importance-tagged evidence curation with a hard cap, adversarial claim verification with durable verdicts, and a marginal-gain stop rule.
- Bridges results back into the core Spec Kit flow by writing `research.md` with a requirement-coverage table — the only core artifact the harness ever writes.
- Declares two optional hooks in `extension.yml`: `after_specify` → `speckit.harness.init` and `after_plan` → `speckit.harness.verify` (both prompt the user first).
- Ships a `config-template.yml` for budgets, curation caps, rendering slice sizes, state location, and stop conditions.

## Start here

1. Read [architecture/overview.md](architecture/overview.md) to understand the manifest, the prompt-file command mechanism, and the six state files.
2. Read [commands/reference.md](commands/reference.md) for what each of the five commands reads, writes, and guarantees.
3. Read [operations/installation-and-config.md](operations/installation-and-config.md) to install the extension, configure it, and cut releases.

## Documentation map

- [quickstart.md](quickstart.md) — this page: what the repository is and where to go next.
- [architecture/overview.md](architecture/overview.md) — extension structure: manifest schema, prompt-file commands, state files, the Harness-1 policy/bookkeeping split, hooks, config precedence.
- [commands/reference.md](commands/reference.md) — per-command reference: purpose, arguments, files touched, key invariants.
- [operations/installation-and-config.md](operations/installation-and-config.md) — installation paths, configuration options, release procedure, and the OpenWiki refresh workflow.

## Key source files

- [`extension.yml`](../extension.yml) — the Spec Kit extension manifest: identity, the 5 command registrations, the config template entry, the 2 hooks, and default budgets.
- [`commands/speckit.harness.init.md`](../commands/speckit.harness.init.md) — prompt that creates the six state files (idempotent).
- [`commands/speckit.harness.explore.md`](../commands/speckit.harness.explore.md) — prompt for the budget-aware decide→act→bookkeep exploration loop.
- [`commands/speckit.harness.verify.md`](../commands/speckit.harness.verify.md) — prompt for adversarial claim verification against primary sources.
- [`commands/speckit.harness.status.md`](../commands/speckit.harness.status.md) — prompt for the read-only, budget-aware state snapshot and next-action recommendation.
- [`commands/speckit.harness.report.md`](../commands/speckit.harness.report.md) — prompt that synthesizes state into `research.md` with a coverage table.
- [`config-template.yml`](../config-template.yml) — user-copyable configuration template (all keys optional).
- [`docs/concepts.md`](../docs/concepts.md) — design document mapping each Harness-1 mechanism onto Spec Kit, with deliberate differences from the paper.
- [`README.md`](../README.md) — full user-facing documentation including an end-to-end tutorial.
- [`CHANGELOG.md`](../CHANGELOG.md) — Keep a Changelog / SemVer history (currently 1.0.0, 2026-06-11).
