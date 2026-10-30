# 交接包结构

使用此参考文档准备可直接复制的交接内容和可持久保存的任务包。保持任务包自包含，并在每个阶段结束后更新状态。

## 本地任务批次

以运行代理的当前会话目录作为任务根目录。目标工作区和修改范围默认是该目录及其子目录，并受实际权限、项目规则和用户约束限制。相对于此根目录解析工作区及允许/排除路径；`allowed_paths` 中的目录条目包含其子目录。`artifacts` 和 `history` 中的路径相对于批次目录解析。

仅为新任务创建新批次：`pir/YYYY-MM-DD-NNN-task-slug/`，使用用户的本地日期和该日期下一个未使用的序号。不得覆盖现有批次。优先选择明确指定的批次，其次是已核验的对话关联批次，最后是唯一匹配的未完成批次。存在多个匹配时先澄清；不得按最新日期推断任务身份。跨代理和日期接续同一任务时，保持使用同一批次。

对于源于 ACP 的批次，匹配条件除任务身份/目标外，还要求 ACP 来源相对路径和已记录摘要相同。绝不将 PIR 自行规划的批次复用于不同的 ACP 来源，也不得仅因某个 ACP 来源最新就选择它。

```text
<current-task-root>/pir/YYYY-MM-DD-NNN-task-slug/
├── state.yaml
├── input/
│   └── acp-plan.md                # 仅用于 ACP 接入；不可变快照
├── plan.md
├── implementation-prompt.md
├── implementation-report.md       # 由实施者创建
├── review-report.md               # 由评审者创建
└── evidence/                      # 可选的持久验证证据
```

规划阶段创建 `state.yaml`、`plan.md` 和 `implementation-prompt.md`。ACP 接入还会创建不可变的 `input/acp-plan.md` 快照，并记录其来源元数据。首次评审读取三个 Markdown 输入（方案、实施提示词和实施报告），以及实际源文件/差异和验证证据；评审报告是输出，不是前置条件。相应阶段生成报告前，后续报告路径保持为 null。修复轮次还需包含修复提示词和此前评审记录。

交给另一个代理或另一台机器时，将 `workspace.root: "."` 映射到该代理的当前任务目录，并在那里定位相同的相对 `bundle`。写入前核验任务 ID、相关源文件及预期版本/修改。发送方的绝对路径不是工作区绑定。材料缺失或无关时，必须解决问题，不得创建重复批次或重置源文件。

## 多轮产物

一个任务可以有任意多轮修复。保留原始 `plan.md` 和 `implementation-prompt.md`，分别存储后续各轮产物：

```text
iterations/
└── 002/
    ├── repair-prompt.md
    ├── implementation-report.md
    └── review-report.md
```

遇到 `CHANGES_REQUESTED` 时，先保存已完成轮次的评审报告，并在 `history` 中保留其产物路径。将 `iteration` 增加一次，创建下一轮的 `iterations/<nnn>/repair-prompt.md`，并让 `artifacts.repair_prompt` 指向该文件。保留原始方案和实施提示词路径。将待生成的实施/评审报告路径设为 null；在新一轮执行时填入。记录修复所需的预期当前版本和已有源文件修改，同时保留原始基线。切换阶段、代理或日期不会增加迭代编号。

修复提示词必须包含完整上下文和所有未通过的验收标准。所有轮次路径都相对于原始批次，不得另以第二个 `pir/` 目录为基准。

## 状态

```yaml
task_id: "2026-10-25-001-order-cancel"
bundle: "pir/2026-10-25-001-order-cancel"
phase: READY_FOR_IMPLEMENTATION
status: PLANNED
iteration: 1
planning_source: PIR
acp_source: null
objective: "用户可见的任务结果"
workspace:
  root: "."
  base_revision: "提交或版本"
  baseline_dirty_files: []
  current_revision: "提交或版本"
  current_dirty_files: []
allowed_paths:
  - "."
excluded_paths: []
constraints:
  - "约束或排除项"
acceptance_criteria:
  - id: AC-1
    criterion: "可观察的条件"
    verification: "如何检查该条件"
artifacts:
  plan: plan.md
  acp_plan: null
  implementation_prompt: implementation-prompt.md
  repair_prompt: null
  implementation_report: null
  review_report: null
history: []
next_action: "将完整任务包交给代理 B"
open_questions: []
blocker: null
```

此示例是规划阶段成功完成后的状态。用户范围或项目规则有要求时，收窄 `allowed_paths` 并填写排除项。任何阶段都不能扩大执行代理的实际权限。如果本地任务包不可写，报告文档写入限制，不得选择外部输出目录。

