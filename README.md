# go-build-tools

Single source of truth for build tooling shared by the KARTIKrocks Go libraries.

| File | Purpose |
|---|---|
| `common.mk` | Pinned `golangci-lint` / `goimports` / `govulncheck` versions, `setup`, and `print-*-version` targets |
| `renovate.json` | Bumps the versions in `common.mk` automatically |
| `.github/sync.yml` | List of repos that receive `common.mk` |
| `.github/workflows/sync.yml` | Opens a PR in each repo when `common.mk` changes |
| `.github/workflows/check-common-mk.yml` | Reusable drift check for consumers (runs on PRs that change `common.mk`) |

## Upgrading a tool

See [MAINTAINING.md](MAINTAINING.md) for when and how. In short: merge the
Renovate PR here, then merge the sync PRs it produces in each repo.

## Using it in a repo

Top of the repo's `Makefile`:

```makefile
include common.mk
```

Keep only repo-specific targets (`lint`, `test`, `vuln`, ...) in the Makefile.
`common.mk` sets `.DEFAULT_GOAL := all`, so the repo must define `all`.

Add the drift check to CI:

```yaml
  common-mk:
    uses: KARTIKrocks/go-build-tools/.github/workflows/check-common-mk.yml@main
```

## One-time setup

- Add a `SYNC_PAT` secret to this repo: a fine-grained PAT with
  *Contents* and *Pull requests* write access on the repos in `.github/sync.yml`.
- Install the Renovate GitHub app on this repo.
