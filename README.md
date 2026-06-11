# Research Harness — a Spec Kit extension

**State-externalizing research harness for spec-driven development**: budgeted
exploration, importance-tagged evidence curation, and adversarial claim
verification — all persisted as files, not context.

Based on **Harness-1** — *"Harness-1: Reinforcement Learning for Search Agents
with State-Externalizing Harnesses"* (Jiang et al.,
[arXiv:2606.02373](https://arxiv.org/abs/2606.02373)) and its reference
implementation [pat-jj/harness-1](https://github.com/pat-jj/harness-1).
This community extension adapts the paper's *harness* design to
[Spec Kit](https://github.com/github/spec-kit) workflows. It is not affiliated
with the paper authors or GitHub.

## Why

The research that feeds `/speckit.specify` and `/speckit.plan` — exploring a
codebase, evaluating libraries, checking API behavior — is long-horizon work,
and the agent's conversation context is a terrible place to keep its state:
findings silently fall out of the window, searches get repeated, claims are
written into the plan without ever being checked, and a new session starts
from zero.

Harness-1's diagnosis is that this is a separation-of-concerns failure: the
model is forced to do **bookkeeping** (tracking what was found, what it
supports, what was verified, what is duplicate) with the same machinery it
uses for **semantic decisions** (what to search, what to retain, when to
stop). Its fix is a stateful harness that holds the working memory
*environment-side* — a candidate pool, an importance-tagged curated set,
compact evidence links, verification records, compressed deduplicated
observations — and renders the model only a compact, budget-aware slice.

This extension applies the same split to spec-driven development. Your coding
agent is the policy; a set of per-feature markdown state files is the harness.

| Harness-1 (paper) | This extension |
|---|---|
| Environment-side working memory | `specs/<feature>/harness/` state files |
| Candidate pool | `candidates.md` — deduplicated, append-only IDs |
| Importance-tagged curated set | `curated.md` — capped, critical→low tags, eviction policy |
| Compact evidence links | `evidence.md` — pointers + ≤25-word excerpts, never bulk content |
| Verification records | `verification.md` — verdict, method, confidence per claim |
| Compressed, deduplicated observations | `observations.md` — ≤3-line entries, `dup-of` marking |
| Budget-aware context rendering | `/speckit.harness.status` + slice-only loading in every command |
| Policy decides: search / retain / verify / stop | The agent's only jobs inside `/speckit.harness.explore` |
| Recoverable search state | Resume any session from files via `/speckit.harness.status` |

## Installation

```bash
specify extension add harness --from https://github.com/formin/spec-kit-harness/archive/refs/tags/v1.0.0.zip
```

> The CLI shows an *Untrusted Source* warning for any URL install and asks
> `Continue with installation? [y/N]` — answer `y`. (In a non-interactive
> shell, pipe the answer: `echo y | specify extension add …`.)

Or for development:

```bash
git clone https://github.com/formin/spec-kit-harness
specify extension add --dev ./spec-kit-harness
```

Verify:

```bash
specify extension list
#  ✓ Research Harness (v1.0.0)
#     harness
#     Commands: 5 | Hooks: 2 | Priority: 10 | Status: Enabled
```

Requires Spec Kit `>=0.2.0`. Works with any agent Spec Kit supports (Claude
Code, GitHub Copilot, Cursor, Gemini CLI, …) — commands are plain prompt
files; no external tools, MCP servers, or network access required.

## Commands at a glance

| Command | What it does | Touches disk |
|---|---|---|
| `/speckit.harness.init [mission] [key=value…]` | Create the six state files with budgets and stop conditions | creates `harness/` (never overwrites) |
| `/speckit.harness.explore [question]` | Budgeted decide→act→bookkeep research loop | updates all state files |
| `/speckit.harness.verify [targets]` | Adversarial verification of load-bearing claims | `verification.md`, `evidence.md`, `budget.md` |
| `/speckit.harness.status [full \| topic]` | Compact snapshot + one recommended next action | **read-only** |
| `/speckit.harness.report [scope]` | Synthesize evidence into `research.md` with a coverage table | `research.md` only |

State lives in `specs/<feature>/harness/` (or `.specify/harness/global/` when
no feature directory exists).

## Where it fits in the Spec Kit workflow

The harness does not replace any core stage. It fills the **research gap
between writing a spec and trusting a plan** — and it hands its results to the
core flow through `research.md`, the exact artifact `/speckit.plan`'s
*Phase 0: Outline & Research* generates and consumes.

```text
/speckit.constitution               project principles — no harness involvement
        │
/speckit.specify ──▶ spec.md
        │  └─ hook after_specify → /speckit.harness.init      (optional prompt)
        │
        │  ┌─ RESEARCH PHASE (harness home turf) ──────────────────────┐
        │  │  /speckit.harness.explore   budgeted evidence gathering   │
        │  │  /speckit.harness.verify    check the spec's claims       │
        │  │  /speckit.harness.report ─▶ research.md + coverage table  │
        │  └────────────────────────────────────────────────────────────┘
        │
/speckit.plan ──▶ plan.md           Phase 0 starts from verified research.md
        │                           instead of one-shot, unverified research
        │  └─ hook after_plan → /speckit.harness.verify       (optional prompt)
        │
/speckit.tasks ──▶ tasks.md         gate: /speckit.harness.status shows no
        │                           unverified critical claims before this
/speckit.implement                  explore/status on demand for unknowns
                                    discovered mid-implementation
```

Stage by stage:

| Core stage | Harness command | How it is used |
|---|---|---|
| `/speckit.constitution` | — | Not used. |
| Right after `/speckit.specify` | `init` (the `after_specify` hook offers it) | Creates the per-feature state files; the spec's open questions become the **mission** and get explicit budgets. |
| **Between specify and plan** | `explore` → `verify` → `report` | The main pass: gather evidence within budget, adversarially verify the spec's load-bearing claims, then write `research.md` with a requirement-coverage table. This is the deep, resumable replacement for plan's ad-hoc Phase 0 research. |
| Right after `/speckit.plan` | `verify` (the `after_plan` hook offers it) | Re-checks the *plan's* factual claims against primary sources. Refuted claims come back as suggested edits — apply them via `/speckit.clarify` or by hand before they harden into tasks. |
| Before `/speckit.tasks` | `status` | Go/no-go gate: the snapshot warns if any `critical` claim is still unverified or contradicted. |
| During `/speckit.implement` | `explore` / `status` as needed | Budget-boxed investigation of unknowns discovered mid-implementation; a fresh session resumes from files, not from a lost context window. |
| Any time | `status` | Read-only snapshot + exactly one recommended next action; the session-resume entry point. |

Two rules keep the integration safe: the harness **never edits**
`spec.md`, `plan.md`, or `tasks.md` (corrections always flow back as suggested
edits), and the only core artifact it writes is `research.md` — between
`<!-- harness:begin/end -->` markers, preserving anything you wrote by hand.

## Usage

### 1. `/speckit.harness.init` — set up the harness

Run once per feature, ideally right after `/speckit.specify` (the
`after_specify` hook offers to do this for you).

```text
/speckit.harness.init How is session state currently handled, and what are the revocation options?
```

- The free text becomes the **mission** — the question this harness exists to
  answer. You can add budget overrides inline: `searches=50 inspections=60`.
- Creates `budget.md`, `candidates.md`, `curated.md`, `evidence.md`,
  `verification.md`, `observations.md` under the feature's `harness/`
  directory, with budgets from your config (defaults: 30 searches,
  40 inspections, 20 verifications, curated cap 25).
- **Idempotent**: if a harness already exists it refuses to overwrite and
  shows status instead; a new mission passed as argument is appended to the
  mission list rather than replacing it.

### 2. `/speckit.harness.explore` — budgeted research loop

```text
/speckit.harness.explore What auth middleware exists and which routes bypass it?
```

(With no argument it uses the mission from `budget.md`.)

Each iteration the agent makes exactly one **policy decision** — `SEARCH` a
new query, `INSPECT` a known candidate, `CURATE` the curated set, or `STOP` —
then performs mandatory bookkeeping: log a ≤3-line compressed observation,
add deduplicated candidates, promote findings into the curated set with an
importance tag plus an evidence pointer, and account for the budget. The loop
stops on budget exhaustion, on the marginal-gain rule (3 consecutive actions
yielding no new curated evidence), or when the mission is answered.

After a session, `budget.md` carries an auditable ledger like:

```markdown
| Resource | Budget | Spent | Remaining |
|----------|-------:|------:|----------:|
| searches | 30 | 1 | 29 |
| inspections | 40 | 1 | 39 |

## Action log
| # | Action | Target | Cost | New evidence? |
|---|--------|--------|------|---------------|
| 1 | SEARCH | .specify tree + templates listing | 1 search | yes (C001, C002 → E001, E002) |
| 2 | INSPECT | .specify/templates/spec-template.md | 1 inspection | yes (E002 confirmed) |
```

and `curated.md` holds the working set, most important first:

```markdown
| ID | Importance | Finding | Source candidate | Evidence |
|----|------------|---------|------------------|----------|
| E003 | critical | Templates ship bundled inside the specify-cli package; init needs no network access. | C003 | evidence.md#E003 |
| E001 | high | init scaffolds .specify/{templates,scripts,memory,…} and copies five artifact templates. | C001 | evidence.md#E001 |
```

### 3. `/speckit.harness.verify` — check the claims before they harden

```text
/speckit.harness.verify            # default: spec.md + plan.md + unverified critical curated entries
/speckit.harness.verify plan.md    # narrow to one artifact
```

Extracts **load-bearing factual claims** ("X is handled by Y", "library Z
supports W", "there is no existing V") and, for each one, *tries to refute it*
against the primary source — never the curated summary. Every check leaves a
durable row:

```markdown
| ID | Claim | Method | Verdict | Confidence | Evidence | Date |
|----|-------|--------|---------|------------|----------|------|
| V001 | Templates are bundled in specify-cli; init performs no network fetch | Re-read `specify init --help`; cross-checked offline scaffold | verified | high | E001 | 2026-06-11 |
```

Refuted claims are demoted in `curated.md` (marked, never deleted — recorded
dead ends prevent re-deriving the same error) and reported back as concrete
suggested edits to `spec.md`/`plan.md`. The command does **not** edit your
artifacts itself.

### 4. `/speckit.harness.status` — resume, or decide what's next

```text
/speckit.harness.status            # compact snapshot
/speckit.harness.status full       # 3× larger slices
/speckit.harness.status sessions   # filter rows to a topic
```

Read-only and budget-free. Renders the Harness-1 "budget-aware context
rendering": mission, remaining budgets, top curated entries, the open
candidate frontier, refuted/unverified-critical warnings, recent observations
— and closes with exactly **one recommended next action** derived from the
state, e.g.:

```text
Recommendation: /speckit.harness.verify — 2 critical claims unverified and
12 verification budget remaining → verify before planning.
```

This is also the **session-resume entry point**: open a fresh agent session,
run `/speckit.harness.status`, and continue exactly where research stopped —
nothing depended on the old context window.

### 5. `/speckit.harness.report` — feed the results back into Spec Kit

```text
/speckit.harness.report
```

Reads the full state (the only command that does), maps every requirement in
`spec.md` to its supporting evidence, and writes the feature's `research.md`
between `<!-- harness:begin/end -->` markers (hand-written sections outside
the markers are preserved). The coverage table makes evidence gaps visible
before `/speckit.plan` consumes the research:

```markdown
### Requirement Coverage — 7/10
| Requirement | Status | Evidence | Verification |
|-------------|--------|----------|--------------|
| FR-001 Token revocation | covered-verified | E003, E007 | V002 (high) |
| FR-004 Admin audit log | covered-unverified | E011 | — |
| FR-006 SSO logout | uncovered | — | — |
```

Statuses: `covered-verified` / `covered-unverified` / `contradicted` /
`uncovered`. Contradictions and uncovered requirements come with suggested
follow-ups (fix the artifact, explore more, or carry the risk explicitly).

## Worked example — end to end

Researching a *session-revocation* feature:

```text
/speckit.specify Build a session-revocation feature for the admin console
   → specs/001-session-revocation/spec.md          (core Spec Kit)
   → hook prompt: "Set up a state-externalized research harness?" → yes

/speckit.harness.init How is session state stored today, and what can invalidate it?
   → specs/001-session-revocation/harness/{budget,candidates,curated,evidence,verification,observations}.md

/speckit.harness.explore
   → SEARCH "session" across src/ ............ 9 candidates (C001–C009)
   → INSPECT C002 src/auth/session.ts ........ curated E001 (critical): "JWTs validated only at the gateway"
   → INSPECT C005 src/cache/redis.ts ......... curated E002 (high): "session cache TTL 24h, no revocation list"
   → SEARCH "revoke OR invalidate" ........... no new candidates (dup-of O-003)
   → STOP: marginal gain exhausted
   → budget: searches 27/30 left · inspections 37/40 left

/speckit.harness.verify
   → V001 "JWTs validated only at gateway" .... verified (high) — re-read gateway middleware
   → V002 "no revocation list exists" ......... refuted — found legacy denylist in src/auth/legacy.ts
   → curated E00x demoted to refuted; suggested spec edit reported

/speckit.harness.status
   → 1 refuted claim propagated, 0 critical unverified, budget healthy
   → Recommendation: /speckit.harness.report

/speckit.harness.report
   → specs/001-session-revocation/research.md — coverage 4/5, FR-003 uncovered (flagged)

/speckit.plan
   → the plan now cites verified evidence and inherits one explicit known-unknown
```

If the session dies anywhere in the middle: open a new one and run
`/speckit.harness.status`. The files are the memory.

## State files

| File | Role | Invariants |
|---|---|---|
| `budget.md` | Mission, budget ledger, stop conditions, action log | every budgeted action accounted |
| `candidates.md` | Everything discovered | dedup by source+topic; statuses `new/inspected/curated/discarded` |
| `curated.md` | What matters | hard cap; importance tags; refuted entries marked, not deleted |
| `evidence.md` | Where proof lives | pointers + locators; excerpts ≤ 25 words |
| `verification.md` | What was checked | verdict + method + confidence; primary sources only |
| `observations.md` | What happened | append-only; ≤3 lines each; duplicates flagged |

The files are ordinary markdown in your repo: diffable, reviewable in PRs, and
shared by every agent and teammate working on the feature.

## Patterns & recipes

**Light vs. deep research** — budgets are per-session levers:

```text
/speckit.harness.init quick sanity check on the websocket layer searches=8 inspections=10 verifications=5
/speckit.harness.init full audit of the billing pipeline searches=60 inspections=80 verifications=40
```

**Several questions, one harness** — running `init` again with a new question
appends it as mission #2 instead of clobbering state; `report` covers all
missions.

**Verify-before-plan gate** — accept the `after_plan` hook (or run
`/speckit.harness.verify` manually) so every load-bearing claim in `plan.md`
has a verdict before `/speckit.tasks` turns it into work items.

**Team workflow** — commit `harness/` with the feature branch. Reviewers see
*why* the plan believes what it believes (evidence pointers + verdicts), and
a teammate's agent can pick up the research mid-flight via
`/speckit.harness.status`.

**Token discipline** — every command loads slices, never full files, within
the `context_tokens` render cap. Long-horizon research stops scaling with
conversation length and starts scaling with file size — which is effectively
unlimited.

## Hooks

Both optional (you are prompted):

- `after_specify` → `speckit.harness.init` — set up the harness when a spec is created.
- `after_plan` → `speckit.harness.verify` — verify the plan's claims before they harden into tasks.

## Configuration

Copy `config-template.yml` to
`.specify/extensions/harness/harness-config.yml` and adjust budgets, the
curated-set cap, slice sizes, state location, and stop conditions:

```yaml
budget:
  searches: 30
  inspections: 40
  verifications: 20
  context_tokens: 4000
curation:
  max_curated: 25
  evict_policy: lowest-importance-first
rendering:
  candidates_slice: 10
  curated_slice: 15
  observations_slice: 8
```

Precedence (lowest → highest): extension defaults → config file →
`SPECKIT_HARNESS_*` environment variables → per-invocation `key=value`
arguments to `init`.

## Troubleshooting & FAQ

**`init` says the harness already exists.** By design — it never overwrites.
To start over, delete the feature's `harness/` directory yourself; to add a
question, pass it to `init` and it is appended as a new mission.

**No `specs/` feature directory yet?** Commands fall back to
`.specify/harness/global/`. State written there stays useful for any feature;
`report` writes `research.md` next to it.

**Exploration keeps stopping "marginal gain exhausted".** That is the
Harness-1 stop rule working: 3 consecutive actions added nothing new to the
curated set. Sharpen the question (`/speckit.harness.explore <narrower question>`)
or raise the window in `stop_conditions.marginal_gain_window`.

**A claim came back `refuted` — now what?** The verify report includes the
suggested artifact edit. Apply it (or run `/speckit.clarify`), keep the
verification record as-is; the recorded dead end is what stops the error from
coming back.

**Does it call any external services?** No. The commands are prompt files;
all "infrastructure" is markdown in your repo. The only network access in the
whole lifecycle is your own `specify extension add --from <url>` download.

**Install prompt blocks my CI.** `specify extension add --from <url>` asks
`Continue with installation? [y/N]` on untrusted URLs; pipe the answer in
non-interactive shells: `echo y | specify extension add harness --from <url>`.

See [docs/concepts.md](docs/concepts.md) for the full design mapping and the
deliberate differences from the paper.

## License

[MIT](LICENSE) © 2026 formin

Credits: Harness-1 by Pengcheng Jiang et al. ([arXiv:2606.02373](https://arxiv.org/abs/2606.02373),
[pat-jj/harness-1](https://github.com/pat-jj/harness-1)); [Spec Kit](https://github.com/github/spec-kit) by GitHub.
