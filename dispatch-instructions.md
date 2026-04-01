# DeepSleep 阶段二：晨报分发指令

⚠️ **时间预算：15 分钟。高效执行。**

## 第-1步：去重检查（防重复发送）
检查 `~/clawd/memory/dispatch-lock.md` 是否存在：
1. 读取文件内容
2. 如果文件中的日期 = 昨天日期（即本次要 dispatch 的目标日期）→ **说明已经发过了，直接退出，不重复发送**
3. 如果文件不存在或日期不匹配 → 继续执行

在所有晨报发送完成后（第2步之后），立即写入锁文件：
```
write(file="~/clawd/memory/dispatch-lock.md", content="YYYY-MM-DD")
```
（内容仅一行：本次 dispatch 的目标日期）

⚠️ 锁文件必须在**发送完成后**写入，不能在开始前写。这样如果中途崩溃，下次重跑还能正常发送。

## 第0步：检查 pack 产出（失败恢复）
读取 `~/clawd/memory/YYYY-MM-DD.md`（**昨天**日期）。

**如果文件不存在，或没有 `## DeepSleep Daily Summary` 章节**：
说明 Phase 1 pack 失败了（超时、崩溃等）。此时**不要跳过**，执行紧急补救：
1. 用 `sessions_list(kinds=['group','main'], activeMinutes=1440, messageLimit=1)` 发现昨天的活跃群
2. 对每个群用 `sessions_history(sessionKey=<key>, limit=30)` 拉取精简历史（并行）
3. 生成精简版每日摘要，写入 `memory/YYYY-MM-DD.md`
   ⚠️ **必须使用与 pack 完全一致的模板**：标题 `## DeepSleep Daily Summary`，每群标题带 `<!-- chat:oc_xxx -->` 注释
4. 然后继续正常的第1-4步

这样即使 pack 失败，dispatch 也能兜底，**不丢失一天的记忆**。

## 第1步：读取昨天的摘要
读取 `~/clawd/memory/YYYY-MM-DD.md`（昨天日期）。经过第0步，此文件应该存在了。

## 第1.5步：解析群映射
从摘要中的 `### 群名 <!-- chat:oc_xxx -->` 注释提取每个群的 chat_id。
如果某个群缺少 chat_id 注释，查阅下方的映射表兜底。

## 第2步：发送每群晨报
对昨天摘要中每个有内容的群，用 `message(action='send', channel='feishu', target='chat:<chat_id>')` 发送：

```
📋 昨日工作小结（YYYY-MM-DD）

[该群专属摘要——描述"昨天做了什么"]

🔮 Open Questions（如有相关的）
📋 今日待办（如有相关的——描述"今天要做什么"）
```

⚠️ **视角规则（必须遵守）**：
dispatch 在凌晨 00:10 发出，读者看到时是**新一天的早晨**。
- 摘要内容 = "**昨天**做了什么"（过去时）
- 待办/行动项 = "**今天**要做什么"（将来时）
- ❌ 绝对不要写"今天做了什么 / 明天要做什么"——这是 pack 时 23:40 的视角，对读者来说时间错位。

⚠️ **隐私规则**：每个群只收到自己的摘要。不要跨群泄露内容，不要包含 MEMORY.md 私人信息。

## 第3步：检查 schedule.md
读取 `~/clawd/memory/schedule.md`，如有今天到期的事项，一并提醒。

## 第4步：生成每群记忆快照（最关键！）— 滚动 Merge + 静默日保护
**这一步决定了 agent 明天是否还有记忆。没有快照 = 明天失忆。**

### Merge 算法（活跃窗口 + 静默日保护）
对每个群：
1. **读取现有快照** `~/clawd/memory/groups/<chat_id>.md`（如果存在）
2. **读取昨天的摘要**（刚才解析的 daily summary 中该群的段落）
3. **Merge 规则**：
   - **活跃日条目**：保留最近 3 天的（按 `[日期]` 前缀判断）。超过 3 天的旧条目 → 删除
   - **静默日延续条目**（标记为"延续自 YYYY-MM-DD"）：**不计入 3 天窗口**。只要对应的 Open Question 或待办仍未完成，延续条目就一直保留，直到：
     a. 有新的活跃对话产生了替代内容，或
     b. 对应的 Open Question / 待办被标记为已解决/已完成
   - 昨天的新内容 → 追加到"最近进展"
   - **Open Questions：保留所有未解决的（永不过期）**，已解决的删除
   - **待办：保留所有未勾选的 `- [ ]`（永不过期）**，已完成的 `- [x]` 超过 3 天则删除
4. **写入合并后的快照**

### ⚠️ 静默日保护规则
如果昨天的 daily summary 标记为静默日（包含"静默日"或"当日无新对话"），在 merge 时：
- **不要清除现有快照中的进展条目**（因为没有新内容替代它们）
- **只更新快照的时间戳**（`> 更新时间：YYYY-MM-DD 00:10`）
- 保留所有 Open Questions 和未完成待办
- 这确保即使连续多天沉默，快照始终保持"活跃"，3天滚动不会误杀

```
write(file="~/clawd/memory/groups/<chat_id>.md", content=合并后的快照)
```

快照格式：
```markdown
# [群名] 近期记忆
> 更新时间：YYYY-MM-DD 00:10

## 最近进展
- [YYYY-MM-DD] 要点1（最近3天，新的在前）
- [YYYY-MM-DD] 要点2

## 🔮 Open Questions
- 问题1（不受3天限制，直到解决）

## 📋 待办
- [ ] 未完成待办（不受3天限制）
- [x] 已完成（3天后自动清除）
```

⚡ **效率**：可以在同一个 tool call batch 中并行写入多个群的快照文件。

## 第5步：写入发送日志
dispatch 完成后，将本次执行结果追加到 `~/clawd/memory/dispatch-log.md`：

```markdown
## YYYY-MM-DD 00:10
- 目标日期：YYYY-MM-DD（昨天）
- 发送群数：N
- 成功：[群名1, 群名2, ...]
- 失败：[群名3（原因：xxx）] 或 无
- 快照更新：N 个文件
- 锁文件：已写入
- 耗时：约 Xs
```

⚡ 用 `read` 先读现有内容，然后 `write` 追加（保留历史记录）。如果文件超过 50 条记录，删除最早的记录只保留最近 30 条。

## 群名与 chat_id 映射（兜底）
优先从摘要的 `<!-- chat:oc_xxx -->` 注释解析。如果注释缺失，读取本地映射文件兜底：

```
read(file="~/clawd/skills/deepsleep/chat-id-mapping.local.md")
```

如果本地映射文件也不存在，从 `sessions_list` 的 session key 中提取：
`agent:main:feishu:group:oc_abc123` → chat_id 是 `oc_abc123`
