# search-playbook

> **An agent-oriented search discipline** — not another search-operator cheat sheet.
> 一套**给 LLM agent 用**的检索纪律，而不是给人类看的搜索技巧速查表。

## 为什么做这个 / Why this exists

大多数检索指南是写给人看的。**LLM agent 的失效方式不一样**：过度搜索、不知何时停、不做验证、把整页原文灌进昂贵的上下文。本仓库把这套纪律打包成 agent 可直接执行的形式。

| 人类检索指南 | 本仓库（agent 向） |
|---|---|
| 记住更多算子 | **什么时候停**（硬预算 + 停手判据） |
| 找得更全 | **少而准**（L1 直取 vs L3 深挖） |
| 靠自觉 | **可核账**（首行自报档位 + 轮次） |
| 不考虑成本 | **上下文成本意识**（取页隔离到子会话） |

## 内容 / What's inside

```
SKILL.md                              ← 执行版纪律（分档 / 预算 / 证据规则 / 停手判据）
reference/manual.md                   ← 完整手册（算子现状、各库语法、模板库、来源、存疑项）
templates/agent-dispatch-tiering.md   ← 写进「编排者」的指令（控制点在这里）
templates/agent-append.example.md     ← 追加到「检索子 agent」的提示词（执行端兜底）
```

## 核心 / Core idea

```
正确率 = 选对源 × 构造对查询 × 读对证据 × 及时止损
```

最容易忽略的是 **第 1 项（选源）** 与 **第 3 项（读证据）** —— 别把力气全花在堆关键词上。

### 按难度分档（降本的关键）

| 档 | 判据 | 预算 | 谁做 |
|---|---|---|---|
| **L1 直取** | 一个事实 / 数字 / 名称，一手源可直接读出 | ≤1 轮 | **编排者自己做**（不委派 = 最省） |
| **L2 对比** | 需 ≥2 个来源，或要比较 / 汇总 | ≤3 轮 | 检索子 agent |
| **L3 深挖** | 结论性 / 争议性判断、多源交叉 | ≤5 轮 + 1 轮证伪 | 检索子 agent |

## 安装（opencode）/ Install

1. 复制到全局 skill 目录：
   ```
   ~/.config/opencode/skills/search-playbook/SKILL.md
   ~/.config/opencode/skills/search-playbook/reference/manual.md
   ```
2. **给你要用的 agent 开白名单**（关键 —— subagent 的 skill 默认是 `deny`）：
   ```json
   "librarian": { "skills": ["search-playbook"] }
   ```
3. 把 `templates/agent-dispatch-tiering.md` 并入**编排者**的指令文件（如 `AGENTS.md`）。
4. 把 `templates/agent-append.example.md` 并入**检索子 agent** 的提示词追加。
5. **重启**（配置只在启动时加载一次）。

> 非 opencode 环境：`SKILL.md` + `manual.md` 本身就是纯 markdown 纪律，可直接喂进任意 agent 的 system prompt。

## 诚实的限制 / Honest limitations

- ⚠️ **这是实战经验沉淀，未经对照实验验证。** 预算阈值（≤2/≤3/≤5 轮、6 个月时效）是**设计值不是实测值**；`manual.md` 附录 B 单列了全部存疑项。
- ⚠️ **预算是提示词级，不是硬闸门。** 实测中 L3 出现过超 1 轮（但**会自报**，所以可控）。
- ⚠️ **算子行为会变**（Google 近两年改动频繁）→ 建议每隔几个月复核附录 B。
- ⚠️ **部分结论属"实测可用但官方未文档化"**（如某些库的字段码）→ 用前自测。
- ⚠️ **CNKI / 万方 / 维普** 通常需要登录态或有头浏览器；无头 agent 会撞墙（手册已给替代路径）。

## 怎么自查它有没有用 / Verify

跑 3 题（L1/L2/L3 各一），核对：

1. **首行是否自报** `档位=L1，用了 N 轮`
2. **轮次是否守住预算**
3. L3 是否有**证伪轮**、是否列出**未覆盖项**

> 自称档位却超轮 = 提示词级不够硬 → 该换**独立轻量 agent**（换更便宜的模型 = 结构性降本，而不是求它自觉）。

## 许可证 / License

- 代码与配置（`SKILL.md`、`templates/`）：**MIT**（见 `LICENSE`）
- 文档正文（`reference/manual.md`）：**CC BY 4.0**

## 来源 / Origin

Derived from field experience in a search-heavy project (2026-09): hundreds of real retrievals across CNKI / Wanfang / VIP / PubMed / government sites / patent databases / web archives (internet archives) — then cross-checked and corrected by an independent review pass. No source-project-specific content is included.
