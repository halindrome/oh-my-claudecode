---
name: ultradebug
description: Stateful, resumable bug-fixing loop - reproduce, hypothesize, fix, verify, driven to completion until the bug is gone
argument-hint: "\"<bug>\" [--competing|--serial] [--no-verify] [--pause] [--interactive] | --resume [--session <id>]"
level: 4
---

[ULTRADEBUG - ITERATION {{ITERATION}}/{{MAX}}]

Your previous attempt did not output the completion promise `ULTRADEBUG_COMPLETE`.
Continue the debug loop from the current session status.

<Purpose>
UltraDebug is a persistence loop that fixes ONE bug per session using the
scientific method — reproduce → hypothesize → evidence → diagnose → fix → verify —
and keeps iterating until the original reproduction passes and the suite is green,
or the loop's bounds trip. State is persisted every phase transition, so the run
survives interruption, compaction, and `--resume` across sessions.

Design reference: `.omc/plans/ultradebug-design.md` (decisions D1–D7; D5 superseded, §13A).
</Purpose>

<Use_When>
- A specific bug needs fixing end-to-end, not just diagnosing.
- The user says "ultradebug", "debug this", "fix this bug and verify it", "find and fix".
- The fix may take several investigate→fix→verify cycles and must be verified, not assumed.
</Use_When>

<Do_Not_Use_When>
- User only wants a diagnosis / root-cause opinion — delegate to the `debugger` or `tracer` agent, or use `/trace`.
- User wants to diagnose the OMC session/repo state itself — use `/debug` (different skill).
- User wants a general quality-gate cycle unrelated to a specific bug — use `/ultraqa`.
- User wants a full idea→code pipeline — use `/autopilot`.
</Do_Not_Use_When>

<Driver>
UltraDebug is driven by the OMC persistence engine (the "ralph engine", design §5A),
NOT by Claude's native `/goal` (its evaluator only reads the transcript and cannot
run tests — see D4). The Stop hook re-injects this continuation prompt every turn
while the session's `active: true`, until either:
- the loop prints the completion promise `ULTRADEBUG_COMPLETE`, or
- `iteration >= max_iterations` (default 8), or
- the thinking-only-streak guard trips (no tool progress).

Completion promise contract: only print `ULTRADEBUG_COMPLETE` once session status is
`complete` (the original repro passes AND `/ultraqa` returned PASS). Print it **alone
on its own line** — the driver recognizes the promise ONLY as a standalone line, so
narrating the token inside a sentence (e.g. "not printing ULTRADEBUG_COMPLETE yet")
will NOT end the loop. Never print it to escape a hard iteration — report the blocker.

Cross-turn autonomy is driven by the persistent-mode Stop hook's `ultradebug` mode
(`checkUltradebug` in `src/hooks/persistent-mode`, mirroring `ralph`): while the
session is `active`, each Stop re-injects the continuation prompt until completion,
`max_iterations`, or the thinking-only guard trips. `--resume` continues a session
that was interrupted or stopped at a bound.
</Driver>

<State>
Per-session file: `.omc/state/sessions/{sessionId}/ultradebug-state.json` (schema: design §6).
(The `state_write`/`state_read` tools normalize mode `ultradebug` to the canonical
`ultradebug-state.json` filename — same convention as every other OMC mode.)
Written via `state_write`, read via `state_read`. Key fields:
`active`, `session_id`, `title`, `issue`, `status`, `path` (A|B), `pause_before_complete`,
`verify_interactive`, `iteration`, `max_iterations`, `head_before`, `hypotheses[]` (confirmed AND rejected,
each with `evidence_for`/`evidence_against`/`conclusion`), `root_cause`, `plan`,
`changed_files[]`, `commit`, `verify{round,result,failures}`.

Session id: `OMC_SESSION_ID` (CLI) or hook `data.session_id`.
Hypothesis preservation is non-negotiable — keep every rejected hypothesis so
`--resume` never re-investigates a dead end.
</State>

<Phases>

**Phase 0 — Resolve session.**
- `--resume` → load active session, else latest unresolved (announce which).
- `--session <id>` → load that session; missing file → STOP with error.
- Neither, and a bug description present → create a new session: parse the bug
  (strip flags), capture `head_before = git rev-parse HEAD`, set `status=investigating`,
  `iteration=1`, `max_iterations=8`, `pause_before_complete` from `--pause` (default `false`),
  `verify_interactive` from `--interactive` (default `false`).
- No description and no resume flag → STOP with usage.
- On resume, dispatch on `status`: `investigating`→Phase 2, `fixing`→Phase 3,
  `verifying`→Phase 4, `complete`→STOP (already done).
- **Re-arm on resume.** A session stopped at a bound has `active:false` (state kept
  by design). To resume it, first write `active:true` and ensure `iteration <
  max_iterations` — raise `max_iterations` if the previous run exhausted it — so the
  Stop-hook driver re-injects continuations again. (Resume normally runs in a fresh
  session, where the in-memory tombstone that suppressed the slot at stop time is
  already gone; within the same session, the tombstone TTL must lapse first.)

