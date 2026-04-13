# autodidact

> 基于 [Karpathy LLM Wiki](https://gist.github.com/karpathy/442a6bf555914893e9891c11519de94f) 构建的自主研究 Agent——给它一个课题，它自动检索互联网、构建结构化知识库。

[![MIT License](https://img.shields.io/badge/license-MIT-blue.svg)](LICENSE)
[![Built with Claude Code](https://img.shields.io/badge/Built%20with-Claude%20Code-black)](https://claude.ai/code)

[English](README.md)

Karpathy 的 LLM Wiki 是一个强大的模式：LLM 把原始素材增量编译为持久化的、互相链接的 wiki，知识随时间不断积累，而不是每次查询都从零开始。

**autodidact 在此基础上更进一步**：你甚至不需要提供素材。给它一个研究课题，它自动驱动完整循环——生成问题、检索网页、编译 wiki、审计漏洞、迭代深入，直到课题被充分覆盖。

```
输入：/autodidact 期权卖方策略，目标年化收益10%

输出：22 篇 wiki 文章，43 篇原始素材 — 自动运行约 1 小时
```

```
your-project/
├── raw/          (43 个文件，自动抓取的原始网页素材)
├── wiki/
│   ├── INDEX.md
│   ├── 期权基础概念.md        # 期权定义、四种形态、核心要素、盈亏结构
│   ├── 期权定价与Greeks.md    # 六大定价因素、隐含波动率、Delta/Gamma/Theta/Vega
│   ├── 国内期权市场.md        # 可交易品种、开户条件（50万门槛）、交易权限等级
│   ├── 期权策略.md            # 备兑开仓、卖出看跌、双轮策略、年化10%可行性
│   ├── 备兑开仓实操.md        # 行权价/到期日选择、滚动操作、被行权处理
│   ├── 期权风控.md            # 止损规则（权利金2-3倍）、仓位管理、开仓检查清单
│   ├── 期权工具与资源.md      # IV分位数查询、交易软件推荐、模拟盘
│   ├── 价差策略.md            # 牛市看跌价差、熊市看涨价差、铁鹰策略构建
│   ├── 历史表现与熊市应对.md  # 各年份实际表现、领口策略、熊市四种应对方案
│   ├── 新手学习路径.md        # 四阶段路径（1-1.5年）、里程碑、税务说明
│   ├── IV分位数实战应用.md    # IV分位数计算、高/低IV策略选择框架
│   ├── 双轮策略实操.md        # Wheel Strategy完整循环、A股实操细节、参数选择
│   ├── 期权流动性指南.md      # 各品种流动性排名、月份/行权价选择
│   ├── 波动率Skew.md          # 负偏斜成因（机构对冲+恐惧溢价）、对行权价选择的影响
│   ├── 双卖宽跨策略.md        # Short Strangle参数、IV要求、止损、单边突破处理
│   ├── 日历价差.md            # 同行权价买远卖近、Theta差+Vega多头、低IV适用
│   ├── 对角价差.md            # PMCC穷人版备兑、与日历价差的区别
│   ├── Pin Risk.md            # 到期日价格压行权价的风险、Pinning Effect、规避
│   ├── 期权保证金计算.md      # 认沽/认购义务仓公式、价差组合节省70-82%保证金
│   ├── 期权合约代码解读.md    # 17位代码结构、C/P方向、到期年月、行权价解析
│   ├── CSP与备兑开仓对比.md   # 年化收益比较、被行权处理、选择决策框架
│   └── 铁鹰策略.md            # Iron Condor=Short Strangle+保护腿、保证金节省70-80%
└── outputs/
    ├── research_state.json    (完整研究轨迹)
    ├── research_summary.md    (最终摘要)
    └── rounds/                (每轮审计报告)
```

---

## 与 Karpathy LLM Wiki 的关系

| | Karpathy LLM Wiki | autodidact |
|---|---|---|
| 核心思想 | LLM 增量编译持久化 wiki | 完全继承 ✓ |
| 素材来源 | 人工收集，手动喂给 LLM | **自动检索互联网** |
| 知识审计 | 手动触发 lint | **每轮自动审计，驱动下一轮** |
| 研究循环 | 无 | **自动提问 → 检索 → 编译 → 审计 → 循环** |
| 使用方式 | 手动 ingest + 手动 query | **一条命令，全自动运行** |

---

## 工作原理

每轮四步，审计报告驱动下一轮提问：

```
┌─────────────┐     ┌────────────┐     ┌──────────────┐     ┌────────────┐
│  questioner │────▶│ researcher │────▶│ wiki compile │────▶│ wiki check │
│             │     │            │     │              │     │            │
│ 读审计报告   │     │ 并行抓取    │     │ 质量加权提炼  │     │ 漏洞审计   │
│ 生成问题列表 │     │ 写入 raw/  │     │ 更新 wiki/   │     │ 输出报告   │
└─────────────┘     └────────────┘     └──────────────┘     └─────┬──────┘
       ▲                                                           │
       └───────────────── 驱动下一轮 ──────────────────────────────┘
```

**终止条件**（全部满足才停止）：审计漏洞边际递减（≤上轮60%）+ 问题重叠率 >70% + 无用户插入问题 + 无新素材

---

## 环境要求

### 必须

- **[Claude Code](https://claude.ai/code)**：本项目是 Claude Code 的 skill 插件，需要先安装 Claude Code

### 推荐

- **[agent-browser](https://github.com/steel-dev/agent-browser)**：无头浏览器 CLI，显著提升检索成功率和内容质量。未安装时自动降级为 Claude 内置 WebSearch，但覆盖率和质量会下降。

  ```bash
  npm install -g agent-browser
  agent-browser install   # 下载 Chromium
  ```

  > macOS 上若 Chrome 启动报错（DevToolsActivePort 文件未生成），加环境变量：`export AGENT_BROWSER_ARGS="--no-sandbox"`

---

## 安装

将四个 skill 复制到 Claude Code 的 skills 目录：

**macOS / Linux**
```bash
git clone https://github.com/your-username/autodidact
cd autodidact
cp -r skills/autodidact  ~/.claude/skills/
cp -r skills/questioner  ~/.claude/skills/
cp -r skills/researcher  ~/.claude/skills/
cp -r skills/wiki        ~/.claude/skills/
```

**Windows**
```powershell
git clone https://github.com/your-username/autodidact
cd autodidact
Copy-Item -Recurse skills\autodidact  "$env:USERPROFILE\.claude\skills\"
Copy-Item -Recurse skills\questioner  "$env:USERPROFILE\.claude\skills\"
Copy-Item -Recurse skills\researcher  "$env:USERPROFILE\.claude\skills\"
Copy-Item -Recurse skills\wiki        "$env:USERPROFILE\.claude\skills\"
```

重启 Claude Code，输入 `/autodidact` 验证安装成功。

---

## 使用

### 开始研究

在任意目录下打开 Claude Code，运行：

```
/autodidact <课题描述>
```

首次运行自动创建 `raw/`、`wiki/`、`outputs/` 目录，无需任何初始化。

### 参数

```
/autodidact <课题描述> [--max-rounds 20] [--background <已知背景>]
```

| 参数 | 说明 | 默认值 |
|---|---|---|
| `课题描述` | 研究主题，越具体越好 | 必填 |
| `--max-rounds` | 最大研究轮次 | 20 |
| `--background` | 你已掌握的背景知识，跳过基础问题直接深入 | 无 |

### 示例

```bash
# 技术研究
/autodidact RAG 技术原理与工程实践难点

# 行业调研
/autodidact 国内陪诊行业市场调研 --max-rounds 5

# 有背景知识时跳过基础问题
/autodidact 期权卖方策略年化10% --background "已了解期权基础概念和Greeks"
```

### 中断与恢复

Ctrl+C 随时中断。下次在同一目录运行相同课题，从中断处自动继续。

### 插入自己的问题

研究进行中，编辑 `outputs/user_questions.md`：

```markdown
## 待处理

- 这个策略在熊市中表现如何？
- 有没有量化回测数据？
```

下一轮开始时自动纳入，优先级最高。

---

## 单独使用各模块

四个 skill 均可独立调用：

```bash
# 编译 raw/ 素材到 wiki/
/wiki compile

# 增量编译（只处理指定文件）
/wiki compile --files 2026-04-09_article.md

# 纯结构审计（不需要问题列表）
/wiki check

# 完整审计（问题覆盖 + 结构审计）
/wiki check outputs/rounds/round_5_questions.json --round 5
```

---

## 常见问题

**Q：没有安装 agent-browser 能用吗？**
可以，自动降级为 Claude 内置 WebSearch + WebFetch。部分网站会被拦截，内容质量会下降，建议安装。

**Q：支持中文课题吗？**
完全支持。researcher 会同时生成中英文搜索关键词，分别检索中英文信息源。

**Q：上下文快用完了怎么办？**
autodidact 每轮结束时检测上下文用量，超过 50% 时自动执行 `/compact` 压缩并继续，无需任何操作。

**Q：研究跑了很多轮但内容不够深入？**
通常是课题描述太宽泛。加 `--background` 告知已有背景，或在 `user_questions.md` 插入更具体的问题引导方向。

---

## 致谢

wiki 核心架构基于 [Andrej Karpathy 的 LLM Wiki](https://gist.github.com/karpathy/442a6bf555914893e9891c11519de94f)。
