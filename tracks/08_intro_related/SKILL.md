---
name: bohr-intro-related
description: "Write and validate source-grounded introductions and related work from a research question and evidence matrix: one claim per paragraph, citations bound to matrix entries, grounded gaps, clustered comparison dimensions. Use for introductions, background, related work, gaps, or contributions."
---

# 08 引言与相关工作写作（intro_related）

围绕一个真实研究问题，把调用方 Agent 提供的证据矩阵组织成可复核的引言 / 相关工作
草稿：每段只承载一个段旨主张，引用落点绑定矩阵条目，空白陈述由已引文献推出，
相关工作按主题簇在同一维度上比较。本包负责**计划校验、引用完整性检查、outline↔
草稿对齐、缺口清单与契约结果组装**；段旨句与正文由调用方 Agent 生成，脚本不硬
编码任何一篇论文的引言。

## 1. 触发条件

满足任一即按本 Skill 流程执行：

- 要求撰写 / 重构论文引言、研究背景、相关工作、研究空白或贡献条目；
- 要求把已有文献矩阵按问题设定 / 方法路线 / 数据与评价方式分组比较；
- 要求核对草稿中的断言是否都挂了引用、空白陈述是否有文献支撑；
- 要求对齐本文贡献条目与正文 / 图表证据，检查引用覆盖与元数据一致性。

## 2. 输入契约

输入为一个 `case.json`（字段详见 `references/outline_schema.md` 与
`examples/input_example.json`）：

- `research_question`：一句话研究问题（条件 / 对象 / 问题）；
- `evidence_matrix[]`：`claim_id` / `ref` / `reference_meta` / `main_conclusion` /
  `support_strength` / `usable_sections` / 逐字 `quote` / `locator`；
- `outline_spec[]`：`para_id` / `section` / `cluster` / `comparison_dimensions` /
  `topic_sentence` / `claim_ids` / `refs` / `boundary_sentence` / `word_budget`；
- `draft.paragraphs[]`：`para_id` + 正文 `text`，引用以 `[n]` 标注；
- `contributions[]`：本文贡献条目，须可核对到正文 / 图表证据。

填写纪律（调用方 Agent 遵守）：

- 矩阵条目必须来自真实文献；作者、年份、标题、DOI 不得编造；
- `quote` 逐字摘录，不改写、不翻译、不补词；
- 对他人工作的转述要写清适用条件与局限，不把相关性写成因果，不把单模型结论
  写成普适规律；
- 空白必须由已引工作的共同限制、未覆盖场景或冲突结论推出，不写“研究较少”式
  空转铺垫。

## 3. 执行流程

```bash
# 1) 校验矩阵与段落计划，产出 outline.json + gaps.json
python scripts/plan_outline.py --input case.json --output out/

# 2) 端到端：计划校验 → 渲染草稿 → 引用完整性 / 空白落地 / 映射 / 预算校验 → result.json
python scripts/pipeline.py --input case.json --output out/ --material-dir case/

# 3) 独立自检（outline.json + intro.md 对账）
python scripts/validate_intro.py --outline out/outline.json --intro out/intro.md \
    --output out/validation.json
```

步骤说明：

1. **装载**：读取 `case.json`；核对被引材料是否存在、可解码、未加密。
2. **计划校验**：段旨句 / 主张 / 引用齐全且互相一致；段落挂边界句；相关工作成簇
   并声明对比维度；空白段落只引用标记为 gap 可用的条目。
3. **草稿渲染**：按 outline 段号组装 `intro.md`；缺正文段落显式标记 pending，
   脚本不补写。
4. **引用完整性**：逐句检查——非白名单陈述句必须带可解析引用；空白陈述必须挂
   gap 可用引用；引用号不得越出本段 outline 集合。
5. **映射与预算**：outline 与草稿段号集合、顺序一一对应；段落字数不超预算。
6. **落盘**：`outline.json`、`intro.md`、`gaps.json`、`result.json`。

## 4. 写作与校验规范

- **立得住**：背景只写与本问题直接相关的脉络，指向具体未解决问题，不做百科式
  重要性叙述。
- **分得清**：相关工作按主题簇比较；每簇给出对比维度（问题设定 / 方法路线 /
  数据与评价方式），在同一维度上归纳各工作取舍，不逐篇摘要拼接。
- **说得实**：空白由已引文献的缺口推出，并说明本文工作如何填补；无依据的判断
  标注为作者判断，不冒充文献结论。
- **对得上**：贡献条目与正文 / 图表证据一一对应；引用元数据以矩阵为准，可逐条
  回查。
- 引用格式、白名单类别、空白标记词表：见 `references/outline_schema.md` §5。
- 输出字段、原因码与状态语义：见 `references/output_contract.md`。

## 5. 异常与降级

四类异常均给出明确 `status`、原因码与人工接管位置；降级路径先落盘已有成果，不
静默重试、不伪造结果。

| 情形 | 判定 | status | 原因码 | 处置与人工接管位置 |
|-|-|-|-|-|
| **输入缺失** | `case.json` 不存在或不可解码 | `unable` | `input_file_missing` / `material_unreadable` / `invalid_record` | 写出仅含状态的 `result.json`；人工在输入阶段补齐可读的 `/input/case.json` 后重跑 |
| **字段缺失** | 段落缺段旨句 / 主张 / 边界句；矩阵条目缺 `claim_id` / `quote`；引用号或主张在矩阵中不存在 | `partial`（无有效矩阵条目时 `unable`） | `missing_topic_sentence` / `missing_claim_ids` / `unknown_claim_id` / `missing_boundary_sentence` / `unknown_reference` / `invalid_record` | 坏段落移出成稿主体并入 `uncertainty`；人工按 `handling` 补齐字段与定位后重跑；不允许把缺证据主张写进草稿 |
| **材料不可读或加密** | 被引文件缺失、解码失败、PDF 加密或损坏 | `partial`（内联矩阵与草稿照常产出） | `material_missing` / `material_encrypted` / `material_unreadable` | 派生主张不得标记为 grounded；人工解密或替换可读材料后重跑；PDF 探测依赖缺失时材料标 `material_unverified`，不阻断文本路径 |
| **超预算** | 段落字数超过 `word_budget`，或整体响应 / 时间预算紧张 | `partial` | `over_budget`；延迟段落记入 `draft_pending` | 先落盘当前计划与已有草稿；超预算 / 待写段落登记在案；人工在预算内压减或增量续跑，不一次性吃满额度 |

补充：触发安全 / 合规边界（如要求伪造参考文献、跑分成绩或预制答案）→
`refused`，交人工合规审查。

## 6. 输出与自查

- 产物清单与字段契约：`references/output_contract.md`。
- 交付前逐项核对：`references/checklist.md`。
- 自检脚本必须通过：`validate_intro.py` 报告 `ok: true`；outline 与草稿段号一致；
  无 `uncited_claim` / `gap_without_reference` / `over_budget` 残留。

## 7. 边界

- 不做自由文本到主张 / 引用的抽取；写作质量由上游 Agent 负责，本包对坏输入做
  拒绝与标注。
- 不生成参考文献条目；矩阵之外的文献需求只进入缺口清单，不自动补条目。
- 不推断材料中未出现的事实、数字、图表结论；不强制期刊引用格式。
- 定位以案例提供的段落 ID / 页码为准；脚本不反推 PDF 页码。
