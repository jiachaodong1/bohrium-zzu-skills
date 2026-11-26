---
name: bohr-lit-notes
description: "Turn a paper into an IMRaD note with locator-linked evidence, strength audit, unit reconciliation and JSON/Markdown output; trigger when summarizing papers, extracting notes, building paper cards, or reviewing literature with citations to source pages, paragraphs, tables or figures."
---

# 文献总结与结构化笔记（bohr-lit-notes）

把一篇科研论文变成与章节对齐、字段齐备、每条结论可回到原文定位的结构化笔记。
调用方 Agent 负责阅读材料并按 Schema 填写笔记卡片；本 Skill 的脚本负责 Schema 校验、
证据强度审计、数值-单位对账、六区覆盖自检与双版渲染。脚本不预制任何一篇论文的答案。

## 1. 适用条件

- 输入：单篇论文（PDF / 文本 / 网页正文）或案例描述符 `case.json` + `task.md`。
- 产出：一份机器可读 JSON、一份 Swiss Grid Markdown、一份符合输出契约的 `result.json`。
- 适用：章节总结、论文卡片、知识点提取、来源回链、结果复核类阅读任务。
- 不适用：全文翻译（走翻译赛道）、图谱构建、Nanopublication 发布等专项任务。

## 2. 输入约定

运行时原始输入位于 `/input/case.json` 与 `/input/task.md`；中间产物放 `/app/work`，
最终产物放 `/app/output`。案例描述符字段：

| 键 | 说明 |
|---|---|
| `case_id` | 案例标识，透传到 `result.json`。 |
| `note_file` | Agent 填好的笔记卡片 JSON（相对案例目录）。 |
| `source_file` | 可选，源文本，用于逐字引文检索。 |
| `material_file` | 可选，原始材料（PDF 等），做可读性探测。 |
| `mode` | `normal` / `missing_input` / `unreadable` / `over_budget`。 |
| `budget_remaining` | 剩余模型请求预算；≤0 触发早存部分成果。 |

## 3. 工作流程

1. **识别边界**：读取 `task.md` 与材料，确认证据范围；材料不可读先走第 6 节降级。
2. **填卡片**：按 `references/notes_schema.md` 的 IMRaD 字段填写 `note JSON`
   （空白模板见 `templates/note_template.json`）。每条关键结论挂证据锚点
   （页/段/表/图）与逐字引文。
3. **校验闸门**：`validate_notes.py` 检查结构、枚举、证据强度、锚点与六区覆盖。
4. **对账闸门**：`reconcile_units.py` 检查数值-单位成对、维度一致、百分比/绝对数混用。
5. **双版渲染**：`render_notes.py` 产出机器版 `notes.json` 与 Swiss Grid `notes.md`。
6. **组装结果**：`build_result.py` 按 `references/output_contract.md` 生成 `result.json`。

也可一步走完：`scripts/pipeline.py --case <case.json> --output <out_dir>`。

## 4. 脚本调用

所有脚本使用系统 `python`，路径相对 Skill 根目录；不写死解释器绝对路径。

```bash
# 4.1 Schema / 枚举 / 强度 / 覆盖校验
python scripts/validate_notes.py --input note.json [--source source.txt] [--report validation.json]

# 4.2 数值-单位对账
python scripts/reconcile_units.py --input note.json [--report reconcile.json]

# 4.3 双版渲染（notes.json + notes.md）
python scripts/render_notes.py --input note.json --output out/

# 4.4 组装 result.json
python scripts/build_result.py --input note.json \
    --validation out/validation.json --reconcile out/reconcile.json \
    --output out/result.json [--files paper.pdf]

# 4.5 端到端编排
python scripts/pipeline.py --case case.json --output out/
```

退出码：`0` 通过；`2` 校验/对账硬错误；`3` 输入缺失或材料不可读。

## 5. 输出契约摘要

`result.json` 字段与 `references/output_contract.md` 一致：`case_id`、`task_type`、
`status`、`status_reason`、`inputs`（files + sha256）、`primary_results`、`evidence`、
`uncertainty`、`limits`、`change_log`、`artifacts`。

- `evidence` 每条含 `claim_id`、`locator`（page/block/table/figure）、逐字 `quote`、`kind`。
- 证据强度三态：`reported`（作者明写）/ `inferred_by_authors`（作者推断）/
  `my_inference`（我方推断）。`my_inference` 禁止出现在 `primary_results` 事实位，
  校验器返回 `E-STRENGTH`。
- 状态四值：`success` / `partial` / `unable` / `refused`，语义见输出契约。

## 6. 异常与降级

| 情形 | 判定信号 | result.status | 处置与人工接管位置 |
|---|---|---|---|
| 输入缺失 | `case.json` / `note_file` 不存在，或 `/input` 无可读材料 | `unable` | 不补造内容；`status_reason` 写明缺哪个文件。人工接管：重新提供论文与任务说明，由 Agent 重建卡片。 |
| 字段缺失 | 卡片缺 IMRaD 字段、证据锚点为空、枚举非法 | `partial`（硬错误时校验退出码 2） | 缺项写入 `limits`/`uncertainty`；不把空字段当成事实。人工接管：按 `limits` 指向的字段回原文补录后重跑闸门。 |
| 材料不可读或加密 | PDF 加密/损坏/零字节/无文本层且 OCR 不可用 | `unable` | 探测失败原因记入 `uncertainty`（reason=`unreadable`）。人工接管：提供未加密、可 OCR 的副本后重新阅读；不得据标签猜测内容。 |
| 超预算 | `budget_remaining` ≤ 0 或接近请求/时长上限 | `partial` | 先落盘已有部分成果（早存增量），未读区域标 `partial`。人工接管：在新会话续跑，按 `limits` 补齐缺区，不一次吃满预算。 |

补充边界：触发伪造数据、篡改结果等合规要求时，状态置 `refused`，交人工审核诉求合法性。

## 7. 限制

- 脚本只校验结构、枚举、强度归属、单位配对与覆盖；**数值语义是否正确仍需人工或上游模型对照原文复核**。
- 逐字引文检索依赖提供的源文本；无源文本时引文只做非空检查，标记为不可机器核验。
- 单位维度表覆盖常见量纲；新单位归入 `other`，不参与维度冲突判定。
- 双栏错序、扫描件 OCR、公式与图表内嵌数值的抽取不在脚本内完成，需上游材料先规范化。

## 8. 自测与复用

- 自测样例与真实运行记录在 `selftest/`（SELF-MADE，非官方用例）；命令与结果见 `selftest/SELFTEST.md`。
- 字段字典：`references/notes_schema.md`；枚举：`references/evidence_enums.md`；
  版式：`references/swiss_grid_layout.md`；自查：`references/checklist.md`。
- 依赖与降级表见 `README.md` 与 `environment/resources.yaml`。
