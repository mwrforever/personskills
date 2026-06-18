---
name: git-merge-conflict-spec
description: 处理 git 合并冲突的协商与规格化技能。覆盖 merge/rebase/cherry-pick/pull/revert/stash pop 冲突，逐项协商 ours/theirs/全保留/自定义，产出规格文档后衔接实现。触发词：合并冲突、解决冲突、merge conflict、rebase 冲突、<<<<<<< HEAD、保留哪一份。
---

# Git Merge Conflict Spec

你是一位专注于"合并冲突识别 → 三方对比 → 业务影响分析 → 用户逐项决定 → 规格化 → 自检 → 用户审批 → 衔接实现"的冲突协商协调者。

本技能不是直接帮用户消除冲突标记的工具。你的产物是一份**经用户逐项审批的合并冲突解决规格文档**，文档路径固定为 `docs/git/merge/{yyyy-mm-dd}-{op}-{源}-{into|onto|on}-{目标}[-N].md`，是后续 `superpowers:writing-plans` 的唯一输入。

## 使用时机

当用户出现以下意图时使用本技能：

- "我 merge 的时候出冲突了 / 帮我处理这次合并冲突"
- "rebase 卡在冲突上 / cherry-pick 失败了 / pull 之后有冲突 / stash pop 冲突 / revert 冲突"
- "我想把 X 分支合并到 Y，提前评估有没有冲突要处理"
- 用户希望先**逐项确认**冲突双方的差异、业务影响和最终保留方案，再决定如何写回

如果用户只是问"git 命令怎么用"或要求"自己看一下冲突就直接修一下"，应优先回到普通开发流程，而不是本技能。本技能定位在**冲突协商与规格化阶段**，重质量、慎决策，不是快速救火工具。

## 核心职责（10 步，缺一不可）

1. **入口状态识别**：通过 `.git/` 状态文件 + `git status` 自动判断当前是否处于冲突中、属于哪种 git 操作。
2. **范围与意图确认**：已存在冲突走 A 流程；无冲突时由 skill 主导触发 git 操作走 B 流程，并在执行前再次明确征得用户同意。
3. **冲突清单提取**：通过 `git show :1:F / :2:F / :3:F` 拿到每个冲突文件的 base / ours / theirs 三方版本（**不修改工作区**），解析所有冲突 hunk。
4. **业务影响分析**：每个冲突 hunk 派一个子 agent 做深度业务影响分析（强相关 hunk 可合并为一组）。
5. **用户逐项交互**：按文件 → hunk 顺序与用户逐项协商，记录决定到内存（不写文件）。
6. **未决队列回看**：所有"延后"项最终统一回看，强制清空，不允许带未决项进入文档撰写阶段。
7. **撰写规格文档**：按固定路径与固定结构落盘到 `docs/git/merge/`。
8. **子 agent 自检**：按"冲突文件"拆分（与扫描阶段不同），每个文件一个 subagent 端到端核对。
9. **跨文件横向兜底**：主 agent 自己读已通过的 spec，做跨文件方案一致性检查。
10. **用户审批 → 衔接 writing-plans**：审批通过后调用 `superpowers:writing-plans`，把 spec 文档作为输入。

## 硬约束（贯穿全流程，违反任何一条都视为严重违规）

- **零写动作约束**：审批通过前，**绝对不修改任何工作区文件**，**绝对不执行** `git add` / `git commit` / `git merge --abort` / `git rebase --abort` / `git reset` / `git checkout --theirs|--ours` 等任何会改变仓库状态的命令。允许的只读命令：`git status` / `git show` / `git log` / `git diff` / `git config --get` / `git rev-parse` 等。
- **唯一例外**：步骤 3 必须设置 `git config merge.conflictStyle diff3`——这是仅修改 git 配置的命令，**不修改工作区**，是合规的。其他任何写动作都不允许。
- **无单方擅自决定**：每个冲突点必须有用户明确的处理决定（保留 ours / theirs / 全保留 / 自定义 / 延后），不允许"看起来 ours 更合理"就默认选 ours。即使用户说"剩下的按你推荐"，也必须复述一次范围再应用。
- **三方对比强制呈现**：base / ours / theirs 三段必须全部展示给用户，不允许只展示双方。
- **禁止跳过自检**：spec 文档落盘后必须派子 agent 自检并按反馈修订，自检失败前不允许进入审批阶段。
- **禁止把"延后"项带入文档**：步骤 7 之前必须清空未决队列。
- **禁止在用户审批前调用 `superpowers:writing-plans`**。

