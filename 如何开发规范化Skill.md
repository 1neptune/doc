# 如何开发规范化 Skill

> 一份面向实战的技术规范文档：从作用定义 → 组成部分 → 各部分作用 → 完整案例 → 检查清单
>
> 适用环境：WorkBuddy / CodeBuddy Skill 体系（向下兼容 Anthropic Agent Skills 规范）

---

## 目录

- [一、作用与定义](#一作用与定义)
- [二、组成部分](#二组成部分)
- [三、各组成部分的作用](#三各组成部分的作用)
- [四、开发流程（六步法）](#四开发流程六步法)
- [五、命名与格式红线](#五命名与格式红线)
- [六、案例](#六案例)
- [七、常见错误清单](#七常见错误清单)
- [八、发布前检查清单](#八发布前检查清单)

---

## 一、作用与定义

### 1.1 什么是 Skill

**Skill（技能）是一个模块化、自包含的能力包**，它通过一段结构化文档 + 可选的可执行资源，把通用 AI 助手改造成某个特定领域的"专业助手"。

官方定义可以概括为一句话：

> Skill 是针对特定领域或任务的"入职指导手册"，它为 AI 补充模型无法完整掌握的程序性知识（procedural knowledge）。

换句话说：**模型知道"写一个 PDF 旋转脚本"大概该怎么做，但不知道你们公司的 PDF 模板规范、不知道那台服务器上装的哪个版本、不知道踩过的坑。Skill 就是把这些东西固化下来。**

### 1.2 Skill 解决的四类问题

| 能力类型 | 解决的问题 | 典型表现 |
|---|---|---|
| **可复用工作流** | 每次都要走一遍的多步流程，容易漏步骤 | 部署、发布、排障、迁移的标准动作 |
| **工具集成指引** | 特定文件格式 / API / CLI 的正确用法 | Excel 生成、PPT 美化、Git 操作规范 |
| **领域专属知识** | 公司内部规范、数据库结构、业务逻辑 | 表结构文档、等保条款、品牌规范 |
| **打包资源** | 反复重写的代码、反复查的文档、反复拷的模板 | 脚本、参考文档、模板文件 |

### 1.3 Skill vs 其他方案：什么时候该写 Skill

Skill 不是唯一选择。下表是决策依据：

| 你的需求 | 推荐方案 | 理由 |
|---|---|---|
| 一句话的一次性指令 | 直接在对话里说 | 不值得固化成文件 |
| 需要调用外部系统 / 实时数据 | **MCP Server / Connector** | Skill 只是说明书，不能提供真实数据通路 |
| 一段固定的问答知识 | Skill `references/` | 轻量、随取随用 |
| 多步 + 有固定判断分支的流程 | Skill `SKILL.md` 正文 | 需要流程编排能力 |
| 每次都要跑同样的确定性代码 | Skill `scripts/` | 脚本不占上下文、可重复执行 |
| 需要长期记忆、跨会话的状态 | `MEMORY.md` / 项目记忆 | Skill 是能力，不是记忆 |

**一句话判断法**：如果这件事你（或团队）**一个月内会重复做三次以上**，且每次的做法基本一致 —— 它应该是一个 Skill。

### 1.4 Skill 的价值收益

| 指标 | 规范化前 | 规范化后 |
|---|---|---|
| 输出稳定性 | 每次执行路径不同，靠临场发挥 | 固定流程 + 固定资源，结果可预期 |
| Token 成本 | 每次重新解释一遍规则 | 三级加载，常驻上下文仅约 100 词 |
| 正确性 | 依赖模型"记得"细节 | 细节存在 `references/`，需要时精确读取 |
| 可维护性 | 规则散落在聊天记录里 | 一个目录，改一处全团队生效 |
| 团队协作 | 口头传授，无法复制 | 目录即文档，随仓库分发 |

---

## 二、组成部分

### 2.1 标准目录结构

一个规范的 Skill 至少包含一个 `SKILL.md`，其余资源按需挂载：

```
skill-name/                     # 目录名必须与 frontmatter 的 name 完全一致
├── SKILL.md                    # 【必需】技能主体：元数据 + 指令正文
├── LICENSE.txt                 # 【可选】许可证（对外分发时建议）
├── scripts/                    # 【可选】可执行代码
│   └── xxx.py
├── references/                 # 【可选】按需加载的参考文档
│   └── xxx.md
└── assets/                     # 【可选】用于产出物的资源文件
    └── template.xxx
```

**注意：不要为了"看起来完整"而创建空目录。** 三个资源目录不是必选项，大部分 Skill 只用 `SKILL.md` + `references/` 就够了。

### 2.2 最小可用 Skill（长这样就够了）

```
linux-audit-baseline/
└── SKILL.md
```

### 2.3 完整形态 Skill

```
db-migration-checker/
├── SKILL.md
├── scripts/
│   ├── check_constraints.py
│   └── gen_report.py
├── references/
│   ├── db-schema.md
│   └── migration-rules.md
└── assets/
    └── report-template.md
```

### 2.4 安装位置与作用域

| 类型 | 路径 | 作用域 | 何时选它 |
|---|---|---|---|
| **用户级 Skill** | `~/.workbuddy/skills/<skill-name>/` | 所有项目可用 | 个人工作流、跨项目的通用能力（**默认选这个**） |
| **项目级 Skill** | `<workspace>/.workbuddy/skills/<skill-name>/` | 随仓库共享 | 团队规范、项目专属 schema、共享流程 |

> 版本差异：CodeBuddy 环境对应路径为 `~/.codebuddy/skills/` 和 `<workspace>/.codebuddy/skills/`。

**选择原则**：不确定时选用户级。只有当明确需要"随仓库分发给团队所有人"时，才用项目级。

---

## 三、各组成部分的作用

### 3.1 `SKILL.md` —— 必需，技能的身份证 + 使用说明书

`SKILL.md` 由两部分组成，两部分职责完全不同：

```
---
（YAML frontmatter：元数据）
---

（Markdown 正文：指令）
```

#### 3.1.1 YAML frontmatter 各字段的作用

| 字段 | 必需 | 作用 | 写错的后果 |
|---|---|---|---|
| `name` | ✅ | 技能唯一标识，决定调用名 | 校验直接失败；与目录名不一致会导致加载异常 |
| `description` | ✅ | **决定"什么时候会被触发"** | 描述含糊 → 技能永远不被加载，等于白写 |
| `description_en` | ○ | 英文描述，多语言环境下的辅助路由 | — |
| `version` | ○ | 版本号，便于迭代追踪 | 无法判断技能是否需要升级 |
| `author` | ○ | 负责人 / 来源团队 | 出问题找不到人 |
| `license` | ○ | 许可证标识 | 对外分发有法律风险 |
| `allowed-tools` | ○ | 限定可用工具范围（如 `Read,Write,Bash`） | 不写则继承全部；**安全敏感场景建议写** |
| `disable` | ○ | 设为 `true` 可临时下线技能而不删除 | — |
| `agent_created` | ○ | 标记为 AI 生成，允许后续被工具修改/删除 | 可能无法被正常管理 |
| `metadata` | ○ | 自定义扩展字段（数据分级、生命周期等） | — |

#### 3.1.2 `description` —— 全文档最重要的 100 个字

**它唯一的职责是回答一个问题：AI 该在什么情况下想起我。**

必须同时写清三件事：

```
做什么 + 什么时候用 + 什么情况下优先用
```

**反面写法（几乎必然失效）：**

```yaml
description: 数据库迁移检查工具
```

问题：没说触发时机，AI 面对任何请求都不会想起它；也没说与其他技能的优先级。

**正面写法：**

```yaml
description: 数据库表结构变更前的迁移合规检查（Schema Migration Review）。
当用户提出修改表结构、增删字段/索引、执行 DDL、涉及大表 DDL 锁表风险，
或询问"这个迁移脚本能不能上生产"时使用本技能。
涉及生产库变更时优先使用本技能而非直接给出 SQL 建议。
```

差异一目了然：后者覆盖了**动作词**（改表、增删字段、执行 DDL）、**场景词**（上线前评估）、**风险词**（锁表、生产库），AI 匹配的成功率高得多。

**description 写作规范：**

| 规则 | 说明 |
|---|---|
| 用第三人称 | 写 `本技能用于…`，不要写 `你应当使用我…` |
| 关键词要具体 | 写出用户真实会说的词，而不是抽象概念 |
| 列出触发文件类型 / 命令 | 如 `.sql` 文件、`kubectl`、`psql` |
| 标注优先级场景 | "XX 场景下优先使用本技能" |
| 禁止尖括号 | 校验脚本会拒绝包含 `<` `>` 的 description |
| 控制长度 | 一到三句，超过 5 行说明你在写正文而不是描述 |

#### 3.1.3 正文（Markdown Body）—— 告诉 AI 具体怎么做

正文回答三个问题：

1. **这个技能是什么？**（几句话说清目的）
2. **什么时候用？**（可以与 description 呼应，但正文的重点是**边界**：什么情况不适用）
3. **实际怎么用？**（分步骤、可执行、引用到具体资源文件）

**正文写作风格规范：**

| 规范 | ✅ 正确 | ❌ 错误 |
|---|---|---|
| 用祈使句 / 不定式 | `执行 gen_report.py 生成报告` | `你应该去运行 gen_report.py` |
| 用客观表述 | `当输入文件超过 10MB 时，分片处理` | `如果文件太大，你可能需要考虑分片` |
| 给出判断分支 | `若 A → 执行 X；若 B → 执行 Y` | `视情况灵活处理` |

#### 3.1.4 正文的四种结构范式

选错范式是新手最常见的问题。按技能性质对号入座：

| 范式 | 适用 | 结构骨架 | 参考 |
|---|---|---|---|
| **工作流型** | 有明确先后步骤的流程 | 概览 → 决策树 → 步骤1 → 步骤2 | 部署、发布、迁移 |
| **任务型** | 一堆并列的操作能力 | 快速开始 → 任务A → 任务B | PDF 合并/拆分/提取 |
| **规范型** | 标准、约定、要求 | 概览 → 准则 → 规格 → 应用方式 | 品牌规范、等保要求 |
| **能力型** | 多个相互关联的特性 | 概览 → 核心能力 → 能力1 → 能力2 | 产品管理、报表系统 |

> 实际多数 Skill 是混合型：**任务型开头 + 复杂任务内部用工作流型展开**。

### 3.2 `scripts/` —— 让 AI 不必每次重写代码

**定位**：存放需要**确定性执行**的代码。AI 不需要把代码读进上下文就能运行它，这是最大的 Token 优势。

| 该放什么 | 不该放什么 |
|---|---|
| 每次都要重写的同一段代码 | 一次性、只调用一次的逻辑 |
| 需要精确结果的计算 / 校验 | 通用的模型能力（总结、翻译、写作） |
| 需要多次复用的工具函数 | 需要人工判断才能决定分支的决策 |
| 环境相关、需要适配的样板代码 | SKILL.md 里的流程说明 |

**命名与规范：**

- 文件名用动宾结构、语义明确：`check_constraints.py`、`gen_report.py`，不要叫 `utils.py`
- 脚本必须能独立运行：自己处理参数、返回退出码、**不要依赖"AI 先读了 SKILL.md"**
- 顶部写清 docstring：这个脚本干什么、输入什么、输出什么
- 参数化优于硬编码：路径、阈值都用参数传入
- 可选：脚本执行失败时应有清晰报错，便于 AI 自我纠正

### 3.3 `references/` —— 把细节从上下文里搬出去

**定位**：存放**需要时才会用到**的详细资料。SKILL.md 永远保持精简，细节放这里按需加载。

| 典型内容 | 说明 |
|---|---|
| 数据库表结构 / Schema | 有完整字段定义，比反复 SHOW TABLE 高效 |
| API 文档 | 接口签名、错误码、限流规则 |
| 内部规范 / 制度文件 | 公司标准、合规条款 |
| 领域长篇指南 | 复杂多步的详细操作手册 |
| 排错案例库 | 历史故障的成因与处置 |

**核心原则：信息只存在于一处。**

同一个信息不能既写在 SKILL.md 又写在 references 里。判断归属的简单标准：

> **它是"必须遵循的执行流程"吗？** → 是 → 写 SKILL.md
> **它是"执行时可能要查的资料"吗？** → 是 → 写 references/

**大文件的额外处理**：如果单个 references 文件超过约 1 万词，在 SKILL.md 中附带 grep 检索模式，告诉 AI 如何精准定位而非全量读取。

### 3.4 `assets/` —— 不进上下文，只进产出物

**定位**：存放**用于生成最终结果**的文件，AI 使用它们但不需要理解其内容。

| 类型 | 示例 |
|---|---|
| 文档模板 | `report-template.md`、`notice.docx` |
| 演示模板 | `slides.pptx` 母版 |
| 代码脚手架 | `frontend-starter/` 整个项目目录 |
| 图片 / 图标 / 字体 | `logo.png`、`icon.svg`、`brand.ttf` |
| 示例数据 | `sample.csv`、`test-data.json` |

**scripts / references / assets 三者最容易混淆，用一张表钉死区别：**

| 对比项 | scripts | references | assets |
|---|---|---|---|
| AI 是否需要**读取内容** | 否（直接执行） | **是**（读进上下文） | 否（直接使用） |
| 是否进入上下文 | 否 | 是 | 否 |
| 是否产生输出 | 是 | 否 | 是 |
| 典型例子 | `rotate_pdf.py` | `api-docs.md` | `logo.png` |
| 判定口诀 | **要跑** | **要读** | **要用** |

### 3.5 渐进式披露：三级加载机制

以上所有设计，最终服务于一个目标：**用最少的上下文成本支撑最完整的能力**。

```
┌─ 第 1 级：元数据（name + description）────────────────── 常驻上下文
│   约 100 词，全量技能都加载，用于"决定要不要用"
│
├─ 第 2 级：SKILL.md 正文 ────────────────────────────── 命中后加载
│   建议 < 5000 词，包含流程、判断分支、资源引用
│
└─ 第 3 级：打包资源（scripts / references / assets）───── 按需加载
    容量无上限：scripts 可不读入直接执行，references 按需检索
```

**推论：不要把所有东西塞进 SKILL.md。** 这是新手最普遍的错误——写出一份 8000 字的 SKILL.md，既臃肿又浪费上下文。正确做法是**把 SKILL.md 当成索引和路线图，细节外包出去。**

---

## 四、开发流程（六步法）

### 第 1 步：收集具体用例

不要凭想象写 Skill。先回答：

- 用户会用什么**原话**来触发它？（写出 3～5 句真实措辞）
- 有哪几类任务需要它？
- 哪些**相似但不属于它**的请求，应该被它排除？

> 用例收集到位，后面的结构基本就确定了。用例模糊，Skill 一定模糊。

### 第 2 步：规划可复用资源

逐个用例分析："要做这件事，我每次都需要重写/重查什么？"

| 分析出的需求 | 归属目录 |
|---|---|
| 每次都要写同一段代码 | `scripts/` |
| 每次都要重查同一份资料 | `references/` |
| 每次都要拷同一份模板 | `assets/` |
| 只是流程顺序不同 | 不建目录，写在 SKILL.md 正文 |

**这一步的目的是：让 SKILL.md 尽量短。**

### 第 3 步：初始化骨架

官方提供了初始化脚本，会生成标准目录 + 带 TODO 占位符的 SKILL.md 模板：

```bash
# 初始化（--path 指向父目录，不是技能目录）
python scripts/init_skill.py <skill-name> --path <目标父目录>
```

初始化后**立即删除示例文件**（`scripts/example.py`、`references/api_reference.md`、`assets/example_asset.txt`），它们只是演示。

### 第 4 步：先写资源，再写 SKILL.md

**顺序很重要。** 先把 `scripts/` `references/` `assets/` 做实，最后写 SKILL.md —— 此时正文只需要"引用"这些资源，天然保持精简。

写正文时回答三问：

1. 这个技能的目的（几句话）
2. 什么时候用 / 什么时候**不**用
3. 具体怎么用（分步骤引用所有资源）

### 第 5 步：校验与打包

```bash
# 快速校验
python scripts/quick_validate.py <skill目录>

# 校验通过后打包为 zip
python scripts/package_skill.py <skill目录> [输出目录]
```

打包脚本会先自动校验（frontmatter 格式、命名规范、目录结构、description 完整性、资源引用有效性），**校验失败则不产出包**并报错。

### 第 6 步：迭代

Skill 好不好用，只能跑出来：

1. 用真实任务触发它
2. 观察哪里卡住、哪里 AI 走偏了、哪里重复劳动
3. 判断该改 SKILL.md 的哪一段，还是该加/改某个资源文件
4. 再测一遍

> **实践建议**：迭代通常发生在首次真实使用之后，且第一次使用后的反馈最有价值 —— 那时用户对"哪里不对"的感受最强烈。

---

## 五、命名与格式红线

以下为**硬性校验规则**，违反会直接导致加载失败：

| 项 | 规则 | 反例 |
|---|---|---|
| 目录名 | 必须与 `name` 字段完全一致 | `db-check/` vs `name: db-checker` |
| `name` 字符集 | 仅允许小写字母、数字、连字符 `^[a-z0-9-]+$` | `DB_Checker`、`db_checker`、`数据库检查` |
| `name` 长度 | 不超过 40 字符 | `a-very-very-...-long-name-here` |
| 连字符位置 | 不能以 `-` 开头/结尾，不能出现连续 `--` | `-db-check`、`db--check` |
| frontmatter | 必须以 `---` 开头，格式合法 | 缺失或 YAML 语法错误 |
| 必需字段 | `name` 和 `description` 缺一不可 | 只有 name |
| description | **不得包含尖括号 `<` `>`** | `处理 <file> 格式` |
| frontmatter 位置 | 必须是文件第一行 | 前面有空行 |

其他强烈建议（非硬校验）：

- SKILL.md 正文控制在 5000 词以内
- 所有代码块标注语言（` ```python `）
- 相对路径引用资源文件，保证跨机器可移植
- 不在 Skill 里写入密钥、Token、个人隐私数据

---

## 六、案例

### 案例 A：重量级 Skill —— `db-migration-checker`

**场景**：团队生产库变更频繁，DDL 上线缺乏统一审查，每次都要人工检查锁表风险和索引缺失。

**用例清单**：

| 用户可能会说 | 触发动作 |
|---|---|
| "帮我 review 一下这个建表 SQL" | 校验建表语句 |
| "这个加字段的脚本能直接上生产吗" | 评估大表 DDL 风险 |
| "删掉这个索引会有什么影响" | 影响面分析 |

**资源规划结论**：

- 每次都要跑一遍的检查逻辑 → `scripts/check_constraints.py`
- 每次都要重查的表结构 → `references/db-schema.md`
- 审查规则汇总 → `references/migration-rules.md`
- 每次都要填的评审报告 → `assets/report-template.md`

#### 目录结构

```
db-migration-checker/
├── SKILL.md
├── scripts/
│   ├── check_constraints.py
│   └── gen_report.py
├── references/
│   ├── db-schema.md
│   └── migration-rules.md
└── assets/
    └── report-template.md
```

#### `SKILL.md` 全文

````markdown
---
name: db-migration-checker
description: 数据库表结构变更（DDL）迁移前的合规与风险检查。当用户提交建表 / 改表 / 增删字段或索引的 SQL、准备执行数据库迁移、询问"这个迁移脚本能不能上生产"、"删索引有什么影响"、或需要对 .sql 文件做变更评审时使用本技能。涉及生产库 DDL 变更时，优先使用本技能产出评审报告，而非直接给出执行建议。
description_en: Pre-flight compliance and risk review for database DDL migrations. Use when the user submits CREATE/ALTER/DROP SQL, plans a schema migration, asks whether a migration script is production-safe, or requests review of a .sql file.
version: 1.2.0
author: dba-team
license: Internal
allowed-tools: Read, Grep, Glob, Bash, Write
---

# 数据库迁移合规检查

## 概览

在 DDL 执行前完成三项检查：**规范合规**、**生产风险**、**索引影响**，并输出统一格式的评审报告。

## 何时使用 / 何时不用

**使用**：用户提交 DDL 语句、.sql 文件，或询问迁移可行性与风险。

**不使用**：
- 仅涉及 DML（增删改数据）→ 用普通 SQL 建议即可
- 用户只是询问某张表的字段含义 → 直接读 `references/db-schema.md` 回答
- 非本项目数据库 → 本技能的表结构参考不适用

## 执行流程

### 第 1 步：确认目标环境

询问用户目标环境（开发 / 测试 / 生产）。生产环境必须执行第 2～4 步全部检查。

### 第 2 步：规范合规检查

运行：

```bash
python scripts/check_constraints.py <sql文件路径> --env <环境> --schema-ref references/db-schema.md
```

脚本检查项：
- 命名规范（表名 / 字段名小写下划线、索引前缀、字段长度上限）
- 禁止操作（生产库禁止 `DROP COLUMN`、禁止无 `WHERE` 的 `DELETE`）
- 字段类型白名单
- 注释与字符集完整性

退出码：`0` 通过；`1` 存在阻断项；`2` 存在警告项。

**存在阻断项时停止流程**，直接输出问题清单，不得继续生成报告。

### 第 3 步：生产风险评估

对照 `references/migration-rules.md` 的「生产风险矩阵」逐项判断：

| 操作 | 大表判定阈值（行数） | 风险等级 | 应对 |
|---|---|---|---|
| 加字段（可为空 / 有默认值） | > 100 万 | 中 | 需低峰期执行 |
| 加字段（非空无默认） | 任何 | 高 | 需分批回填，禁止直接执行 |
| 加索引（非并发） | > 100 万 | 高 | 必须改为 `CONCURRENTLY` |
| 删列 / 改类型 | > 100 万 | 高 | 需先删依赖，先影子表切换 |

大表行数从 `references/db-schema.md` 中读取；若为实时库，先执行：

```sql
SELECT reltuples::bigint FROM pg_class WHERE relname = '<表名>';
```

### 第 4 步：生成评审报告

以 `assets/report-template.md` 为模板，填充后用脚本渲染：

```bash
python scripts/gen_report.py --input <检查结果.json> --template assets/report-template.md --output <报告路径>
```

报告包含：变更摘要、检查结果明细、风险等级判定、回滚方案、审批人签字栏。

### 第 5 步：交付

向用户呈现报告路径，并口头说明**阻断项数量**与**最高风险等级**。不要输出未通过检查的迁移执行命令。

## 参考资料

| 文件 | 内容 | 何时读取 |
|---|---|---|
| `references/db-schema.md` | 全部表结构、索引、行数 | 需确认表大小 / 字段是否存在时 |
| `references/migration-rules.md` | 规范条目与风险矩阵 | 第 3 步风险判定时 |

大表行数与字段定义请用 grep 精准检索，不要整文件读取。

## 脚本

| 脚本 | 作用 |
|---|---|
| `scripts/check_constraints.py` | 规范合规检查 |
| `scripts/gen_report.py` | 模板渲染为最终报告 |
````

#### `scripts/check_constraints.py` 节选

```python
#!/usr/bin/env python3
"""
数据库迁移规范合规检查器。

输入：一个或多个 .sql 文件
输出：JSON 格式检查结果；同时以退出码表达结论
      0 = 全部通过 / 1 = 存在阻断项 / 2 = 仅存在警告项

用法：
    python check_constraints.py <sql文件...> --env <dev|test|prod> --schema-ref <表结构文件>
"""

import argparse
import json
import re
import sys

# 生产库禁止的操作
FORBIDDEN_IN_PROD = [
    (r"\bDROP\s+COLUMN\b", "生产库禁止删除列，请走影子表切换方案"),
    (r"\bTRUNCATE\b", "生产库禁止 TRUNCATE"),
    (r"\bALTER\s+TABLE\s+\S+\s+ADD\s+COLUMN\s+\S+\s+NOT\s+NULL(?!\s+DEFAULT)",
     "非空无默认值字段需分批回填，禁止直接 ADD"),
]

# 索引需使用 CONCURRENTLY
INDEX_RE = re.compile(r"\bCREATE\s+(UNIQUE\s+)?INDEX\b(?!\s+CONCURRENTLY)", re.IGNORECASE)


def check_file(path: str, env: str) -> dict:
    """检查单个 SQL 文件，返回违规项列表。"""
    issues = []
    with open(path, "r", encoding="utf-8") as f:
        sql = f.read()

    if env == "prod":
        for pattern, message in FORBIDDEN_IN_PROD:
            for match in re.finditer(pattern, sql, re.IGNORECASE):
                line_no = sql[: match.start()].count("\n") + 1
                issues.append({"level": "block", "line": line_no, "message": message})
        for match in INDEX_RE.finditer(sql):
            line_no = sql[: match.start()].count("\n") + 1
            issues.append({
                "level": "warn",
                "line": line_no,
                "message": "生产库建索引应使用 CREATE INDEX CONCURRENTLY",
            })

    return {"file": path, "issues": issues}


def main() -> int:
    parser = argparse.ArgumentParser(description="DDL 迁移规范合规检查")
    parser.add_argument("sql_files", nargs="+")
    parser.add_argument("--env", choices=["dev", "test", "prod"], default="dev")
    parser.add_argument("--schema-ref", help="表结构参考文件，用于校验字段存在性")
    args = parser.parse_args()

    results = [check_file(p, args.env) for p in args.sql_files]
    print(json.dumps(results, ensure_ascii=False, indent=2))

    levels = [i["level"] for r in results for i in r["issues"]]
    if "block" in levels:
        return 1
    if "warn" in levels:
        return 2
    return 0


if __name__ == "__main__":
    sys.exit(main())
```

#### `assets/report-template.md`

```markdown
# 数据库迁移评审报告

**迁移单号**：{{ticket_id}}
**目标环境**：{{env}}
**评审人**：{{reviewer}}
**评审日期**：{{date}}

## 一、变更摘要
{{summary}}

## 二、规范合规检查
| 检查项 | 结果 | 说明 |
|---|---|---|
{{compliance_rows}}

## 三、风险评估
**最高风险等级**：{{risk_level}}

{{risk_detail}}

## 四、回滚方案
{{rollback_plan}}

## 五、审批
- [ ] DBA 复核
- [ ] 业务方确认
- [ ] 低峰期窗口确认
```

### 案例 B：轻量级 Skill —— `meeting-minutes-formatter`

**场景**：团队每周会议手写记录格式混乱，需要统一整理成规范纪要。这个 Skill **没有 scripts，也没有 assets**，只有一份 SKILL.md。

```
meeting-minutes-formatter/
├── SKILL.md
└── references/
    └── minute-templates.md
```

`SKILL.md` 全文：

````markdown
---
name: meeting-minutes-formatter
description: 将零散的会议手写记录、口述要点整理为结构化会议纪要。当用户提供会议记录、要求"整理成纪要"、"出个会议纪要"、或需要把聊天记录转成待办清单时使用。输出采用固定五段式结构，并自动抽取行动项与责任人。
---

# 会议纪要整理

## 概览

把任意格式的会议原始记录，转换为统一五段式纪要，并确保每个行动项都有责任人与截止日期。

## 输出结构（五段式，顺序固定）

1. **会议主题与时间地点**
2. **参会人员**
3. **议题与讨论要点**（按议题归并，保留原始表述中的关键结论）
4. **行动项**（表格：事项 / 责任人 / 截止日期 / 状态）
5. **待确认事项**（讨论中未达成一致的部分）

## 执行流程

1. 通读原始记录，识别所有议题。
2. 按议题归并讨论内容，**不新增记录中不存在的信息**。
3. 抽取行动项，按 `references/minute-templates.md` 中的表格格式输出。
4. 记录中缺失的责任人或日期，填入「待定」，**不要猜测或编造**。
5. 未达成一致的议题放入第 5 段，不要强行写进结论。

## 输出要求

- 直接输出 Markdown，不加解释性前后缀
- 行动项表格必须完整四列，缺失值填「待定」
- 若原始记录缺少会议时间或参会人，第 1、2 段填「未提供」并告知用户

## 参考资料

`references/minute-templates.md` —— 各类会议（周会 / 评审会 / 需求会）的纪要模板差异。
````

**这个案例说明什么**：

- 不需要脚本就别建脚本 —— 97% 的 Skill 属于这一类
- 规范的价值主要在**正文**（固定的五段式结构 + 明确的反编造规则）
- 模板差异这种"执行时可能要查"的细节，放 `references/`

### 案例 C：三种 description 写法对比

| 写法 | 示例 | 评价 |
|---|---|---|
| ❌ **抽象名词堆砌** | `description: 部署助手` | 只有一个模糊的词，AI 永远匹配不上 |
| ⚠️ **有动词但缺场景** | `description: 帮助用户完成 Kubernetes 集群部署和扩缩容` | 说清了做什么，但没说"什么请求会触发它"，仍可能漏触发 |
| ✅ **动作 + 场景 + 文件 + 优先级** | `description: Kubernetes 集群部署、扩缩容与版本升级。当用户要求部署/更新集群、调整副本数、升级 k8s 版本，或需要解读 deployment.yaml、排查 Pod 调度失败时使用。生产环境变更前优先使用本技能生成变更方案。` | 覆盖动作词、场景词、文件名、优先级，触发稳定 |

### 案例 D：正文的结构范式选择对照

| 你的技能是… | 用哪个范式 | 正文骨架 |
|---|---|---|
| 数据库迁移检查 | **工作流型** | 概览 → 使用边界 → 步骤1..5 → 参考资料 → 脚本 |
| 会议纪要整理 | **规范型** | 概览 → 输出结构 → 执行流程 → 输出要求 |
| PPT 美化 | **任务型** | 快速开始 → 排版美化 → 图表优化 → 母版替换 |
| 产品需求文档生成 | **能力型** | 概览 → 核心能力 → 需求分析 → 方案设计 → 验收标准 |
| 安全合规审计 | **工作流型 + 规范型** | 概览 → 检查项清单 → 执行步骤 → 判定标准 |

---

## 七、常见错误清单

| # | 错误 | 后果 | 修正 |
|---|---|---|---|
| 1 | description 只写做了什么，不写何时用 | 技能永不被触发 | 补齐动作词、场景词、文件类型、优先级 |
| 2 | SKILL.md 写成 8000 字长文 | 每次触发烧掉大量上下文 | 细节外移到 `references/` |
| 3 | 同一信息在两处重复 | 改了一处另一处过时，AI 读到矛盾版本 | 信息单一归属 |
| 4 | 目录名与 `name` 不一致 | 加载异常 / 校验失败 | 严格保持一致 |
| 5 | 用第二人称写正文（"你应该…"） | AI 执行不稳定 | 改用祈使句 |
| 6 | 建了空的 `scripts/` `assets/` 目录 | 无意义结构噪音 | 删掉不用的目录 |
| 7 | 没删初始化脚本生成的示例文件 | AI 可能误用示例逻辑 | 第 3 步后立即清理 |
| 8 | 脚本依赖"AI 先读了 SKILL.md"才能跑 | 在别的场景下直接崩 | 脚本自包含 + 参数化 |
| 9 | 把大表结构全塞进 SKILL.md | 同 #2 | 放 `references/` 并附 grep 检索提示 |
| 10 | Skill 里写死密钥 / 内网地址 | 安全风险、跨环境失效 | 改为参数或环境变量 |
| 11 | `allowed-tools` 留空但涉及敏感操作 | AI 可执行超出预期的工具 | 安全敏感技能显式限定工具范围 |
| 12 | 写完就发布，不做真实任务验证 | 纸面完美、实际不好用 | 第 6 步迭代，跑通真实用例 |

---

## 八、发布前检查清单

### 元数据

- [ ] 目录名与 `name` 完全一致
- [ ] `name` 为小写 + 连字符，无首尾连字符、无连续连字符、≤ 40 字符
- [ ] `description` 覆盖「做什么 + 何时用 + 优先级场景」，含具体触发词
- [ ] `description` 中不含 `<` `>`
- [ ] frontmatter 在文件第一行，`name` / `description` 均存在
- [ ] 填了 `version` 和 `author`

### 内容

- [ ] 正文使用祈使句 / 客观表述，无第二人称
- [ ] 正文 < 5000 词
- [ ] 每条信息只有一处归属（SKILL.md 或 references，不重复）
- [ ] 写明了「何时**不**使用本技能」的边界
- [ ] 正文引用了所有 `scripts/` `references/` `assets/` 资源，路径为相对路径
- [ ] references 中超过 1 万词的文件附了 grep 检索模式

### 结构

- [ ] 没有多余的空目录
- [ ] 初始化脚本生成的示例文件已全部删除
- [ ] 脚本命名语义明确（无 `utils.py` 这类模糊名）
- [ ] 脚本可独立运行、有 docstring、有退出码

### 安全与质量

- [ ] `allowed-tools` 按需限定
- [ ] 无密钥 / Token / 隐私数据硬编码
- [ ] 已通过 `quick_validate.py`
- [ ] 已用至少 2 个真实用例端到端跑通

---

## 附：一句话总结

> **规范化的 Skill = 精准的 description + 精简的 SKILL.md + 该外置的资源和该内置的流程。**
>
> frontmatter 决定它会不会被想起，正文决定它会不会做对，`scripts/references/assets` 决定它做得多省、多稳。
