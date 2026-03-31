# DeepSleep 设计文档

## 版本
- v2.0 (2026-03-29): 初始发布，三阶段架构
- v2.1 (2026-03-31): 7 项健壮性改进（详见下方）

## 架构

纯 Agent tool-call 方案。无外部脚本依赖，全部通过 OpenClaw cron → isolated agentTurn → agent 读指令文件执行。

```
23:40  cron "deepsleep-pack"
         → agentTurn (isolated session, timeout 900s)
         → 读 pack-instructions.md
         → 第-1步: 锁定日期（防 midnight 竞态）
         → sessions_list + sessions_history (并行) → 筛选总结
         → 写 memory/YYYY-MM-DD.md + 更新 MEMORY.md (guard rail) + schedule.md (Key 去重)

00:10  cron "deepsleep-dispatch"
         → agentTurn (isolated session, timeout 900s)
         → 读 dispatch-instructions.md
         → 第-1步: 去重检查（dispatch-lock.md）
         → 读昨天 memory/YYYY-MM-DD.md
         → message(send) 各群晨报（昨天做了什么/今天要做什么）
         → write 各群快照 memory/groups/<chat_id>.md（3天滚动 merge）
         → 写发送日志 dispatch-log.md
         → 写锁文件 dispatch-lock.md

随时   Phase 3: Session Memory Restore
         → 群 session 收到消息时
         → 读 memory/groups/<chat_id>.md 恢复记忆
         → 配置在 AGENTS.md
```

## 关键设计决策

1. **纯 Agent，无 Python 脚本**：早期设计过 Gemini 粗加工 + Claude 精加工的双 LLM 管道，但实测发现 agent 直接用 sessions_history tool 拉取 + 自行总结更简单可靠，且不需要维护额外的 Python 依赖和 API key。

2. **schedule.md 是唯一调度源**：`memory/schedule.md`（Markdown 表格），由 pack 写入、dispatch 读取。不再使用 JSON 格式。Key 列实现去重。

3. **群映射表写在 dispatch-instructions.md**：用户自行填入。新群通过 sessions_list 自动发现后手动加入映射表。

4. **隐私隔离**：每日摘要可能被 dispatch 到各群 → 不能包含 MEMORY.md 私人内容。每群只收到自己的摘要。DM 内容只写入 MEMORY.md，不进入 daily summary。

5. **幂等性**：pack 检查 daily file 中是否已有 DeepSleep 章节，存在则替换而非追加。dispatch 通过 lock 文件防重复发送。

## v2.1 改进清单 (2026-03-31)

| # | 类型 | 问题 | 方案 |
|---|------|------|------|
| 1 | 🔴 | Midnight 竞态：pack 跨午夜写错日期 | pack 第-1步锁定 PACK_DATE，全程不再重新取时间 |
| 2 | 🔴 | Dispatch 重复发送 | dispatch-lock.md 锁文件，发完写入，重跑检测同日跳过 |
| 3 | 🟡 | DM 隐私泄露到群 | daily summary 中 DM 只写条数，Open Questions/Today 不含 DM 项 |
| 4 | 🟡 | Schedule 重复条目 | Key 列作唯一标识，写入前查重 |
| 5 | 🟡 | Snapshot 无滚动机制 | 3天窗口保留进展，Open Questions/未完成待办不受限 |
| 6 | 🟡 | 无发送日志 | dispatch-log.md 记录每次成功/失败，滚动保留 30 条 |
| 7 | 🟡 | MEMORY.md 被 LLM 自由改写 | Guard rail：只允许追加/更新，禁止删除/重组/改写已有段落 |
| + | 🟡 | 晨报时间视角错误 | 「今天做了/明天做」→「昨天做了/今天做」（读者视角） |

## 运行时文件

```
memory/
├── YYYY-MM-DD.md          # 每日摘要（pack 产出）
├── schedule.md            # 中长期任务（Key 去重）
├── dispatch-lock.md       # dispatch 去重锁（一行日期）
├── dispatch-log.md        # dispatch 发送日志（滚动 30 条）
├── groups/
│   ├── <chat_id>.md       # 每群记忆快照（3天滚动）
│   └── ...
```

## 已废弃

`references/archived/` 下的 Python 脚本是早期方案残留（含硬编码数据，已通过 .gitignore 排除）：
- `sleep_phase.py` — 直接读 JSONL transcript + Gemini API 总结
- `wake_phase.py` — 读 sleep 报告 + 生成晨醒消息 JSON
- `extract_conversations.py` — 从 JSONL 提取对话
- `schedule_engine.py` — JSON 格式的任务调度器
- `schedule.json` — 旧调度数据

这些不再使用，保留仅供参考。