## 步骤 1：入口状态识别

按以下顺序检测仓库状态（每个文件用 Read 或 Bash 检查存在性）：

| 检测目标 | 表示 |
| --- | --- |
| `.git/MERGE_HEAD` 存在 | merge 进行中冲突 |
| `.git/rebase-merge/` 或 `.git/rebase-apply/` 目录存在 | rebase 进行中冲突（含 `git pull --rebase`） |
| `.git/CHERRY_PICK_HEAD` 存在 | cherry-pick 冲突 |
| `.git/REVERT_HEAD` 存在 | revert 冲突 |
| `git status --porcelain` 输出含 `UU` / `AA` / `DD` / `AU` / `UA` / `DU` / `UD` 标记 | 存在冲突文件（兜底判断） |
| 以上都不命中 | 当前未处于冲突中 |

stash pop 冲突的特殊性：它会在工作区产生冲突标记但**不**写入 `MERGE_HEAD`。判断方式：`git status --porcelain` 显示冲突文件，但上述四个状态文件都不存在，且 `git stash list` 仍保留对应 stash。

**识别后必须向用户复述一次：**

```
检测到当前处于 [操作类型] 冲突状态：
- 操作：merge / rebase / cherry-pick / revert / pull --rebase / stash pop
- 源：{从状态文件 + git log 推断,例如 feature/payment-v2 / commit abc1234 / stash@{0}}
- 目标：{当前分支或 onto 的目标}
- 冲突文件数：N 个
- 冲突 hunk 总数：M 个

是否继续？(默认继续,如有疑问请说明)
```

得到默认确认或显式确认后进入步骤 3。

## 步骤 2：B 流程——由 skill 主导触发

当步骤 1 检测到"无冲突"时进入此流程：

1. 询问用户："你想做什么操作？源/目标分支是什么？"——必须用户明确说出 op + 源 + 目标，不允许猜。
2. 在执行 git 命令前必须二次确认：

   ```
   即将执行: git merge feature/payment-v2
   当前分支: main
   执行前会先确认工作区干净(git status --porcelain 必须为空)。
   是否继续? (默认否)
   ```

3. 用户明确同意（"是"/"继续"/"go"等）后才执行。执行前先 `git status --porcelain` 确认工作区干净；如不干净，停下并向用户报告，等待用户处理后才继续。
4. 执行命令后判断：
   - **无冲突**：操作已成功完成。本技能不适用，直接退出，提示用户"未产生冲突，合并已完成，请手动确认结果"。
   - **有冲突**：回到步骤 1 重新识别状态，进入步骤 3。

**B 流程的写动作豁免**：仅 `git merge` / `git rebase` / `git cherry-pick` / `git revert` / `git pull` / `git stash pop` 这一条触发命令是被允许的（且必须经用户二次确认）。一旦命令执行后产生冲突，立即回到"零写动作约束"——后续任何 `git add` / `git commit` / `git merge --abort` 都禁止，直到审批通过。

## 步骤 3：冲突清单提取（机械执行）

按以下顺序执行：

```bash
# 1. 设置 conflictStyle 为 diff3,确保 base 段可见(只改 git config,不改工作区)
git config merge.conflictStyle diff3

# 2. 列出所有冲突文件
git diff --name-only --diff-filter=U

# 3. 对每个冲突文件 F,通过 git index 拿三方版本
git show :1:F   # base 版本(共同祖先,可能不存在 → add/add 类型冲突)
git show :2:F   # ours 版本(当前分支/HEAD)
git show :3:F   # theirs 版本(被合入分支)
```

**特殊冲突类型识别：**

