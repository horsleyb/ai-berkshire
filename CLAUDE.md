# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## 项目概述

基于 Claude Code 的价值投资研究 Skill 合集。四大师框架：巴菲特、芒格、段永平、李录。
GitHub: xbtlin/ai-berkshire。同时维护 Codex 兼容层（见下方“Skill 双系统架构”），两套客户端共用 `tools/` 下的数据/校验工具。

## 常用命令

### 安装 Skill 到本地客户端

```bash
./scripts/install-claude-commands.sh   # 复制 skills/*.md 到 ~/.claude/commands/（可用 CLAUDE_COMMANDS_DIR 覆盖目标目录）
./scripts/install-codex-skills.sh      # 安装 codex-skills/ 到本地 Codex
./scripts/install-codex-prompts.sh     # 安装 codex-prompts/ 到本地 Codex
```

### 修改 `skills/*.md` 后必须同步 Codex 产物

```bash
python3 scripts/sync-codex-skills.py             # 从 skills/*.md 重新生成 codex-skills/*/SKILL.md
python3 scripts/sync-codex-skills.py --check      # 只校验生成产物是否最新，不写文件（改动前/CI用）
python3 scripts/sync-codex-prompts.py             # 重新生成 codex-prompts/*.md
python3 scripts/sync-codex-prompts.py --check
```

### 金融工具 CLI（零外部依赖，仅 Python stdlib，要求 Python ≥ 3.7）

```bash
# 市值/估值精确验算（Decimal 精度，禁止心算替代）
python3 tools/financial_rigor.py verify-market-cap --price 510 --shares 9.11e9 --reported 4.65e12 --currency HKD
python3 tools/financial_rigor.py verify-valuation --price 510 --eps 23.5 --bvps 120 --fcf-per-share 18 --dividend 2.4
python3 tools/financial_rigor.py cross-validate --field revenue --values '{"年报": 7518, "Yahoo": 7500}' --unit 亿
python3 tools/financial_rigor.py three-scenario --price 510 --eps 23.5 --shares 91.1 --growth 15 8 -5 --pe 30 22 15
python3 tools/financial_rigor.py benford --values '[1234, 2345, 3456]'

# 报告发布前的数据抽检门禁（提取 → 抽样15% → 核验 → 准出/打回）
python3 tools/report_audit.py extract --report reports/xxx.md --dry-run
python3 tools/report_audit.py verdict --results '[...]'

# 台股数据（FinMind API 封装）
python3 tools/twstock_data.py quote 2330
python3 tools/twstock_data.py valuation 2330
python3 tools/twstock_data.py financials 2330
python3 tools/twstock_data.py revenue 2330
python3 tools/twstock_data.py dividend 2330
python3 tools/twstock_data.py search 台積

# A股行情/财务（腾讯行情 + 东方财富）
python3 tools/ashare_data.py quote 600519
python3 tools/ashare_data.py financials 600519
python3 tools/ashare_data.py valuation 600519
python3 tools/ashare_data.py search 关键词
```

无测试套件、无 lint 配置、无 CI workflow——本仓库的“正确性”校验就是上述工具的输出（市值验算、交叉验证、抽检）以及手动跑一遍对应 skill 确认输出可用。

## 架构

### Skill 双系统架构（Claude Code 为源，Codex 为生成产物）

- `skills/*.md` 是**唯一权威源**——Claude Code slash-command 格式，写作/修改研究流程都改这里。
- `codex-skills/*/SKILL.md` 与 `codex-prompts/*.md` 是从 `skills/*.md` **自动生成**的 Codex 兼容层（见 `scripts/sync-codex-skills.py`、`scripts/sync-codex-prompts.py`）。改完 `skills/` 后必须重新生成，禁止直接手改生成产物。
- 例外：`codex-skills/investment-memo-craft/` 是 Codex-only 手写包，没有对应的 `skills/*.md` 源文件，明确标注为 Codex-only，新增此类文件时同理标注。
- `AGENTS.md` 记录 Codex 行为规范，`CLAUDE.md`（本文件）记录 Claude Code 行为规范——两者内容需要保持一致的项目约定，但各自面向不同客户端。

### Skill 执行模式：单 Agent 分析 vs 多 Agent 团队

