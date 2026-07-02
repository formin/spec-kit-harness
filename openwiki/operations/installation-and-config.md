# Installation, configuration, and release operations

## Installing the extension

Research Harness is listed in the Spec Kit [community extension catalog](https://github.com/github/spec-kit/blob/main/extensions/catalog.community.json) under the name `harness`. Requires Spec Kit `>=0.2.0`; works with any agent Spec Kit supports. No external tools, MCP servers, or network access are required at runtime.

**By name from the community catalog.** The community catalog is discovery-only by default; opt in once via `.specify/extension-catalogs.yml` (per project) or `~/.specify/extension-catalogs.yml` (per user) by setting `install_allowed: true` on the community catalog entry, then:

```bash
specify extension add harness
```

**By release URL (zero config, pinned version):**

```bash
specify extension add harness --from https://github.com/formin/spec-kit-harness/archive/refs/tags/v1.0.0.zip
```

URL installs show an *Untrusted Source* warning and ask `Continue with installation? [y/N]`; in non-interactive shells (CI), pipe the answer: `echo y | specify extension add …`. Catalog installs skip this prompt.

**For development:**

```bash
git clone https://github.com/formin/spec-kit-harness
specify extension add --dev ./spec-kit-harness
```

Verify with `specify extension list` — expect `Research Harness (v1.0.0)`, `Commands: 5 | Hooks: 2`, status Enabled.

## Configuration

Copy [`config-template.yml`](../../config-template.yml) to `.specify/extensions/harness/harness-config.yml` and customize. Every key is optional; missing keys fall back to the extension defaults in `extension.yml`.

- `budget` — per-session hard ceilings: `searches: 30`, `inspections: 40`, `verifications: 20`, and `context_tokens: 4000` (soft cap on harness state rendered into context per iteration).
- `curation` — `max_curated: 25` (hard cap on `curated.md` entries), `importance_levels: [critical, high, medium, low]`, `evict_policy: lowest-importance-first`.
- `rendering` — slice sizes used by `/speckit.harness.status` and each explore iteration: `candidates_slice: 10`, `curated_slice: 15`, `observations_slice: 8`.
- `state` — `directory: harness` (subdirectory of the active feature dir) and `fallback_directory: .specify/harness/global` (used when no feature directory exists).
- `stop_conditions` — `marginal_gain_window: 3` (stop after N consecutive actions adding no curated evidence) and `require_critical_verified: true` (research is not "done" while critical claims are unverified).

**Precedence chain** (lowest → highest):

1. Extension defaults (`defaults` block in `extension.yml`)
2. `.specify/extensions/harness/harness-config.yml`
3. `SPECKIT_HARNESS_*` environment variables (e.g. `SPECKIT_HARNESS_BUDGET_SEARCHES=50`)
4. Per-invocation `key=value` arguments to `/speckit.harness.init` (e.g. `searches=50`)

## Releasing a new version

1. Update `extension.version` in `extension.yml` and add a section to `CHANGELOG.md` (Keep a Changelog format, Semantic Versioning).
2. Tag the commit `vX.Y.Z` and push the tag.
3. Create a GitHub release for the tag. The release archive URL (`https://github.com/formin/spec-kit-harness/archive/refs/tags/vX.Y.Z.zip`) is what `specify extension add --from` consumes; catalog users pick up the new version through the community catalog entry.

## OpenWiki documentation refresh (`.github/workflows/openwiki-update.yml`)

The docs under `openwiki/` are refreshed by the `OpenWiki Update` GitHub Actions workflow:

- **Triggers**: `workflow_dispatch` (manual) and a daily schedule at `0 21 * * *` UTC (06:00 KST).
- **Steps**: checks out the repository, sets up Node.js 22, installs OpenWiki globally (`npm install --global openwiki`), runs `openwiki --update --print` with `OPENWIKI_PROVIDER: anthropic` and `OPENWIKI_MODEL_ID: anthropic/claude-sonnet-5`, then opens a pull request (branch `openwiki/update`, paths limited to `openwiki/`) via `peter-evans/create-pull-request`.
- **Requirement**: the repository secret `ANTHROPIC_API_KEY` must be set, or the OpenWiki step fails. The workflow has `contents: write` and `pull-requests: write` permissions so it can push the branch and open the PR.