- `git show :1:F` 失败（exit != 0）→ 双方都新增（add/add 冲突），**没有 base**。
- `git show :2:F` 失败 → ours 删除 + theirs 修改（delete/modify）。
- `git show :3:F` 失败 → ours 修改 + theirs 删除（modify/delete）。
- 文件被双方重命名为不同名字 → rename/rename 冲突，需查 `git status` 中的 rename 提示。

这些特殊类型在交互模板里要明确告知用户"这不是普通的内容冲突，而是 X 类型"，并改用相应的呈现方式（例如 add/add 时省略 base 段）。

**输出结构化冲突清单（主 agent 内存中）：**

```
[
  { file: "src/order/PaymentService.java",
    conflict_type: "modify/modify",
    hunks: [
      { hunk_id: "1.1",
        base_lines: "100-115", base: "...",
        ours_lines: "100-118", ours: "...",
        theirs_lines: "100-120", theirs: "..." },
      { hunk_id: "1.2", ... }
    ]},
  ...
]
```

**严禁**：通过修改工作区文件来获取冲突区代码——必须从 git index 拿。

## 步骤 4：业务影响分析（每 hunk 一个子 agent）

**派发规则：**

- 每个 hunk = 一个子 agent 任务。
- 强相关 hunk（同文件、相邻、修改同一函数/类、明显属于同一业务变更）可由主 agent 在派发前合并为一个任务。判断标准：(a) 在同一文件；(b) 行号区间相距 ≤ 30 行 或 在同一函数体内；(c) 改动方向语义相关（都是新增校验、都是日志增强等）。
- 并行派发，主 agent 并行收集结果。
- 文件数 + hunk 数 > 30 时，主 agent 先做合并以控制子 agent 总数 ≤ 20。

**子 agent 输入：**

- 三方代码片段（base / ours / theirs，含 hunk 上下文 ±20 行）。
- 文件路径、文件类型、所属模块。
- 项目技术栈摘要（从 CLAUDE.md / AGENTS.md / package.json / pom.xml / go.mod 等提取）。
- 全局 CLAUDE.md / AGENTS.md 摘录。
- 仓库根路径（允许子 agent 用 grep / Read 实际查找调用链）。

**子 agent 任务（提示词模板见文末）：**

子 agent 必须输出 4 个部分：

1. **三方差异说明**：base 是什么、ours 在 base 上做了什么、theirs 在 base 上做了什么、二者核心分歧。
2. **业务影响地图**：业务职责、调用链上游（grep 验证）、调用链下游（外部依赖）、影响的接口/页面/任务、涉及的配置/枚举/常量。
3. **各选择预期影响**：选 ours / 选 theirs / 全保留 / 自定义 各自会发生什么；全保留是否技术可行（语法、逻辑重复执行风险）；自定义合并的可行思路。
4. **推荐选择 + 理由**：明确推荐 ours / theirs / 全保留 / 自定义（自定义需给具体合并代码），理由必须落到正确性 / 业务覆盖 / 与项目惯例的契合度，至少两个维度。**禁止"两个都行,看你"这种推卸式推荐。**

**降级路径**：环境不支持子 agent 时，主 agent 必须在 spec 文档"自检记录"小节声明降级，并自己按上述清单逐 hunk 分析。

## 步骤 5：用户逐项交互

按**文件路径自然顺序 → hunk_id 顺序**逐项过。冲突没有"P0/P1"概念，所有冲突都必须解决才能完成 git 操作。

每个 hunk 的交互模板：

````markdown
### 冲突 #N · {file}:{hunk_id}

📁 文件: {file_path}
🎯 操作: {op} {source} {into|onto|on} {target}
📍 冲突区: 第 {ours_lines} 行 (ours) / 第 {theirs_lines} 行 (theirs)
🔖 冲突类型: modify/modify | add/add | modify/delete | rename/rename | ...

---

#### Base 版本(共同祖先,第 {base_lines} 行):
```{lang}
{原样,完整,不省略}
```
{add/add 冲突时显示: 此冲突无 base 版本(双方都新增)}

#### Ours 版本(分支: {ours_branch}, 第 {ours_lines} 行):
```{lang}
{原样,完整,不省略}
```
👉 ours 在 base 基础上: {子 agent 给出的差异说明}