- 多数 skill（investment-research、investment-checklist、thesis-tracker、portfolio-review 等）是单会话内直接执行的结构化分析框架。
- `investment-team`、`earnings-team` 用 TeamCreate/TaskCreate 拉起 4 个并行子 Agent，分别以段永平（商业模式）/巴菲特（财务估值）/芒格（行业竞争）/李录（风险与管理层）视角研究，**目的是互相挑战制造张力，而非简单分工**。
- **多 Agent skill 有强制前置检查**：用 `run_in_background: true` 启动子 Agent 前，必须确认 `.claude/settings.local.json`（项目级或用户级）的 `permissions.allow` 中已放行 `WebSearch`。后台 Agent 无法弹出交互式权限确认，一旦 WebSearch 被静默拦截，子 Agent 会退化为仅凭训练知识作答，却仍输出一份看似完整、实则未联网的伪研究（这是历史上出现过的最危险失败模式，详见 `skills/investment-team.md` 的“WebSearch 权限预检”步骤）。

### 数据获取与交叉验证分层

- 权威规范见 `skills/financial-data.md`：每个关键数据必须来自 ≥2 个独立来源，误差 >1% 须标记。按市场划分数据源优先级（美股 macrotrends/stockanalysis、港股 aastocks、A股东方财富、台股 FinMind）。
- 台股：`tools/twstock_data.py` 封装 FinMind API，零依赖。Token 读取顺序为环境变量 `FINMIND_TOKEN` → 本地文件 `local/finmind_token.txt`（`local/` 已被 `.gitignore` 永久排除）→ 匿名访问（小时级限额）。Token 绝不能出现在报告、skill 或 commit 中。
- A股：`tools/ashare_data.py` 走腾讯行情 + 东方财富接口获取实时行情、估值、近5年财务。
- 港股/美股：直接访问公开数据源网页，无专用脚本封装。

### 金融计算精确性与发布门禁

- `tools/financial_rigor.py` 用 `decimal.Decimal`（非 float）做市值验算、估值比率、三情景估值、Benford's Law 检测，避免浮点误差；skill 执行中禁止心算替代，必须通过 Bash 调用该工具并把输出嵌入报告。
- `tools/report_audit.py` 是报告发布前的抽检门禁：`extract` 用正则从 Markdown 报告提取数据点并随机抽样 15%（最少3个最多30个）→ 人工/Claude 对照可靠信源填入实际值 → `verdict` 输出准出/打回判决及原因。`investment-team`、`industry-research`、`industry-funnel` 等 skill 在收尾阶段都会跑这一步。

### 报告目录结构

所有报告按**公司名**建文件夹，公司相关的所有报告放在对应文件夹内：

```
reports/
├── AI产业研究/              — AI产业链全景研究（置顶）
│   ├── AI五层蛋糕-产业全景研究-20260605.md
│   └── AI五层蛋糕-公众号-20260605.md
├── 腾讯/                    — 腾讯所有研究报告
│   ├── 腾讯-research-20260408.md
│   ├── 腾讯-earnings-2025Q4.md
│   ├── 腾讯-management-20260409.md
│   └── 腾讯-thesis.md
├── 拼多多/                  — 拼多多所有研究报告
├── 泡泡玛特/                — 泡泡玛特所有研究报告
├── community/               — 外部贡献者用本框架产出的报告（PR 提交，见 CONTRIBUTING.md）
├── 核电-industry-20260409.md — 行业报告放根目录
├── AI算力-funnel-20260509.md  — 漏斗筛选报告放根目录
├── AI-轮动判断-20260509.md    — 主题级综合判断报告放根目录
├── portfolio-latest.md       — 组合报告放根目录（.gitignore 排除，只存本地）
└── 多公司对比-checklist-20260408.md — 多公司报告放根目录
```

`实盘记录/`（公开权重口径的实盘操作记录）与 `筛选公司/`（全市场初筛结果）是根目录下的独立目录，同样是个人研究产出，不属于 `reports/`。

### 报告命名规范

