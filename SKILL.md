---
name: fable-relay
description: >
  Anti-downgrade subagent dispatch ("downgrade relay") for Claude Code. When your main
  session gets silently downgraded from a premium model (e.g. Fable 5 -> Opus 4.8 under
  capacity pressure), this skill has the (downgraded) main session summarize the day's
  task faithfully, then dispatch the work to a BARE unnamed subagent pinned to the premium
  model — empirically far more resistant to mid-run downgrade than named teammate spawns —
  and verify the actually-served model from the transcript before claiming success.
  Triggers: "fable relay", "anti-downgrade dispatch", "relay to fable", "must be done by
  fable", 防降级派车, 降级接力, 派fable子代理接着干, 让fable接力.
  Hard rule: once triggered, the main session is a PURE DISPATCHER — no pre-diagnosis, no
  "quick look at the logs first"; diagnosis is part of what gets relayed.
---

# fable-relay — Anti-downgrade subagent dispatch

When the main session's model has been silently downgraded, hand the day's work to a
**downgrade-resistant premium-model subagent** instead of grinding on with the fallback model.

Core empirical finding (2026-07-12, n=11 subagents across 3 conversations, same account):
**downgrade risk is not about the dispatch channel — it is about *naming*.** Subagents
spawned WITH a `name` parameter (teammate / in-process agents) downgraded mid-run in 4 of 5
cases (all within their first 6-9 messages). Subagents spawned WITHOUT a name — plain bare
`Agent` tool spawns or Workflow `agent()` workers — stayed on the requested premium model in
6 of 6 cases, including a 117-message marathon that ran clean through the worst capacity
window while named siblings dispatched minutes apart were being swapped. Full evidence table
in the repo README.

## When to trigger

- The user explicitly asks: "fable relay" / "anti-downgrade dispatch" / "this must be done
  by <premium model>" / 防降级派车 / 降级接力.
- Or the user reports the main session has been downgraded. Note: the main session **cannot
  perceive its own downgrade** — `settings.json` reflects the *requested* model, not the
  served one. Only per-message `message.model` in the transcript is ground truth. Trust the
  user, not your config.

## Procedure (main session follows this even if it is itself the fallback model)

> Whether to relay at all is the **user's call** — invoking this skill IS the decision, so
> don't second-guess it or lecture about overhead. (Advice for humans on when a relay is
> worth its ~20-50K fresh-context overhead lives in the README fine print, not here.)

### Step 0 — You are a dispatcher, not a detective

Once this skill triggers, the main session's job shrinks to four things: **summarize,
quote, dispatch, verify**.

- **MUST NOT do any substantive diagnosis / analysis / debugging / fixing before
  dispatching** — including the very reasonable-sounding "let me just take a quick look at
  the logs first to save the subagent some tokens" (real incident, 2026-07-12: a session
  triggered this skill and still started its own bug diagnosis; the user had to stop it).
  Two reasons: the downgraded model's pre-diagnosis is exactly the quality you're trying to
  avoid, and a wrong first guess anchors the premium model's judgment. Token-saving is not
  a reason — the user manages quota.
- The brief carries **raw materials only**: the user's verbatim words, error text already
  visible in the chat, file/log paths, repro steps — never the main session's own analysis
  or conclusions. **Diagnosis is itself part of the relayed task** — put it on the
  subagent's task list ("first read the logs at X to find why the model connection fails,
  then fix it").
- Boundary: facts *already established* in the conversation before the trigger (what's
  done, which files were touched, what the user decided) belong in a faithful summary —
  include them. Any *new* investigation after the trigger is out of bounds — stop.

### Step 1 — Self-summarize the task (fidelity rule)

Review the conversation and write a one-shot brief for the relay subagent. The brief MUST:

- Quote the user's **original prompt and key follow-up instructions verbatim** (in quotes,
  labeled as the user's original words). Your paraphrase is supplementary only; on any
  conflict the subagent follows the original words.
- Include full context: absolute file paths, done/not-done boundaries, hard constraints
  (do-not-touch), expected deliverable format.
