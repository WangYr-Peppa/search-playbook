# 跨平台接入指南

> **核实日期：2026-09-27**（除注明"搜索快照/UGC"外均为官方文档直读）
> 目标：把本仓库的**纪律**接到你实际在用的平台上 —— 而不是绑死在某一个。

---

## TL;DR

1. 🎯 **`<name>/SKILL.md` 正在成为跨平台事实标准**。Claude Code、OpenAI Codex、DeepSeek Harness、WorkBuddy **都用这个约定** → **本仓库的 `SKILL.md` 几乎可原样移植**。
2. 🔑 **`.agents/skills/`** 这个路径被 **Codex 与 DeepSeek Harness 同时支持** —— 放这里可以一份给两家用。
3. 📄 **纯对话版**（无 agent）→ 用 [`chat-only-prompt.md`](./chat-only-prompt.md)。
4. ⚠️ 平台格式**变动很快**（本次就发现 Gemini CLI 已对个人账户停服）→ 用前请复核官方文档。

---

## 总表

| 平台 | 技能(skill)放哪 | 指令/规则放哪 | 子 agent | 移植难度 |
|---|---|---|---|---|
| **OpenCode**（本仓库原生） | `~/.config/opencode/skills/<n>/SKILL.md` | `AGENTS.md`（全局或项目） | 需插件（如 OMO-Slim） | — |
| **Claude Code** | `~/.claude/skills/<n>/SKILL.md` 或 `.claude/skills/` | `~/.claude/CLAUDE.md`（v2.1.277+ 也认 `AGENTS.md`） | ✅ `~/.claude/agents/*.md` | ⭐ 极低 |
| **OpenAI Codex** | `.agents/skills/<n>/SKILL.md`（扫到 repo root）或 `~/.agents/skills/` | `~/.codex/AGENTS.md`；项目逐目录 `AGENTS.md` | ⚠️ 无文件规范（未证） | ⭐ 极低 |
| **DeepSeek Harness** (`dsh`) | `.dsh/skills/` > `.agents/skills/` > `~/.dsh/skills` | `cordis.yml` / preset | ✅ 插件 | ⭐ 极低 |
| **WorkBuddy**（腾讯云） | `.codebuddy/skills/<n>/SKILL.md` | `~/.workbuddy/`（**推断**） | ✅ 有 subagents | ⭐ 低 |
| **Cursor** | 有 Skills 专页，**路径未证** | `.cursor/rules/*.mdc`（**必须 `.mdc`**）；也读 `AGENTS.md`/`CLAUDE.md` | ✅ `.cursor/agents/*.md` | 中 |
| **Gemini CLI** ⚠️ | 扩展内打包：`~/.gemini/extensions/<n>/` | `~/.gemini/GEMINI.md` | ✅ 扩展可含 sub-agents | 中 |
| **纯对话版** | — | 各平台"自定义指令" | — | 见 §7 |

---

## 1. Claude Code ⭐ 首选移植目标

| 机制 | 路径 | 关键点 |
|---|---|---|
| **Skills** | `~/.claude/skills/<name>/SKILL.md`（用户）／`.claude/skills/`（项目，向上扫到 repo root） | YAML frontmatter：`description`（用于自动触发）、`name`；可选 `disable-model-invocation` / `user-invocable` / `allowed-tools`。正文支持 `$ARGUMENTS`、`${CLAUDE_SKILL_DIR}`、`!`cmd`` 注入 |
| **Subagents** | `~/.claude/agents/**/*.md` / `.claude/agents/**/*.md` | frontmatter：`name`、`description`（必需）、`tools`、`model`、`permissionMode`、**`skills`**（启动时预载全文）、`maxTurns`、`effort`、`background`… |
| **指令** | 加载序（宽→具体）：Managed → `~/.claude/CLAUDE.md` → `./CLAUDE.md` → `./CLAUDE.local.md` | v2.1.277+ **支持 `AGENTS.md`**（默认二者取一，可在 `/config` 改成同时读） |

