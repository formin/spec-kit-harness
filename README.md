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

## What you gain

One thing to be precise about up front: the harness does **not** make your
agent's conversation context persist — nothing can, and every agent starts a
new session empty. What it does is make the conversation **disposable**:
everything worth keeping is written to files at the moment it is learned, and
`/speckit.harness.status` re-renders the working picture into any new session
in one step. State survives; context is rebuilt on demand.

What that buys you, concretely:

| With the harness | Without it — research state lives in the conversation |
|---|---|
| **Session immortality** — resume after a restart, compaction, window overflow, or crash with one `status` call | whatever fell out of the context window is gone for good |
| **Token efficiency** — each step re-renders a bounded slice (default cap: 4,000 tokens), never the full history | continuing means re-reading an ever-growing transcript |
| **No repeated searches** — the candidate pool dedups by source + topic; repeats are flagged `dup-of` | the same query gets re-run days later because nothing remembers it ever ran |
| **Claims with verdicts** — every load-bearing claim carries `verified` / `refuted` / `unverifiable`, plus method and confidence | "I checked that somewhere" hardens into the plan unexamined |
| **Remembered dead ends** — refuted claims stay on file, marked, never deleted | the same wrong conclusion gets re-derived in the next session |
| **Bounded research** — explicit budgets and a marginal-gain stop rule decide when exploration ends | research ends when the agent drifts or the context fills up |
| **Auditable, shareable evidence** — plain markdown: diffable, PR-reviewable, and a teammate (or their agent) can take over mid-research | the evidence behind a plan exists only inside one person's expired chat |

The tutorial's [step 6](#6-speckitimplement--boxing-the-mid-build-unknown)
shows the resume scene in action.

## Plain Spec Kit vs. Spec Kit + Research Harness

First, the honest framing: **this is not an alternative to Spec Kit — it is an
extension that plugs into it.** Spec Kit already does research — `/speckit.plan`
opens with *Phase 0: Outline & Research*, which prompts the agent to fan out
into parallel research tasks and write the result to `research.md`. The harness
does not remove that step; it makes the same step **budgeted, verified, and
resumable**. Everything below compares *the research phase of a plain Spec Kit
workflow* against *that same phase with the harness installed* — the rest of
Spec Kit (`constitution`, `specify`, `tasks`, `implement`) is untouched.

| Research dimension | Plain Spec Kit (`/speckit.plan` Phase 0) | + Research Harness |
|---|---|---|
| Where the working state lives | Inside the planning agent's conversation context | Externalized to `specs/<feature>/harness/*.md` |
| After the session ends | Gone on restart, compaction, or window overflow | Survives in files; `/speckit.harness.status` rebuilds it in one step |
| How far it explores | Unbounded prompt; stops when context fills or the agent drifts | Explicit budgets (searches / inspections / verifications) + a marginal-gain stop rule |
| Repeated searches | Nothing remembers what was already queried | Candidate pool dedups by source + topic; repeats flagged `dup-of` |
| Claim checking | Findings are written into `research.md` as the agent stated them | Adversarial pass *tries to refute* each load-bearing claim → `verified` / `refuted` / `unverifiable` + method + confidence |
| Dead ends | Re-derived from scratch in the next session | Refuted claims kept on file, marked — never re-investigated blind |
| Output | `research.md`, free-form | The same `research.md` — **plus a requirement-coverage table** (`covered-verified` … `uncovered`) |

