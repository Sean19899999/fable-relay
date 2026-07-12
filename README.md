# fable-relay

Keep your Claude Code work on the good model — even after your session quietly stops using it.

让你的活儿留在高档模型上干完——哪怕你的会话已经悄悄"变笨"了。

---

## English

### Ever feel like Claude suddenly got dumber mid-conversation?

You might not be imagining it. When capacity gets tight, Claude Code can quietly swap the
model serving your session — you picked Fable 5, your settings still say Fable 5, even the
model itself believes it's Fable 5. But behind the scenes, every reply is coming from
Opus 4.8. There's no banner, no warning. The only place the truth lives is a little
`"model"` field buried in your session's log file.

I found this out the hard way: I spent a day dispatching "Fable 5 subagents" from a session
that had already been downgraded, and kept getting results that were... fine, but not
*Fable* fine. So I dug through the logs of three days of conversations — eleven subagents in
total — to figure out which ones actually stayed on Fable 5, and why.

### The surprising part

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

### So this skill does three things for you

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

### Install

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

### The honest fine print

This is a pattern from eleven subagents, one account, two days, one model pair. It's real
evidence, not a guarantee — if you hit a case where a bare spawn downgrades anyway, I'd
genuinely love to hear about it in an issue. And of course, none of this helps if the
premium model isn't available on your plan at all; the skill is designed to tell you that
honestly rather than pretend. It doesn't bypass quotas, and it doesn't bypass safety —
it only changes how the work gets handed off.

---

## 中文

### 有没有觉得 Claude 聊着聊着突然变笨了？

不一定是错觉。容量紧张的时候，Claude Code 会悄悄换掉给你干活的模型——你选的是 Fable 5，
设置里写的还是 Fable 5，连模型自己都以为自己是 Fable 5，但实际上每条回复都是 Opus 4.8
出的。没有任何提示。真相只藏在会话日志文件里一个不起眼的 `"model"` 字段里。

我是踩了坑才发现的：有一天我从一个（其实已经被降级的）会话里连派了好几个"Fable 5 子代理"，
拿回来的活总觉得……能用，但不是 Fable 该有的水平。于是我把三天的对话日志翻了个底朝天——
一共 11 个子代理——逐条核对到底谁真的是 Fable 5 干的，谁半路被换了人。

### 意外的发现

跟系统忙不忙关系不大，跟一个小细节关系极大：**派子代理的时候，给没给它起名字。**

凡是用"组队"方式派出去、带了 `name` 参数的子代理，5 个里有 4 个半路被静默换成了降级模型，
基本都撑不过开头几条消息。而**裸派**的——不起名、不组队、把任务简报一次给全就让它去干的——
**6 个全程都是纯 Fable 5**。最夸张的对照：在容量最紧张的时间窗里，一个裸派子代理连跑 117
条消息全程纯 Fable，而同一账号、就差几分钟派出去的命名子代理，一分钟内就被换掉了。

服务端我看不到，机制只能推测（命名组队似乎会共享主会话已降级的状态，裸派则是独立重新分配）。
但日志里的规律非常硬。

### 所以这个 skill 帮你干三件事

装好之后，发现会话被降级了，你只要说一句 **「防降级派车」** 或 **「降级接力」**（英文说
"fable relay" 也行），它就会：

1. **替你写好交接简报。**（已经降级的）主会话自己总结你今天在干什么——但必须一字不改地
   引用你的原话，它自己的转述只能当补充，保证活儿不走样；
2. **用抗降级的方式把活派出去。** 裸派一个钉住高档模型的子代理，要求成果直接写进文件——
   完全照着日志里 6 战 6 胜的那套派法；
3. **交活前先验货。** 跑完以后去查子代理的日志，看每条消息到底是谁答的。如果半路被降级了，
   直接告诉你：从第几条开始降的、降级之后写了哪些文件——而不是把降级货贴上高档标签糊弄你。

### 安装

```bash
git clone https://github.com/Sean19899999/fable-relay ~/.claude/skills/fable-relay
```

装完就行，Claude Code 会自动识别。默认加载的 `SKILL.md` 是英文版，旁边的
`SKILL.zh-CN.md` 是完整中文版——想让 Claude 读中文版，把两个文件名对调一下就好。

（小注：skill 里提到的 `USER-APPROVED-FABLE` 是我自己机器上一个权限 hook 的放行暗号。
你没装这种 hook 的话，它就是几个无害的多余单词，留着删掉都行。）

### 丑话说在前面

这是 11 个子代理、单账号、两天、一对模型（Fable 5 / Opus 4.8）跑出来的规律——是真实证据，
但不是保证。如果你遇到裸派也被降级的反例，真心欢迎开 issue 告诉我。另外，高档模型压根不在
你订阅里的时候，这招也无能为力——skill 会如实告诉你，而不是硬装。它不绕额度、不绕安全，
只改变交接活儿的方式。

---

## License

MIT
