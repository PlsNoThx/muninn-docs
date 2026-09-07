# Muninn — Progress

Working state across Claude Code sessions. Updated at the end of every session
by `/session-end`, read at the start of every session by `/session-start`.

This is a *state* file, not a changelog — git history is the changelog. Keep it
under ~100 lines; delete finished items rather than accumulating them.

**Mirrored to a public repo. No keys, no env values, no user emails, no user
data.**

---

## Current focus

Workflow reset: lean `CLAUDE.md`, per-feature sessions, `PROGRESS.md` as memory
between them. Feature work resumes after this lands.

## Last session

- Rewrote `CLAUDE.md` from the stale personal-MVP draft to match current
  multi-user state. Now rules-only (commands, architecture pointers,
  conventions, hard rules, session workflow) — the feature history lives in
  `README.md`.
- Added `.github/workflows/mirror-docs.yml` — mirrors `PROGRESS.md`,
  `CLAUDE.md`, `README.md` and `docs/` to the public `muninn-docs` repo on
  every push to `main`.
- Added `/session-start` and `/session-end` slash commands and
  `dev/briefs/_TEMPLATE.md`.

## Next up

- [ ] Run `/init` then `/doctor` and prune anything in `CLAUDE.md` that Claude
      can derive from the codebase.
- [ ] Confirm the `README.md` line pointing at `CLAUDE.md` as "the full brief"
      — that framing is stale now.
- [ ] Write the first real brief in `dev/briefs/` for the next globe task and
      run a session against it end to end.
- [ ] Cost guardrails: per-SKU quota caps on Places, spend limit in the
      Anthropic console.

## Open questions / blocked on

- Whether the globe `/beta` path is close enough to promote, or whether the
  graduation loop comes first.

## Standing context

Things worth not rediscovering each session.

- Places and Claude API calls are ~98% of cost at scale; hosting is a rounding
  error. Any change to call frequency matters more than any change to infra.
- The eval harness is `npx tsx scripts/eval.ts --engine v2 --limit 30`.
- No test suite yet.
