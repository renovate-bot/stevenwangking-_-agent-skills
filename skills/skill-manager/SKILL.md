---
name: skill-manager
description: |
  管理 Agent Skills 全生命周期：多源发现、去重溯源、评测对比、安全审查、经 skills CLI 安装/更新/卸载、各 Agent 链接验证与状态诊断。
  TRIGGER（中文）："找个适合 X 的技能""有没有现成的 X 技能""对比/评测一下这几个技能""安装这个技能""更新已装技能""卸载/删掉某技能""检查技能状态""技能装不上/看不见了""skill manager"。
  DO NOT TRIGGER：从零创作自研技能（用 skill-creator）；把自家技能发布到 GitHub 等多平台（用 skill-publisher）；项目版本发布流程（用 release-skills）；普通编码任务虽提及 skill 一词但无管理意图。
metadata:
  version: 1.2.0
  created: 2026-10-02
---

# Skill Manager：技能全生命周期管理

一条链路管到底：发现 → 去重溯源 → 评测（安全硬门槛）→ 推荐 → skills CLI 安装 → 链接验证 → 状态诊断。本技能是**决策编排层**，`npx skills` CLI（vercel-labs/skills）是**唯一执行层**。

## 核心原则

1. **CLI 是唯一执行层。** 安装/更新/卸载一律走 `npx skills`，禁止把 git clone / cp -R 当常规安装方式；只有 CLI 已完成安装但 Agent 链接缺失或损坏时，才允许安全地手动修复软链接。
2. **~/.agents/skills 是唯一实体。** 全局技能的真实文件只存在 `~/.agents/skills/<name>/`；symlink 类 Agent 目录（~/.claude/skills、~/.openclaw/skills 等）只能是软链接，绝不允许同一技能存在两份实体副本。
3. **安装前必过安全审查。** 安装与更新前必须读候选技能完整源码并按 `references/security-vetting.md` 审查；安全是硬门槛判定（PASS / WARNING / DANGER）而非评分项，DANGER 直接淘汰。
4. **只读自由，写入需意图。** 搜索、读源码、读元数据、查状态、非破坏诊断无需确认；安装、卸载、替换文件必须来自用户明确意图。
5. **验证通过才算完成。** 流程进度用 discovered / evaluated / recommended / installed / linked / verified 刻画（第八章的 OK / broken-link 等则是安装健康状态，两套词用途不同）；未完成验证的安装不得报告为成功。

## 本机 Agent 可见性模型

本机 Agent 分两类（CLI 安装时自己也这么分组，实测行为）：

**直读类（universal）**：CLI 注册表里技能目录就是 `.agents/skills` 的 Agent，安装**不建链、也不核查其链接**——Codex、Cursor、Gemini CLI、Antigravity、GitHub Copilot、OpenCode（其全局目录 `~/.config/opencode/skills` 独立，无链接即不核查）等。可见性判据：`~/.agents/skills/<name>/SKILL.md` 存在即可。

**ZCode 特例**：ZCode 应用自身直读 `~/.agents/skills`（新技能须新会话生效），但 CLI 注册表把 zcode 归为 symlink 类（全局目录 `~/.zcode/skills`，本机现为空）——安装**勿用 `-a zcode`**；装后发现 `~/.zcode/skills` 出现链接时如实报告（与直读叠加成双路可见）。

**symlink 类**：CLI 安装时在其目录创建指向 canonical 的软链接——Claude Code（`~/.claude/skills/`）、OpenClaw（`~/.openclaw/skills/`）、Qwen（`~/.qwen/skills/`）、Kilo Code（`~/.kilocode/skills/`）等。可见性判据：对应目录下 `<name>` 为软链接且解析后指向 `~/.agents/skills/<name>`。

两类归属与目录清单**以实际枚举为准，勿硬编码**：

```bash
ls -d ~/.claude/skills ~/.openclaw/skills ~/.qwen/skills ~/.kilocode/skills ~/.config/opencode/skills ~/.cursor/skills ~/.gemini/skills ~/.gemini/antigravity/skills ~/.codex/skills ~/.copilot/skills ~/.factory/skills 2>/dev/null
```

注意：Codex/Cursor/Gemini 虽有目录但本机无用户级链接，属直读类。`npx skills list -g` 的 Agents 字段记录的是安装时选择的 Agent（含直读类），**只对 symlink 类可当"链接本应存在"的预期**，对直读类一律不核查链接。链接拓扑以观测为准，不凭假设；不存在的目录直接跳过该列。

## 工作流路由

