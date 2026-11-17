# Bookandry

Local-first personal money manager: multi-profile, multi-currency, English and Hebrew with full RTL, account-backed with a full offline copy per device. Money is integer minor units from a frozen ISO 4217 table; dates are `"YYYY-MM-DD"` string maths with no date library. `README.md` and the documents under `docs/` are the source of truth; this file only points into them.

## Where to look

- Architecture, the three memo layers, money and date rules: `docs/design-decisions.md` → "Architecture".
- Why things are the way they are: `docs/design-decisions.md` → "Design decisions" and "Non-goals". Check before proposing a change that touches them.
- File layout: `docs/sync-and-security.md` → "Code map". Features live in `src/features/<name>/`, pure logic in `src/domain/`.
- Auth, Supabase, threat model: `docs/sync-and-security.md` → "Accounts" and "Threat model".
- Background research (card issuers, interest rules): `docs/research/`.
- Specs and plans from earlier sessions: `docs/superpowers/`.

## Before saying done

`pnpm verify` is the gate (typecheck, lint, format, the three bespoke gates, unit tests with coverage, build). CI runs the same command on every push and PR; the pre-commit hook runs eslint and prettier on staged files. `pnpm verify:full` adds Playwright, Lighthouse, knip and the hostile-timezone run; use it before a release.

## Agent skills

### Issue tracker

GitHub Issues on `Doofess/bookandry` via `gh`. See `docs/agents/issue-tracker.md`.

### Triage labels

The five default labels (`needs-triage`, `needs-info`, `ready-for-agent`, `ready-for-human`, `wontfix`). See `docs/agents/triage-labels.md`.

### Domain docs

Single-context: `CONTEXT.md` at the root and `docs/adr/`, created lazily by `/domain-modeling`. See `docs/agents/domain.md`.
