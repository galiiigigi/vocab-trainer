# Vocab App · S2–S9 方案细化

> 锁定日期：2026-04-26 / 细化日期：2026-04-27
> 起点状态：库 ~1,187 词（236 测试样本 + 951 NGSL 1K），用户 Vocab Map 测试 3,100 词 / B2（自评 B1.1–B1.2）
> 路径：hybrid 🅲️ — 自建 + Langua 7 天试用并行
> 硬规矩 🔵：每个 session 上线后，用户必须在 1 周内用新功能 ≥ 5 次，否则暂停新开发先诊断

---

## ⚠️ 重要更新 (2026-04-27 晚间) — 产品定位转变

经用户深入体验 Langua 后，方案从"**自建 Langua 替代品**"转向"**Langua 配套的 gap-detection 工具**"。

### 新定位
```
Langua    = 学习层（深度沉浸、对话、内容、SRS 实践）
Vocab-app = 情报层（找出哪些词需要学 + 生成喂给 Langua 的文章）
```

### 新工作流
1. 用户在 vocab-app 跑 Vocab Map test / 浏览 NGSL 词库 → 找出 gap 词
2. vocab-app Story Mode 生成一篇包含 gap 词的自然文章
3. 用户点 "Copy for Langua" → 粘到 Langua
4. Langua 分析文章，把 gap 词推到学习流（聊天/SRS/练习）
5. 用户在 Langua 实际学习

### 关键洞察：Langua 不接受批量词导入，但接受文章
所以 vocab-app 的 Story Mode（S2）变成了"导入 Langua 的桥"——这个洞察让 S2 的工作从"过度建设"变成了"核心引擎"。

---

## S1-S9 在新定位下的价值重估

| 功能 | 旧定位 | 新定位 | 状态 |
|---|---|---|---|
| S1 Free Chat 2-agent + Vocab Flagging | 核心 | ❌ 0（用户不在这聊天，会在 Langua 聊） | 已建，沉睡 |
| **S2 Story Mode 2-agent** | 核心 | ⭐⭐⭐⭐⭐ **核心引擎**（喂 Langua 的桥） | 已建，**载荷** |
| S3 Profile + SRS 注入聊天 | 核心 | ⭐ 部分用（profile 用于词选择 bias） | 已建，部分价值 |
| S4 Grammar Tracker | 核心 | ❌ 0（用户语法错误在 Langua 发生） | 已建，沉睡 |
| **S5 Drill Generator** | 核心 | ❌ 0（Langua 干这个） | **取消** |
| S6 Curator Agent | 中 | ⭐⭐⭐ 中高（健康监控 + decay detection） | 待建 |
| **S7-S8 NGSL 2-3K 词库** | 中 | ⭐⭐⭐⭐⭐ **核心**（gap 雷达数据源） | 进行中 |
| **S9 COCA 4-6K 词库** | 中 | ⭐⭐⭐⭐ **核心**（同上） | 待建 |

---

## 新位置下的下一步行动清单

### 等待验证（next session 第一件事）
**Langua 端到端测试结果**：
- Story Mode 生成 → "Copy for Langua" → 粘到 Langua → Langua 分析吐出的词，跟 vocab-app 目标词的重合度
- **>60%** = 方案 work，进入"扩张阶段"
- **<40%** = 调 Auditor prompt 或重思整体方案

### 验证通过后的小迭代（按优先级）

1. **`STORY_WORD_COUNT` 5 → 10-15** — 减少粘贴次数。50 个 gap 词从 10 篇降到 4-5 篇文章
2. **`pickNonMasteredWords()` profile 加权** — 命中 `userProfile.jobContext` / `interests` 关键词的 gap 词权重 ×2
3. **Mark-as-learned 半自动同步** — 文章生成后立即把目标词标记 `learning`，避免反复重复推荐
4. **Smart Curator (S6)** — Smart Import 去重 + decay detection（曾掌握现错的词重新进 gap）