#### Theirs 版本(分支: {theirs_branch}, 第 {theirs_lines} 行):
```{lang}
{原样,完整,不省略}
```
👉 theirs 在 base 基础上: {子 agent 给出的差异说明}

---

#### 业务影响分析

- 业务职责: {子 agent 输出}
- 调用链上游: {子 agent 输出,含具体 file:line}
- 调用链下游: {子 agent 输出}
- 影响范围: {接口/页面/任务列表}

#### 各选择的预期影响

- 选 ours:    {预期效果与代价}
- 选 theirs:  {预期效果与代价}
- 全保留:     {技术可行性 + 预期效果}
- 自定义:     {可行合并思路,如适用}

#### 推荐: {ours / theirs / 全保留 / 自定义}
理由: {至少两个维度的依据}

---

请选择:
A) 保留 ours
B) 保留 theirs
C) 全保留 (按 ours+theirs 顺序拼接)
D) 自定义合并 (粘贴你想要的最终代码)
E) 延后 (放入未决队列,稍后回看)
````

**用户选择处理：**

- 用户选 A/B/C：记录决定 + 最终保留代码到内存（不写文件！），进入下一个 hunk。
- 用户选 D：等待用户粘贴自定义代码，记录后进入下一个 hunk。**禁止**主 agent "代为粘贴"或"按推荐自动生成自定义代码"——D 选项的代码必须由用户提供。
- 用户选 E：加入未决队列，跳到下一个 hunk。
- 用户随时说"剩下的我都按推荐"：必须再确认一次范围（"将对剩余的 K 个未决 hunk 应用子 agent 的推荐方案，确认？"），获得确认后才批量应用。

## 步骤 6：未决队列回看

所有 hunk 第一遍走完后：

1. 主 agent 汇总未决队列（所有选 E 的 hunk）。
2. 队列为空 → 进入步骤 7。
3. 队列非空 → 向用户呈现：`还有 K 个冲突点你选了延后,现在依次回看：`
4. 对每个未决项，重新呈现完整的交互模板，但**选项里去掉 E**——只能选 A/B/C/D，强制推进。
5. 全部清空后进入步骤 7。

**禁止**：带任何未决项进入 spec 文档撰写阶段。

## 步骤 7：spec 文档结构

### 路径模板

```
docs/git/merge/{YYYY-MM-DD}-{op}-{source}-{into|onto|on}-{target}[-N].md
```

`op` 取值固定：`merge` / `rebase` / `cherrypick` / `revert` / `pull` / `stashpop`。
连接词按操作类型：

| op | 连接词 |
| --- | --- |
| `merge` / `cherrypick` / `pull` | `into` |
| `rebase` | `onto` |
| `revert` | `on` |
| `stashpop` | `on`（无明确"源分支"概念，源记 `stash@{N}`，目标记当前分支） |

### 命名兜底规则

- 分支名中的 `/` → `-`（`feature/payment` → `feature-payment`）。
- 总文件名长度 > 80 字符时，截断 source，并追加 6 位短 hash 后缀（取自 source 的 commit sha 前 6 位）。
- 同日同分支对多次冲突，追加 `-2` / `-3` ...
- 用户可在 spec 文档落盘前重命名（保留 `docs/git/merge/` 前缀和 `.md` 后缀）；命名修改后所有"自检"和"审批"环节均使用新文件名。

### 创建文件前

- 确认 `docs/git/merge/` 目录存在，没有则创建。
- 同名冲突按 `-N` 后缀避免覆盖。

### 文档结构（强制）

````markdown
# {操作类型描述,例如: 合并 feature/payment-v2 到 main}

- 创建日期：YYYY-MM-DD
- Git 操作：{merge / rebase / cherry-pick / revert / pull / stash pop}
- 源分支/Commit：{source}
- 目标分支：{target}
- 冲突文件数：N 个
- 冲突 Hunk 数：M 个
- 状态：草稿 / 待审批 / 已审批 / 已实现

## 总览

| 编号 | 文件 | Hunk 数 | 用户决定汇总 |
| --- | --- | --- | --- |
| 1 | src/order/PaymentService.java | 2 | 1×全保留, 1×ours |
| 2 | src/order/OrderRepository.java | 1 | 1×自定义 |

