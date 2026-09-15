# council

A multi-seat planning body for AI coding agents: every registered seat states its own direction, reviews every other seat's, and a meeting closes only when all participating seats are on the same page -- bound to one plan revision and one frozen roster, with zero open items. Execution stays separate: this skill plans and reviews, an external executor (in the reference environment, the `infinity-gauntlet` skill's arm registry) builds.

Forked from Jens Heitmann's `NulightJens/rocket-fuel-skill` (MIT); see `PROVENANCE.md` for the full lineage and what changed from a two-seat Visionary/Integrator pair to an open N-seat registry.

## Install

Copy this directory to your skills directory (for Claude Code: `~/.claude/skills/council/`). No build step; it is plain Markdown + one JSON template.

On first activation, the skill creates its data home (`$COUNCIL_HOME`, default `~/.claude/council/`) and seeds `seats.json` from `templates/seats.template.json` if one does not already exist there. Edit `seats.json` to name your own seats -- membership is deliberately kept out of the skill directory so it is never overwritten by an update.

Read `SKILL.md` first, then `MEETING.md` for the meeting protocol and `SEAT-CALLS.md` for provider-specific call recipes (the shipped example covers Codex CLI; see "Adding a provider seat" in `SEAT-CALLS.md` to add another).

## License

MIT -- see `LICENSE`; the upstream notice is reproduced in `NOTICE.md`.
