# The Council Meeting (every seat, IDS protocol)

The N-seat planning loop. Every participating seat states its own direction first, reviews every other seat's, and the meeting iterates until every verdict line agrees or the pass cap hits. A seat may endorse another seat's direction instead of manufacturing a competing one; any seat may originate a complete replacement plan or supply exact section text at any pass; the seats jointly choose the starting proposal. The coordinator scribes (no editorial veto); the Owner breaks deadlocks; unresolved alternatives are preserved, never quietly dropped.

## Inputs

- The canonical plan file: `PLAN.md` for pipeline runs, or the file the user pointed at (copied into the repo as `RF-PLAN.md` if it lives outside). One file for the whole meeting; every revision lands in it. Call it `PLAN_FILE` below.
- The Core Focus: one sentence from `VTO.md`, or from the user's request if there is no VTO
- The frozen roster: every participating seat's id + model + effort, fixed at pass 1 (see the roster rules in [SKILL.md](SKILL.md)).
- `MAX_PASSES` = 10 by default per meeting kind (L1-L3 meeting, L6 review meeting counted separately); whatever the Owner set is a hard cap, not a suggestion.
- A PASS is one numbered exchange: every participating seat's response to ONE revision id, collected before the next revision is cut. Individual seat CALLS are recorded separately and are not passes: an attempt, a retry under the failure ladder, or a mechanical re-open receipt does not advance the count; the count advances only when the exchange is complete.
- The OPEN ITEMS REGISTER lives at the TOP of `PLAN_FILE`, before any plan content, under the heading `## Open items register`: one line per open item (id, originating seat, one-line state, the pass it was raised in), rewritten every pass. It is part of `PLAN_FILE` and is hashed with it: the file's content, register included, is frozen before each exchange is collected, and that frozen content is what the pass's revision id names.
- The revision id: `<unit> / <gauntlet round> / <meeting purpose> / <pass> / <hash of PLAN_FILE at that pass>`. The hash covers PLAN_FILE's content at that pass ONLY -- never the log, never the seat responses. Two passes that produce byte-identical PLAN_FILE content share a hash; any edit to PLAN_FILE changes it.

## The seat-response contract

Every seat, whatever its provider, returns the same shape (provider mechanics -- flags, `-o` files, resume -- live in [SEAT-CALLS.md](SEAT-CALLS.md), never here):

```
IDENTITY: <seat id> / <model> / <effort>
ROSTER + REVISION REVIEWED: <the frozen roster> / <the revision id this response answers>

DIRECTION: <this seat's own direction, its assumptions, and its strongest reason>
  (or) ENDORSE: <another seat's id> -- <why, in one sentence>
  (any seat may ORIGINATE a complete replacement plan, or supply exact section
  text, at any pass -- not only pass 1)

ITEMS:
- [KILL|DEFER|FIX|CLARIFY] <item id> <root cause in one sentence> -> <one-line fix or question>
- ALT <item id>: <direction> -> <sketch> / Core Focus test: <...> / cost: <...> / replaces: <...>

DISPOSITIONS (on every OTHER seat's open items from the prior pass):
- <item id>: AGREE
- <item id>: DISAGREE -- <reason>
- <item id>: MERGE -- <how>

OPEN ITEMS: <ids still open after this response>

VERDICT: SAME PAGE
(or) VERDICT: NOT YET
```

`VERDICT:` is always the last line. `OPEN ITEMS:` always comes immediately before it, never after.

Item ids are stable and collision-free: `<seat id>-p<pass>-<n>` assigned by the originating seat at first mention (an ALT is `ALT-<seat id>-p<pass>-<n>`). An id is never changed, and a COLLISION is assigning an id already registered anywhere in the meeting's id history to a DIFFERENT item; references (dispositions, open-items lines, later passes) reuse ids by design and never replace a DISAGREE's reason or a MERGE's how. A colliding response is still logged verbatim; the coordinator returns it for correction and the corrected response is appended as a further entry, never substituted.

## Pass 1: fresh seat sessions (read-only)