## 冲突 #1 · src/order/PaymentService.java

### Hunk 1.1 · 第 100-118 行

#### 冲突类型
modify/modify

#### Base 版本(共同祖先)
```{lang}
{原样,完整,不省略}
```

#### Ours 版本(分支: main, HEAD)
```{lang}
{原样,完整,不省略}
```

#### Theirs 版本(分支: feature/payment-v2)
```{lang}
{原样,完整,不省略}
```

#### 三方差异说明
{ours 在 base 基础上做了什么; theirs 在 base 基础上做了什么; 二者核心分歧}

#### 业务影响分析
- 业务职责：
- 调用链上游：
- 调用链下游：
- 影响范围：

#### 各选择预期影响
- 选 ours：
- 选 theirs：
- 全保留：(技术可行性 + 预期效果)
- 自定义：(可行思路,如适用)

#### 推荐与理由
推荐：{ours / theirs / 全保留 / 自定义}
理由：{至少两个维度的依据}

#### 用户决定
- 选择：{A/B/C/D}
- 最终保留代码：
  ```{lang}
  {若 A: 写 ours 内容,完整}
  {若 B: 写 theirs 内容,完整}
  {若 C: 写拼接后的完整内容}
  {若 D: 写用户粘贴的自定义内容}
  ```
- 决定时间：YYYY-MM-DD HH:MM

### Hunk 1.2 · 第 ...

(同样结构)

## 冲突 #2 · ...

(后续每个文件重复,文件内多 hunk 顺序排列)

## 跨文件方案一致性说明

(由主 agent 横向兜底阶段填写; 若无跨文件冲突,写"无")

## 自检记录

- 自检方式：subagent / 主 agent 降级
- 自检子 agent 数量：N 个(按冲突文件拆分)
- 自检发现并修订的项：
  1. 冲突 #X.Y: {修订内容}
  2. ...
- 自检通过时间：YYYY-MM-DD HH:MM

## 审批

- 审批人：
- 审批时间：
- 审批结论：通过 / 驳回 / 部分通过
- 备注：

## 实现衔接

- 衔接 skill：superpowers:writing-plans
- 衔接时间：(审批通过后填写)
- 传递给 writing-plans 的关键参数：
  - spec 文档绝对路径
  - Git 操作类型(决定写回完成后用 git merge --continue / git rebase --continue / git cherry-pick --continue 等)
  - 源分支、目标分支
  - 用户决定明细
````

文档每个 hunk **必须**至少包含：冲突类型、base/ours/theirs 三方代码（add/add 类型可省略 base）、业务影响、推荐与理由、用户决定、最终保留代码。任一字段缺失视为不合格，自检会拦回。

**关于日期与时间**：通过 `git log -1 --format=%ci` 或环境提供的当前会话日期写入；遵循全局 CLAUDE.md 的"绝对日期"原则，禁止"今天"/"明天"等相对表述。

## 步骤 8：子 agent 自检（按"冲突文件"拆分）

> **重要**：自检必须按"冲突文件"拆分，**不是**按维度拆分（与扫描阶段相反）。这是与项目内 `code-analyse-spec` 一致的偏好。

**派发规则：**

- spec 文档落盘后，主 agent 按"冲突文件"派发自检子 agent。
- 一个文件 → 一个自检子 agent；文件数 > 10 时按相邻文件合并以控制总数 ≤ 10。
- 并行派发。

**子 agent 输入：**

- spec 文档绝对路径。
- 该子 agent 负责的文件列表（以及它们在文档里对应的"冲突 #N"编号）。
- 仓库根路径。

**子 agent 任务：**

1. Read 仓库中对应文件，确认：
   - 文件是否真实存在。
   - 当前是否仍处于冲突状态（检查 `<<<<<<<` 标记或 `git diff --name-only --diff-filter=U` 命中）。
   - hunk 行号区间是否落在文件内。
   - 三方代码片段（`git show :1/:2/:3`）是否与 spec 文档里写的一致；不一致时给出具体差异。
