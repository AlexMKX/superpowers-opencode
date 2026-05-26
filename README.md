# superpowers (OpenCode patcher)

Automated fork of [obra/superpowers](https://github.com/obra/superpowers) with
patches applied for OpenCode-specific subagent routing.

## Why this matters

To swat a fly with a bazooka is, of course, a triumph of ambition over proportion. Much like summoning the most expensive model in the house merely to replace a single variable in a file.

Yet out of some lingering courtesy to both nature and the budget, it may be wiser to entrust such honest drudgery to cheaper minds (minding model's don't have ones).

But, cheap models can provide low-quality results.

Current superpowers flow uses general-purpose tool for the subagents. If you use expencive model for brainstorming (correct), 
and if you don't switch model during implementation stage - all gruntwork will be done by expencive model.

You can change the subagent-driven stage to the cheaper model (say, sonnet) and it would save you costs and electricity. 
In this case, the review stage, which is a part of development process will be executed by sonnet as well. So results most likely suffer. 

To maintain better quality we prefer following roles:
Architecture/brainstorming : Opus; Coding: Sonnet; Reviewing: Opus (or GPT as a second pair of eyes).

## The limitations of superpowers

Although virtually any agent harness can allow to use specific models for specific agents, the superpowers prohibits harness-specific code in their repo. 
For opencode you can specify implementer as a sonnet, but you can't push such a PR to upstream (at least according to the Obra's CLAUDE.md).

This is why this patcher came.

## What this does

The upstream `superpowers` plugin hardcodes `general-purpose` as the subagent
type for all dispatched tasks. This fork patches the templates to use named
agents (`@implementer`, `@code-reviewer`) with a soft fallback to
`general-purpose` when those agents aren't configured.

This lets you pin specific models per role via `opencode.json`.

## Installation

In `~/.config/opencode/opencode.json`:

**Option A — always current (recommended):**

```json
{
  "plugin": [
    "superpowers@git+https://github.com/AlexMKX/superpowers-opencode.git#latest"
  ]
}
```

The `latest` tag is moved to the newest release on every workflow run.
Note: git-based npm/bun packages are cached locally; you may need to clear
`~/.cache/opencode/node_modules` (or equivalent) after a new release lands
for the update to take effect.

**Option B — pin to a specific release:**

```json
{
  "plugin": [
    "superpowers@git+https://github.com/AlexMKX/superpowers-opencode.git#v5.1.0-202605260727"
  ]
}
```

Pick a version from [Releases](https://github.com/AlexMKX/superpowers-opencode/releases).

**Agent config (same for both options):**

```json
{
  "agent": {
    "implementer": {
      "model": "anthropic/claude-sonnet-4-6",
      "mode": "subagent"
    },
    "code-reviewer": {
      "model": "anthropic/claude-sonnet-4-6",
      "mode": "subagent"
    }
  }
}
```

## How it works

- `main` branch contains only the patcher: patches, workflow, this README.
- Each release lives in an orphan branch `release/<version>` + matching tag.
- Release version format: `{upstream_tag}-YYYYMMDDHHMM` (UTC), e.g.
  `v5.1.0-202601020304`.

## Releasing

**Manual:** GitHub Actions → "Release" workflow → "Run workflow".
Inputs:
- `upstream_ref`: upstream tag (e.g. `v5.1.0`) or `latest` (default).
  Only published tags matching `^v\d+\.\d+\.\d+$` or the literal string
  `latest` are accepted. Branches and SHAs are rejected.

**Automatic:** Any push to `main` that touches `patches/**` or
`.github/workflows/**` triggers a release against upstream `latest`.

## Adding a new patch

1. Clone upstream at the version you want to patch:
   ```bash
   git clone --branch v5.1.0 https://github.com/obra/superpowers.git
   cd superpowers
   ```
2. Apply existing patches first:
   ```bash
   git am <path-to-this-repo>/patches/*.patch
   ```
   (Using `git am` preserves authorship and commit metadata, so step 3
   becomes "make your changes and commit them normally".)
3. Make your changes, commit them in the upstream clone.
4. Generate the patch:
   ```bash
   git format-patch -1 HEAD --stdout > <path-to-this-repo>/patches/NNNN-short-desc.patch
   ```
   Naming: `NNNN-short-description.patch` where `NNNN` is a 4-digit ordinal
   and `short-description` uses `[a-z0-9-]+`.
5. Open a PR against `main` of this repo.

## Attribution

All upstream code in release branches is property of `obra/superpowers`
contributors and licensed under upstream terms (see `LICENSE` in release
branches). The patches in `patches/`, CI configuration in `.github/`, and
this `README.md` are MIT-licensed.
