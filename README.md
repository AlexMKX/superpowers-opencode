# superpowers (OpenCode patcher)

Automated fork of [obra/superpowers](https://github.com/obra/superpowers) with
patches applied for OpenCode-specific subagent routing.

## What this does

The upstream `superpowers` plugin hardcodes `general-purpose` as the subagent
type for all dispatched tasks. This fork patches the templates to use named
agents (`@implementer`, `@code-reviewer`) with a soft fallback to
`general-purpose` when those agents aren't configured.

This lets you pin specific models per role via `opencode.json`.

## Installation

In `~/.config/opencode/opencode.json`:

```json
{
  "plugin": [
    "superpowers@git+https://github.com/AlexMKX/superpowers.git#<version>"
  ],
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

Pick a `<version>` from [Releases](https://github.com/AlexMKX/superpowers/releases).

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
   for p in <path-to-this-repo>/patches/*.patch; do git apply "$p"; done
   ```
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