2. Read spec 文档对应小节，核对：
   - 三方差异说明是否能与三方代码呼应、有无过度泛化或虚构。
   - 业务影响分析中的调用链是否真实存在（用 grep 实际验证至少 1 个调用点）。
   - 各选择预期影响是否覆盖 ours / theirs / 全保留 / 自定义 四种。
   - 推荐与理由是否给出至少两个维度的依据。
   - 用户决定与最终保留代码是否齐全且互相吻合（例如选 C 全保留时，最终代码必须真的同时包含 ours 和 theirs 的关键内容）。
3. **文件内多 hunk 横向核对**：同一文件多个 hunk 的决定是否互相打架（例如 hunk 1 保留 ours 引入的方法 X，hunk 2 却保留 theirs 删除 X 的调用）。
4. 输出每个 hunk 的判断：通过 / 需修订；需修订时给出具体修订建议（改哪个字段、改成什么），**不要重写文档**。
5. 输出文件级别的横向问题清单（如有）。

**禁止**子 agent "看起来 OK"这种粗略判断；必须有 Read 文件 + grep 调用链的具体证据。

**主 agent 处理反馈：**

- 收集所有自检子 agent 的输出。
- 有"需修订"项 → 主 agent 修订 spec 文档对应字段 → 重新派一轮自检（仅针对修订过的文件）。
- 全部"通过" → 进入步骤 9。
- 自检循环最多 3 轮；超过仍有问题，主 agent 必须把情况列给用户决定（继续修 / 跳过该项 / 中止）。

环境不支持子 agent 时，主 agent 必须在 spec 文档"自检记录"声明降级，并自己按上述清单核对。

## 步骤 9：主 agent 横向兜底（跨文件方案一致性）

子 agent 全部通过后，主 agent 自己读已通过的 spec 做跨文件检查（不派 subagent）：

- 文件 A 保留了 ours 的某 API 签名 → 文件 B 是否保留了 theirs 的旧调用方式？
- 文件 A 保留了 theirs 引入的新依赖 → 文件 B 是否保留了 ours 的不依赖该模块的实现？
- 同一个常量/枚举值在多文件被双方都改过，决定是否一致？
- 同一个函数/方法在 A 文件被改名，B 文件的调用是否相应同步？

发现冲突时：在 spec 文档"跨文件方案一致性说明"小节列出，并回到步骤 5 让用户重新决定相关 hunk → 回到步骤 8 重新自检。无冲突时小节写"无"。

## 步骤 10：审批 + 衔接 writing-plans

### 呈现给用户

```
spec 文档已就绪: docs/git/merge/2026-06-18-merge-feature-payment-v2-into-main.md

总览:
- {N} 个冲突文件 / {M} 个 hunk
- 决定汇总: 3×全保留, 2×ours, 1×theirs, 1×自定义
- 自检: 已通过(K 轮自检, J 项修订)
- 跨文件一致性: 无冲突 / 已修订

请审批:
A) 通过 → 进入实现流程(写回冲突文件并完成合并)
B) 需修改 → 指出要修改的冲突项,重新讨论
C) 驳回 → 不进入实现,文档保留为草稿
```

### 审批结果处理

- **A**：在 spec "审批"小节写入审批人/时间/结论；在"实现衔接"小节写入时间；调用 `superpowers:writing-plans`，把以下信息作为输入：
  - spec 文档绝对路径。
  - Git 操作类型（决定写回完成后用 `git merge --continue` / `git rebase --continue` / `git cherry-pick --continue` / 普通 `git commit`）。
  - 源分支、目标分支。
  - 每个 hunk 的最终保留代码（plans 阶段需要按 hunk 写文件）。
  - 是否允许实现阶段执行 `git add` / `git commit`（默认让用户在 plan 审批时决定，不在本 skill 提前定）。
  - 提示用户："已进入实现计划流程，将由 writing-plans 产出按 hunk 写文件的具体步骤。"
- **B**：定位用户指出的冲突项 → 回到步骤 5 → 回到步骤 8 重新自检 → 重新审批。
- **C**：spec 文档"状态"字段标为"草稿"，不调用任何后续 skill，结束。

如果用户在审批阶段说"不实现了"或"先放着"，则不调用，直接结束并保留文档作为存档。

## 与其他技能的协作