### 词库继续扩
- NGSL 2K 剩 ~400 词（理论上限 1000）
- COCA 4-6K：1000 词，B2-C1 难度
- 优先级：根据 gap 检测命中率决定，如果 NGSL 2K 600 已经够用就直接 COCA

### 明确取消
- ~~S5 Drill Generator~~（Langua 做练习）
- ~~更多 Free Chat / Story Mode 学习功能~~（学习在 Langua 发生）

---

## 历史决策记录（保留供 reference）

---

## S2 · 2-agent Story Mode

### 当前问题
单 agent 一次出稿，prompt 里塞"必须用这些词" + vibe，结果：
- 词强行出现（"Freezie/Housecoat" 那种突兀感）
- 套话结尾（"It truly taught me a valuable lesson"）
- 难度失控（B1 用户读到 15–18% 不认识的词 = 超出 i+1）

### 新架构

```
Agent 1: Story Writer          Agent 2: Vocab Auditor
─────────────────────          ─────────────────────
输入: 选中的词 + vibe          输入: Agent 1 的 passage + 词单
任务: 自然写 ~300 词            任务: 检查每个词是否自然
注: "如果某词不自然就别用"      输出: { fits, forced, rewrite_suggestions }

                               ↓ forced 占比 > 30% → 触发 Agent 1 重写
                               ↓ 否则 → 接受，passage + 命中率显示给用户
```

### 关键参数
- **目标词命中率显示**：`Used 4/6 naturally · 1 skipped · 1 forced (rewriting...)`
- **难度门**：生成后扫描 passage 所有词，未掌握词比例 > 12% → Auditor 发回重写（i+1 原则，B1 ≤ 10–12%）
- **去 AI 腔黑名单**：prompt 硬性禁用 "It truly taught me", "valuable lesson", "in conclusion", "I realized that..."

### UI 改动
Story 卡片上方增加 1 行 meta：
```
目标词 6 · 自然命中 4 · 跳过 2 · 未知词 8% (适配你的水平)
```

---

## S3 · Memory/Profile Agent + 主动注入到期复习词

### Profile Agent 数据结构
```js
state.userProfile = {
  level: 'B1.1',          // 来自 Vocab Map test
  job_context: 'environmental sector, Canadian workplace',
  interests: [],          // 从聊天历史提取
  recent_topics: [],      // 最近 10 次聊天主题
  weak_grammar: [],       // 来自 S4
}
```

### 到期复习词注入到聊天

**触发条件**：Free Chat 每次 AI 回复前，Profile Agent 跑一次：
1. 从 SRS 拉今日 due 的词（`nextReviewDate <= today`）
2. 挑 2–3 个语义可融入当前对话的
3. 把这几个词作为"软提示"塞进 Conversational Agent 的 system prompt：
   `"Try to naturally weave 1-2 of these words into your reply if context allows: [reclaim, deploy, mitigate]"`
4. **不强制**：如果对话语义不合，不用塞

### UI
- 聊天界面右上角 badge：`复习中: 3 词`（点开看是哪几个）
- AI 回复里命中的复习词自动加下划线高亮 + 点击弹"今日刚复习过"小提示

### 记忆持久化
Profile 每次聊天结束后增量更新（job/interests），用 ~50 token 让 Agent 自己写 diff

---

## S4 · Grammar Correction Agent 增强

### 当前
Free Chat 每条用户消息后给 `[Correction] X/O/Tip` 一条

### 增强 — 分类系统（覆盖 90% B1 错误）
1. Tense（时态）
2. Article（a/an/the）
3. Preposition（in/on/at/for）
4. Subject-Verb Agreement
5. Word Choice（近义混用）
6. Collocation（搭配）
7. Word Order
8. Plural / Countability

### 数据结构
```js
state.errorLog = [
  { date, sentence_user, sentence_correct, category: 'preposition',
    pattern: 'in→on (days of week)', resolved: false }
]
```

