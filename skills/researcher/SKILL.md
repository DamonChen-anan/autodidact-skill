---
name: researcher
description: 信息检索 skill。接收问题列表，通过浏览器自动搜索并抓取高质量网页内容，存入 raw/ 文件夹。含两层质量筛选（抓取前过滤低质来源，抓取后评分）。通常由 autodidact skill 调用，也可单独使用。依赖 agent-browser CLI。
---

# Researcher — 信息检索器

接收问题列表，通过浏览器检索互联网，将高质量信息抓取为 Markdown 存入 raw/。

---

## 前置检查

执行前先确认 agent-browser 可用：

```bash
agent-browser --version
```

**若不可用，直接静默降级到 WebSearch + WebFetch 工具组合，行为保持一致，不提示用户安装，不中断流程。** 在最终输出摘要中记录"agent-browser 不可用，已使用 WebSearch 降级"即可。

---

## 输入

从调用上下文读取：
- **问题列表文件**：`outputs/rounds/round_<N>_questions.json`（questioner 的输出）
- **当前轮次 N**：从文件名或 research_state.json 获取

若直接调用，也可接受用户直接提供的问题列表。

---

## 执行流程

**并行策略**：对问题列表中的每个问题，**同时启动多个 subagent 并行处理**，而不是串行逐一执行。这是提速的关键——5 个问题串行需要 5 倍时间，并行只需 1 倍。

### 总体步骤

1. 读取问题列表（`outputs/rounds/round_N_questions.json`）
2. **为每个问题启动一个 subagent**（用 Agent 工具，在同一条消息中并发发出所有调用）
3. 等待所有 subagent 完成，收集各自产出的文件列表
4. 输出汇总摘要

每个 subagent 的 prompt 模板：

```
你是一个信息检索 agent。请完成以下检索任务：

工作目录：<知识库根目录绝对路径>
今日日期：<YYYY-MM-DD>
当前轮次：<N>

**问题**：<问题文本>

请按以下步骤执行：
1. 生成 2-3 组搜索关键词（中文 + 英文）
2. 用 agent-browser 搜狗搜索获取 URL 列表
3. 筛选 3-5 个高质量 URL（过滤广告、登录墙、内容农场）
4. 逐一抓取正文
5. 质量评分（高/中/低），写入 raw/ 文件

搜索命令：
  REAL_UA="Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/124.0.0.0 Safari/537.36"
  agent-browser --user-agent "$REAL_UA" batch \
    "open https://www.sogou.com/web?query=<关键词>" \
    "wait --load networkidle" "get text body"

抓取正文：
  agent-browser --user-agent "$REAL_UA" batch \
    "open <url>" "wait --load networkidle" "get text body"

raw/ 文件格式（保存至 <工作目录>/raw/YYYY-MM-DD_<slug>.md）：
  # <页面标题>
  > 来源：<URL>
  > 抓取日期：<YYYY-MM-DD>
  > 关联问题：<问题文本>
  > 质量评分：高/中/低
  > 质量说明：<一句话>
  ---
  <正文>

完成后将新增文件路径列表作为你的最终输出（每行一个路径），以便主 agent 汇总。若所有 URL 均失败，输出"检索失败：<问题>"。
```

**并行注意事项**：
- 所有 subagent 共享同一个 `raw/` 目录，文件名用 slug 去重，同名跳过
- 若问题数量 ≤ 2，可在主 agent 中串行处理，无需启动 subagent
- **失败恢复**：某个 subagent 失败不阻断其他，等所有完成后再汇总
  - 若某问题的 subagent 完全失败（0 个文件），自动降级：用 WebSearch + WebFetch 重试
  - 若降级也失败，在摘要中标注"检索失败：<问题>"，继续后续流程
  - 若总体成功率 < 50%（超过一半问题无产出），在摘要中记录警告"本轮信息覆盖不足"，继续执行后续流程，不等待用户输入

---

### 单问题处理流程（subagent 内执行）

### 第一步：生成检索关键词

将问题转化为 2-3 组搜索关键词：
- 中文关键词（适合中文信息源）
- 英文关键词（适合英文学术/技术信息源）
- 专业术语变体（如有）