| 用户意图 | 走哪条链路 |
|---|---|
| 找技能 / 有没有现成的 | 发现 → 候选池去重 → 评测 → 推荐 |
| 安装 | 安全审查 → CLI 安装 → 三重验证 |
| 更新 | CLI 更新 → 重新安全审查 → 复验 |
| 卸载 | CLI 卸载 → 残留检查 |
| 查状态 / 装不上 / 看不见 | status 状态表 / 诊断修复 |

## 一、发现（Discovery）

1. **需求分析**：明确能力域、具体任务、使用场景（个人/公司项目）、要在哪些 Agent 里用。
2. **先查本机已装**：`npx skills list -g` 已有功能匹配的技能时直接纳入候选并优先评估；已装技能满足需求时如实告知，不重复安装（可建议 update）。
3. **关键词族**：围绕需求生成多组搜索词——能力名、任务名、领域名、常见技能命名、技术实现词、中英同义词（如 logo → icon / brand / mark / app icon）。
4. **多源搜索**（某源网络不可达则跳过并在报告注明，不静默丢弃）：
   - ① skills.sh：`npx skills find <关键词>`（可 `--owner <owner>` 限定），配合 https://skills.sh 排行榜——安装量数据以 skills.sh 为准，第一站。注意 `find` 必须带关键词参数，不带参数会进入交互式选择并把非交互会话挂起；误入交互态立即终止，改用 skills.sh 网页检索
   - ② GitHub 搜索（仓库与代码）
   - ③ SkillsMP（skillsmp.com，公开 SKILL.md 聚合索引）
   - ④ 腾讯 SkillHub（中文/国内生态、企业技能）
   - ⑤ ClawHub（Agent/自动化方向）
   - ⑥ NanoSkill（补充源）
5. **溯源**：聚合站找到的候选一律追溯回 GitHub 上游仓库，以仓库为评审对象；追不到上游的候选标记为风险项。

## 二、候选池与去重

- 节奏：10–20 个候选 → 去重（fork / 镜像 / 改名 / 二次封装一律认上游原始仓库）→ 初筛 5–8 个进入评测。
- 每候选采集（可得范围内）：名称、上游仓库、作者、GitHub stars、skills.sh 安装量、license、最近更新、依赖、支持的 Agent、是否含 scripts/、是否有网络请求、需要的环境变量、已知局限。

## 三、评测（四维）

细则见 `references/evaluation-framework.md`，要点：

- **A 功能匹配度（核心）**：把需求拆成能力项加权评分；能力项每次按实际需求现拆，勿套固定模板；无证据的能力项记 0 分，不猜测给分。
- **B 工程质量**：description 是否写明触发时机；SKILL.md 结构（scripts/references 划分、progressive disclosure）；单文件巨 prompt 扣分。
- **C 安全**：按 `references/security-vetting.md` 全量审查，**硬门槛判定（PASS / WARNING / DANGER）**，DANGER 淘汰、不进评分。
- **D 实用性**：安装复杂度、依赖、维护活跃度、与本机环境和在用 Agent 的匹配度（输出对比表里的"活跃度/易用性"两列即本维度拆出的子项）。

stars / 安装量只作活跃度参考，不当质量证明：1K+ 安装、官方源（anthropics、vercel-labs 等）是正面信号；<100 安装或无名作者则加强审查，而非直接否决。

## 四、推荐

输出：推荐候选 + 备选，各附优势、局限、安全结论（PASS/WARNING 及要点）、来源仓库、安装要求。禁止只凭 stars 或安装量推荐。

多个强候选难分胜负时，向用户建议做 Benchmark（协议见 evaluation-framework.md；属重流程，默认不主动跑，用户同意才执行）。

生态中确无合适技能时如实说明，fallback 建议用 skill-creator 自建。

## 五、安装

**前提**：用户明确表达安装意图；"帮我找一个" ≠ "装"。仅发现未点名安装时，停在推荐并给出安装命令。

1. **安全审查**：读候选仓库完整源码（含 scripts/ 全部文件），按 security-vetting.md 判定；WARNING 列出全部疑点请用户裁决，DANGER 淘汰并说明证据。
2. **执行安装**（本机约定一律全局安装，命令显式带 `-g`——注意 CLI 不带 `-g` 时默认装到项目目录）：
   ```bash
   # 默认：canonical 落 ~/.agents/skills；symlink 类 Agent 自动建链，直读类 Agent 直读无需链接（实测行为）
   npx skills add <owner/repo> --skill <skill-name> -g -y
   # 收窄：只给指定 Agent 安装（技能名含空格须加引号，如 --skill "Convex Best Practices"）
   npx skills add <owner/repo> --skill <skill-name> -g -a claude-code -a codex -y
   ```
   CLI 行为异常时，先换 `npx skills@latest ...` 重试一次再排查。
