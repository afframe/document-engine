# Headless Document Engine

Design stage: there is no engine code yet. `ARCHITECTURE.md` holds the approved design and nothing else; update it after each of Hleb's rulings.

## Commands

- `pnpm install`: install dev tools.
- `pnpm preflight`: the local gate (Prettier, oxlint, `fallow audit`). CI runs the same command.
- `pnpm format`: fix formatting.

`pnpm install` also turns on `.githooks/pre-push`, which blocks a push if gitleaks finds secrets or Czech personal identifiers (`.gitleaks.toml`). The hook needs `gitleaks` installed; CI runs the same scan.

## Repo rules

- The remote is the source of truth. Prepare commits on a branch after `pnpm preflight` passes, and merge to `main` only when Hleb approves. PRs are only for Dependabot, external contributions and releases.
- This repo is public. Never commit client names, real contracts, real personal or accounting data, or generated documents; examples use invented data. `/output/` and `/tmp/` are ignored for local runs.
- Licensed MPL-2.0. Check a new dependency's license is compatible before adding it.