示例：
- 问题："RAG 的主要技术挑战是什么？"
- 中文关键词：`RAG 检索增强生成 技术挑战 问题`
- 英文关键词：`RAG retrieval augmented generation challenges limitations`

### 第二步：获取目标 URL

**实测结论（2026-04）**：设置真实 User-Agent 是绕过反爬的关键。**所有 agent-browser 命令都应加上 `--user-agent` 选项**：

```bash
REAL_UA="Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/124.0.0.0 Safari/537.36"
```

各搜索引擎实测结果：

| 搜索引擎 | 无 UA | 真实 UA | 备注 |
|---|---|---|---|
| **搜狗** sogou.com | CAPTCHA | ✅ 可用 | 中英文均好，`get text body` 直接获取 |
| **百度** baidu.com | CAPTCHA | ✅ 可用 | 中文结果丰富，需用 `snapshot -i` 获取（`get text body` 不返回结果） |
| **cn.bing.com** | 部分可用 | 部分可用 | 仅中文通用词有效，英文和专业术语常跑偏 |
| **DuckDuckGo** | 超时 | 超时 | 国内不可访问 |
| **Google** | CAPTCHA | CAPTCHA | 不可用 |

**推荐搜索策略（按优先级）：**

1. **搜狗**（首选，中英文均可用）：
```bash
agent-browser --user-agent "$REAL_UA" batch \
  "open https://www.sogou.com/web?query=<关键词>" \
  "wait --load networkidle" \
  "get text body"
```

2. **百度**（中文备选，结果丰富）：
```bash
agent-browser --user-agent "$REAL_UA" batch \
  "open https://www.baidu.com/s?wd=<中文关键词>" \
  "wait --load networkidle" \
  "snapshot -i -c -d 3"
```

3. **直接访问已知高质量站点**（当搜索引擎结果质量不佳时，按课题类型选择）：

   **技术 / AI**
   - 论文：arxiv.org（`arxiv.org/search/?query=<关键词>&searchtype=all`）
   - 论文+代码：paperswithcode.com
   - 开源项目：github.com/trending、github.com/search?q=<关键词>
   - 技术问答：stackoverflow.com、segmentfault.com、v2ex.com
   - 开发者社区：juejin.cn、developer.aliyun.com、cloud.tencent.com/developer

   **金融 / 投资**
   - 上市公司公告（中国官方）：cninfo.com.cn（巨潮资讯）
   - 行情与数据：eastmoney.com（东方财富）、xueqiu.com（雪球）
   - 财经深度：caixin.com（财新）、yicai.com（第一财经）
   - 创投动态：36kr.com、itjuzi.com（IT桔子）
   - 美股财报：sec.gov/cgi-bin/browse-edgar（SEC EDGAR）

   **行业调研**
   - 国内调研机构：iresearch.com.cn（艾瑞）、analysys.cn（易观）
   - 企业信息：qcc.com（企查查）
   - 商业观察：huxiu.com（虎嗅）、tmtpost.com（钛媒体）
   - 国际咨询：mckinsey.com/insights、bcg.com/publications

   **政策 / 法规（中国）**
   - 中央政策：gov.cn（国务院）、xinhuanet.com（新华社）
   - 各部委：ndrc.gov.cn（发改委）、miit.gov.cn（工信部）、mof.gov.cn（财政部）
   - 法律法规：chinalaw.gov.cn（中国政府法制信息网）
   - 判决案例：wenshu.court.gov.cn（中国裁判文书网）

   **学术研究**
   - 跨学科：semanticscholar.org（有API，AI驱动）
   - 医学：pubmed.ncbi.nlm.nih.gov（有Entrez API）
   - 预印本：medrxiv.org、biorxiv.org

   **医疗健康**
   - 临床试验：clinicaltrials.gov（有API）
   - 官方政策：nhc.gov.cn（国家卫健委）、nmpa.gov.cn（国家药监局）
   - 医学社区：dxy.com（丁香园）

   **创业 / 商业**
   - 创投新闻：techcrunch.com、cyzone.cn（创业邦）
   - 消费数据：cbndata.com（CBNData）
   - 思想观点：medium.com、36kr.com

