---
name: fable-relay
description: 主会话被静默降级（Fable5→Opus4.8）时的降级接力派车。用户说"防降级派车/降级接力/派fable子代理接着干/让fable接力/fable relay"时触发——由（已降级的）主会话自己总结当天任务、忠实引用用户原话，按防降级铁律派 Fable5 裸子代理续活，完工后 transcript 验明正身。硬规矩：触发后主会话是纯调度员，MUST NOT 自己预诊断/预分析/先看日志——诊断本身也是派给子代理的任务。
---

# fable-relay — 降级接力派车

主会话模型被静默降级后，把当天的活交给**不易降级的 Fable5 子代理**接力完成。核心发现（2026-07-12 三对话取证，n=11）：**降级风险不在通道，在"组队"——带 `name` 的 teammate 派法 4/5 中途降级；不带 name 的裸子代理 6/6 全程纯 Fable**（含压力窗口 117 条马拉松）。证据明细 → auto-memory `model-fallback-forensics`。

## 触发条件

- 用户明说：防降级派车 / 降级接力 / 派 fable 子代理接着干 / 必须 fable 干
- 或用户发现主会话已降级（自查法：本 skill 不主动扫，用户说了才信；主会话**无法感知**自己被降级，settings.json 不算数）

## 执行流程（主会话照做，即使你自己是 Opus 4.8）

### Step 0 — 角色铁律：主会话只当调度员，不当侦探

触发本 skill 后，主会话的职责只剩四件：**总结、引原话、派车、验收**。

- **MUST NOT 在派车前自己做任何实质性诊断/分析/排查/修复**——包括"我先快速看看日志/查查根因，好省子代理额度"这类看似合理的开场（实际案例：2026-07-12 某会话触发本 skill 后仍先自行诊断 bug，被用户纠正）。理由有二：降级模型的预诊断质量不可信；错误结论会锚进简报污染 Fable 的判断（anchor bias）。省额度不是理由——额度由用户人工掌握。
- 简报里只放**原材料**：用户原话、对话里已出现的报错原文、文件/日志路径、复现步骤——不放主会话自己的分析结论。**诊断本身就是子代理任务的一部分**，写进任务清单（如"先读 X 处日志定位'连不上模型'的根因，再修"）。
- 边界：触发前对话里**已确立**的事实（做到哪了、哪些文件动过、用户之前拍板过什么）属于忠实总结，照常写进简报；触发后**新开**的调查属于越权，停手。

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