Write the pass-1 prompt to a temp file first (never inline-quote a long prompt), then launch every participating seat per its call recipe in [SEAT-CALLS.md](SEAT-CALLS.md), capturing each seat's identity (thread/session id where the provider has one). Every seat's prompt is the complete seat-response contract above, plus: the Core Focus, `PLAN_FILE`'s content, the frozen roster, and this pass's revision id. The simpler-alternative flag (does a materially simpler alternative exist?) and the broken-proof flag (would this rock's "proof" pass while the feature is actually broken?) are asked of every seat. Every seat runs read-only.

## Passes 2..N

Resume the SAME seat sessions (never a fresh one, except under the failure ladder in [SEAT-CALLS.md](SEAT-CALLS.md)). The request carries the explicit outstanding state -- the current revision id, the frozen roster, every open item, and every other seat's dispositions from the prior pass -- never session memory: a seat's provider may or may not retain context between calls, and the protocol never depends on it remembering anything the request does not restate.

## Between passes: IDS every item

For each finding, run Identify / Discuss / Solve:

1. **Identify.** Restate the real issue in one sentence, root cause over symptom -- any seat may do this, not only the seat that raised it.
2. **Discuss once.** Every seat dispositions the item (AGREE / DISAGREE: reason / MERGE: how). There is no final arbiter: a DISAGREE keeps the item OPEN regardless of which seat raised it or which seat disagrees. Re-arguing a settled item in a later pass is politicking.
3. **Solve.** Items every seat AGREEs or MERGEs on are applied to `PLAN_FILE` at once. DEFER items go to `ISSUES.md` (or `RF-ISSUES.md` under the collision rule) with their destination and reason. Merging or deferring an item requires unanimity among participating seats; the originating seat may withdraw its own item, but withdrawal never erases a responding seat's standing objection to it -- an objection stands until its own seat retracts it.

## The log

Per pass, every participating seat's response section lands verbatim (subject only to the armor rule's secret/injection redaction), followed by the open-items register and the revision/roster line for that pass:

```markdown
## Pass N -- revision <id> -- roster <seat ids>
### <seat id> (verbatim)
<the seat's complete response>
### <seat id> (verbatim)
<the seat's complete response>
### Open items after this pass
- <item id>: <one-line state>
```

The per-pass register in the log is a snapshot; the running register at the top of `PLAN_FILE` is the canonical one (design A9).

Decision content already recorded (a closed item, a prior pass's verbatim sections) is frozen once written; the log itself is append-only -- corrections land as a new pass's entry, never an edit to a past one.

## Exit conditions

- **Close:** every participating seat's `VERDICT: SAME PAGE`, bound to ONE revision id and to the frozen roster, with ZERO open items. A substantive edit to `PLAN_FILE` after a close re-opens the affected decision. A membership change invalidates prior consensus while preserving every recorded objection and the passes already consumed (unless the Owner also changes the cap).
- **Cap hit without close:** stop-and-report to the Owner with the unresolved items and a recommendation; the Owner extends the pass cap or decides. Record the decision as `USER OVERRIDE: <decision>` -- scoped to exactly what it resolves, never a blanket approval and never fabricating a seat's agreement it did not give. A flagged deadlock beats a fake approval.
- **A seat's call fails twice in a row:** follow [SEAT-CALLS.md](SEAT-CALLS.md)'s failure ladder (retry, then a LOUDLY-announced degraded fresh session carrying the meeting summary, then stop and surface). Log `DEGRADED: fresh session from pass N` if the fallback fires. Never silently continue without that seat's review; a seat that cannot be reached is an open item, not a pass.
- **The mechanical gate is unchanged:** it validates a receipt's freshness and identity only -- never the verdict text. SAME PAGE on all fronts is meeting discipline, established by every seat's own verdict line in the log, not by the gate.
- **Recovery** (resuming a meeting after any interruption) always carries the revision id, the frozen roster, every standing objection, every disposition already recorded, and the count of passes already consumed -- whether the interruption was a scheduled retry with a known reset, genuine seat unavailability, or a harness fault. A scheduled retry with a known reset is neither a pass nor a failure; genuine seat unavailability is an open item and, at the cap, a stop-and-consult.
- **Seat-built rocks (Standalone execution):** for standalone L6, the electorate is the frozen roster minus the builder and must contain at least one seat; throughout pass collection, dispositions, availability and closure, participating reviewers means this electorate, while revision and roster binding, zero open items and the existing caps remain mandatory; the builder supplies factual answers in its own log section without a verdict or veto.

## Hygiene

- One finding, one line, one disposition. No finding disappears without a logged reason.
- No seat's call edits files during a meeting. Every call recipe in [SEAT-CALLS.md](SEAT-CALLS.md) snapshots status before every call; if a read-only pass changes files anyway, paths that were clean before get reverted, and paths that were already dirty (tracked or untracked) get a STOP, never an auto-revert -- surface it, let the Owner judge -- then restart the pass with the sandbox forced per the call recipe.

## Meeting discipline

(kept from the prior system's weekly-review integrations; obs numbers retained as provenance)

- **Source-verify checkable claims (76):** findings asserting engine behavior, API semantics, or math identities are verified against primary sources BEFORE disposition - acceptance and rejection alike cite the check; verification also UPGRADES fixes, not just filters them.
- **Altitude re-anchor:** when a pass's items become predominantly execution-detail refinements, the coordinator's next prompt names the boundary that applies under Equal voice (execution-detail discretion still requires every seat's agreement -- there is no assigned tie-break seat) and asks for a build-sufficiency judgment from every seat -- this prevents fake deadlocks at the cap.
- **Hot-document revision (105):** past round ~3, or when any finding supersedes prose: apply by SECTION REWRITE, never incremental patches; grep afterward for the retired phrases and old numbering; verify every cross-reference the round touched.