- Be self-contained: the subagent is a fresh context and knows nothing from this chat.
- **Carry pointers, not payloads (token-saving rule #1):** the brief holds exactly three
  kinds of material — verbatim user quotes, file/log **paths**, and error text already
  visible in the chat. MUST NOT paste file bodies or long documents into it; the subagent
  reads from the paths itself, and only what it actually needs. Healthy size:
  **≤ 2,500 characters** (real numbers: one fat brief that quoted documents in full burned
  517,883 uncached input tokens in a single run, while same-day pointer-style briefs of
  1.8-2.4K characters carried full-size tasks just fine).
- **Write the brief to a file and reuse it:** save it as
  `~/.claude/tmp/relay-brief-<YYYYMMDD>-<topic>.md`; the spawn prompt is then just the
  guard marker + one or two lines of task framing + the brief's file path. If the run
  downgrades or needs a respawn, point the fresh agent at the same file — don't rewrite
  the brief.

### Step 2 — Dispatch rules (the core)

Spawn a **bare, unnamed** subagent:

```
Agent({
  model: "<premium-model>",          // e.g. "fable"
  subagent_type: "general-purpose",
  prompt: "<optional guard marker> <brief>",
  description: "<3-5 words>"
})
```

- **MUST NOT pass a `name` parameter.** Naming creates a teammate (in-process team
  infrastructure); empirically 4/5 named spawns downgraded mid-run, 6/6 unnamed spawns
  stayed clean. One dispatch = one bare agent; collect and dismiss.
- **Batch the day's items:** pack accumulated to-dos into ONE brief for ONE agent (working
  example: five review items relayed as a single package, finished by a single agent) —
  never one agent per item; every extra spawn re-pays the ~20-50K fresh-context overhead.
- MUST NOT use SendMessage team-coordination patterns for the relay.
- If you run a model-guard PreToolUse hook (as the author does), put its approval marker
  (e.g. `USER-APPROVED-FABLE`) at the head of the prompt. Harmless noise otherwise.
- If reasoning effort must be pinned too, use a single-agent Workflow instead:
  `agent(prompt, {model: 'fable', effort: 'xhigh'})` — the inline Agent tool cannot pin
  effort. Both unnamed channels tested 100% clean.
- Require in the brief that the subagent **writes its deliverables to disk** (survives any
  relay loss), plus a final report message.
- **No round-trips (token-saving rule #2):** dispatch it and let it run to completion — do
  not exchange messages with the relay agent mid-run; every round-trip burns a turn on both
  sides. A small gap in the finished result → the main session patches it itself; a big gap
  → edit the brief file and respawn fresh, never follow up with the old agent.

### Step 3 — Verify the served model before claiming success

After the subagent finishes:

```bash
# transcript: <session-dir>/subagents/agent-*.jsonl
# (Workflow spawns: <session-dir>/subagents/workflows/wf_*/agent-*.jsonl)
grep -o '"model":"[^"]*"' <agent-transcript>.jsonl | sort | uniq -c
```

- All premium-model messages → report "clean run, N messages on <model>".
- Any fallback-model messages → say so plainly: at which message it downgraded, and which
  on-disk writes happened after the switch (match tool_use entries to the timeline).
  **Never** cite `settings.json`, team config, or the subagent's self-report as evidence —
  per-message `message.model` is the only ground truth.
- A downgraded run is respawned as a fresh bare agent by default (do not resume the
  downgraded one), unless the user accepts the mixed result.

### Step 4 — Rate-limit errors

This skill does no quota prediction (the user manages quota). On a "session limit / resets"
error: relay the error and reset time verbatim, then wait for instructions. Never silently
swap models or pass off fallback output as premium work.

## Known boundaries

- Works only while the premium model is callable on your plan at all. During a full
  delisting, pinning it falls back 100% — say so honestly instead of pretending.
- n=11 empirical pattern, single account, specific dates — not an official guarantee. If a
  bare spawn does downgrade, report it truthfully and record the counter-example.
- **This is a quality tool, not a money-saver.** Losing the prompt cache is inherent to the
  relay pattern; the economy gate, pointer-style briefs, and batching only compress the
  waste back down to the 20-50K base overhead — never to zero. If quota is tight and the
  task doesn't actually need the premium tier, not relaying (letting the main session do
  it) is the optimal move.
