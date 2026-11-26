---
name: bohr-09-methods-writer
description: "Write or audit an IMRaD Methods section from notes: six-field checklist (unit, n, control, randomization/blinding, statistics, availability), missing/conflicting placeholders, and validation blocking fabricated values. Trigger: methods writing, reproducibility checklists, repeatability review."
---

# 写作09｜方法与实验章节写作

## 1. 用途与适用条件

依据所供实验记录（文本/Markdown/TXT，可选 PDF）起草或复核论文 Methods 章节，覆盖六类要素：实验单位、样本量与重复 n、对照与分组、随机化与盲法、统计检验与多重比较、数据与代码可得性。

适用：
- 把零散实验记录整理成有序、可执行的方法段落；
- 复核已有方法稿是否齐全、是否存在无来源数值；
- 产出可追溯到材料原句的草稿与缺口清单。

不适用：结果章节写作、引言/相关工作、参考文献管理；这些走对应赛道 Skill。

## 2. 输入契约

输入为 `case.json`（评测环境位于 `/input/case.json`）：

```json
{
  "case_id": "case-001",
  "task_type": "methods_writer",
  "constraints": {"audience": "…", "format": "markdown", "language": "zh"},
  "materials": [{"id": "M1", "file": "notes.md", "kind": "text"}]
}
```

材料路径相对 `case.json` 所在目录；支持 `text` / `markdown` / `txt`，`pdf` 走可选 PyMuPDF。

## 3. 执行流程

1. **读取输入**：加载 `case.json`，校验 `case_id` / `task_type` / `materials` 三个必需键；材料逐个读取并记录可读性状态。
2. **六项清单核对**（`scripts/checklist.py`）：逐字段在材料中检索依据，判定 `reported` / `missing` / `conflicting`，证据逐字摘录并定位到文件与行号。
3. **生成草稿**（`scripts/draft_methods.py`）：按 IMRaD 结构成稿——材料与受试者 → 实验设计与分组 → 实验流程 → 测量与读出 → 统计分析 → 数据与代码可得性。有据项引用原句与来源；缺口项写 `[[MISSING: <field>]]`；矛盾项写 `[[CONFLICT: <field>]]`。
4. **校验草稿**（`scripts/validate_draft.py`）：核对每个占位符已登记且类型与报告一致；拦截未在材料证据中出现的具体数值；标记无来源套话。
5. **落盘结果**（`scripts/run_methods.py`）：写出 `methods.md`、`checklist_report.json`、`validation_report.json`、`result.json`，字段符合 `references/output_contract.md`。

命令（系统 `python`，路径相对 Skill 根目录）；`--case` 与 `--input` 是同一参数的等价别名，指向同一个 case.json：

```bash
python scripts/run_methods.py --case /input/case.json --output /app/output
python scripts/run_methods.py --input /input/case.json --output /app/output
```

分步调用：

```bash
python scripts/checklist.py --case case.json --output checklist_report.json
python scripts/draft_methods.py --report checklist_report.json --case case.json --output methods.md
python scripts/validate_draft.py --draft methods.md --report checklist_report.json --output validation_report.json
```

## 4. 产物

| 文件 | 内容 |
|-|-|
| `methods.md` | 可编辑草稿，含占位符与来源定位 |
| `checklist_report.json` | 六字段状态、证据、冲突明细 |
| `validation_report.json` | 占位符登记、未登记数值、套话警告 |
| `result.json` | 契约规定的全部字段（见 `references/output_contract.md`） |

## 5. 状态判定规则

- `reported`：材料中有可定位的原句依据；若原句明确说“未随机/未设盲”，记为已陈述的缺失（stated absence），不改写为已实施。
- `missing`：材料未提及；草稿必须留 `[[MISSING: <field>]]`，禁止用“standard protocol / 常规方法 / 若干次重复”糊过去。
- `conflicting`：同一字段出现两个及以上互不相同的具体取值（当前对样本量 n 做数值冲突检测）；草稿留 `[[CONFLICT: <field>]]`，不得择一写入。

## 6. 异常与降级

| 情形 | 判定 | status | 人工接管位置 |
|-|-|-|-|
| **输入缺失**：`case.json` 不存在、为空、JSON 非法或缺必需键 | 不进入核对流程，仍落盘一份诚实 result.json | `unable` | 检查 `/input/case.json` 是否就位、结构是否合法后重跑；`status_reason` 写明具体校验错误。 |
| **字段缺失**：某六字段在材料中未提及 | 草稿留 `[[MISSING: <field>]]`，报告登记进 `uncertainty`（reason=`not_reported`） | `partial` | 作者按草稿占位提示补录实验单位、n、对照、随机/盲法、统计或可得性信息；补录后重跑校验。 |
| **材料不可读或加密**：文件缺失、编码错误、PDF 加密需密码、无文本层、解析异常 | 该材料记 `unreadable` 并写明原因；只要 case.json 可解析，仍出 checklist 与部分草稿，六项按可判定部分给状态，其余记 missing | `partial`（仅 case.json 本身不可解析才 `unable`） | 人工提供可读文本/解密 PDF 后重跑；不得据领域标签补造内容。 |
| **安全/合规边界**：诉求要求伪造实验参数、编造缺失项或补造无来源数据 | 拒绝补造，不生成杜撰内容；`status_reason` 写明诉求不合法，缺口仍以占位符保留 | `refused` | 交人工审核；不得按诉求补写参数、数值或结论。 |
| **超预算**：模型请求/时长/token 接近上限 | 增量落盘优先：先写 `checklist_report.json` 与带占位符的 `methods.md`，再补校验与 `result.json`；未完成项写入 `limits` 与 `uncertainty` | `partial` | 下一轮从已落盘产物继续，不重跑已完成步骤；`change_log` 记录断点。 |

**exit 约定**：编排器只要写出合法 `result.json` 即 `exit=0`，包括 `partial`、`unable` 与 `refused`；非零退出只保留给"根本产不出 result.json"（case.json 缺失不可读、输出目录不可写）。

注意区分两类"拒绝"，不要混为一谈：
- **`refused`（诉求不合规）**：诉求本身要求伪造/编造，编排器拒绝执行并落盘诚实 result.json，**退出码仍为 0**。
- **校验器拦截信号（forged 场景）**：`validate_draft.py` 发现草稿含未登记数值或缺占位符时保持非零退出；这是"拦截能力生效"的证明，针对的是草稿内容，不是诉求合规性。

降级原则：任何未执行的步骤记 `not_run`，不补写工具调用与日志；任何不可得的信息不写成事实。

## 7. 硬规则

- 不编软件版本、试剂货号、accession、随机种子、样本量；缺则占位。
- 不把生物学重复与技术重复混成一个 n；实验单位不是独立样本时，n 必须改正。
- 不写 `standard protocol` / `as described previously` / “常规方法”而不给出处。
- 不把“统计显著”写成“机制成立”。
- 草稿中出现的每个具体数值都必须能在材料证据中找到；校验脚本拦截未登记数值。
- 证据 `quote` 逐字来自材料，不改写、不补词。

## 8. 自测与复用

- 自测样例与真实运行记录见赛道目录 `selftest/`；重跑命令见 `selftest/SELFTEST.md`。
- 复用资源：`references/six_field_checklist.md`（核对清单与禁止事项）、`references/output_contract.md`（结果契约）、`examples/`（最小输入输出示例，已标注 EXAMPLE ONLY）。
- 已知边界：字段判定基于中英关键词启发式，不理解实验语义；PDF 解析依赖可选 PyMuPDF；仅支持中英文材料。
