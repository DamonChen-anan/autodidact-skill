---
name: autodidact
description: 自动课题研究器。给定一个研究课题，自动循环执行"提问→检索→整理→评估"，持续更新知识库直到研究目标达成。适用于行业调研、技术研究、概念理解、创业方案构建等场景。触发方式：用户说"帮我研究 X"、"调研一下 X"、"我想深入了解 X"、"/autodidact X"。
user-invocable: true
args: "<课题描述> [--max-rounds 20] [--background <已知背景>]"
---

# Autodidact — 自动课题研究器

协调 questioner、researcher、wiki 三个模块，驱动研究循环，直到达成终止条件。

---

## 目录结构约定

```
<知识库根目录>/
├── CLAUDE.md                    ← 知识库规则（必须存在）
├── raw/                         ← 原始抓取素材（不修改）
├── wiki/                        ← 结构化知识库
│   ├── INDEX.md
│   └── <主题>.md
└── outputs/
    ├── research_state.json      ← 持久状态（唯一真相来源）
    ├── research_summary.md      ← 最终研究摘要
    ├── user_questions.md        ← 用户插入问题（可随时编辑）
    └── rounds/                  ← 每轮过程产物（可归档/删除）
        ├── round_N_questions.json
        └── round_N_check.md
```

**过程产物统一放 `outputs/rounds/`，持久状态放 `outputs/` 根目录。**

---

## 启动流程

### 第一步：初始化

1. 解析参数：
   - `课题描述`（必须）
   - `--max-rounds`：最大轮次，默认 20
   - `--background`：用户已知背景（可选，传给 questioner 用于跳过基础问题）

2. 检查并初始化环境（冷启动）：

   **目录**：`raw/`、`wiki/`、`outputs/`、`outputs/rounds/` 不存在则自动创建，无需提示。

3. 初始化或读取 `outputs/research_state.json`：
   - 若文件不存在：创建新的研究状态（见数据结构）
   - 若文件已存在且 topic 匹配：从上次中断处继续（读取已完成轮次）
   - 若文件已存在但 topic 不匹配：提示用户是否覆盖

4. 向用户确认研究计划：
   ```
   ## 开始研究：<课题描述>
   - 最大轮次：N
   - 当前状态：全新开始 / 从第 X 轮继续
   - 知识库现状：wiki/ 中有 X 篇文章 / 空
   ```

---

## 外部输入检测（每轮开始前）

在调用 questioner 之前，先检查两类外部输入，任何一类有新内容都应触发本轮循环（即使原本已达终止条件）：

**① 用户插入问题**
检查 `outputs/user_questions.md` 是否存在且有未处理的问题：

```markdown
## 待处理

- 期权的波动率曲面是什么？
- 国内有没有专门做期权策略的私募基金？

## 已处理

- ~~备兑开仓被行权后怎么操作？~~（已纳入第2轮）
```

用户只需在 `## 待处理` 下加一行 `- 问题文本` 即可。

若有待处理问题，将文件路径传给 questioner，questioner 将其合并入本轮问题列表，优先级设为 high。

**questioner 完成后，由 autodidact 负责更新 user_questions.md**：
读取 questioner 输出 JSON 中的 `incorporated_user_questions` 字段，将对应条目从 `## 待处理` 移至 `## 已处理`，格式：`~~问题文本~~ [r<N>]`。

**② 用户贡献的新 raw 文件**
对比 research_state.json 中 `all_raw_files_processed` 列表与当前 `raw/` 目录的实际文件，找出新增文件。
若有新文件，通知 questioner 扫描这些文件内容，提取新主题和问题。

---

## 研究循环

每轮执行以下步骤，完成后更新 research_state.json：

### Step 1：调用 questioner

传入：
- 课题描述
- 当前轮次 N
- 上一轮审计报告路径（`outputs/rounds/round_<N-1>_check.md`，第1轮无）
  - **若上一轮 check_report 为 null**（如因中断未完成），回退查找最近一轮非 null 的 check_report，并在问题列表摘要中标注"⚠️ 使用第 X 轮审计报告（第 N-1 轮审计缺失）"
- 用户插入问题文件路径（若 `outputs/user_questions.md` 有待处理问题）
- 新增 raw 文件列表（若有用户贡献素材）
- 历史所有问题列表（`all_questions_ever`，含 id 和 text，去重用）
- 用户背景（若有）

等待输出：`outputs/rounds/round_N_questions.json`

### Step 2：调用 researcher

传入：`outputs/rounds/round_N_questions.json`

等待输出：raw/ 新增文件列表

### Step 3：调用 wiki compile

传入：本轮新增的 raw/ 文件列表（`--files` 参数，增量编译）
注意：若 Step 1 检测到用户贡献的新 raw 文件，也一并传入

等待输出：新增/更新的 wiki 主题列表

### Step 4：调用 wiki check（完整审计）

