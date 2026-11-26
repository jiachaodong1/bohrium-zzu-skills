---
name: bohr-academic-polish
description: "Polish academic prose sentence by sentence, locking terms, numbers, units, citations, stats, negations; output revised draft, token-level diff and risk list for overreach/drift/term_swap. Trigger when polishing manuscripts, abstracts, grants or replies without altering claims, data or evidence."
---

# 学术语言润色（bohr-academic-polish）

面向科研稿件的逐句润色与审计：调用方 Agent 负责改写文本，本 Skill 的脚本负责**锁定令牌登记、词级比对、
违规拦截、风险标注与契约组装**。脚本不预制任何一篇稿件的答案，也不替代作者判断科学结论真伪。

## 1. 适用条件

- 输入：原始稿件（纯文本 / Markdown / 章节稿）+ Agent 产出的润色草稿 JSON；可选术语表。
- 产出：润色稿（句编号与原文对齐）、逐句 diff 表（机器 JSON + 人读 MD）、风险清单、契约 `result.json`。
- 适用：论文摘要 / 引言 / 方法 / 结果 / 讨论章节润色、基金申请书语句收紧、审稿回复语域调整、
  中式英语与直译腔修改、术语全篇一致性回扫。
- 不适用：全文翻译（走翻译赛道）、文献总结、图谱构建等专项任务。

## 2. 输入约定

运行时原始输入位于 `/input/case.json` 与 `/input/task.md`；中间产物放 `/app/work`，
最终产物放 `/app/output`。案例描述符字段：

| 键 | 说明 |
|---|---|
| `case_id` | 案例标识，透传到 `result.json`。 |
| `source_file` | 原始稿件（相对案例目录）；缺失或不可读时回退到内联 `source_text` / `source_sentences`。 |
| `polished_file` | Agent 产出的草稿 JSON，结构见下；缺失或不可读时回退到内联 `draft`。 |
| `draft` | 内联草稿 JSON（与 `polished_file` 同结构），作为文件缺失时的回退。 |
| `source_text` | 内联原稿文本，作为 `source_file` 缺失时的回退。 |
| `source_sentences` | 内联句清单（字符串数组），作为 `source_text` 也缺失时的回退。 |
| `glossary_file` | 可选术语表 `{terms: [{canonical, aliases}]}`。 |
| `material_file` | 可选，原始材料（PDF 等），做可读性探测。 |
| `mode` | `normal` / `field_missing` / `unreadable` / `over_budget`。 |
| `budget_remaining` | 剩余模型请求预算；≤0 触发早存部分成果。 |

材料路径解析顺序：绝对路径 → `--material-dir`（默认 case.json 所在目录）→ case.json 所在目录 → 当前工作目录 → `/input`。

草稿 JSON 结构：

```json
{
  "meta": {"audience": "journal manuscript", "register": "research-article"},
  "sentences": [
    {"sid": "S1", "text": "润色后句子", "change_type": "grammar", "reason": "为什么这样改"}
  ],
  "risks": [
    {"sid": "S1", "kind": "overreach", "note": "风险位置与说明"}
  ]
}
```

- `change_type` 枚举：`grammar` / `minimal_reorder` / `term_consistency` / `tone` /
  `tighten` / `overreach` / `drift` / `term_swap` / `unchanged`。
- `kind` 枚举：`overreach`（过度推断）/ `drift`（语义漂移）/ `term_swap`（术语替换）。

## 3. 工作流程

1. **锁定登记**：从原文登记五类 locked token——引用编号、统计量（p/n/r/OR/HR/RR/SD/SEM/CI）、
   数值+单位、否定词、术语表术语；任何一类在润色稿中被改动、丢失或无据新增即判违规。
2. **逐句对齐**：按 `sid`（缺失时按序号）对齐原句与改句，标准库 `difflib` 做词级 token 比对。
3. **字段审计**：检查草稿是否具备 `sid` / `text` / `reason`，`change_type` 与风险 `kind` 是否合法。
4. **锁定审计**：比对锁定令牌；硬违规（丢失 / 改写 / 无据新增）拒绝接受草稿，附词级 diff 证据；
   术语大小写 / 别名形态差异记为软警告，交作者确认。
5. **风险审计**：校验 Agent 标注的风险条目；并跑启发式标记——主张强化（hedge 消失 + 强断言出现）、
   队列范围漂移、术语被同义替换。
6. **组装结果**：按 `references/output_contract.md` 生成 `result.json` 与三件套产物。

## 4. 脚本调用

所有脚本使用系统 `python`，路径相对 Skill 根目录；不写死解释器绝对路径。

