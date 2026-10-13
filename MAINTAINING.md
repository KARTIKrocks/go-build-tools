# Maintaining go-build-tools

This repo controls the lint, format and vulnerability-scan tooling of 13
published libraries. A bad bump here breaks CI everywhere, so changes are
deliberate and staged.

## Principles

- **Pin exact versions. Never use `@latest`.** Builds must be reproducible: the
  same commit lints the same way today and in a year.
- **One source of truth.** Versions live only in `common.mk`. CI reads them with
  `make print-*-version`; nothing else hardcodes a number. `make setup` checks
  the version actually installed and reinstalls on mismatch, so local runs use
  the pinned tools.
- **Pin third-party actions to a commit SHA** (with the tag in a comment),
  especially in `sync.yml`, which has access to `SYNC_PAT`.
- **Bump in batches, security fixes immediately.** Routine bumps happen
  monthly; they don't need to chase every patch release.
- **Canary first.** New tool versions are proven in one repo before the rest.

## When to upgrade

| Trigger | Action | Urgency |
|---|---|---|
| Security advisory in a tool, or `govulncheck` itself flagged | Bump now | Same day |
| New Go release adopted by the libraries | Bump all three tools (see below) | Before CI moves to the new Go |
| Monthly Renovate PRs (`dev tooling`, and `golangci-lint` on its own) | Review and merge | First week of the month |
| New patch release only | Ignore unless it fixes something you hit | None |

### Per tool

- **golangci-lint** is the riskiest bump. Minor releases can add linters or
  change what existing ones report, which turns green repos red. Read the
  release notes first. It must be built with a Go version at least as new as
  the one the libraries target, or it refuses to run, so bump it whenever the
  Go version goes up.
- **goimports** (`golang.org/x/tools`) is low risk. It moves with Go releases
  and newer versions require a newer Go, so it follows the Go bump.
- **govulncheck** is a security gate. Keep it current; new releases track new
  Go versions and standard-library analysis. The vulnerability database itself
  is fetched live on every run, so the pin affects the scanner, not the data.

## How to upgrade

1. Take the Renovate PR (or edit `common.mk` by hand). golangci-lint always
   arrives in its own PR; goimports and govulncheck are grouped. Renovate waits 7 days
   after a release before proposing it, so very fresh releases with
   regressions are skipped.
2. Read the release notes for breaking or behavior changes.
3. Merge to `main`. The sync workflow opens a PR in each repo.
4. **Canary:** merge one repo's sync PR first. If CI is green, merge the rest.
   If a tool now reports new findings, fix them in that repo's PR or, if the
   new rule is wrong for the library, adjust its `.golangci.yml` with a
   comment saying why.
5. If a bump is bad, revert the commit here. The sync reverts every repo.

The drift check only runs on PRs that change `common.mk`, so repos that haven't
merged their sync PR yet are not blocked while you canary.

## Go version policy

Libraries test on their minimum supported Go and on the latest stable release.
The tools here must support both. When raising a library's minimum Go,
check that the pinned tools still run on it.

## Rules for editing common.mk

- Only things that must be identical everywhere belong in `common.mk`:
  versions, `setup`, `print-*-version`. Repo-specific targets (`lint`, `test`,
  `vuln` for multi-module repos) stay in each repo's Makefile.
- Never edit `common.mk` in a consuming repo. The drift check will fail the PR.
- A new repo: add it to `.github/sync.yml`, add `include common.mk` and the
  `common-mk` CI job (see README).

## Credentials

- `SYNC_PAT` is a fine-grained PAT limited to the repos in `.github/sync.yml`
  (Contents and Pull requests: read/write).
- Rotate it if it leaks: revoke the old token in GitHub settings first, then
  create a new one and update the secret.
- When a repo is added to `.github/sync.yml`, also add it to the token's
  repository list, or the sync fails for that repo.
