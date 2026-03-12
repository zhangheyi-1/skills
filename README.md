# batch-runner-skill

Claude Code Skill：上下文隔离的批量任务执行引擎。

## 解决什么问题

在 Claude Code 中对同一个 skill 批量执行多个任务时，由于所有任务共享同一个对话上下文，前面任务的工具调用历史、中间产物会逐渐污染后续任务的上下文，导致 skill 指令不再被严格遵循。

**batch-runner** 通过为每个任务启动独立的 subagent 来解决这个问题：
- 每个任务在全新的上下文中执行
- Skill 指令直接内嵌到 subagent prompt 中，不依赖触发机制
- 任务完成后 subagent 销毁，不会累积上下文

## 核心特性

- **上下文隔离**：每个任务独立 subagent，彻底消除上下文串扰
- **Skill 注入**：将指定 skill 的完整指令嵌入 subagent prompt，确保 100% 遵循
- **灵活输入**：支持 JSON 任务列表文件或对话中直接描述
- **三种调度模式**：parallel（并行）/ serial（串行）/ wave（按依赖分层）
- **结果汇总**：自动收集所有任务结果并生成汇总报告
- **失败重试**：失败任务可选择性重新派发

## 安装

将 `SKILL.md` 和 `references/` 目录复制到 Claude Code 的 skills 目录：

```bash
# 方式1：直接复制
cp -r batch-runner-skill ~/.claude/skills/batch-runner

# 方式2：clone后复制
git clone <repo-url> /tmp/batch-runner-skill
cp -r /tmp/batch-runner-skill ~/.claude/skills/batch-runner
```

## 使用方式

### 方式1：任务列表文件

创建 `tasks.json`：

```json
{
  "tasks": [
    {
      "id": "insight-saas",
      "prompt": "深度分析中国企业级SaaS市场",
      "skill": "business-insight"
    },
    {
      "id": "insight-ai",
      "prompt": "深度分析AI基础设施市场",
      "skill": "business-insight"
    }
  ]
}
```

然后在 Claude Code 中：
```
/batch-runner 请执行 tasks.json 中的任务
```

### 方式2：直接描述

```
帮我用 tech-insight 分别深度分析 Redis 的持久化机制、Kafka 的分区策略、etcd 的 Raft 实现
```

### 方式3：混合 skill

```json
{
  "tasks": [
    {
      "id": "redis-deep-dive",
      "prompt": "深度拆解Redis持久化机制",
      "skill": "tech-insight"
    },
    {
      "id": "cache-compare",
      "prompt": "对比Redis和Memcached用于会话缓存",
      "skill": "tech-evaluation"
    }
  ]
}
```

## 任务文件格式

详见 [references/task-schema.md](references/task-schema.md)

### 关键字段

| 字段 | 必填 | 说明 |
|------|------|------|
| `id` | 是 | 唯一标识（英文slug） |
| `prompt` | 是 | 任务描述 |
| `skill` | 否 | 要调用的 skill 名称 |
| `dependencies` | 否 | 依赖的其他任务 ID |
| `isolation` | 否 | 是否使用 git worktree 隔离 |
| `strategy` | 否 | 调度策略：auto/parallel/serial/wave |

## 架构设计

```
主对话（batch-runner 编排层）
│
├─ 读取 tasks.json
├─ 读取目标 skill 的 SKILL.md
├─ 为每个任务构造自包含 prompt：
│   ┌─────────────────────────┐
│   │ Subagent Prompt         │
│   │ ├─ skill 指令（全文注入）│
│   │ ├─ 本次任务描述         │
│   │ └─ 输出格式要求         │
│   └─────────────────────────┘
├─ 启动独立 subagent 执行
└─ 收集结果，生成汇总报告
```

## License

MIT