对于 ACP 接入，使用相同的状态结构，并填写以下字段：

```yaml
planning_source: ACP
acp_source:
  original_path: "acp/2026-10-28-001-order-cancel/merges/merged-final.md"
  copied_path: "input/acp-plan.md"
  source_phase: COMPLETE
  source_status: COMPLETE
  source_digest: "sha256:..."
  source_task_id: "2026-10-28-001-order-cancel"
  decision_owner_confirmed: true
  verification_limits:
    - "ACP 方案未包含仓库版本"
artifacts:
  acp_plan: input/acp-plan.md
```

复制的 ACP 文件是规划输入的证据，不能证明其中的路径、假设或实施声明仍与当前仓库一致。规划期间重新检查，并将任何不匹配记录到 `plan.md` 或 `blocker` 中。

将 `workspace.base_revision` 和 `workspace.baseline_dirty_files` 保持为原始基线。在交接时刷新 `current_revision` 和 `current_dirty_files`。对于非 Git 工作，版本使用 null，并在方案和提示词中描述等效的源文件基线。将本批次生成的文档与源文件修改分别跟踪，避免将任务包本身误判为意外代码偏移。

每个 `history` 条目记录一个已完成的迭代编号，以及该轮实施/修复提示词、实施报告和评审报告的路径。第一轮报告位于批次根目录；后续报告位于 `iterations/<nnn>/` 下。产物路径标识已生成的文件；null 表示尚未生成。

`phase` 必须是 `NEW`、`PLANNING`、`READY_FOR_IMPLEMENTATION`、`IMPLEMENTING`、`READY_FOR_REVIEW`、`REVIEWING` 或 `ITERATING` 之一。

`status` 必须是 `PLANNED`、`IMPLEMENTING`、`REVIEWING`、`CHANGES_REQUESTED`、`APPROVED` 或 `BLOCKED` 之一。

验证规划来源时，使用以下组合：

| 条件 | phase | status | 提示词 |
|---|---|---|---|
| PIR 自行制定的有效方案，或已核验的规范 ACP 方案 | `READY_FOR_IMPLEMENTATION` | `PLANNED` | `Prompt status: READY` |
| ACP 缺失/无效、身份或基线不匹配，或自行规划缺少关键输入 | `PLANNING` | `BLOCKED` | `Prompt status: NOT_READY` |
| ACP 阻塞项后续已解决 | 复用相同批次和迭代 | 仅在核验后重新生成 `PLANNED` | 仅用新的不可变提示词替换 |

## ACP 接入记录

当 `planning_source: ACP` 时，在 `plan.md` 的常规工作区方案之前添加以下部分：

```markdown
## ACP 规划输入
- 原始 ACP 路径：
- 复制的输入路径：input/acp-plan.md
- ACP 任务 ID、来源 phase 和 status：
- ACP 来源摘要：
- ACP 决策负责人及接受确认：
- ACP 目标：
- 本 PIR 批次采用的 ACP 最终决策：
- 沿用的 ACP 验收标准：
- ACP 假设与核验限制：
- 支持或反驳 ACP 方案的工作区事实：
- 已解决的路径/范围差异：
- 剩余阻塞项：
```

当 `planning_source: PIR` 时，记录 `未提供 ACP 方案；根据用户请求和工作区检查制定方案。`，不创建占位 ACP 文件。

## 规划者输出

将方案和实施提示词作为独立文件保存到所选批次中。所有关键输入都根据当前工作区填写；关键输入缺失时，生成 NOT_READY 提示词和 BLOCKED 状态，不得编造上下文。

```markdown
# 方案

## 目标
<!-- 说明请求的结果以及受益对象。 -->

## 已核验上下文
<!-- 从工作区或用户请求中确认的事实。 -->

## 工作区与基线
- 任务根目录：当前会话目录（`.`）
- 任务 ID 和相对批次路径：
- 所在仓库和原始基线版本（如可获取）：
- 初始未提交文件及归属：
- 允许路径：
- 排除路径：
- 生成的 PIR 文档（与源文件修改分开）：

## 假设与待解决问题
<!-- 标明假设。如果缺失的事实会实质改变工作，在实施前停止。 -->

## 范围
### 范围内
-

### 范围外
-

## 验收标准
- [ ] AC-1: <可观察的条件>；通过<命令、检查或证据>验证

## 实施方案
1.

## 风险与依赖
-

## 验证计划
-
```