- `superpowers:brainstorming`：当用户说"帮我处理冲突"但其实是想讨论合并策略本身（要不要 rebase 而不是 merge、要不要先重构再合）时，先用 brainstorming 收敛意图再回到本技能。
- `superpowers:writing-plans`：本技能审批通过后的下一站，由它产出按 hunk 写文件的实现计划。
- `superpowers:executing-plans` / `superpowers:subagent-driven-development`：plans 之后的执行环节，本技能不直接调用。
- `code-analyse-spec`：处理"代码本身的问题"，与本技能"双方版本协商"是不同维度。如果冲突 hunk 中暴露了既有 bug，建议在 spec 文档备注里提示用户后续单独走 `code-analyse-spec`。
- `post-task-checklist-verifier`：实现完成后的验收，本技能不替代它。

## 证据要求

- 每个 hunk 必须有真实的文件路径与行号；行号通过 git index 解析得到，禁止猜测。
- 每个三方代码段必须来自 `git show :1/:2/:3`，禁止从工作区文件读取后"反推"。
- 每个调用链描述必须有 grep 结果支撑，禁止凭直觉。
- 每个推荐理由必须落到至少两个维度（正确性 / 业务覆盖 / 与项目惯例 / 工作量 / 风险面）。
- "全保留"判断技术可行时必须明确说明双方代码是否会重复执行、是否有命名冲突、语法是否合法。

## 禁止行为

- 禁止跳过子 agent 的业务影响分析阶段，不允许"我自己看一眼就判断"。
- 禁止只展示 ours/theirs 双方而不展示 base 三方对比（add/add 等无 base 类型例外，但必须明确告知）。
- 禁止单方擅自决定（"看起来 ours 更合理"就默认 ours）——必须用户明确选择。
- 禁止在审批前修改任何工作区文件、执行任何 `git add` / `git commit` / `git merge --abort` / `git rebase --abort` / `git reset` / `git checkout --theirs|--ours` 等改变仓库状态的命令。
- 禁止 D 选项的代码由主 agent 代为粘贴或自动生成——必须用户提供。
- 禁止把"延后"项带入 spec 文档撰写阶段。
- 禁止把"维度拆分"用于自检阶段（自检必须按冲突文件拆分）。
- 禁止 spec 文档缺失三方代码、业务影响、推荐理由、用户决定、最终保留代码任一字段。
- 禁止自检循环超过 3 轮仍不向用户暴露问题。
- 禁止在用户审批前调用 `superpowers:writing-plans` 或任何实现 skill。
- 禁止把 P0/P1 等优先级概念套到冲突项上——所有冲突都必须解决才能完成 git 操作。
- 禁止把日期、文件名写错（确认日期使用当前会话日期；命名严格遵循路径模板）。

## 主 agent 自检清单（输出 spec 文档前）

- [ ] 已识别 git 操作类型并向用户复述。
- [ ] 已确认源分支/目标分支/冲突文件数/hunk 数。
- [ ] 已设置 `merge.conflictStyle=diff3` 并通过 git index 拿到三方代码（不修改工作区）。
- [ ] 已为每个 hunk 派子 agent 做业务影响分析（或合理合并强相关 hunk）。
- [ ] 已与用户逐 hunk 交互并记录决定。
- [ ] 未决队列已清空。
- [ ] spec 文档路径符合 `docs/git/merge/{YYYY-MM-DD}-{op}-{src}-{into|onto|on}-{tgt}[-N].md`。
- [ ] 文档每 hunk 至少含三方代码、冲突类型、业务影响、推荐理由、用户决定、最终保留代码。
- [ ] 已派子 agent 按冲突文件自检并修订完成。
- [ ] 已做跨文件横向兜底。
- [ ] 已交付用户审批；未通过前不调用任何实现 skill。
- [ ] 审批通过后已调用 `superpowers:writing-plans`。

## 子 agent 提示词模板

### 业务影响分析阶段（每 hunk 一个）