### 错误档案 UI
`Words` tab 旁边加 `Errors` 子 tab：
- 按类别折叠列表，每类显示出现次数
- 点开看具体例子
- "已解决"打勾 → 7 天内同类错不再出现就移除

### Prompt 升级（JSON 输出）
```json
{
  "correct": false,
  "user_said": "I go to office on Monday",
  "should_be": "I went to the office on Monday",
  "categories": ["tense", "article"],
  "pattern": "past action → past tense",
  "tip_zh": "动作发生在过去，要用过去式"
}
```

---

## S5 · Drill Generator

### 输入
S4 的 `errorLog`，按类别频次排序

### 生成逻辑
**最高频错误类别 → 生成针对性 drill**

例：preposition 错 5 次 → 生成 5 题 prep drill：
- 形式 1：填空 `She arrived ___ Tuesday morning.`
- 形式 2：选择 `The meeting is ___ 3pm. (in/on/at)`
- 形式 3：改错 `I'm working in this project since 2020.`

### 触发方式
- **主动**：每天 1 次系统提示"今天有 3 个高频错可以集中练 5 分钟" → 进入 Drill 模式
- **被动**：Sub Drill 现有界面增加"Error-driven"按钮，从错误档案抽题而不是随机词

### 成功机制
- 连续答对同类 5 题 → S4 的 `resolved: true`，Profile 里 `weak_grammar` 移除该类
- 后续 Drill 不再生成这类，但持续 monitor 复发

### 与 SRS 集成
错误本身用 SM-2 算法：错过的句型有 interval/easeFactor，到期重复出现

---

## S6 · Library Curator Agent

### 范围（明确锁定）
- ✅ Smart Import · 健康监控 · 衰退检测
- ❌ AI 生成定义 · 主题词包生成

### Smart Import
**当前痛点**：库已有 1,187 词，未来导 NGSL 2-3K 时会有重复

**Curator 工作流**：
1. 用户粘贴新词列表（如 NGSL 2K，500 词）
2. Curator 拉本地 library，做 **去重 + 词形归并**（"manage/manages/managing/management" → 一个根词）
3. 输出 preview：`新词 380 · 重复 95 · 词形变体 25`
4. 重复词：若新词条带新例句/定义，提示是否合并增强；否则跳过
5. 用户确认后 commit

### 健康监控
后台定期扫描（每次开 app 时）：
- **Stale**：`lastReviewed > 60 天 && status !== mastered` → 提示"30 个词被遗忘了"
- **Suspicious**：定义和单词字面冲突（如词是 "deploy" 但定义里出现 "deploy" → cleanDefinition 漏网）
- **Orphan**：没有例句的词
- **Duplicate**：用户后续手动加词时撞了已有词

### 衰退检测
- 之前 Mastered 的词，最近 3 次复习准确率 < 60% → 自动降级到 Familiar，重新进入 SRS
- 纯客户端逻辑可做，但 Curator agent 的额外价值：给出"为什么衰退"的诊断
  （"这词多义，你只学了 sense 1，最近遇到 sense 2 答错了"）

### UI
Settings 加 "Library Health" 面板：
```
✓ 健康: 980 词
⚠ 衰退: 30 词 (查看)
⚠ 60天未复习: 45 词 (恢复 / 归档)
✗ 异常条目: 4 词 (修复)
```

---

## S7–S8 · NGSL 2-3K 手写（1800 词，2 个 session）

### 数据来源
NGSL 2K (1001–2000) + NGSL 3K (2001–3000) = 1,800 词
（NGSL = New General Service List，基于 273M 词语料库的高频通用词）

### 每词字段（与 NGSL 1K 保持一致）
```js
{
  word: 'allocate',
  definition: 'to set aside money/resources for a specific purpose',
  example: 'The company allocates $50K per quarter for safety training.',
  domain: ['business', 'workplace'],
  level: 'B1',
}
```