| Skill | 文件命名格式 | 示例 |
|------|---------|------|
| /investment-team | `{公司名}/` 目录内含4个视角+最终报告 | `reports/拼多多/最终报告.md` |
| /investment-research | `{公司名}-research-{YYYYMMDD}.md` | `reports/腾讯/腾讯-research-20260408.md` |
| /investment-checklist | `{公司名}-checklist-{YYYYMMDD}.md` | `reports/腾讯/腾讯-checklist-20260408.md` |
| /industry-research | `{行业名}-industry-{YYYYMMDD}.md`（根目录） | `reports/核电-industry-20260409.md` |
| /industry-funnel | `{行业名}-funnel-{YYYYMMDD}.md`（根目录） | `reports/AI算力-funnel-20260509.md` |
| /private-company-research | `{公司名}-private-{YYYYMMDD}.md` | `reports/字节跳动/字节跳动-private-20260408.md` |
| /earnings-review | `{公司名}-earnings-{期间}.md` | `reports/腾讯/腾讯-earnings-2025Q4.md` |
| /earnings-team | `{公司名}/` 目录内含4个大师视角+研究底稿+公众号文章+读者评审 | `reports/腾讯/腾讯-earnings-2025Q4.md`（公众号定稿） |
| /thesis-tracker | `{公司名}-thesis.md`（长期维护） | `reports/腾讯/腾讯-thesis.md` |
| /portfolio-review | `portfolio-latest.md`（根目录，持续更新） | `reports/portfolio-latest.md` |
| /management-deep-dive | `{公司名}-management-{YYYYMMDD}.md` | `reports/腾讯/腾讯-management-20260409.md` |

### /investment-team 文件结构

```
reports/{公司名}/
├── README.md                         — 研究框架概览+核心结论
├── 01-商业模式分析-段永平视角.md
├── 02-财务估值分析-巴菲特视角.md
├── 03-行业竞争分析-芒格视角.md
├── 04-风险管理层评估-李录视角.md
└── 最终报告.md                       — Team Lead 综合报告
```

## 投研分析核心原则（最高优先级）

- **客观、客观、客观**——所有投研分析必须基于事实和数据，严禁主观臆断
- 严格区分"事实"与"观点"：事实用数据支撑，观点必须明确标注为"观点"或"推测"
- **不预设立场**：不预设看多或看空，先摆数据、再推逻辑、最后得结论。结论必须从数据中自然推出
- 禁止使用"我认为"、"我觉得"、"显然"等主观表述，改用"数据显示"、"证据表明"、"根据XX来源"
- **呈现正反两面**：每个核心判断都必须附带反面论据（"但另一方面..."），让读者自己权衡
- 对不确定的事情诚实说"不确定"或"数据不足"，不要用推测填充确定性
- 所有skill（investment-team、investment-research、earnings-review等）在执行时都必须遵守以上原则

## 报告语言与风格

- 所有报告使用**中文**
- 风格：直接、犀利、不说废话
- 数据必须标注来源，关键数据至少2个来源交叉验证
- 估计值必须注明"估计"
- 评分使用★符号（★1-5），不含半星
- 穿插巴菲特/芒格/段永平/李录的语录点评

## 仓库编辑边界

- `reports/` 下已有报告（`community/` 除外）、`实盘记录/`、`筛选公司/` 是维护者本人的研究产出和交易记录，**不接受结论性修改**（如“这家公司不该是这个评级”）。外部贡献的研究报告只能提交到 `reports/community/{公司名}/`，详见 `CONTRIBUTING.md`。
- 改动 `skills/*.md` 后必须跑 `scripts/sync-codex-skills.py`（及需要时 `scripts/sync-codex-prompts.py`），保持 Codex 产物同步；不要手改 `codex-skills/`、`codex-prompts/` 下的生成文件。
- `local/` 目录、任何 `*_token.txt` / `.env*` 文件永不提交（已在 `.gitignore` 中永久排除），FinMind 等 API token 只能落在本机。

## GitHub 操作

- 本地克隆路径：`~/ai-berkshire/`
- 远程仓库：`https://github.com/xbtlin/ai-berkshire.git`
- 推送前先 `git pull --rebase origin main`（远程经常有新提交）
- commit message 用中文，描述清楚改了什么
- 不要推送中间过程文件（如 data_collection.md），只推最终报告

```bash
# 推送报告到GitHub
cd ~/ai-berkshire
git add reports/xxx.md
git commit -m "添加xxx报告"
git pull --rebase origin main
git push origin main
```

## 注意事项

- 市值必须手算校验：股价 × 总股本，与报告市值对比
- 货币单位要明确（港币/人民币/美元/新台币），防止混淆
- PE/ROE等指标用 tools/financial_rigor.py 精确计算
- 台股数据用 tools/twstock_data.py（FinMind）获取，并按 skills/financial-data.md 台股章节交叉验证
- 报告写完后主动询问是否推送到GitHub
