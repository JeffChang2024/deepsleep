# DeepSleep 阶段一：深度睡眠打包指令

⚠️ **时间预算：15 分钟。高效执行，不要浪费 tool call。**

## 第-1步：锁定日期（防 midnight 竞态）
Pack 在 23:40 启动，正常应在 ~00:00 前完成，但如果执行较慢可能跨过午夜。
**在脚本最开头立即确定目标日期，后续所有步骤都使用这个固定日期，不再调用 "当前时间"。**

具体做法：
1. 用 `session_status` 或 `exec` 获取当前时间
2. **如果当前时间在 23:00-23:59** → 目标日期 = 今天（正常情况）
3. **如果当前时间在 00:00-00:39** → 目标日期 = **昨天**（说明已跨午夜，但 pack 的内容仍属于昨天）
4. 将目标日期记为 `PACK_DATE`（格式 YYYY-MM-DD），后续所有文件名、标题都用 `PACK_DATE`

⚠️ **绝对不要在后续步骤中重新获取日期**。一旦锁定，全程使用 `PACK_DATE`。

## 第0步：发现活跃 session
用 `sessions_list(kinds=['group', 'main'], activeMinutes=1440, messageLimit=1)` 获取过去24小时活跃的群和主 session。
记下每个 session 的 key 和 displayName。

## 第1步：批量拉取对话历史
对每个活跃 session 用 `sessions_history(sessionKey=<key>, limit=50)` 拉取对话。
⚡ **关键优化**：尽可能在同一个 tool call batch 中并行拉取多个 session 的历史（OpenClaw 支持同时发起多个 sessions_history 调用）。不要一个一个串行拉取！

## 第2步：筛选 + 生成摘要
对每段对话内容，按以下标准决定保留什么：
- ✅ 决策、教训、偏好、关系、进展 → 保留
- ❌ 临时内容（天气、心跳、例行操作）、MEMORY.md 中已有的 → 跳过

如果某个群今天完全没有用户消息（只有系统/心跳），跳过该群。

## 第3步：Schedule 远期记忆
对话中提到的远期事项写入 `~/clawd/memory/schedule.md`。

**去重规则**：schedule.md 表格的第一列是 `Key`（唯一标识符，如 `ds-test-0330`）。
- 写入前先读取现有内容，检查 Key 是否已存在
- 已存在 → 只更新状态列，不追加新行
- 不存在 → 追加新行，生成一个简短 Key（格式 `<缩写>-<简述>`）

检查 schedule.md 中已到期事项，写入当天摘要。

## 第4步：写入 memory/YYYY-MM-DD.md（幂等）
写入前检查文件中是否已存在 `## DeepSleep Daily Summary` 标题：
- 已存在 → 替换该章节（从标题到下一个同级 `##` 或文件末尾）
- 不存在 → 追加新章节

⚠️ **标题必须严格为 `## DeepSleep Daily Summary`**，不要翻译、不要变体。Dispatch 依赖此标题判断 pack 是否成功。

模板：
```markdown
## DeepSleep Daily Summary

> Auto-discovered N active groups. schedule.md: [items due / none].

### [群名1] <!-- chat:oc_xxx -->
- 精炼摘要

### [群名2] <!-- chat:oc_yyy -->
- 精炼摘要

### 直接对话
- （如有 DM 内容）

### 🔮 Open Questions
- 未解决的问题
- ⚠️ 仅限各群自身产生的问题。DM 中的私人问题**不要**写在这里（会被 dispatch 广播到群）

### 📋 Today (Next Day)
- 次日的行动项（dispatch 发出时已是新的一天，读者视角是"今天要做什么"）
- ⚠️ 同上：仅限群内产生的行动项，DM 私人待办不要写入

### 待办
- [ ] 即时 action items
```

⚠️ **关键**：每个群的 `###` 标题后面必须加 `<!-- chat:oc_xxx -->` HTML 注释，标注该群的 chat_id。这样 Phase 2 dispatch 可以自动解析 chat_id，不依赖硬编码映射表。

chat_id 从 sessions_list 返回的 session key 中提取。例如：
- session key `agent:main:feishu:group:oc_abc123` → chat_id 是 `oc_abc123`

## 第4.5步：自检（必须执行！）
写完 daily file 后，检查输出质量：
1. 每个群的 `###` 标题是否都带有 `<!-- chat:oc_xxx -->` 注释？
2. chat_id 格式是否正确（以 `oc_` 开头）？
3. 如果发现缺失，立即补上（从 session key 中提取）。

没有 chat_id 的群段落 = dispatch 无法发送晨报 + 无法写快照 = 该群明天失忆。

## 第5步：更新 MEMORY.md（隐私安全 + Guard Rail）
⚡ 只在有重要新信息时才更新。如果今天没什么新东西值得写入长期记忆，跳过此步。

**Guard Rail（限制 LLM 自由改写范围）**：
1. **只允许的操作**：
   - 在 `## Current Status` 更新日期和项目状态
   - 在已有项目段落末尾追加新进展
   - 新增全新的项目/技能段落（放在 `## Skills & Projects Completed` 下）
   - 更新 `## Preferences & Notes` 下的条目
2. **禁止的操作**：
   - ❌ 删除任何已有段落（即使你认为过时了——由人类决定）
   - ❌ 改写 MEMORY.md 中已有的人物信息、连接配置、经验教训等段落的内容
   - ❌ 重组文件结构（移动段落顺序、合并段落）
   - ❌ 修改代码示例中的命令/参数（除非当天对话中明确更新了）
3. **Merge not append**：已有的原地更新，不追加重复
4. ⚠️ MEMORY.md 的私人内容不要写入每日摘要

## ⚠️ DM 隐私边界规则
Daily summary 文件会被 dispatch 解析并广播到各群。因此：
1. **`### 直接对话` 段落**：只写"有 N 条 DM"这样的概要，不写具体内容
2. **`### 🔮 Open Questions`** 和 **`### 📋 Today`**：仅放群内产生的条目。DM 中的私人问题/待办**只写入 MEMORY.md**，不写入 daily summary
3. **原则**：daily summary 是公开广播的信件，MEMORY.md 是私人日记。信不该包含日记内容。