**接入动作**
```bash
# 1) 技能
mkdir -p ~/.claude/skills/search-playbook
cp SKILL.md            ~/.claude/skills/search-playbook/
cp -r reference        ~/.claude/skills/search-playbook/
# 2) 分档调度规则 → 并进 ~/.claude/CLAUDE.md（见 templates/agent-dispatch-tiering.md）
# 3) 执行端兜底 → 做成一个 subagent：~/.claude/agents/searcher.md
#    正文用 templates/agent-append.example.md，并在 frontmatter 里写 skills: [search-playbook]
```
> ✅ **几乎零改动**：`SKILL.md` 的 `name`/`description` 字段本来就符合。

---

## 2. OpenAI Codex

| 机制 | 路径 | 关键点 |
|---|---|---|
| **Skills** | 仓库：`.agents/skills/<name>/`（向上扫到 repo root）；用户：`~/.agents/skills/` | `SKILL.md` 必含 `name` + `description`；可带 `scripts/ reference/ assets/`；调用 `/skills` 或 `$skill`；**技能列表预算 = 上下文 2%** |
| **指令** | 全局 `~/.codex/AGENTS.override.md`（优先）或 `~/.codex/AGENTS.md`；项目逐目录 `AGENTS.md`（越靠近 CWD 越强） | 纯 markdown，无 frontmatter |

**接入动作**：把 `SKILL.md` + `reference/` 放到 `.agents/skills/search-playbook/`；分档调度规则并进 `AGENTS.md`。
> ⚠️ Codex **未见文件型自定义 subagent 官方规范** → "编排者派子 agent"那半套仅能做**提示词级**模拟。

---

## 3. DeepSeek Harness（`dsh`）—— 官方开源 agent 框架

- **真实存在**：官方仓库 `github.com/deepseek-ai/deepseek-harness`（MIT，开发者预览），`npx @deepseek-ai/dsh web`。
  ⚠️ `dshagent.com` / `deepseekagent.io` 等是**仿站/社区站**，不是官方。
- **技能目录优先级**（高→低）：
  `<projectRoot>/.dsh/skills` → `<projectRoot>/.agents/skills` → 配置 `customSkillDirs` → `<dshHome>/skills` → `<agentsHome>/skills` → 内置
- **格式**：`<name>/SKILL.md` 或扁平 `<name>.md`（kebab-case）；支持 `modelInvocable` / `userInvocable`
- **子 agent / hook**：以**插件**形式提供（含 Claude Code / Codex 的 hook 桥）

**接入动作**：把 `SKILL.md` 放进 `<项目>/.agents/skills/search-playbook/`（**与 Codex 共用同一份**）。
> ✅ 这是最省事的平台 —— 直接复用 `.agents/skills/` 约定。

---

## 4. WorkBuddy（腾讯云，AI Agent 办公工作台）

- **真实存在**：`workbuddy.cn` / 国际站 `workbuddy.ai`；基于 OpenClaw 架构，**支持多 Agent 并行**。
- **Skills**：工作区 `.codebuddy/skills/<skill-name>/SKILL.md`；frontmatter `name`、`description`（必需）+ 可选 `allowed-tools`、`disable`；可带 `scripts/ references/ assets/`；对话内用 `/` 调用；设置页有 Project / User 两级管理（项目级优先）
- **子 agent**：官方文档导航含「subagents 使用指南」+ Hooks ✅
- ⚠️ 用户级 `~/.workbuddy/skills/` 与 `npx skills add` 属**推断**（来自腾讯云社区文章，非产品核心文档）

**接入动作**：`SKILL.md` 放 `.codebuddy/skills/search-playbook/`。

---

## 5. Cursor

| 机制 | 路径 | 关键点 |
|---|---|---|
| **Project Rules** | `.cursor/rules/**/*.mdc`（**必须 `.mdc`** —— 纯 `.md` 会被 rules 系统忽略） | frontmatter：`description` / `globs` / `alwaysApply` → 决定 4 种应用模式 |
| **指令** | 项目根与子目录的 `AGENTS.md`；`CLAUDE.md` 也读且**无条件 always applied** | 纯 markdown |
| **Subagents** | 项目 `.cursor/agents/*.md`；用户 `~/.cursor/agents/*.md`（**兼容读 `.claude/agents/`、`.codex/agents/`**） | frontmatter：`name`、`description`、`model`（可带 `[effort=high]`） |
| Skills | 有专页，**路径未证** | — |

