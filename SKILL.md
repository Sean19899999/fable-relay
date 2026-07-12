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

### Step 1 — Self-summarize the task (fidelity rule)

Review the conversation and write a one-shot brief for the relay subagent. The brief MUST:

- Quote the user's **original prompt and key follow-up instructions verbatim** (in quotes,
  labeled as the user's original words). Your paraphrase is supplementary only; on any
  conflict the subagent follows the original words.
- Include full context: absolute file paths, done/not-done boundaries, hard constraints
  (do-not-touch), expected deliverable format.
- Be self-contained: the subagent is a fresh context and knows nothing from this chat.

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
  stayed clean. One task = one bare agent; collect and dismiss.
- MUST NOT use SendMessage team-coordination patterns for the relay.
- If you run a model-guard PreToolUse hook (as the author does), put its approval marker
  (e.g. `USER-APPROVED-FABLE`) at the head of the prompt. Harmless noise otherwise.
- If reasoning effort must be pinned too, use a single-agent Workflow instead:
  `agent(prompt, {model: 'fable', effort: 'xhigh'})` — the inline Agent tool cannot pin
  effort. Both unnamed channels tested 100% clean.
- Require in the brief that the subagent **writes its deliverables to disk** (survives any
  relay loss), plus a final report message.

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