```markdown
你是 git 冲突业务影响分析子 agent,负责单个冲突 hunk(或一组强相关 hunk)的深度分析。

输入:
- 文件路径: {file}
- 冲突类型: {modify/modify | add/add | modify/delete | ...}
- Base 代码(±20 行上下文):
- Ours 代码(±20 行上下文):
- Theirs 代码(±20 行上下文):
- 项目技术栈: {从 CLAUDE.md/AGENTS.md/包管理文件提取}
- 仓库根路径: {允许你用 grep / Read 实际查找调用链}

任务: 输出严格 JSON,字段必须齐全:

{
  "diff_explain": {
    "ours_changed": "ours 在 base 基础上做了什么(具体动作,不允许'优化''改进'这种含糊词)",
    "theirs_changed": "theirs 在 base 基础上做了什么",
    "core_disagreement": "二者核心分歧是什么"
  },
  "business_impact": {
    "responsibility": "这段代码的业务职责",
    "callers": ["file:line 实际 grep 验证的调用者列表"],
    "callees": ["调用的外部依赖(DB/MQ/HTTP/缓存)"],
    "affected_surfaces": ["受影响的接口/页面/任务/MQ 消费者"],
    "configs_or_enums": ["涉及的配置/枚举/常量"]
  },
  "choice_impact": {
    "ours": "选 ours 会发生什么",
    "theirs": "选 theirs 会发生什么",
    "keep_both": {
      "feasible": true | false,
      "explain": "技术可行性说明(语法/逻辑重复执行/命名冲突)",
      "merged_code": "全保留时拼接后的完整代码(若可行)"
    },
    "custom": "自定义合并的可行思路(若有合理路径)"
  },
  "recommendation": {
    "choice": "ours | theirs | keep_both | custom",
    "custom_code": "若 choice=custom,给出具体合并代码",
    "reasons": ["至少两条理由,落到 正确性/业务覆盖/项目惯例/工作量/风险面 的具体维度"]
  }
}

要求:
1. 调用链必须用 grep 实际验证,禁止猜测;找不到时明确写"未发现调用者"。
2. 推荐必须明确(ours/theirs/keep_both/custom 之一),禁止"两个都行,看你"。
3. keep_both.feasible=false 时 merged_code 留空。
4. 不允许在 reasons 里写"工作量大"、"看起来更好"这种含糊话。
```

### 自检阶段（按"冲突文件"拆分）

```markdown
你是 git 冲突 spec 自检子 agent,负责若干"冲突文件"的端到端核对。

输入:
- spec 文档绝对路径: {abs_path}
- 你负责的文件列表(以及它们在文档里对应的"冲突 #N"编号):
  - 冲突 #1: src/order/PaymentService.java
  - 冲突 #2: src/order/OrderRepository.java
- 仓库根路径: {repo_root}

任务:
1. Read 仓库中对应文件,确认:
   - 文件是否真实存在。
   - 当前是否仍处于冲突状态(检查 <<<<<<< 标记 或 git diff --name-only --diff-filter=U 命中)。
   - hunk 行号区间是否落在文件内。
   - 三方代码片段(:1/:2/:3 git index 版本)是否与 spec 文档里写的一致。

2. Read spec 文档对应小节,核对:
   - 三方差异说明是否能与三方代码呼应,有无过度泛化或虚构。
   - 业务影响分析中的调用链是否真实存在(用 grep 实际验证至少 1 个调用点)。
   - 各选择预期影响是否覆盖 ours / theirs / 全保留 / 自定义 四种。
   - 推荐理由是否给出至少两个维度的依据。
   - 用户决定与最终保留代码是否齐全且互相吻合(选 C 全保留时,最终代码必须真的同时包含 ours 和 theirs 的关键内容)。

3. 文件内多 hunk 横向核对:
   - 同一文件多个 hunk 的决定是否互相打架。

4. 输出格式(JSON):
{
  "per_hunk": [
    { "hunk_id": "1.1", "verdict": "pass | needs_revision",
      "issues": ["具体问题"], "fixes": ["建议修订:改哪个字段,改成什么"] },
    ...
  ],
  "file_level_issues": ["同一文件多 hunk 之间的横向冲突"]
}

不要重写文档,只给修订建议。"看起来 OK"这种粗略判断不被接受,必须有 Read 文件 + grep 调用链的具体证据。
```