**接入动作**：分档调度规则做成 `.cursor/rules/search-discipline.mdc`；`SKILL.md` 的**正文**可转为 rule 或放到 agents 的 prompt 里（Cursor 的 Skills 路径请自查当前文档）。

---

## 6. Gemini CLI ⚠️ 时效警告

> 🔴 **2026-06-18 起，Gemini CLI 对免费 / Google AI Pro / Ultra 个人账户停服**，迁移到闭源 **Antigravity CLI**（承接了 Skills / Hooks / Subagents / Extensions → plugins）。企业许可与付费 API key 路径仍可用。

仍在用的路径：
- 上下文：`~/.gemini/GEMINI.md` → workspace 及父目录 → JIT 子目录；文件名可在 `settings.json` 的 `context.fileName` 改（例如改成 `AGENTS.md`）
- 自定义命令：`~/.gemini/commands/**.toml`（必填 `prompt`）
- **扩展**：`~/.gemini/extensions/<name>/` + `gemini-extension.json`（可打包 custom commands / hooks / **sub-agents** / **agent skills**）

**接入动作**：把技能与规则打包成一个 extension。

---

## 7. 纯对话版（无 agent、无 skill）

→ 直接用 **[`chat-only-prompt.md`](./chat-only-prompt.md)**：贴进各平台的"自定义指令"。

| 平台 | 入口（官方名） | 字数 |
|---|---|---|
| ChatGPT | Settings → Personalization → **Custom instructions**；另有 Projects / GPTs | Free 1,500 / 付费 5,000 |
| Claude 网页版 | Settings → **Instructions for Claude**；Projects → Set project instructions；Styles | Org 指令 3,000（官方明载）；profile/project **未公布** |
| Gemini 网页版 | 侧栏 **Gems** → New Gem（Instructions + Knowledge）；Settings → **Personal Intelligence → Instructions for Gemini** | 官方**未公布**（社区实测数字互相冲突，未采信） |
| DeepSeek 网页版 | ⚠️ **未找到官方自定义指令入口**（负证据：官网/api-docs 均无） | — |

> 若平台没有持久"自定义指令"，就把 prompt 作为**每次对话的第一条消息**贴上。

---

## 未证 / 注意（下次复核）

1. **Codex 的文件型自定义 subagent 规范** —— 官方未见专页。
2. **Cursor Skills 的确切路径与 frontmatter** —— 仅确认文档有专页，未取页。
3. **ChatGPT Projects/GPTs、Claude profile/project、Gemini Instructions/Gems 的官方字符数** —— 官方不公布，社区数字冲突。
4. **Cursor 的 `.mdc` vs `.md`** —— 两处官方页说法冲突，本文取"现行主页：必须 `.mdc`"。
5. **Antigravity CLI 自身的配置路径/格式** —— 未查。
6. ⚠️ **平台格式变动极快**（本次就撞上 Gemini CLI 停服）→ **每隔几个月复核一次**。

---

## 移植原则（为什么不用为每个平台重写）

```
本仓库的 SKILL.md / manual.md = 【内容】（平台中立）
        │
        ├─ 各平台只负责回答一个问题：这份内容【放哪、叫什么、什么格式】
        │
        └─ 于是接入工作 = 复制文件 + 改放置位置，而不是重写
```

**唯一真正依赖平台架构的是"分档调度"** —— 它要求存在"编排者 + 子 agent"。
- 有 subagent 的平台：Claude Code / Cursor / DeepSeek Harness / WorkBuddy → 用 `templates/agent-dispatch-tiering.md`
- 只有单 agent 的（Codex 未证）：退化为**提示词级自述档位**（仍然有用，只是少了硬隔离）