```bash
# 4.1 端到端编排（推荐）
python scripts/pipeline.py --case case.json --output out/

# 4.2 独立闸门：锁定 + 字段 + 风险审计（audit.json）
python scripts/audit.py --case case.json --output out/audit.json

# 4.3 契约校验（result.json 字段、状态枚举、引文逐字回溯、产物落盘）
python scripts/validate_result.py --result out/result.json --source source.md
```

- `--case` 与 `--input` 为等价别名；`--material-dir` 可选，默认取 case.json 所在目录。
- 退出码：`0` 合法 `result.json` 已写出（success / partial / unable 均含）；`2` 草稿被拒绝（锁定或字段硬错误，拒绝信号）。成败以 `result.json` 的 `status` 为准。

## 5. 输出契约摘要

`result.json` 字段与 `references/output_contract.md` 固定一致：`case_id`、`task_type`、
`status`、`status_reason`、`inputs`（files + sha256）、`primary_results`、`evidence`、
`uncertainty`、`limits`、`change_log`、`artifacts`。

- `evidence` 每条含 `claim_id`（句编号 S1…）、`locator.block`、逐字 `quote`、`kind`。
- `primary_results.draft_status`：`accepted`（通过）/ `held`（有风险待作者确认）/
  `rejected`（锁定或字段硬错误，不接受为最终稿）。
- 状态四值：`success` / `partial` / `unable` / `refused`，语义见输出契约。

## 6. 异常与降级

| 情形 | 判定信号 | result.status | 处置与人工接管位置 |
|---|---|---|---|
| **输入缺失（描述符缺失 / 不可解析）** | `case.json` 路径不存在，或内容不是合法 JSON | `unable`（退出码 0，结果文件已产出） | 不补造内容；`status_reason` 写明缺哪个文件或解析错误。人工接管：重新提供可读案例描述符后重建。 |
| 单一材料缺失 | `source_file` / `polished_file` / `glossary_file` 引用的文件不存在，且无内联回退 | `partial`（退出码 0） | 缺项写入 `uncertainty`（reason=`missing_material`）；有内联字段时继续产出部分结果。人工接管：补齐引用文件后重跑完整审计。 |
| **材料不可读或加密** | 零字节、UTF-8 解码失败、PDF 头不符、`/Encrypt` 标记 | `partial`（退出码 0） | 探测失败记入 `uncertainty`（reason=`unreadable`）；不据标签猜测内容。人工接管：提供未加密、可解码副本后重跑。 |
| **字段缺失** | 草稿缺 `sid`/`text`/`reason`，或 `change_type`/`kind` 非法、句编号与原文对不上 | `partial`（草稿 `rejected`，退出码 2） | 缺项写入 `uncertainty`；不把空字段当成事实。人工接管：按 `audit.json` 的 `field_errors` 修复 JSON 行后重跑闸门。 |
| **超预算** | `budget_remaining` ≤ 0 或接近请求 / 时长上限 | `partial`（退出码 0） | 先落盘已有部分成果（早存增量），未处理句段标 `partial`。人工接管：在新会话续跑，按 `limits` 补齐，不一次吃满预算。 |
| 要求伪造 / 篡改数据 | 诉求要求伪造数据、篡改结果或突破安全 / 合规边界 | `refused`（退出码 0，若已写出 `result.json`） | 不执行违规诉求；人工接管：交人工审核诉求合法性。 |

**退出码约定**：只要写出合法 `result.json`，退出码即为 `0`，含 `partial` / `unable` / `refused` 这类诚实降级情形；非零退出只保留给拒绝信号（草稿被锁定 / 字段硬错误拒绝，退出码 2）或根本产不出合法结果文件。**调用方必须以 `result.json` 的 `status` 判定成败，不能只看退出码。**

## 7. 限制

- 脚本只核对事实 / 术语 / 数字 / 证据边界是否被保留；**科学结论真伪与语义恰当性仍需作者或上游模型对照原文复核**。
- 扫描件 OCR、双栏错序、图表内嵌数值、中英日混排公式的规范化不在核心路径内。
- 启发式风险标记仅为建议；最终措辞由作者或编辑确认。
- 术语表缺失时，术语锁定只覆盖显式提供的条目；跨章节同义混用需作者补充术语表后回扫。

## 8. 自测与复用

- 自测样例与真实运行记录在 `selftest/`（SELF-MADE，非官方用例）；命令与结果见 `selftest/SELFTEST.md`。
- 规则词表：`references/style_rules.md`；交付自查：`references/checklist.md`；
  字段字典：`references/output_contract.md`。
- 依赖与降级表见 `README.md` 与 `environment/resources.yaml`。