### Claude 手写质量标准
- **定义**：避免循环（不用 word 本身解释）、口语化、≤ 15 词
- **例句**：加拿大职场场景优先（环保/矿业/办公），≤ 20 词，自然不像教材
- **domain**：最多 2 个标签

### 工作流（每 session）
1. Claude 一次写 100 词 batch（约 12–15 分钟生成）
2. 用户 spot check 10 词（随机抽）
3. 通过 → commit；不通过 → 调 prompt 重写
4. 每 session 9 个 batch = 900 词
5. **S7 = 1001–1900，S8 = 1901–2800**（凑整 1800）

### 成本估算
~1,800 词 × ~150 tokens/词 = 270K tokens 输出 ≈ Sonnet $4 / Opus $20
建议 Sonnet 4.6 写，Opus 抽审

---

## S9 · COCA 4-6K 手写（1000 词，1 个 session）

### 数据来源
COCA (Corpus of Contemporary American English) 4001–5000 + 5001–6000 高频词
（COCA 比 NGSL 更广，覆盖学术/新闻/小说/口语，4–6K 段更多是低频但仍重要的词）

### 与 S7–S8 的差异
- 这段词更多 **形态复杂 + 多义**（如 "render" 有 ≥4 个常见 sense）→ 每词可能要 1–3 个定义条目
- 例句更倾向 **学术/正式语境**（适合环保/技术报告写作）
- 字段加 `register: formal/informal/neutral` 标签（B2+ 用户开始要区分语域）

### 工作流
- 单 session 1,000 词 = 10 个 batch × 100 词
- Spot check 比例提到 15%（多义词易出错）
- 9-session 的收尾，做完 library = ~3,800–4,000 独立词条 → C1 词汇覆盖

### 多义词存储格式
```js
{
  word: 'render',
  senses: [
    { def: 'to provide a service', example: '...', domain: ['business'] },
    { def: 'to cause something to be in a state', example: '...', domain: ['general'] },
    { def: 'to depict in art/animation', example: '...', domain: ['tech'] },
  ]
}
```
（注：当前 schema 已支持多义编号，S9 只是更系统地用上）

---

## 执行顺序逻辑

```
S1 (Free Chat 2-agent)  ───┐
                           ├─→ 给 S3/S4 提供"对话流量"
S2 (Story Mode 2-agent) ───┘

S3 (Profile + 注入复习)  ───→ 让 Free Chat 真的循环 SRS
S4 (Grammar 分类)        ───→ 错误数据
S5 (Drill Generator)     ───→ 消费 S4 数据，闭环

S6 (Curator)             ───→ 为 S7-S9 大批量导入做准备
S7-S8 (NGSL 2-3K)        ───→ B2 词库
S9 (COCA 4-6K)           ───→ C1 词库
```

**关键依赖**：
- S3 必须在 S1 后
- S5 必须在 S4 后
- S6 必须在 S7 前

其余可调。

---

## 潜在调整建议（待用户决策）

1. **S6 提前到 S2 后**？目前 1,187 词其实已需要健康监控，不必等到 S6
2. **S4 + S5 合并为 1.5 session**？语法错误链条较紧，分开会让 S4 数据空转
3. **S9 是否必要**？若 S7–S8 跑完已到 B2–C1 边缘，COCA 4–6K 可能用 Curator + Smart Import 引第三方现成数据更经济

---

## 明确否决

- AI 自动生成定义/例句（用户只要 Claude 手写质量）
- 主题词包生成器
- Edge-read（粘文章点词加入）
- 多语言（学日韩 → 直接推荐 Langua）
- 单独的 UX 打磨 session

---

## Langua 试用并行计划

- 2026-04-27 晚上：用户买 Langua 7 天免费试用
- 试用与 S1+ 并行
- **试用结束后**：用户报告具体 Langua wins/losses → feedback 可调整 S2–S5 设计
- 成本对比：自建 ~$60/yr API + 9 sessions  vs  Langua $240/yr 永久
