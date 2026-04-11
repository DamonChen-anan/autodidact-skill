# autodidact

> An autonomous research agent built on [Karpathy's LLM Wiki](https://gist.github.com/karpathy/442a6bf555914893e9891c11519de94f) — give it a topic, it researches the internet and builds a structured wiki automatically.

[![MIT License](https://img.shields.io/badge/license-MIT-blue.svg)](LICENSE)
[![Built with Claude Code](https://img.shields.io/badge/Built%20with-Claude%20Code-black)](https://claude.ai/code)

[中文文档](README_CN.md)

Karpathy's LLM Wiki is a powerful pattern: instead of RAG, the LLM incrementally compiles raw sources into a persistent, interlinked wiki. The knowledge compounds over time.

**autodidact takes it one step further**: you don't even need to supply the sources. Give it a research topic, and it drives the entire loop autonomously — generating questions, searching the web, compiling the wiki, auditing gaps, and iterating until the topic is thoroughly covered.

```
Input:  /autodidact options selling strategies, targeting 10% annualized return

Output: 22 wiki articles, 43 sources — ~1 hour unattended
```

```
your-project/
├── raw/          (43 files — auto-fetched web sources)
├── wiki/
│   ├── INDEX.md
│   ├── options-basics.md              # definitions, payoff structure, key terms
│   ├── options-pricing-greeks.md      # Black-Scholes factors, IV, Delta/Gamma/Theta/Vega
│   ├── domestic-options-market.md     # tradable instruments, account requirements
│   ├── options-strategies.md          # covered call, CSP, wheel, 10% feasibility
│   ├── covered-call-execution.md      # strike/expiry selection, rolling, assignment handling
│   ├── options-risk-management.md     # stop-loss rules (2-3x premium), position sizing
│   ├── options-tools-resources.md     # IV percentile lookup, broker software
│   ├── spread-strategies.md           # bull put spread, bear call spread, iron condor
│   ├── historical-performance.md      # year-by-year results, bear market playbook
│   ├── beginner-roadmap.md            # 4-stage path (1–1.5 years), tax notes
│   ├── iv-percentile-guide.md         # IV percentile calc, high/low IV strategy selection
│   ├── wheel-strategy-execution.md    # full Wheel cycle, parameter selection
│   ├── options-liquidity-guide.md     # liquidity ranking by instrument and strike
│   ├── volatility-skew.md             # negative skew mechanics, impact on strike selection
│   ├── short-strangle.md              # construction, IV requirements, stop-loss, adjustment
│   ├── calendar-spread.md             # same-strike buy-far-sell-near, Theta+Vega long
│   ├── diagonal-spread.md             # PMCC poor man's covered call, vs calendar spread
│   ├── pin-risk.md                    # expiry pinning risk, Pinning Effect, avoidance
│   ├── margin-calculation.md          # put/call seller formulas, spread saves 70–82% margin
│   ├── contract-code-decoder.md       # 17-digit code structure, direction, expiry, strike
│   ├── csp-vs-covered-call.md         # annualized return comparison, assignment handling
│   └── iron-condor.md                 # Short Strangle + protective legs, margin savings
└── outputs/
    ├── research_state.json            (full research trajectory)
    ├── research_summary.md            (final summary)
    └── rounds/                        (audit reports per round)
```

---

## How it extends Karpathy's LLM Wiki

| | Karpathy LLM Wiki | autodidact |
|---|---|---|
| Core idea | LLM incrementally compiles a persistent wiki | Fully inherited ✓ |
| Source material | You collect and feed sources manually | **Auto-retrieved from the web** |
| Knowledge audit | Manual lint pass | **Automated every round, drives next round** |
| Research loop | None | **Auto question → search → compile → audit → repeat** |
| Usage | Manual ingest + query | **One command, fully autonomous** |

---

## How it works

Four steps per round, audit output drives the next round's questions:

```
┌─────────────┐     ┌────────────┐     ┌──────────────┐     ┌────────────┐
│  questioner │────▶│ researcher │────▶│ wiki compile │────▶│ wiki check │
│             │     │            │     │              │     │            │
│ reads audit │     │ parallel   │     │ quality-     │     │ gap audit  │
│ generates   │     │ web search │     │ weighted     │     │ outputs    │
│ questions   │     │ → raw/     │     │ → wiki/      │     │ report     │
└─────────────┘     └────────────┘     └──────────────┘     └─────┬──────┘
       ▲                                                           │
       └──────────────── drives next round ───────────────────────┘
```

**Stops when all conditions are met**: audit gaps diminishing (≤60% of previous round) + question overlap >70% + no pending user questions + no new raw files

---

## Requirements

### Required

- **[Claude Code](https://claude.ai/code)** — autodidact is a Claude Code skill plugin

### Recommended

- **[agent-browser](https://github.com/steel-dev/agent-browser)** — headless browser CLI that significantly improves retrieval success rate and content quality. Without it, autodidact falls back to Claude's built-in WebSearch, which works but with lower coverage.

  ```bash
  npm install -g agent-browser
  agent-browser install   # downloads Chromium
  ```

  > On macOS, if Chrome fails to start (DevToolsActivePort error), set: `export AGENT_BROWSER_ARGS="--no-sandbox"`

---

## Installation

Copy the four skills into Claude Code's skills directory:

**macOS / Linux**
```bash
git clone https://github.com/DamonChen-anan/autodidact
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

Restart Claude Code and type `/autodidact` to verify the installation.

---

## Usage

### Start researching

Open Claude Code in any **empty directory** and run:

```
/autodidact <topic>
```

On first run, `raw/`, `wiki/`, and `outputs/` are created automatically. No setup needed.

### Options

```
/autodidact <topic> [--max-rounds 20] [--background <what you already know>]
```

| Option | Description | Default |
|---|---|---|
| `topic` | Research subject — the more specific, the better | required |
| `--max-rounds` | Maximum number of research rounds | 20 |
| `--background` | Your existing knowledge, skips basic questions | none |

### Examples

```bash
# Technical research
/autodidact RAG architecture and engineering challenges in production

# Market research
/autodidact home care industry market landscape --max-rounds 5

# Skip basics with background knowledge
/autodidact options selling strategies 10% annual return --background "familiar with options basics and Greeks"
```

### Pause and resume

Ctrl+C to stop at any time. Re-run the same command in the same directory to resume from where it stopped.

### Insert your own questions

While research is running, edit `outputs/user_questions.md`:

```markdown
## Pending

- How does this strategy perform in a bear market?
- Is there any quantitative backtest data?
```

These are picked up at the start of the next round with highest priority.

---

## Use modules independently

All four skills can be used standalone:

```bash
# Compile raw/ sources into wiki/
/wiki compile

# Incremental compile (specific files only)
/wiki compile --files 2026-04-09_article.md

# Structural audit only (no question list)
/wiki check

# Full audit (question coverage + structural audit)
/wiki check outputs/rounds/round_5_questions.json --round 5
```

---

## FAQ

**Q: Does it work without agent-browser?**
Yes. It falls back to Claude's built-in WebSearch + WebFetch automatically. Some sites will be blocked and content quality may be lower. Installing agent-browser is recommended.

**Q: Does it support non-English topics?**
Yes. The researcher generates both Chinese and English search queries and retrieves from both language sources.

**Q: What if the context window fills up?**
autodidact checks context usage after each round. If it exceeds 50%, it prompts you to run `/compact` before continuing.

**Q: The wiki isn't deep enough after many rounds — what's wrong?**
Usually the topic is too broad. Use `--background` to skip basics, or insert specific questions via `user_questions.md` to steer the research.

---

## Acknowledgements

Wiki architecture based on [Andrej Karpathy's LLM Wiki](https://gist.github.com/karpathy/442a6bf555914893e9891c11519de94f).