(Some rows echo [What you gain](#what-you-gain) — that section frames these as
benefits in the abstract; here each one is pinned to the exact Spec Kit step it
changes.)

### The same question, two ways

Say the spec hinges on one assumption: *can we ship `better-sqlite3`, and will
it run on our deploy target?*

**Plain Spec Kit** — Phase 0 researches inline and records the conclusion:

```text
research.md
  Decision: use better-sqlite3 (fast, synchronous SQLite binding).
```

Plausible, and probably right — but nobody checked the part that actually bites
(native compilation on the deploy image), the search trail evaporates when the
session ends, and the next session starts over.

**Spec Kit + Harness** — the same research, externalized and checked:

```text
/speckit.harness.explore
  C001 better-sqlite3 README   C002 prebuild-install notes
  → curated E001 (critical): "ships prebuilt binaries for common platforms;
    falls back to compiling via node-gyp when none match"

/speckit.harness.verify
  V001 "no compiler needed on linux/x64 + glibc"  → verified  (high)
  V002 "works as-is on Alpine/musl"               → refuted   (high)
        → needs build-base to compile; suggested edit to spec.md

/speckit.harness.report
  research.md + coverage row:
  FR-004 persistence → covered-verified (E001 / V001)
```

```text
# next morning, brand-new session
/speckit.harness.status
  → mission answered · 1 refuted assumption on record (Alpine) · budgets healthy
```

Same decision — but now the load-bearing part carries a verdict, the Alpine
dead end is **remembered** instead of re-hit during `/speckit.implement`, and
the whole trail survives into a fresh session as plain, PR-reviewable markdown.

### Bottom line

The harness is a **rigor layer for the research phase**. For a throwaway
script, plain Spec Kit's Phase 0 is fine. For anything where a wrong assumption
hardens into a plan — an unfamiliar library, a load-bearing "there is no
existing X", or research too long to fit one context window — the budget, the
verification verdicts, and the resumable state are the difference between *a
plausible sentence* and *a checked one*. See [Where it fits in the Spec Kit
workflow](#where-it-fits-in-the-spec-kit-workflow) for which commands run where.

## Installation

Research Harness is listed in the
[Spec Kit community extension catalog](https://github.com/github/spec-kit/blob/main/extensions/catalog.community.json).

**Option 1 — by name, from the community catalog.** Spec Kit treats the
community catalog as discovery-only by default, so allow installs from it once
(per project, or per user via `~/.specify/extension-catalogs.yml`):

```yaml
# .specify/extension-catalogs.yml
catalogs:
  - name: default
    url: https://raw.githubusercontent.com/github/spec-kit/main/extensions/catalog.json
    priority: 1
    install_allowed: true
  - name: community
    url: https://raw.githubusercontent.com/github/spec-kit/main/extensions/catalog.community.json
    priority: 2
    install_allowed: true
```

```bash
specify extension add harness
```

**Option 2 — zero config, pinned version.** Install straight from a release URL:

```bash
specify extension add harness --from https://github.com/formin/spec-kit-harness/archive/refs/tags/v1.0.0.zip
```

> URL installs show an *Untrusted Source* warning and ask
> `Continue with installation? [y/N]` — answer `y` (in a non-interactive
> shell, pipe it: `echo y | specify extension add …`). Catalog installs skip
> this prompt.

**Option 3 — for development:**

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
        │       └─ creates specs/<feature>/harness/{budget, candidates,
        │          curated, evidence, verification, observations}.md
        │
        │  ┌─ RESEARCH PHASE (harness home turf) ──────────────────────┐
        │  │  /speckit.harness.explore   budgeted evidence gathering   │
        │  │    ↳ appends: candidates · curated · evidence ·           │
        │  │      observations · budget ledger (every iteration)       │
        │  │  /speckit.harness.verify    check the spec's claims       │
        │  │    ↳ appends: verification records (+ evidence, ledger)   │
        │  │  /speckit.harness.report ─▶ research.md + coverage table  │
        │  │    ↳ the only harness write to a core artifact            │
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

And when each harness file comes into existence, and who touches it afterward:

| File | Created at | Written during | Consumed by |
|---|---|---|---|
| `budget.md` | `init` — mission, budgets, stop conditions | every `explore`/`verify` action (ledger + action log row) | every command |
| `candidates.md` | `init` — empty pool | `explore`: one row per discovery, deduplicated | `explore`, `status` |
| `curated.md` | `init` — empty set | `explore` (promotions, evictions); `verify` (demotes refuted entries) | every command, `report` |
| `evidence.md` | `init` — empty | `explore` and `verify`: pointer entries | `verify`, `report` |
| `verification.md` | `init` — empty | `verify`: one row per checked claim | `status` (gate), `report` |
| `observations.md` | `init` — empty log | every `explore`/`verify` action: ≤3-line compressed entry | `explore`, `status` |
| `research.md` | **`report`** — the last harness step before `/speckit.plan` | re-running `report` (only between the `harness:begin/end` markers) | `/speckit.plan` Phase 0 |

All six state files exist from `init` onward — created once and never
clobbered (`init` is idempotent). `status` writes nothing, ever. `research.md`
is the one file born later, at `report` time — deliberately right before
planning consumes it.

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

## Tutorial — a simple todo app, end to end

This walks **every core Spec Kit stage** and shows exactly where the harness
is used in each one. The app: a minimal single-user web todo list — add,
complete, delete; todos must survive a restart.

### 0. One-time setup

```bash
specify init todo-app --integration claude     # or your agent
cd todo-app
specify extension add harness                  # see Installation for the one-time catalog opt-in
```

### 1. `/speckit.constitution` — principles (no harness yet)

```text
/speckit.constitution Keep it simple: smallest stack that works, test-first,
and no claim enters a plan without a checked source.
```

The harness is not involved at this stage — but that last principle is
precisely what it operationalizes from stage 3 onward.

### 2. `/speckit.specify` — spec, then harness init via the hook

```text
/speckit.specify Single-user web todo app: add, complete, delete; todos must
survive an app restart; keyboard-friendly UI.
```

→ `specs/001-todo-app/spec.md` with FR-001..FR-005 and two open questions
(persistence mechanism? UI stack?). The `after_specify` hook fires:

```text
Set up a state-externalized research harness for this feature? → yes

/speckit.harness.init Which persistence option and minimal web stack satisfy
the constitution? searches=15 inspections=20 verifications=8
```

→ `specs/001-todo-app/harness/` is created, and `budget.md` now reads:

```markdown
## Mission
1. Which persistence option and minimal web stack satisfy the constitution?

| Resource | Budget | Spent | Remaining |
| searches | 15 | 0 | 15 |
```

### 3. Between specify and plan — explore → verify → report

The harness's home turf: answer the spec's open questions *before* planning.

```text
/speckit.harness.explore
```

A few iterations, each externalized to the state files as it happens:

```text
SEARCH  "sqlite vs json file vs localStorage, single-user persistence"
        → C001..C004 added to candidates.md
INSPECT C002 better-sqlite3 docs
        → curated E001 (high): "zero-config embedded DB; sync API fits a tiny app"
INSPECT C004 MDN localStorage
        → curated E002 (critical): "per-browser storage — todos would not survive
          switching browsers; conflicts with FR-004 as written"
SEARCH  "minimal node static file serving"   → dup-of O-002, nothing new
STOP    marginal gain exhausted · budget left: searches 12, inspections 17
```

Now verify the spec's load-bearing assumptions before they harden:

```text
/speckit.harness.verify
```

```markdown
| ID | Claim | Verdict | Confidence |
| V001 | A SQLite file survives app restart (FR-004) | verified | high |
| V002 | "Saving a JSON file is atomic" (spec note)  | refuted — partial-write risk; use write-then-rename | high |
```

The refuted claim comes back as a **suggested edit** — apply it via
`/speckit.clarify` or by hand. Then bridge the results into the core flow:

```text
/speckit.harness.report
```

→ `specs/001-todo-app/research.md`:

```markdown
### Requirement Coverage — 5/5
| Requirement | Status | Evidence | Verification |
| FR-004 persist across restart | covered-verified | E001 | V001 (high) |
```

### 4. `/speckit.plan` — planning from verified research

```text
/speckit.plan SQLite via better-sqlite3, small Express server, vanilla JS frontend
```

Plan's *Phase 0: Outline & Research* finds `research.md` already populated
with verified, cited findings instead of re-deriving everything in one shot.
When the plan lands, the `after_plan` hook fires:

```text
Verify plan claims against primary sources? → yes

/speckit.harness.verify plan.md
→ V003 "express.static serves the frontend with zero config" — verified (high)
```

### 5. `/speckit.tasks` — gated by status

```text
/speckit.harness.status
→ 0 unverified critical claims · 1 corrected assumption (V002) already applied
→ Recommendation: proceed to /speckit.tasks

/speckit.tasks
```

### 6. `/speckit.implement` — boxing the mid-build unknown

```text
/speckit.implement
```

Halfway through, a real unknown appears: *should completed todos be deleted
or archived, given the keyboard-undo flow?* Box it instead of guessing:

```text
/speckit.harness.explore Does undo require archived todos, or is hard delete enough? searches=5 inspections=5
→ E007 (medium): FR-003's undo wording implies soft-delete · 2 actions spent
```

Next morning you open a **brand-new session**. The previous *conversation* is
gone — that is true of every agent, with or without this extension. The
difference: the harness never kept the research state in the conversation in
the first place, so nothing of value was lost. One command re-renders the full
working picture from the files:

```text
/speckit.harness.status
→ mission answered · budgets healthy · Recommendation: finish T012, T013
```

The files are the memory. A conversation context can die at any moment
(session restart, compaction, window overflow); the harness state survives
all of them, and `/speckit.harness.status` rebuilds your context from it in
one step.

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