从搜索结果中提取：标题、摘要、URL（通常 10 条左右）。

**Chrome 启动问题**：若 Chrome 报错退出（DevToolsActivePort 文件未生成），设置环境变量后重启：
```bash
AGENT_BROWSER_ARGS="--no-sandbox" agent-browser --user-agent "$REAL_UA" open ...
```

### 第三步：第一层质量筛选（抓取前）

对每条搜索结果，根据标题、摘要、域名判断质量：

**优先保留**：
- 学术来源：arxiv.org, scholar.google.com, 论文网站
- 官方文档：docs.*, *.readthedocs.io, 官方 GitHub
- 知名媒体/博客：知名技术博客、行业媒体
- 问答社区高赞回答：stackoverflow.com, zhihu.com（高赞）

**过滤掉**：
- 内容农场：标题党、关键词堆砌
- 纯广告页、落地页
- 登录墙（摘要中含"请登录"、"订阅后查看"）
- 与问题明显不相关的结果

每个问题选 **3-5 个** URL 进入下一步。多个关键词组的结果合并去重。

### 第四步：抓取正文

对每个筛选后的 URL，用 `batch` 合并多步命令减少启动开销：

```bash
agent-browser batch "open <url>" "wait --load networkidle" "get text body"
```

若页面内容过长（>50000 字符），改用 snapshot 并限制范围：
```bash
agent-browser batch "open <url>" "wait --load networkidle" "snapshot -c -d 5 -s article"
```

注意：`-s` 是 selector 参数（短选项），`get text` 的选择器直接跟在后面不加引号（如 `get text body`）。

提取正文，去除导航、页脚、广告等噪音。

### 第五步：第二层质量评分（抓取后）

对抓取到的正文评估：

| 维度 | 高质量 | 低质量 |
|---|---|---|
| 信息密度 | 有具体数据、案例、代码 | 空话、泛泛而谈 |
| 问题相关度 | 直接回答问题 | 擦边相关 |
| 来源可信度 | 作者可查、有引用 | 匿名、无出处 |
| 时效性 | 近 2 年内 | 过时信息（技术类尤其注意）|

评分结果：**高 / 中 / 低**，附一句话说明。

低质量内容仍然保存（保留原始记录），但在文件头部标注，wiki compile 时会降权处理。

### 第六步：写入 raw/

文件命名：`YYYY-MM-DD_<问题id>_<slug>.md`
- **问题 id 作为前缀**（如 `r1q3`），确保不同 subagent 并行写入时不会发生 slug 碰撞，同时方便追溯每个文件来自哪个问题
- slug 从页面标题生成（小写、空格转 `-`、去除特殊字符）
- 若同名文件已存在，先检查来源 URL 是否相同：
  - URL 相同：跳过（真正的重复）
  - URL 不同：在文件名末尾加 `-2`、`-3` 依此类推，保留不同来源

文件格式：

```markdown
# <页面标题>

> 来源：<URL>
> 抓取日期：<YYYY-MM-DD>
> 关联问题：<触发这次检索的问题文本>
> 质量评分：高/中/低
> 质量说明：<一句话评分依据>

---

<正文内容>
```

---

## 输出

所有文件写入完成后，输出摘要：

```
## 检索完成（第 N 轮）

处理问题：X 个
新增文件：X 个（列出文件名）
跳过（已存在）：X 个
抓取失败：X 个（列出 URL 和原因）

质量分布：高 X 个 / 中 X 个 / 低 X 个

新增文件列表：
- raw/2026-04-09_xxx.md [高质量]
- raw/2026-04-09_yyy.md [中质量]
```

同时返回新增文件列表供 autodidact 传给 wiki compile。

---

## 异常处理

- **页面加载超时**：跳过该 URL，记录失败
- **反爬拦截**（403/验证码）：跳过，尝试下一个 URL
- **正文为空**：跳过，记录原因
- **单个问题全部 URL 抓取失败**：在摘要中标注该问题"检索失败，需人工处理"