传入：
- `outputs/rounds/round_N_questions.json`
- `--round N`

等待输出：`outputs/rounds/round_N_check.md`（问题覆盖状态 + 结构性审计）

### Step 5：更新 research_state.json

将本轮完整信息写入状态文件，包括：
- 本轮所有问题（含 id 和 text）追加到 `all_questions_ever`
- 本轮处理的所有 raw 文件（含用户贡献的）追加到 `all_raw_files_processed`
- 审计报告路径（`outputs/rounds/round_N_check.md`）
- 终止判断结果（含 `overlap_ratio` 数值）

### Step 6：终止判断

终止条件**全部满足**才终止（AND 关系）：

| 条件 | 判断方式 |
|---|---|
| 审计漏洞边际递减 | 本轮漏洞数 ≤ 上轮的 60%，或连续两轮漏洞数均 < 3 |
| 无用户插入问题 | `user_questions.md` 无 pending 问题 |
| 无新 raw 文件 | 用户没有贡献新素材 |
| 问题重叠率 > 70% | questioner 输出的 `overlap_ratio` 字段值 > 0.7 |

> **为什么用"边际递减"而非"绝对值 < 3"**：知识库越深，审计发现越少是自然规律。但如果用绝对值，审计工具稍微努力一点就能找出 3+ 条漏洞，导致循环永不终止。边际递减才能真正捕捉"已无明显新增价值"的信号。

**硬上限**：达到 max_rounds 无条件终止

**继续**：进入下一轮前，先执行 Step 7 上下文检查，然后 round N+1

**终止**：执行收尾流程

### Step 7：上下文检查（继续时执行）

在进入下一轮之前，检查当前上下文使用量。若上下文已超过 50%，**提示用户**：

```
⚠️ 上下文已使用超过 50%，建议在继续前执行 /compact 压缩对话。
   输入 /compact 后再继续，或直接回复"继续"跳过（可能在后续轮次中断）。
```

等待用户确认后继续。这能防止长时间研究循环因上下文耗尽而中断。

---

## 收尾流程

研究循环结束后：

1. 生成研究摘要，写入 `outputs/research_summary.md`：

   ```markdown
   # 研究摘要：<课题>

   ## 研究概况
   - 研究轮次：N 轮
   - 检索文章：X 篇（raw/）
   - 知识主题：X 个（wiki/）
   - 终止原因：<漏洞边际递减达标 / 重叠率超标 / 达到最大轮次 / ...>

   ## 核心发现
   （基于 wiki/ 内容，提炼 3-5 个最重要的发现）

   ## 未解决问题
   （列出所有 open 状态的问题）

   ## 知识图谱
   （列出 wiki/ 中的主题及其关联关系）
   ```

2. 向用户输出：
   ```
   ## 研究完成：<课题>

   共 N 轮，检索 X 篇文章，整理 X 个知识主题。

   产出文件：
   - wiki/                        ← 结构化知识库
   - outputs/research_summary.md  ← 最终摘要
   - outputs/rounds/              ← 各轮审计报告
   - outputs/research_state.json  ← 完整研究轨迹
   ```

---

## research_state.json 数据结构

```json
{
  "topic": "课题描述",
  "started_at": "2026-04-09T10:00:00",
  "max_rounds": 20,
  "status": "in_progress",
  "rounds": [
    {
      "round": 1,
      "timestamp": "2026-04-09T10:05:00",
      "questions": [
        {
          "id": "r1q1",
          "text": "问题文本",
          "layer": "L1-定义",
          "priority": "high",
          "source": "framework",
          "status": "resolved"
        }
      ],
      "new_raw_files": ["2026-04-09_xxx.md"],
      "user_contributed_raw_files": [],
      "wiki_topics_updated": ["主题A", "主题B"],
      "check_report": "outputs/rounds/round_1_check.md",
      "termination_check": {
        "audit_issues_count": 5,
        "pending_user_questions": 0,
        "new_raw_files": 0,
        "overlap_ratio": null,
        "decision": "continue",
        "reason": "audit_issues_count 5，边际递减条件未满足"
      }
    }
  ],
  "all_questions_ever": [
    {"id": "r1q1", "text": "问题文本"},
    {"id": "r1q2", "text": "问题文本"}
  ],
  "all_raw_files_processed": ["2026-04-09_xxx.md"],
  "final_summary": null
}
```

---

## 注意事项

- **课题边界**：每轮传给 questioner 时，明确提醒"围绕原始课题：<课题描述>"，防止范围漂移
- **中断恢复**：如果某轮中途失败，下次启动时从该轮重新开始（不重复已完成的步骤）
- **进度展示**：每轮开始时输出 `## 第 N 轮 / 最多 M 轮`，让用户了解进度
- **用户可随时中断**：Ctrl+C 中断后，research_state.json 保留已完成轮次，下次可继续
