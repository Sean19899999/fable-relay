# fable-relay

**Anti-downgrade subagent dispatch for Claude Code** — when your session silently falls back
from a premium model (e.g. Fable 5 → Opus 4.8), relay the work to a *bare unnamed* subagent
pinned to the premium model, which empirically resists mid-run downgrade far better than
named teammate spawns. Then verify what model *actually* served the run before believing it.

**Claude Code 防降级派车 skill** —— 主会话被静默降级（如 Fable 5 → Opus 4.8）时，把活交给
钉住高档模型的「裸」子代理接力，实测比带名字的 teammate 派法抗降级得多；完工后从 transcript
验明正身，不信自称。

---

## English

### The problem

Under capacity pressure, Claude Code sessions can be **silently** downgraded: you requested
Fable 5, `settings.json` still says Fable 5, the model *believes* it is Fable 5 — but every
response is actually served by Opus 4.8. The only ground truth is the per-message
`"model"` field in the session transcript (`~/.claude/projects/<project>/<session>.jsonl`).

Worse: a downgraded main session that dispatches "Fable 5 subagents" may hand you
downgraded subagents too — or not, depending on *how* it dispatches them.

### The finding (n=11, 3 conversations, same account, 2026-07-11/12)

| Spawn style | Result |
|---|---|
| With `name` param (teammate) | 4 of 5 downgraded mid-run, at message 6-9 |
| Bare unnamed `Agent` spawn | 4 of 4 stayed clean on Fable 5 |
| Workflow `agent()` worker (inherently unnamed) | 2 of 2 stayed clean (one killed by hard rate limit — died as Fable, never silently swapped) |

The decisive contrast: during the *same* stressed window, a bare unnamed spawn ran a
**117-message marathon fully on Fable 5** (14:28→15:05), while named teammates dispatched at
14:46 and 14:51 were silently swapped to Opus 4.8 within their first minute. Same account,
same minutes — timing confounds excluded.

Mechanism (speculative, not provable from disk): named spawns join the in-process team
infrastructure and appear to share the lead session's degraded serving state; unnamed spawns
are independent sessions with independent model resolution.

### What the skill does

1. **Self-summarize with fidelity**: the (possibly downgraded) main session writes a
   one-shot brief, quoting the user's original prompts **verbatim** — its own paraphrase is
   supplementary only.
2. **Dispatch bare**: `Agent({model, subagent_type, prompt, description})` — **never** a
   `name` param, never SendMessage team choreography. One task, one bare agent. Pin effort
   via a single-agent Workflow if needed.
3. **Deliverables to disk**, not just chat.
4. **Verify before claiming**: `grep '"model"' agent-*.jsonl | sort | uniq -c` — report the
   actual served-model counts; if it downgraded, say at which message and which writes
   happened after the switch. Downgraded runs get respawned, not resumed.
5. **No quota games**: rate-limit errors are relayed verbatim; fallback output is never
   passed off as premium work.

### Install

```bash
git clone https://github.com/Sean19899999/fable-relay ~/.claude/skills/fable-relay
```

Then in any Claude Code session, say **"fable relay"** / **"anti-downgrade dispatch"**
(or in Chinese: 防降级派车 / 降级接力). `SKILL.md` is the English edition Claude loads;
`SKILL.zh-CN.md` is the full Chinese edition — swap file names if you prefer Chinese as
the loaded one.

Note: the `USER-APPROVED-FABLE` marker mentioned in the skill exists for the author's local
PreToolUse model-guard hook. Without such a hook it is harmless prompt noise; keep or drop.

### Honest caveats

- n=11, one account, two days, one model pair (Fable 5 / Opus 4.8). An empirical pattern,
  not an official guarantee. Counter-examples welcome — file an issue.
- Only works while the premium model is callable on your plan at all. During a full
  delisting, pinning it falls back 100%; the skill reports that instead of pretending.
- This is not a quota bypass and not a safety bypass. It changes *dispatch structure* only.

---

## 中文

### 问题

容量压力下 Claude Code 会**静默**降级：你点的是 Fable 5，`settings.json` 写的还是 Fable 5，
模型自己也**以为**自己是 Fable 5——但每条回复实际由 Opus 4.8 服务。唯一的事实来源是会话
transcript（`~/.claude/projects/<项目>/<会话>.jsonl`）里每条消息的 `"model"` 字段。

更麻烦的是：已降级的主会话去派「Fable 5 子代理」，派出来的可能也是降级货——**取决于怎么派**。

### 取证结论（n=11，三个对话，同一账号，2026-07-11/12）

| 派法 | 结果 |
|---|---|
| 带 name 参数（teammate 组队） | 5 个里 4 个中途降级，都在第 6-9 条 |
| 裸 Agent 派（不带 name） | 4/4 全程纯 Fable 5 |
| Workflow agent() 工人（天然无名） | 2/2 干净（其一被硬限额杀死——以 Fable 身份死掉，没被静默换） |

决定性对照：同一个压力窗口里，裸派子代理跑了 **117 条消息全程纯 Fable**（14:28→15:05），
而 14:46、14:51 派出的命名 teammate 一分钟内就被换成 Opus 4.8。同账号、同几分钟——时间
混杂已排除。

机制推测（磁盘证据无法证实）：命名派进了 in-process 组队设施、疑似与已降级的主会话共享服务
状态；裸派是独立会话、独立解析模型。

### skill 做什么

1. **忠于原话的任务自总结**：（可能已降级的）主会话写 one-shot 简报，**一字不改引用**用户
   原始 prompt，转述只作补充；
2. **裸派**：`Agent({model, subagent_type, prompt, description})`——**绝不**带 `name`、
   不搞 SendMessage 协作编排；一个活一个裸代理。要钉 effort 就走单代理 Workflow；
3. **产出落盘**，不只留在回话里；
4. **验明正身再交差**：`grep '"model"' agent-*.jsonl | sort | uniq -c`——报实际服务模型
   条数；降了就明说降在第几条、哪些写入是降级后干的。降级的活重派新裸代理，不续跑；
5. **不玩额度**：限额报错原样上报；绝不拿降级产出冒充高档模型交差。

### 安装

```bash
git clone https://github.com/Sean19899999/fable-relay ~/.claude/skills/fable-relay
```

之后在任意 Claude Code 会话里说 **「防降级派车」/「降级接力」**（或英文 "fable relay"）即可。
Claude 加载的是 `SKILL.md`（英文版）；`SKILL.zh-CN.md` 是完整中文版，想让 Claude 读中文版
就把两个文件名对调。

注：skill 里提到的 `USER-APPROVED-FABLE` 标记是作者本地 PreToolUse 模型守卫 hook 的放行
暗号；没装这类 hook 的话它只是无害的提示词噪音，留删随意。

### 诚实边界

- n=11、单账号、两天、单一模型对（Fable 5 / Opus 4.8）——经验规律，不是官方保证。欢迎提
  issue 补充反例。
- 前提是高档模型在你的订阅里可调；官方彻底下架期间钉了也会 100% 回退，skill 会如实报告而
  不是硬装。
- 这不是额度绕过、也不是安全绕过——它只改变**派车结构**。

---

## License

MIT
