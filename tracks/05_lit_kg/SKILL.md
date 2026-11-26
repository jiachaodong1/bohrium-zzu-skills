---
name: bohr-lit-kg
description: "Build a source-grounded literature KG from extracted entities/relations: normalize aliases, validate evidence locators, export JSON/GraphML, run traceable queries. Use for KG construction, entity-relation alignment, alias normalization, or GraphML export from papers."
---

# 05 文献知识图谱构建（lit_kg）

把一组论文中由调用方 Agent 抽取的实体与关系，归一为可复核的知识图谱：
每条边都能回到论文来源段落与逐字片段，导出 JSON / GraphML / CSV，并执行
带证据的查询。本包负责**归一、校验、建图、导出、自检、查询**；自由文本到
实体/关系的抽取由调用方 Agent 完成，脚本不硬编码任何一篇论文的图。

## 1. 触发条件

满足任一即按本 Skill 流程执行：

- 要求从一篇或多篇论文构建知识图谱 / 主张图 / 实体关系图；
- 要求做别名对齐、缩写/全称/大小写/连字符变体归一、跨篇实体合并；
- 要求导出 GraphML（Cytoscape / yEd / Gephi）或带证据的关系查询；
- 要求图谱节点与边可回溯到原文位置与片段。

## 2. 输入契约

输入为一个 `case.json`（字段详见 `references/schema.md` 与 `examples/input_example.json`）：

- `sources[]`：`source_id` / DOI / 标题 / 作者 / 许可证；
- `entities[]`：`label`、`type`（枚举见 schema）、`source_id`、`paragraph_id`、
  逐字 `quote`、可选 `aliases`、可选 `do_not_merge`；
- `relations[]`：`source`、`target`、`type`（枚举）、`confidence ∈ [0,1]`、
  `source_id`、`paragraph_id`、逐字 `quote`、可选 `condition`；
- `queries[]`：`id`、`description`，可选 `entity` / `entity_type` / `relation` /
  `direction` / `source_id`。

抽取纪律（调用方 Agent 在填 `case.json` 时遵守）：

- 实体必须是论文中真实出现的科研对象；禁止凭常识补全未出现的概念；
- 关系只写有原文依据的；共现、综述转述、推测不得当成直接证据；
- `quote` 必须逐字，不改写、不翻译、不补词；
- 通用方法/材料术语在两篇论文中同名，不等于相同物理样品、模型实例、数据集或条件。

## 3. 执行流程

```bash
# 1) 建图、归一、校验、导出
python scripts/build_kg.py --input case.json --output out/

# 2) 独立自检（枚举 / 证据 / 孤立节点 / GraphML 回读对账）
python scripts/validate_kg.py --kg out/kg.json

# 3) 查询（结果已并入 kg.json；需要单独导出时）
python scripts/query_kg.py --kg out/kg.json --output out/queries.json
```

步骤说明：

1. **装载**：读取 `case.json`；核对 `sources` 引用材料是否存在、可解码、未加密。
2. **校验实体**：类型必须在枚举内；缺定位/短引的实体保留但标 `unverified`。
3. **校验关系**：类型在枚举内、端点存在、置信度 ∈ [0,1]、证据三件套齐全；
   任一不满足则移出主图，记入 `uncertainty` 与原因码。
4. **归一合并**：大小写/连字符/空格/希腊字母归一；别名表与显式别名对齐；
   记录合并依据；类型冲突保持分立；跨篇通用术语只列待复核候选。
5. **统计与查询**：节点/边数、平均度、孤立节点清单；执行 `queries[]`，
   返回子图、边与证据；空查询如实标注，不补猜。
6. **落盘**：`kg.json`、`kg.graphml`、`nodes.csv`、`edges.csv`、`result.json`。

## 4. 实体与关系规范

- 实体类型、关系类型、字段与归一规则：见 `references/schema.md`。
- 输出字段与状态语义：见 `references/output_contract.md`。
- 合并记录含 `merged_from` 与 `basis`；冲突记录含 `kept_ids` 与原因；
  跨篇候选含 `sources` 与复核提示。

## 5. 异常与降级

四类异常均给出明确 `status`、原因码与人工接管位置；降级路径先落盘已有成果，
不静默重试、不伪造结果。

| 情形 | 判定 | status | 原因码 | 处置与人工接管位置 |
|-|-|-|-|-|
| **输入缺失** | `case.json` 不存在或不可读 | `unable` | `input_file_missing` / `invalid_record` | 写出仅含状态的 `result.json`；人工在输入阶段补齐可读的 `/input/case.json` 后重跑 |
| **字段缺失** | 实体缺 label、关系缺端点/证据/置信度，或记录非对象 | `partial`（无有效实体时 `unable`） | `missing_label` / `unknown_endpoint` / `missing_evidence` / `missing_confidence` / `invalid_record` | 坏记录移出主图并入 `uncertainty`；人工按 `handling` 补齐定位或短引后重跑；不允许把缺证据边写进主图 |
| **材料不可读或加密** | 引用文件缺失、解码失败、PDF 加密；case 内联仍有有效实体/边 | `partial` | `material_missing` / `material_encrypted` / `material_unreadable` | 图按 case.json 内联字段继续构建，材料问题入 uncertainty；人工解密或替换可读材料后复核派生节点/边；若材料不可读导致无任何有效实体，才降为 `unable`。可选 PDF 探测依赖缺失时材料标 `material_unverified` |
| **超预算** | 达到 `--max-edges` 上限或响应/时间预算紧张 | `partial` | 预算截断（`deferred_edges` 记录） | 先落盘当前主图与部分成果；延迟边登记在案；人工在预算内续跑或增量合并，不一次性吃满额度 |

补充：触发安全/合规边界（如要求伪造数据、编造图谱答案或 benchmark 分数）→
`refused`，交人工合规审查。

## 6. 输出与自查

- 产物清单与字段契约：`references/output_contract.md`。
- 交付前逐项核对：`references/checklist.md`。
- 自检脚本必须通过：`validate_kg.py` 报告 `ok: true`，GraphML 节点/边数与
  `kg.json` 一致，端点齐全。

## 7. 边界

- 不做自由文本抽取；不推断图像、补充附件、底层临床/实验数值。
- 不生成 PDF 页码；定位以案例提供的段落 ID 为准。
- 查询为一跳子图；多跳推理、图谱推理不在本包范围。
- 内置别名表仅覆盖常见生物医学缩写；领域专属别名须由输入 `aliases` 提供。
