---
name: batch-runner
description: 批量任务执行引擎，每个任务在独立subagent中执行，保证上下文完全隔离。支持调用指定skill、并行/串行调度、任务依赖。触发场景：用户说"批量执行"、"并行跑这些任务"、"帮我同时处理这几个事情"、"batch run"、"跑一批任务"、"对这个列表里每一个都执行XX"、提供了tasks.json或任务列表文件、或者一次性描述了多个需要分别处理的独立任务。即使用户只说"帮我同时做这几件事"也应触发。不触发场景：单个任务、需要严格顺序的流水线、交互式对话。
---

# Batch Runner — 上下文隔离的批量任务执行引擎

## 核心设计原则

**每个任务 = 一个独立subagent = 一个干净上下文。**

这是本框架存在的根本原因：当你在同一个对话中连续执行多个任务时，前面任务的输出、工具调用历史、中间状态会污染后续任务的上下文，导致skill指令逐渐失效。通过为每个任务启动独立的subagent，每个任务都从零开始，只看到自己的skill指令和任务输入，彻底消除上下文串扰。

## 工作流程

### Step 1: 解析任务来源

支持两种输入方式，自动识别：

**方式A：任务列表文件**

用户提供文件路径（JSON格式），读取并解析。格式参见 `references/task-schema.md`。

**方式B：用户直接描述**

用户在对话中直接描述多个任务。你需要：
1. 从用户描述中拆解出每个独立任务
2. 为每个任务生成 `id`（简短英文slug）
3. 识别用户是否要求对每个任务调用特定skill
4. 识别任务间的依赖关系
5. 将解析结果展示给用户确认：

```
| # | ID | 任务摘要 | 调用Skill | 执行模式 |
|---|-----|---------|----------|---------|
| 1 | analyze-market-a | 分析A市场竞争格局 | business-insight | 串行 |
| 2 | analyze-market-b | 分析B市场竞争格局 | business-insight | 串行 |
| 3 | compare-ab | 对比A和B市场结论 | - | 串行(依赖1,2) |
```

等用户确认后执行。用户说"直接跑"则跳过确认。

### Step 2: 构建执行计划

**调度策略选择**：

根据任务特性自动选择调度策略，也可由用户或任务文件显式指定：

- **parallel（并行）**：无依赖的独立任务，同时启动多个subagent。适用于互不干扰的research/分析类任务。单批最多 **8** 个并发。
- **serial（串行）**：逐个执行，前一个完成后再启动下一个。适用于需要保证顺序、或任务间有文件系统级交互的场景。
- **wave（分层）**：按依赖关系分批，同层并行、跨层串行。适用于有DAG依赖的任务集。

**默认策略**：如果任务间无依赖且无冲突风险 → parallel；如果有依赖 → wave；如果用户明确要求逐个执行 → serial。

### Step 3: 构造subagent prompt（关键步骤）

这是保证skill正确执行的核心。每个subagent的prompt必须是**自包含的**，包含执行该任务所需的全部信息，不依赖主对话的任何上下文。

**Prompt构造规则**：

#### 3a. 需要调用skill的任务

当任务指定了 `skill` 字段时，你需要：

1. **读取目标skill的SKILL.md**：在派发前，先读取 `~/.claude/skills/{skill_name}/SKILL.md` 的完整内容
2. **将skill指令内嵌到subagent prompt中**：不要依赖subagent自己去触发skill，而是直接把skill的指令作为subagent prompt的一部分注入

构造模板：

```
你是一个独立的执行代理，负责完成一个特定任务。请严格遵循以下工作指令。

===== 工作指令（来自 {skill_name} skill）=====

{SKILL.md的完整内容，去掉frontmatter}

===== 本次具体任务 =====

任务ID：{task.id}
任务描述：{task.prompt}
工作目录：{task.cwd}
输入文件：{task.input_files}

===== 输出要求 =====

1. 将所有产出保存到：{output_dir}/{task.id}/
2. 完成后在 {output_dir}/{task.id}/result.json 写入：
   {{
     "task_id": "{task.id}",
     "status": "success" 或 "failed",
     "summary": "一句话描述完成了什么",
     "outputs": ["产出文件路径列表"],
     "error": null 或 "错误信息"
   }}

{task.extra_instructions}
```

这样做的原因：subagent是全新的上下文，它不会自动看到主对话中的skill列表，也不会自动触发skill。把skill指令直接嵌入prompt，等价于让subagent"天生就知道"这个skill的全部知识，确保每个任务都100%遵循skill规范。

#### 3b. 不需要调用skill的普通任务

