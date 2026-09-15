# Seat calls: per-provider invocation contracts

Every seat call in this skill goes through the exact patterns below for its provider. They encode real traps; do not improvise around them. The Codex recipe below is verified live on the version stated in "Verified on this rig."

## The non-negotiables (every call)

1. **Prompts go in via a file on stdin** (`- <"$FILE"`), never as an argv string. This kills two traps at once: shell-quoting bugs and the stdin hang (`codex exec` reads stdin in addition to any argv prompt; under a non-TTY harness an unredirected call blocks forever at 0 CPU). If you ever must pass an argv prompt, append `< /dev/null`.
2. `2>/dev/null` always. Codex streams thinking tokens to stderr; they would flood the driving context.
3. **Capture the JSONL stream to a file and parse it after exit.** Never pipe the stream through `grep` directly: `grep -m1` exits at first match, closes the pipe, and can kill Codex mid-run before the output file is written.
4. `--json -o <outfile>`: the answer is read from the `-o` file, the stream file is parsed only for the thread id.
5. **One fresh temp dir per run, fresh filenames per ATTEMPT** (a retry is not a pass; never reuse an output path across attempts -- after a timeout or failure, a reused path serves the PREVIOUS attempt's verdict and looks exactly like success). The receipt root is work-dir-scopable: run under `${COUNCIL_RUN_ROOT:-/tmp}` so the shipped recipe honours `COUNCIL_RUN_ROOT` when the caller sets it, and falls back to `/tmp` otherwise.
6. Resume by explicit thread id, never `--last`. A wrong session looks exactly like success.
7. Never pin `-m`. Model pins can 400 on some auth configurations (historical note, not a capability claim -- this has been observed on ChatGPT-account auth). Before pass 1, echo the active model if `~/.codex/config.toml` has a `model` line, else say "Codex CLI default."
8. Bash tool `timeout: 600000` (10 min) on every FOREGROUND call. Codex writes output only at completion; a killed foreground call is silently empty -- no `-o` file, no thread id. Long calls (max-effort or document-scale) run DETACHED and are polled from bounded foreground loops instead of one long foreground call (see Call discipline, below).
9. `--skip-git-repo-check` on every call (Codex refuses untrusted directories without it).
10. **Snapshot before every call** (the three snapshot lines below, together, in every snippet): status, tracked diff, and untracked-file hashes. If a read-only call changes files anyway: paths that were CLEAN before get reverted; tracked-dirty or untracked paths whose content no longer matches the snapshot get a STOP, not an auto-revert (surface it, let the user judge). Never blanket-revert: the user or the Visionary may have legitimate uncommitted work.

(The item above names "the Visionary" verbatim from the carried source; in this skill it means whichever seat is driving the call.)

```bash
snap() { git status --porcelain > "$RUN/pre-$1.txt"; git diff > "$RUN/pre-$1.patch";
  git ls-files --others --exclude-standard -z | xargs -0 shasum > "$RUN/pre-$1.sha" 2>/dev/null || true; }
```

11. A stream event complaining about "skills context budget" is benign Codex housekeeping, not a failure.

## Preflight (once per session, every registered seat)

Every registered seat runs this once before the first meeting of a session. **Codex seat:** `codex --version` (>= 0.130 printed), then a fresh read-only probe with the exact fresh-session snippet below and the prompt `Reply with exactly: PREFLIGHT-OK`; the probe's `-o` file must contain `PREFLIGHT-OK` and the thread id must extract from the stream. **Session-native seat:** the driving session states its model and effort from its own runtime (see Session-native seat, below) -- no external call. A failed probe enters the RECOVERY LADDER of ## Failure handling below (retry once with a fresh attempt filename; a usage limit with a known reset is scheduled per that section); a seat still unreachable after the ladder is UNAVAILABLE FOR NOW -- an open item under MEETING.md's exit conditions, recorded with the failing output, never a substitute and never a silent skip. SKILL.md Phase 0's "version/health check from SEAT-CALLS.md" resolves here.

## Planning calls (read-only)

Fresh session:

```bash
RUN=$(mktemp -d "${COUNCIL_RUN_ROOT:-/tmp}/rf.XXXXXX")
# write the pass prompt to "$RUN/prompt-p1-a1.md" first
snap p1a1
codex exec -s read-only --skip-git-repo-check --json -o "$RUN/out-p1-a1.txt" \
  - <"$RUN/prompt-p1-a1.md" > "$RUN/stream-p1-a1.jsonl" 2>/dev/null
```

After it exits, extract the thread id from the stream file and echo it visibly:

```bash
THREAD_ID=$(grep -m1 '"type":"thread.started"' "$RUN/stream-p1-a1.jsonl" \
  | sed 's/.*"thread_id":"\([^"]*\)".*/\1/')
```

Resume the SAME session for later passes. THE SAFETY LINE: `resume` rejects `-s`; without `-c sandbox_mode="read-only"` Codex inherits the config default and can WRITE files mid-review. **Verified resume form (proven live on this rig -- see "Verified on this rig" below): `codex exec resume --help` lists `-c`/`--config`, so the explicit sandbox-forcing form is tried first; if the CLI rejects it, the bare form is the fallback, and whichever form actually worked on this rig is the one recorded there.**

```bash
# write this attempt's prompt to "$RUN/prompt-pN-aA.md" first
snap pNaA
codex exec resume "$THREAD_ID" -c sandbox_mode="read-only" --skip-git-repo-check --json \
  -o "$RUN/out-pN-aA.txt" - <"$RUN/prompt-pN-aA.md" > "$RUN/stream-pN-aA.jsonl" 2>/dev/null
# fallback if the CLI rejects the -c form (see "Verified on this rig"):
# codex exec resume "$THREAD_ID" --skip-git-repo-check --json \
#   -o "$RUN/out-pN-aA.txt" - <"$RUN/prompt-pN-aA.md" > "$RUN/stream-pN-aA.jsonl" 2>/dev/null
```

Then read the pass's `-o` file: `OPEN ITEMS:` before the last line, and grep the last line for `VERDICT:`.

**Execution is not a seat call.** Build/execution recipes belong to the gauntlet's arm registry (`~/.claude/gauntlet/arms.json`), not to this file -- see Arm contracts in [SKILL.md](SKILL.md).

## L6 review mechanics (the non-author seat)

1. `git diff --stat` first. If the diff is large (over ~400 changed lines), review file by file in `--stat` order, skipping lockfiles and generated/vendor output, but always read every hand-written source change in full. The arm's report is a claim, not evidence.
2. Run `PROOF_CMD` yourself. Only your run counts.
3. **Deviations:** agreed implementation discretion (what the contract left to the arm) stays delegated -- it is not re-litigated at L6. A MATERIAL deviation (one that changes behavior, scope, or an interface) returns to the meeting as an item, not a unilateral accept or revert.
4. **Commits:** the coordinator proposes, the Owner approves per instance, conventional format. No seat commits without that approval.

## Failure handling

**A call succeeded only if ALL of:** exit code 0, the pass's `-o` file exists and is non-empty, and (for meeting passes) its last line greps a `VERDICT:`. Anything less is a failure, even if the stream showed `thread.started`. Never read a verdict from a file the current attempt did not freshly write.

- Fresh call fails or times out (no thread id captured): retry ONCE with a fresh session and a new attempt filename. Do not "resume" a thread that never started.
- Resume call fails or times out: retry ONCE with the same explicit thread id and a new attempt filename. Second failure: fall back to a fresh session carrying a one-paragraph summary of the meeting so far (the recovery state: revision, roster, objections, dispositions, consumed passes), and SAY SO in one line (session continuity broke; the pass count continues, the new session cannot verify its own prior findings).
- Still failing: stop and surface the error (rerun the identical command WITHOUT `2>/dev/null` to capture stderr). Never silently continue without the review.
- Auth errors: tell the user to run `codex login`. Broken install (`spawn ... ENOENT`): `npm i -g @openai/codex@latest`.
- Prerequisite floor: Codex CLI >= 0.130; this recipe verified live on the version in "Verified on this rig," below.
- **Usage-limit with a known reset:** schedule the retry at reset + 60 s in a detached, polled compound (never a background task while a gate round is open), keep the explicit thread id, use a fresh receipt file per attempt, and fold any Owner input that arrived during the wait into the re-queued prompt; record the usage-limit event and the scheduled retry in the log; the unit is not classified as failed by the limit (a known reset is a schedule); a scheduled retry is neither a pass nor a failure, and genuine seat unavailability (no known reset) is an open item and, at the cap, a stop-and-consult.

## Call discipline

- **Background long calls:** max-effort or document-scale review/build calls run as DETACHED tasks, polled from bounded foreground loops (consume the `-o` file on completion) -- never a background task while a gate round is open. The 10-minute figure in the non-negotiables is foreground-only guidance. A foreground timeout kill yields NO `-o` file and NO thread id -- the attempt is lost, not recoverable.
- **CLI version drift (97):** codex-cli 0.149+ removed --full-auto (--sandbox workspace-write suffices in exec mode). On the session's first BUILD call, exit 2 + "unexpected argument" -> re-run a trivial probe without stderr suppression before consuming a retry. Record the verified-on version per snippet.

## Verified on this rig

The no-pin policy (non-negotiable 7) is a policy choice, distinct from capability: a pin may work on a given auth configuration without that making it authorized for planning calls. `--skip-git-repo-check` is a local grant an Owner may record in `HOUSE-HARDENING.md` (this shipped file states only the upstream scope, never a portable default).

- Windows: run all contract snippets through the Bash tool (POSIX). `shasum` may be
  absent — `sha256sum` is present and equivalent for the snapshot line.
- **codex-cli version and resume form verified live for this shipped recipe (P4, 2026-09-11):** codex-cli 0.153.4. `codex exec resume --help` lists `-c`/`--config`; the explicit `-c sandbox_mode="read-only"` resume form (as shown above) WORKS on this rig -- exit 0, a fresh non-empty `-o` file, thread continuity proven (the resumed reply repeated a token planted in the fresh call). The bare fallback form was not needed. A read-only-resumed thread's attempt to write a file via its shell tool was ATTEMPTED and DENIED in the execution record (`item.completed` / `status:"failed"` / `exit_code:1` / "Access ... is denied"), the file stayed absent; a control fresh thread in the same directory with `-s workspace-write` completed the identical write (`status:"completed"` / `exit_code:0`, no escalation prompt), file present, both cleaned up afterward.
- **Identity evidence (verified in the P4 live test, 2026-09-11):** the `--json` stdout stream carries no model or reasoning-effort field on any event type; the seat's effective model, effort and sandbox policy are read from the PROVIDER SESSION RECORD -- `~/.codex/sessions/<YYYY>/<MM>/<DD>/rollout-<timestamp>-<thread id>.jsonl`: line 1 carries the session metadata (the thread identity); each `turn_context` event carries `payload.model`, `payload.effort` and `payload.sandbox_policy` for its turn. FRESHNESS BINDING: the evidence for an attempt is the `turn_context` event of THAT attempt's turn in the record named by the thread id the attempt's own stream extracted (the record's turn count advances with the attempt; the entry is matched by turn, never "the last one seen"); a missing, earlier, or unmatched entry = UNVERIFIED, never PASS. The IDENTITY line keeps the seat/model/effort shape; the sandbox policy is reported alongside the proof evidence. Non-negotiable 7's config echo is a pre-call expectation, never identity evidence. This satisfies ## Adding a provider seat's requirement for the Codex recipe.

## Session-native seat (the driving session)

The coordinator seat (today: Fable) is the session model itself. Its "call" is composing its seat response directly into the transcript and the canonical log -- no external process, no `-o` file, no thread id. IDENTITY comes from the session's own runtime model/effort line, never assumed from history or from a prior pass. It makes no external call and, during a meeting, edits only `PLAN_FILE` and the log -- Rule 2 (No End Runs) and the hygiene rule bind it exactly like any other seat. Its preflight is the identity statement above (## Preflight): stating its current model and effort satisfies the session-native preflight, once per session.

## Adding a provider seat

A new provider's call recipe must give the same guarantees as the Codex recipe above, regardless of its actual flags: prompts delivered via a stdin file, output captured to a file the recipe controls, an explicit session/thread identity extracted and never inferred, read-only planning calls enforced by the provider's own sandboxing (never by prompt instruction alone), the snapshot rule (before/after comparison, STOP on unexpected dirt, never blanket-revert), the failure ladder (retry once, then a announced degraded fallback, then stop-and-surface), and the seat's identity (model/effort) read from that call's own execution record, never echoed from a config file or assumed from history. A recipe missing any of these is not ready to register as a seat's call recipe in `seats.json`.
