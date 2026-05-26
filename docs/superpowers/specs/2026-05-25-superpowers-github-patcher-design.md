# superpowers GitHub patcher — design

**Repo:** `AlexMKX/superpowers`
**Status:** live (first release `v5.1.0-202605260656`)

## Why this exists

Upstream [`obra/superpowers`](https://github.com/obra/superpowers) hardcodes
`general-purpose` as the subagent type in skill-prompt templates. This is
fine for Claude Code but bypasses OpenCode's named-agent system, where you
configure per-role models via `opencode.json` `agent:` entries (e.g.,
`@implementer`, `@code-reviewer`).

The patcher consumes upstream releases, applies our patches, and republishes
the result. Consumer (`opencode.json` plugin URL) gets `obra/superpowers`
code with our local routing changes — without manually maintaining a fork.

## Architecture

```
AlexMKX/superpowers
├── main                                 # the patcher itself
│   ├── README.md
│   ├── .gitignore
│   ├── patches/
│   │   └── 0001-route-subagent-dispatches.patch
│   ├── .github/workflows/release.yml
│   └── docs/                            # this file lives here
│
├── release/v5.1.0-202605260656          # orphan branch, one per release
│   └── <patched upstream snapshot>      # full obra/superpowers + patches
│
└── tags
    └── v5.1.0-202605260656              # immutable, points to orphan commit
```

- **`main`**: the patcher only. Never contains upstream code.
- **`release/<version>`**: orphan branches (no history), one per release.
  Each is a full snapshot of `obra/superpowers@<tag>` with our patches
  applied. Consumers install plugin via git ref pointing here.
- **Tags**: `v{upstream_tag}-{YYYYMMDDHHMM}` (UTC). Granular to the minute —
  no collisions in practice.

## Workflow (.github/workflows/release.yml)

Triggers:
- `workflow_dispatch` with `upstream_ref` input (default `latest`)
- `push` to `main` touching `patches/**` or `.github/workflows/**`

Steps (in order):
1. Checkout patcher into `./patcher/`
2. Resolve `upstream_ref` → real upstream tag (`latest` → `gh release view -R obra/superpowers`)
   Only `latest` or tags matching `^v\d+\.\d+\.\d+$` accepted. Branches/SHAs rejected.
3. Compute version `{upstream_tag}-$(date -u +%Y%m%d%H%M)`
4. Check tag collision via `git ls-remote --exit-code --tags <repo_url>`
5. Clone upstream shallow at the tag
6. Apply patches from `patcher/patches/*.patch` (fail if empty or any `git apply --check` fails)
7. Build orphan branch `release/<version>`, snapshot commit with multi-paragraph message
8. Push branch and tag to origin
9. Generate release notes (markdown, with install snippet)
10. Create GitHub Release

All failures abort with `::error::` messages. No retries, no auto-fix.

Security hardening: user inputs (`inputs.upstream_ref`) and
`github.repository` are passed via `env:` blocks, never interpolated into
shell scripts.

## Patches

Format: `git format-patch` output. Naming: `NNNN-short-description.patch`
where `NNNN` is a 4-digit ordinal, `short-description` is `[a-z0-9-]+`.
Applied in alphabetical order via glob.

Current set:
- `0001-route-subagent-dispatches.patch` — replaces hardcoded
  `general-purpose` subagent dispatch in 7 skill template files with
  named-agent references (`@implementer`, `@code-reviewer`) and a soft
  fallback to `general-purpose` when the named agent isn't configured.

## Adding a new patch (workflow)

1. Clone upstream at a tagged release.
2. Apply existing patches: `git am <patcher>/patches/*.patch`.
3. Make changes, commit normally.
4. Generate: `git format-patch -1 HEAD --stdout > <patcher>/patches/NNNN-short.patch`.
5. PR against `main`. Push triggers auto-release.

## Versioning rules

- Version = `{upstream_tag}-{YYYYMMDDHHMM}` (UTC).
- Upstream bumped to v5.2.0 → `v5.2.0-{ts}`.
- Patcher updated, upstream unchanged → `v5.1.0-{newer-ts}`.
- Just re-run workflow → new timestamp regardless.
- No semver of our own — upstream tag is the source of truth for X.Y.Z.

## Open assumptions (handled)

- **Empty `patches/`**: workflow fails explicitly (`::error::No patches found`).
- **Tag collision**: workflow fails on `git ls-remote --exit-code` match.
- **Multi-line `Subject:` headers**: handled via awk fold-unfold in the
  release notes / commit message generation (fixed in workflow).
- **Concurrent runs**: GitHub Actions serializes by workflow file. Same-minute
  collision fails on tag check. Acceptable.

## Out of scope

- npm publishing (OpenCode installs from git directly)
- Auto-detection of upstream changes (no cron / scheduled triggers)
- Multi-target patching (single upstream: `obra/superpowers`)
- Conflict auto-resolution (fail loudly, fix patches manually)
- Retention / GC of old release branches (manual cleanup, branches are cheap)
- Force-push or release overwrites (releases are immutable)

## Migration history

The repo was originally a regular fork mirror (`main` = obra/superpowers
content). Migration replaced `main` content with the patcher scaffold in
a single additive commit. The transitional `superpowers-opencode` branch
(legacy manual fork) was deleted after the first release was verified.
27 mirrored upstream branches were also deleted from origin.