```markdown
# 实施提示词
<!-- 保存为 implementation-prompt.md。内容必须自包含。 -->

Prompt status: READY
<!-- 缺少必要输入或核验门槛未满足时，将 READY 替换为 NOT_READY。 -->

你是任务 `<task_id>` 第 `<iteration>` 轮的实施者。

### 目标

### 已核验背景与当前状态

### 规划来源
- 来源：PIR 自行规划或 ACP 接入
- ACP 输入（适用时）：`input/acp-plan.md`
- ACP 来源 phase/status、摘要、决策负责人确认及核验限制：

### 工作区与基线
- 根目录：你当前会话的任务目录（`.`）
- 批次：pir/<batch>/（相对于该根目录）
- 原始基线版本或等效源文件基线：
- 预期当前版本和源文件修改：
- 初始未提交文件及归属：

在本地解析批次，核验任务身份、源文件和预期状态。阅读适用的项目规则。如果提示词为 NOT_READY 或任务材料不匹配，返回 BLOCKED，不修改源文件。保留已有工作；不得将工作区重置到原始基线。

### 允许和排除路径
- 允许：
- 排除：

### 约束与假设

### 所需修改
1.

### 验收标准
- AC-1:

### 验证
- 运行：
- 明确报告未运行的检查。

### 报告要求
将实施报告保存到所选批次当前轮次的报告路径，并更新 state.yaml。包含状态、修改文件及归属、当前版本、验证命令及结果和证据、偏离项、未运行的检查、已知问题，以及评审建议。PIR 证据保存在批次内。如果工作区或必要输入不匹配，返回 BLOCKED 并停止。
```

## 实施者报告

```markdown
# 实施报告

## 状态
COMPLETED | PARTIAL | BLOCKED

## 摘要

## 工作区与基线
- 任务 ID 和相对批次路径：
- 本地任务根目录：
- 提示词基线版本：
- 当前版本：
- 已有未提交文件：
- 本任务负责的修改文件：
- 生成的 PIR 文档：

## 修改文件
- `<path>` - <修改内容>

## 关键决策
-

## 验证
| 检查项 | 命令或检查 | 结果 | 证据或退出码 |
|---|---|---|---|
| | | PASS / FAIL / SKIPPED / ENV_BLOCKED | |

## 相对于提示词的偏离
- 无，或逐项描述偏离及其原因。

## 未解决问题与风险
-

## 建议评审重点
-
```

## 评审者输出

```markdown
# 评审报告

## 结论
APPROVED | CHANGES_REQUESTED | BLOCKED

## 已检查的任务包与证据
- 任务 ID、相对批次路径和迭代编号：
- 目标和方案：
- 实施提示词：
- 当前修复提示词和此前评审（适用时）：
- 实施报告：
- 实际差异或文件：
- 基线和当前版本：
- 验证输出：

## 验收标准
| ID | 标准 | 结果 | 证据 |
|---|---|---|---|
| AC-1 | | PASS / FAIL / UNVERIFIED | |

## 发现的问题
### [P1] <简短标题>
- 位置：
- 原因：
- 所需修复：

## 核验限制
-

## 下一步行动

## 修复提示词
<!-- 遇到 CHANGES_REQUESTED 时，还要将这份自包含提示词保存到 iterations/<next_iteration>/repair-prompt.md。 -->

Prompt status: READY
<!-- 无法安全执行本次修复时，将 READY 替换为 NOT_READY。 -->

你是任务 `<task_id>` 第 `<next_iteration>` 轮的实施者。

### 目标

### 已核验背景与当前状态

### 规划来源
- 来源：PIR 自行规划或 ACP 接入
- ACP 输入（适用时）：`../../input/acp-plan.md`
- ACP 来源 phase/status、摘要、决策负责人确认及核验限制：

### 工作区与基线
- 根目录：你当前会话的任务目录（`.`）
- 任务 ID 和相对批次路径（复用现有批次）：
- 原始基线版本或等效源文件基线：
- 预期当前版本和已有源文件修改：
- 初始未提交文件及归属：

在本地核验当前任务。保留此前的实施修改；原始基线是历史证据，不是重置目标。修复报告和评审输出使用本批次的 iterations/<next_iteration>/ 目录。

### 允许和排除路径
- 允许：
- 排除：

### 约束与验收标准
- 约束：
- 未通过的标准：
- 所需修改：

### 验证
- 重新运行：
- 明确报告未运行或因环境受阻的检查。

### 报告要求
在本轮保存新的实施报告，并更新现有 state.yaml。包含状态、本任务负责的修改文件、当前版本、修复内容、验证证据、剩余问题和偏离项。如果预期当前源文件状态或任务身份不匹配，返回 BLOCKED。
```