```
你是一个独立的执行代理，负责完成一个特定任务。

===== 任务 =====

任务ID：{task.id}
任务描述：{task.prompt}
工作目录：{task.cwd}
输入文件：{task.input_files}

===== 输出要求 =====

1. 将所有产出保存到：{output_dir}/{task.id}/
2. 完成后在 {output_dir}/{task.id}/result.json 写入：
   {{
     "task_id": "{task.id}",
     "status": "success" 或 "failed",
     "summary": "一句话描述完成了什么",
     "outputs": ["产出文件路径列表"],
     "error": null 或 "错误信息"
   }}

{task.extra_instructions}
```

#### 3c. 带前置任务产出的任务（有依赖）

如果任务依赖其他已完成任务的输出，在prompt中追加前置任务的结果摘要：

```
===== 前置任务结果（只读参考）=====

任务 "{dep_task.id}" 的结果：
- 状态：{dep_task.status}
- 摘要：{dep_task.summary}
- 产出文件位置：{dep_task.outputs}

你可以读取上述产出文件获取详细信息。
```

### Step 4: 派发执行

**派发规则**：

1. **创建输出目录**：`mkdir -p {output_dir}/{task.id}/`
2. **启动subagent**：使用 Agent 工具，参数设置：
   - `prompt`：按Step 3构造的完整prompt
   - `subagent_type`：`"general-purpose"`（需要完整工具集来执行skill）
   - `run_in_background`：parallel模式下为 `true`，serial模式下为 `false`
   - `isolation`：如果任务指定 `isolation: true`，设为 `"worktree"`
   - `description`：`"batch-{task.id}"`
3. **并发控制**：parallel模式下，单批最多8个subagent。一条消息中可以包含多个Agent tool call来并行启动。

**serial模式的执行循环**：
```
for each task in task_list:
    1. 构造prompt（包含skill指令 + 任务输入 + 前置结果）
    2. 启动subagent（run_in_background: false，同步等待完成）
    3. 读取 result.json，记录结果
    4. 向用户报告进度："任务 3/10 完成：{summary}"
    5. 启动下一个任务（全新subagent，干净上下文）
```

### Step 5: 收集结果与进度报告

**任务完成时**：
1. 从通知中记录 `total_tokens` 和 `duration_ms`
2. 读取 `{output_dir}/{task.id}/result.json`
3. serial模式下实时报告进度

**全部完成后汇总**：

控制台输出：
```
## 批量执行结果

| # | 任务 | Skill | 状态 | 耗时 | 摘要 |
|---|------|-------|------|------|------|
| 1 | analyze-market-a | business-insight | ✅ | 32s | 完成A市场深度分析 |
| 2 | analyze-market-b | business-insight | ✅ | 28s | 完成B市场深度分析 |
| 3 | compare-ab | - | ❌ | 5s | 缺少输入数据 |

成功: 2/3 | 失败: 1/3 | 跳过: 0/3
总耗时: 65s（并行实际32s）| 总Token: 185,432
```

结果文件保存到 `{output_dir}/batch-summary.json`：
```json
{
  "batch_id": "batch-{timestamp}",
  "strategy": "serial|parallel|wave",
  "total": 3,
  "success": 2,
  "failed": 1,
  "skipped": 0,
  "tasks": [
    {
      "task_id": "analyze-market-a",
      "skill": "business-insight",
      "status": "success",
      "duration_ms": 32000,
      "tokens": 85000,
      "summary": "完成A市场深度分析",
      "outputs": ["batch-output/analyze-market-a/report.md"]
    }
  ],
  "total_duration_ms": 65000,
  "total_tokens": 185432
}
```

**失败处理**：
- 汇总中高亮失败任务和错误原因
- 询问用户是否重试失败任务
- 重试时只派发失败的任务（仍然是独立subagent）
- 如果被依赖的任务失败，依赖它的任务标记为 `skipped`

## 关键设计决策

### 为什么把skill指令内嵌到prompt而不是让subagent自己触发skill？

三个原因：
1. **可靠性**：subagent的skill触发依赖Claude的判断，可能不触发或触发错误的skill。直接注入消除了这个不确定性。
2. **上下文纯净**：subagent只看到一份skill指令 + 一个任务，不会被其他任务的残留信息干扰。
3. **一致性**：10个任务用同一个skill，每个subagent拿到的skill指令完全相同，执行行为一致。

### 为什么serial模式是逐个启动subagent而不是在同一个对话中循环？

因为同一个对话中的循环执行会累积上下文。第5个任务执行时，前4个任务的工具调用记录、中间产物、错误信息都在上下文中，会干扰第5个任务对skill指令的遵循。每个任务用独立subagent，上下文从零开始。

## 默认配置

- **输出目录**：`./batch-output/`
- **最大并发**：8个subagent
- **超时**：单任务默认10分钟
- **调度策略**：无依赖→parallel，有依赖→wave，用户指定→serial
