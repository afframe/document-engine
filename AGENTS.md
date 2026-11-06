# Headless Document Engine

Design stage: there is no engine code yet. `README.md` describes the planned design. Add `ARCHITECTURE.md` with the first code.

## Commands

- `pnpm install`: install dev tools.
- `pnpm preflight`: the local gate (Prettier, oxlint, `fallow audit`). CI runs the same command.
- `pnpm format`: fix formatting.

## Repo rules

- Commit straight to `main` after `pnpm preflight` passes. Don't create a branch or PR for our own work. PRs are only for Dependabot, external contributions and releases.
- This repo is public. Never commit client names, real contracts, real personal or accounting data, or generated documents; examples use invented data. `/output/` and `/tmp/` are ignored for local runs.
- Licensed AGPL-3.0-only. Check a new dependency's license is compatible before adding it.
