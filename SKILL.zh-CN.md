---
name: fable-relay
description: 主会话被静默降级（Fable5→Opus4.8）时的降级接力派车。用户说"防降级派车/降级接力/派fable子代理接着干/让fable接力/fable relay"时触发——由（已降级的）主会话自己总结当天任务、忠实引用用户原话，按防降级铁律派 Fable5 裸子代理续活，完工后 transcript 验明正身。
---

# fable-relay — 降级接力派车

主会话模型被静默降级后，把当天的活交给**不易降级的 Fable5 子代理**接力完成。核心发现（2026-07-12 三对话取证，n=11）：**降级风险不在通道，在"组队"——带 `name` 的 teammate 派法 4/5 中途降级；不带 name 的裸子代理 6/6 全程纯 Fable**（含压力窗口 117 条马拉松）。证据明细 → auto-memory `model-fallback-forensics`。

## 触发条件

- 用户明说：防降级派车 / 降级接力 / 派 fable 子代理接着干 / 必须 fable 干
- 或用户发现主会话已降级（自查法：本 skill 不主动扫，用户说了才信；主会话**无法感知**自己被降级，settings.json 不算数）

## 执行流程（主会话照做，即使你自己是 Opus 4.8）

### Step 1 — 任务自总结（忠于原话铁律）

回看本次对话，总结当天要续的活。简报里 MUST：
- **一字不改引用用户最原始的 prompt 和后续关键指令**（引号括起，标"用户原话"）——你的转述只能作补充，两者冲突时子代理听原话；
- 列全上下文：涉及文件绝对路径、已完成/未完成边界、硬约束（do-not-touch）、产出格式；
- 一次给全（one-shot）：子代理是全新 context，不知道对话里的任何事。

### Step 2 — 防降级派车铁律

用 Agent 工具派**裸子代理**，参数照抄：

```
Agent({
  model: "fable",
  subagent_type: "general-purpose",
  prompt: "USER-APPROVED-FABLE —— 用户明确要求本活必须 Fable 5 干。<简报>",
  description: "<3-5词>"
})
```

- **MUST NOT 传 `name` 参数**（铁律核心——命名=组队teammate，实测 4/5 中途静默降级；裸派 6/6 纯 Fable）。
- **MUST NOT** 用 SendMessage 组队协作模式；一个活一个裸代理，跑完即收。
- prompt 开头必须带 `USER-APPROVED-FABLE`（过 agent-model-guard 闸门；用户点名 fable 即视为已批准）。
- 需要钉死 effort=xhigh 时改走 Workflow 单代理：`agent(prompt, {model:'fable', effort:'xhigh'})`，脚本带 `USER-APPROVED-WORKFLOW` 注释（Agent 工具钉不了 effort）。实测两通道均 100% 纯 Fable。
- 简报里要求子代理：**产出落盘到文件**（防中继丢失）+ final message 给报告。

### Step 3 — 完工验明正身（不许自称）

子代理跑完后 MUST：

```bash
# transcript 在 <会话目录>/subagents/agent-*.jsonl（Workflow 则在 subagents/workflows/wf_*/）
grep -o '"model":"[^"]*"' <最新的agent-*.jsonl> | sort | uniq -c
```

- 全部 `claude-fable-5` → 如实报"全程 Fable，N 条"。
- 出现 `claude-opus-4-8` → 明说降级了、降在第几条、哪些落盘写入是降级后干的（按 message 时序对 tool_use）。**禁止**拿 settings.json、team config 的 model 字段或子代理自称当证据——每条消息的 `message.model` 才是事实。
- 中途降级的活：默认重派一个新裸代理重跑（别在降级代理上续），除非用户说要了。

### Step 4 — 额度报错处理

本 skill 不做额度预判（用户人工掌握）。撞上 "session limit / resets" 报错时：原样报错误和重置时间给用户，等指令；不偷偷换模型重跑，不拿降级产出冒充 Fable 交差。

## 已知边界

- 有效前提 = Fable 在订阅可调池内；官方彻底下架期间钉 fable 会全量回退，本 skill 失效（如实告知用户）。
- n=11 的经验规律非官方保证；若实测失效（裸派也降级），如实报告并把新样本记入 auto-memory `model-fallback-forensics`。