**Phase 1 — Classify + route (D6).**
- Ambiguity signals (2+ = ambiguous): intermittent/flaky/random/sporadic wording;
  multiple plausible root-cause areas; generic/missing error; prior reverted fixes in `git log`.
- `--competing` → Path A always. `--serial` → Path B always.
- Else: Path A if ambiguous, Path B otherwise.
- **Emit the routing decision + reason** so the 3-lane cost of Path A is visible.
- Persist `path`.

**Phase 2 — Investigate (read-only).**
- **Path A (competing):** run `/trace` orchestration — 3 `tracer` lanes, deliberately
  different hypotheses, evidence for/against, rebuttal, synthesis picks the winner.
  (Dispatch backend = teams via `/trace` for v1, D3.)
- **Path B (standard):** spawn one `debugger` agent (scientific method).
- Model selection follows OMC's per-role defaults — `tracer`/`debugger` run at their
  own default tier (no global effort knob; see §13A R2). Escalate a specific agent to
  `opus` only when the subtask is genuinely hard.
- Persist ALL hypotheses (confirmed + rejected) and `root_cause`.
- Confident root cause → `status=fixing`, go to Phase 3.
- No confident root cause → stay `investigating`; loop re-injects for the next
  hypothesis (or `--resume`). Never fabricate a fix.

**Phase 3 — Fix.**
- Spawn `executor` (default tier; `opus` for complex work) with the root cause + minimal
  fix. It applies the change and commits `fix({scope}): {desc}` using OMC's git-trailer
  protocol (D7: `Constraint:`/`Rejected:`/`Confidence:`/`Scope-risk:`).
- Persist `changed_files`, `commit`. Set `status=verifying`.

**Phase 4 — Verify (always delegate, D1).**
- Delegate to `/ultraqa` — never inline — with the goal scoped to the changed files
  plus a repro of the original bug (default `--tests`). Read back its verdict.
- When `verify_interactive` (`--interactive`) is set, forward `--interactive` to the
  `/ultraqa` call so its verify phase uses `qa-tester` for interactive CLI/service
  testing. (This is `ultraqa`'s own flag, passed through — not the completion gate,
  which is `--pause`.)
- PASS → `status=complete`, go to Phase 5.
- FAIL/exhausted → back to `status=investigating` with the failure context prepended
  to the next investigation prompt; increment `iteration`; loop.
- `--no-verify` → skip straight to `status=complete` after the fix (records `verify: skipped`).

**Phase 5 — Complete.**
- **User acceptance gate (only when `pause_before_complete`).** Before declaring done,
  present ONE `AskUserQuestion`: does the original bug still reproduce / any visible
  regression / is the fix right from your view? Accept → continue. Reject (with notes) →
  `status=investigating` with the note as failure context; increment `iteration`; loop.
  (This reuses OMC's autopilot pause pattern — a single default-off confirmation gate —
  rather than a bespoke UAT phase; see §13A R1.)
- Emit the result summary (mode + backend, issue, root cause, outcome, files, commit).
- Print `ULTRADEBUG_COMPLETE`.
- Delete `.omc/state/sessions/{sessionId}/ultradebug-state.json` (never leave `active:false`).

</Phases>

<Exit_Conditions>
| Condition | Action |
|---|---|
| Repro passes + `/ultraqa` PASS (+ user accept if `--pause`) | `complete` → print `ULTRADEBUG_COMPLETE`, delete state |
| `iteration >= max_iterations` | STOP: report the strongest hypothesis + evidence + remaining unknowns; keep state for `--resume` |
| Same verify failure 3× | STOP early: surface the fundamental blocker; keep state |
| No confident root cause | stay `investigating`; loop or STOP with `--resume` hint |
| User `/oh-my-claudecode:cancel` | clear state, stop |
</Exit_Conditions>

<Rules>
1. Author/review separation: investigators (`tracer`/`debugger`) are read-only; only
   `executor` mutates and commits. Verification is a separate lane (`/ultraqa`).
2. Never claim done without evidence in the transcript — `/ultraqa` PASS is the gate.
3. Minimal fixes only: root cause, not symptom; add/adjust a regression test.
4. Preserve every hypothesis (audit trail). Log the Path A routing choice + cost.
5. One bug per session. Do NOT reuse a session for a second bug: the driver's
   completion scan reads the transcript tail, so a prior run's standalone
   `ULTRADEBUG_COMPLETE` line could false-complete a new run started in the same
   session. Start a fresh session (or `--resume` the intended one) instead.
</Rules>

<Cancellation>
`/oh-my-claudecode:cancel` clears ultradebug state. On `complete`, delete the session file.
</Cancellation>
