**English** | [简体中文](README.zh-CN.md)

# fable-relay

Keep your Claude Code work on the good model — even after your session quietly stops using it.

## Ever feel like Claude suddenly got dumber mid-conversation?

You might not be imagining it. When capacity gets tight, Claude Code can quietly swap the
model serving your session — you picked Fable 5, your settings still say Fable 5, even the
model itself believes it's Fable 5. But behind the scenes, every reply is coming from
Opus 4.8. There's no banner, no warning. The only place the truth lives is a little
`"model"` field buried in your session's log file.

I found this out the hard way: I spent a day dispatching "Fable 5 subagents" from a session
that had already been downgraded, and kept getting results that were... fine, but not
*Fable* fine. So I dug through the logs of three days of conversations — eleven subagents in
total — to figure out which ones actually stayed on Fable 5, and why.

## The surprising part

It had almost nothing to do with *how busy* the system was, and everything to do with one
tiny detail: **whether the subagent was given a name.**

Subagents I spawned as named "teammates" (the team-style dispatch with a `name` parameter)
got silently swapped to the fallback model four times out of five — usually within their
first few messages. Subagents spawned *bare* — no name, no team, just "here's your brief,
go" — stayed on Fable 5 **six times out of six**. The most dramatic case: during the worst
capacity crunch, a bare subagent ran 117 messages straight on pure Fable 5, while named
teammates launched minutes apart from the same account were downgraded almost immediately.

I can't see Anthropic's servers, so the mechanism is educated guesswork (named teammates
seem to share the parent session's degraded state; bare spawns get a fresh die roll). But
the pattern in the logs is hard to argue with.

## So this skill does three things for you

Once installed, when you notice your session has been downgraded, you just say
**"fable relay"** (or in Chinese, 防降级派车 / 降级接力). Then:

1. **It writes the handoff brief for you.** The (downgraded) main session summarizes what
   you were working on — and it must quote *your original words verbatim*, not its own
   paraphrase, so nothing gets lost in translation.
2. **It dispatches the work the resilient way.** A bare, unnamed subagent pinned to the
   premium model, told to write its results to actual files on disk — following exactly the
   pattern that went 6-for-6 in the logs.
3. **It checks the receipts before declaring victory.** After the run, it greps the
   subagent's log for which model *actually* answered each message. If the run got
   downgraded halfway, it tells you straight — at which message, and which files were
   written after the swap — instead of quietly handing you fallback work with a premium
   label on it.

And one thing it deliberately *won't* do: let the downgraded main session "just take a
quick look at the logs first". Diagnosis is part of the work being relayed — the whole
point is that the good model does the thinking, and a wrong first guess from the fallback
model would only lead the good one astray.

## Install

```bash
git clone https://github.com/Sean19899999/fable-relay ~/.claude/skills/fable-relay
```

That's it. Claude Code picks it up automatically; trigger it by saying "fable relay" or
"anti-downgrade dispatch". The loaded `SKILL.md` is in English; a full Chinese edition
(`SKILL.zh-CN.md`) sits right next to it — swap the filenames if you'd rather Claude read
the Chinese one.

(One small note: the skill mentions a `USER-APPROVED-FABLE` marker. That's the password for
a permission hook I run on my own machine. If you don't have such a hook, it's just a few
harmless extra words in the prompt.)

## The honest fine print

This is a pattern from eleven subagents, one account, two days, one model pair. It's real
evidence, not a guarantee — if you hit a case where a bare spawn downgrades anyway, I'd
genuinely love to hear about it in an issue. And of course, none of this helps if the
premium model isn't available on your plan at all; the skill is designed to tell you that
honestly rather than pretend. It doesn't bypass quotas, and it doesn't bypass safety —
it only changes how the work gets handed off.

One more thing worth saying plainly: this is a quality tool, not a money-saver. A relayed
subagent starts from a blank context, so your session's prompt cache does nothing for it —
every relay pays a real chunk of tokens before any work happens. Keep the handoff brief
lean (quotes and file paths, not pasted documents), batch the day's items into one dispatch
instead of one agent per item — and if a task doesn't actually need the premium model, the
cheapest good answer is to just let the main session do it.

## License

MIT