3. **三重验证**：
   - `~/.agents/skills/<name>/SKILL.md` 存在且可读；
   - symlink 类 Agent 的链接正确（`ls -la` + `readlink`，对照上方可见性模型；直读类不核查链接，canonical 存在即可）；
   - frontmatter 的 name 与目录名一致。
4. **回报**：安装位置、来源、各 Agent 可见性、安全结论；提醒 ZCode 须**新会话**才加载新技能（实测结论），其他 Agent 的加载时机按其自身机制提示。

## 六、更新

```bash
npx skills update [技能名...] -g -y
```

**update 与 remove 同红线**：自制技能与身份存疑技能不执行 CLI update——simplify 实测 Source 为 brianlovin/agent-config 但属自制，update 会以上游版本覆盖本地修改（`-y` 免交互，跳过 scope 提示）。全量 update 前先核对将被触碰的技能清单，只点名更新第三方技能。

更新后必须：① 重新安全审查——v1.0 安全不代表 v1.1 安全，重点 diff 新增的 scripts 与网络请求；② 链接复验；③ 报告实质变化（description 改动、新增依赖、行为变化），无实质变化则一句话带过。

## 七、卸载

```bash
npx skills remove <skill-name> -g -y
```

- 卸载后验证：`~/.agents/skills/<name>` 与各 Agent 链接均不存在。
- **孤儿链接清理**：canonical 已删但 Agent 链接残留时——先确认该路径确为软链接且指向已删除的 canonical，再仅 `rm` 链接本体；未验证指向关系前禁止 `rm -rf`。
- **红线（无确认例外）**：`~/.agents/skills` 下的文件永不物理删除。注意 `npx skills remove <name> -g` 会**递归强删 canonical 实体**，且在 AI agent 环境下自动跳过确认（实测源码行为）——因此对 `~/.agents/skills` 的 CLI remove 仅限"用户点名卸载的第三方技能"（Benchmark 临时目录内自装自清不在此列）；自制技能与身份存疑技能**一律不执行 CLI remove**（哪怕它出现在 list 里、哪怕用户催促确认），只给替代方案（移入 ~/.agents/skills-disabled 停用、仅移除 Agent 链接、用户自行归档）。自制技能的参考判据是 `npx skills list -g` 中 Source 为 `local`（如 commit、reply-polish）；注意经 CLI 安装过的自制技能会带上游 Source（如 simplify 实测显示为 brianlovin/agent-config）——身份存疑时向用户求证，勿只认 Source 字段。

## 八、status（技能状态表）

`npx skills list -g` 取 CLI 清单（其 Agents 字段是"该技能面向哪些 Agent 安装"的参考依据，注意会截断），再逐一做文件系统核查。Agent 列按枚举结果动态生成，示例：

| Skill | Canonical | Claude Code | Codex | 状态 |
|---|---|---|---|---|
| \<name-a\> | ✓ | ✓ | - | OK |
| \<name-b\> | ✓ | ✗ | - | broken-link |
| \<name-c\> | ✗ | 残留 | - | orphan-link |
| \<name-d\>（自制） | ✓ | ✓ | - | local |
| \<name-e\> | ✓ | 冲突 | - | conflict |

单元符号：`✓` 链接存在且解析正确；`✗` 本应有链接但缺失或断裂；`残留` 链接还在但 canonical 已删；`冲突` 该位置是实体目录或链接指向别处；`-` 该技能从未面向此 Agent 安装或该 Agent 属直读类（正常，不是问题）。

状态定义：

- `OK`：canonical 存在，且本应存在的 symlink 类链接均正常（直读类只需 canonical 存在）
- `broken-link`：canonical 存在，但本应存在的 symlink 类链接（Agents 字段列出该 symlink 类 Agent，或 add 时用 `-a` 指定过）缺失或断裂。**只核查 symlink 类**——Agents 字段里的直读类 Agent（Codex/Cursor/Gemini CLI 等）从不据此报 broken-link，否则会整列误报；zcode 虽在 CLI 注册表归 symlink 类，但 ZCode 应用直读 canonical，`~/.zcode/skills` 为空属正常态，同样不据此报 broken-link（该目录出现链接时仅如实备注双路可见）。Agents 字段会截断（前 5 个 + "+N more"）：无法确认某 symlink 类 Agent 是否在列时以链接实测为准——链接在记 ✓；链接不在且该 Agent 未直接出现在字段里，记 `-` 并注明"字段截断无法确认"，不凭猜记 ✗
- `orphan-link`：canonical 不存在，但 Agent 目录下链接残留
- `conflict`：Agent 路径上是实体目录，或链接解析后目标 ≠ `~/.agents/skills/<name>`（比较前对目标做 realpath/去尾斜杠归一化；绝对路径链接归一化后指向 canonical 即正常，不算 conflict）
- `local`：`npx skills list -g` 显示 Source 为 `local` 的自制/手工安装技能（正常现象，不视为问题）。状态列**优先标 `local`**；其在 Agent 目录的既有实体目录布局（如 agent-reach）单独备注即可，不按 conflict 给修复建议

