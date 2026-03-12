# 任务列表文件格式 (tasks.json)

## 完整Schema

```json
{
  "output_dir": "./batch-output",        // 可选，全局输出目录，默认 ./batch-output
  "max_concurrency": 8,                  // 可选，最大并发数，默认 8
  "strategy": "auto",                    // 可选，调度策略：auto|parallel|serial|wave，默认auto
  "tasks": [
    {
      "id": "task-slug",                 // 必填，唯一标识，英文slug
      "prompt": "任务描述",               // 必填，subagent要执行的具体指令
      "skill": "skill-name",             // 可选，要调用的skill名称（对应 ~/.claude/skills/{name}/SKILL.md）
      "cwd": "/path/to/dir",            // 可选，工作目录，默认继承当前目录
      "input_files": ["a.py", "b.js"],   // 可选，需要读取的输入文件
      "dependencies": ["other-task-id"], // 可选，依赖的其他任务ID
      "isolation": false,                // 可选，是否使用git worktree隔离，默认false
      "timeout_ms": 600000,              // 可选，超时时间(毫秒)，默认600000
      "extra_instructions": ""           // 可选，追加给subagent的额外指令
    }
  ]
}
```

## 示例1：对多个市场执行同一个skill

所有任务调用同一个 `business-insight` skill，互不依赖，自动并行：

```json
{
  "tasks": [
    {
      "id": "insight-saas",
      "prompt": "深度分析中国企业级SaaS市场",
      "skill": "business-insight"
    },
    {
      "id": "insight-ai-infra",
      "prompt": "深度分析AI基础设施市场",
      "skill": "business-insight"
    },
    {
      "id": "insight-low-code",
      "prompt": "深度分析低代码平台市场",
      "skill": "business-insight"
    }
  ]
}
```

## 示例2：不同skill混合使用

```json
{
  "tasks": [
    {
      "id": "redis-deep-dive",
      "prompt": "深度拆解Redis的持久化机制，重点分析RDB和AOF的trade-off",
      "skill": "tech-insight"
    },
    {
      "id": "cache-comparison",
      "prompt": "对比Redis、Memcached、Dragonfly三个方案用于会话缓存场景",
      "skill": "tech-evaluation"
    }
  ]
}
```

## 示例3：串行执行 + 任务间依赖

前一个任务的产出作为后一个任务的输入：

```json
{
  "strategy": "serial",
  "tasks": [
    {
      "id": "scan-cves",
      "prompt": "扫描项目中使用的所有第三方依赖，列出已知CVE",
      "skill": "kernel-cve-analysis"
    },
    {
      "id": "fix-critical",
      "prompt": "根据 batch-output/scan-cves/result.json 中的高危CVE，修复受影响的依赖版本",
      "dependencies": ["scan-cves"]
    }
  ]
}
```

## 示例4：纯任务列表（不调用skill）

```json
{
  "tasks": [
    { "id": "count-py", "prompt": "统计 src/ 下所有 .py 文件的总行数" },
    { "id": "count-ts", "prompt": "统计 src/ 下所有 .ts 文件的总行数" },
    { "id": "count-go", "prompt": "统计 src/ 下所有 .go 文件的总行数" }
  ]
}
```

## 示例5：带隔离的并行代码修改

多个任务修改同一仓库的不同模块，使用worktree隔离避免冲突：

```json
{
  "tasks": [
    {
      "id": "refactor-auth",
      "prompt": "重构auth模块，将session改为JWT",
      "isolation": true
    },
    {
      "id": "refactor-db",
      "prompt": "将数据库查询从raw SQL迁移到ORM",
      "isolation": true
    }
  ]
}
```

## 字段说明

### strategy 字段

| 值 | 行为 | 适用场景 |
|----|------|---------|
| `auto` | 无依赖→parallel，有依赖→wave（默认） | 大多数场景 |
| `parallel` | 所有无依赖任务同时启动 | 互不干扰的独立任务 |
| `serial` | 逐个执行，前一个完成再启动下一个 | 需要严格顺序或观察中间结果 |
| `wave` | 按依赖DAG分层，层内并行，层间串行 | 有依赖关系的任务集 |

### skill 字段

- 值为skill目录名（如 `business-insight`、`tech-evaluation`）
- batch-runner会读取对应的 `~/.claude/skills/{skill}/SKILL.md`
- skill指令会被完整注入到subagent的prompt中，确保每个任务在干净上下文中严格遵循skill规范
- 如果不指定，subagent以通用模式执行任务

### dependencies 字段

- 值为其他任务的 `id` 数组
- 形成DAG（有向无环图），检测到循环依赖则报错
- 被依赖的任务成功后才会启动后续任务
- 如果被依赖的任务失败，依赖它的任务自动标记为 `skipped`
- 前置任务的 result.json 摘要会注入到后续任务的prompt中