发现 broken-link / orphan-link / conflict 时列 Problems 清单并给出修复建议，等用户指示再动手。批量核查示例（逐 Agent 目录替换执行）：

```bash
for d in ~/.claude/skills/*; do printf '%s -> %s\n' "$(basename "$d")" "$(readlink "$d" 2>/dev/null || echo 非链接)"; done
```

## 九、诊断修复（装不上 / 看不见）

按序排查：① `npx skills --version` 确认 CLI 可用 → ② 查 canonical 是否存在 → ③ 查对应 Agent 目录是否为链接、指向是否正确（③⑤ 仅针对 symlink 类 Agent；直读类无链接概念，canonical 在即可见）→ ④ 问题在 CLI 环节则重跑 add（必要时 `skills@latest`）→ ⑤ 仅当 CLI 已装好而链接缺失/损坏时手动修复：

```bash
# 相对路径形式，与本机既有链接（../../.agents/skills/<name>）及 CLI 行为一致
ln -sfn ../../.agents/skills/<name> ~/.claude/skills/<name>
```

手动修复仅允许三种情形：目标不存在 / 目标是断链 / 用户确认替换；目标是实体目录或他人链接时停下报告。修复后复验 SKILL.md 可读。禁止不诊断就重装覆盖。

## 确认规则

**需要用户确认**：安装未被点名的技能；用户未明确要求的卸载；替换任何实体文件/目录或冲突链接；带 WARNING 的安装；执行候选技能自带的 scripts（vetting 阶段只读不执行）。

**无需确认**：搜索、读源码、读元数据、查安装状态、查链接、非破坏性诊断命令。

## 输出模板

搜索/推荐报告（**结论先行 → 查询摘要 → 对比表**三段式，2026-10-02 用户四轮反馈定稿）：

```
## 结论
推荐 <name>：一句话核心理由 + 关键前提/代价（如依赖、付费）。
安装命令放表格之后，标"确认后执行"。

## 查询摘要
查了哪些源、什么关键词族、共 N 个候选、几个进入评测；被排除/未评测的候选在此点名（名称 + 一句话原因），不进对比表——摘要是它们的唯一去处，保证不静默丢弃。2–3 句以内。

## 对比（仅评测过的候选）

| 候选 | 来源 | 功能匹配 | 安全 | 易用性 | 结论 |
|---|---|---|---|---|---|
| <name> | owner/repo · 安装量 · license | 高（覆盖…）/ 低（缺…） | PASS / WARNING(…) / DANGER(淘汰) | 高/中/低（依赖…） | 推荐/备选/淘汰 |
```

- **表格只收进入评测的候选**：每行须有真实判定（读源码或官方描述核验过）；未评测候选进表只是整行"未评测/未审/—"，零信息且稀释可信度（2026-10-02 用户第四轮反馈）。被评测后淘汰（含 DANGER）的候选属评测产物，留在表中标"淘汰"。
- **表格默认 6 列**：候选、来源（上游/作者 + 安装量/stars + license 合一列）、功能匹配、安全、易用性、结论；工程质量 / 活跃度等信息并入"来源"列短语，确有横向对比价值才另开列——列多是难读的另一来源。
- **表格行自明**：不用合并行 + 编号脚注（读者须视线跳转解码）；少量同类小候选直接并入查询摘要点名。
- 评测候选 ≥2 个必附对比表；单一候选免表，判定写进结论段。
- 安全列只写判定不写分数；行内个别维度证据不足时如实标"未评测"，不猜测给分；细节（schema、安全要点、依赖）按需在表格后简述。

安装报告：Skill、来源、安装位置、各 symlink 类 Agent 链接状态（✓/✗/修复记录；直读类 canonical 在即可见）、安全结论、"新会话生效"提示。
卸载报告：已移除项 + 残留检查结果。
状态表：见第八章格式。

## 边界

- 不修改任何第三方技能的内容；发现质量问题时报告给用户，不代改。
- Benchmark 属可选重流程，仅在用户要求或同意时执行，协议见 evaluation-framework.md。
- Web 搜索不可用时降级为 skills.sh CLI 单源搜索，并在报告中注明覆盖受限。
- 与共存技能分工：英文"find a skill for X"式的简单快速查找可由 find-skills 承接；OpenClaw 风格手动安全清单可由 skill-vetter 承接；本技能负责全生命周期——多源发现、评测、经 CLI 的安装/更新/卸载、多 Agent 链接管理与诊断，以及安装前的安全把关。
