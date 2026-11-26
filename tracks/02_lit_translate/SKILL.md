---
name: bohr-lit-translate
description: Translate papers into block-aligned Chinese-English bilingual text: segment with anchors, lock a term glossary, keep gene/software names verbatim, audit numbers, citations, figures, term consistency and English residue. Use when translating a paper, building a bilingual reader, or checking a draft.
---

# 文献翻译（bohr-lit-translate）

面向科研论文的中英对照翻译 Skill。脚本负责分段、术语抽取与锁定、防漂移校验和结果回填；
译文本身由调用方 Agent 按 `references/translation_rules.md` 生成，脚本不内置预制译文。

## 1. 用途与适用条件

- **适用**：单篇论文（或明确段落区间）的中英对照翻译；已有草稿译文的对照核验；术语表的抽取与全篇一致性校验。
- **输出形态**：段落级双语对照（block_id + 页码锚定），可逐段回看、修改与引用核对。
- **不适用**：版式保真的图片型中文 PDF（另走逐页渲染路线）；超文献体裁的文学翻译。
- **评分口径**：核对术语、指代、否定与限定、数字单位、公式与文献编号是否与原文一致，不以译文通顺度为准。

## 2. 输入与输出

平台运行时：原始输入位于 `/input/case.json` 与 `task.md`；中间产物落 `/app/work`；最终结果与可编辑产物落 `/app/output`，`result.json` 必须在 `/app/output` 下。

**输入（case.json）**

| 字段 | 说明 |
|-|-|
| `case_id` | 案例标识，原样写入 result.json。 |
| `source.kind` / `source.path` | 材料类型（pdf / markdown / text）与相对或绝对路径。 |
| `scope.pages` / `scope.sections` | 翻译范围，可选。 |
| `glossary.entries` | 案例专用术语表，与种子表合并并覆盖同名词条。 |
| `do_not_translate.entries` | 案例专用不可译清单。 |

**输出（output/）**

| 文件 | 说明 |
|-|-|
| `result.json` | 结构化结果，字段对齐 `references/output_contract.md`。 |
| `reader.md` | 人读双语对照读本，每段带 block_id 与页码注释。 |
| `source_map.json` | 机读对齐表，供复核与回填。 |
| `audit_issues.json` | 全部校验问题与修订建议。 |

## 3. 执行流程

1. **分段建表**：读取案例材料，切成带锚点的 block，合并术语表与不可译清单并回填出现位置。

   ```bash
   python scripts/segment.py --input case.json --work work/
   ```

2. **逐段翻译**：调用方 Agent 对照 `references/translation_rules.md` 翻译 `work/blocks.json` 中的每个 block，写入译文文件（支持字典或 blocks 数组两种格式）：

   ```json
   {"blocks": [{"block_id": "p1-b1", "target": "中文译文", "note": "存疑备注，可空"}]}
   ```

3. **防漂移校验**：运行七类探针，装配 result.json 与问题清单；`--strict` 下存在 error 返回非零。

   ```bash
   python scripts/audit_translation.py --work work/ --translations translations.json --output output/ --strict
   ```

4. **装配读本**：生成双语 reader 与 source map。

   ```bash
   python scripts/assemble_reader.py --work work/ --translations translations.json --output output/
   ```

5. **复核闭环**：`error` 全部修订后再交付；`warn` 逐条确认或修订；存疑术语在 `note` / `uncertainty` 标注。

## 4. 校验探针与分级

| 探针 | 检查内容 | 级别 |
|-|-|-|
| `coverage` | 每个 block 都有译文 | error |
| `number_unit` | 数字、百分比、单位、统计量未丢失、未换算 | error |
| `citation_figure` | 文献引用编号、图号、表号与原文一一对应 | error |
| `term_consistency` | 锁定译名在术语出现处真实出现，全篇一致 | warn |
| `dnt_preserved` | 基因 / 软件 / 试剂 / 缩写等不可译词保留原词 | warn |
| `english_residue` | 无整段未译英文残留（≥4 个连续非白名单英文词） | warn |
| `length_ratio` | 译文 / 原文长度比在 0.4–3.0 区间 | warn |

每条 issue 带 `source_quote`、`target_quote`、`message` 与可直接执行的 `suggestion`。

## 5. 异常与降级

**路径解析顺序**（相对材料路径，不依赖调用时的 cwd）：绝对路径 → case.json 所在目录 → `--material-dir`（若传入）→ 当前工作目录 → `/input`。

| 情形 | 检测点 | status | 处置与人工接管位置 |
|-|-|-|-|
| **输入缺失（case.json 缺失）** | 案例文件本身不存在 | `unable` | 写明缺失文件名；人工提供 case.json 后重跑。 |
| **case.json 不可解析** | JSON 语法损坏、顶层不是对象 | `unable` | 报告指明解析错误；人工修正结构后重跑。 |
| **材料文件缺失** | 声明的材料在候选目录均未找到；case.json 可解析 | `partial` | 有内联文本时用内联文本分段；缺失登记 `uncertainty`（reason=`missing_material`）。 |
| **材料不可读或加密** | PDF 打不开、口令保护、编码损坏；无文字层扫描页；PDF 依赖缺失 | `partial` | 已读到的部分 block 保留，失败项登记 `uncertainty`（`unreadable` / `missing_dependency`）；加密件人工解密。 |
| **字段缺失** | 缺 `case_id` / `source` 等 | `partial` | `case_id` 缺省回退 `unknown`；`source` 缺失按材料缺失处理，不判 `unable`。 |
| **超预算** | 请求数 / 时间 / 单条 token 逼近上限（64 / 900s / 16384） | `partial` | 先增量落盘已完成 block 与证据定位，剩余范围写入 `limits`；不得跳过校验。 |
| **合规边界** | 要求伪造译文、编造数据或绕过出处核对 | `refused` | 停止执行，交人工合规复核。 |

仅 case.json 本身缺失 / 不可解析允许落到 `unable`；材料级问题一律先降级 `partial` 并登记 uncertainty。

**退出码约定**：只要合法 `result.json` 已落盘，退出码即为 `0`（含 `partial` / `unable` 两种诚实降级）；非零退出只保留给“根本产不出合法结果”或 `--strict` 下存在 `error` 的拦截信号。调用方应读 `status`，不要以退出码替代 status。

## 6. 硬规则

- 不编造原图 / 原文中没有的文字、数字或结论；`quote` 必须逐字可检索。
- 不把推断术语写成作者原话；`needs_review` 术语必须显式标注。
- 参考文献条目本身保留原文，不翻译；正文中的引用编号原样保留。
- 基因 / 蛋白符号、细胞系、软件名、试剂名、缩写按 DNT 清单保留原词。
- 脚本不内置任何正式案例的预制答案；示例产物在 `examples/` 中明确标注 EXAMPLE。

## 7. 依赖与降级行为

- **标准库**：路径、JSON、哈希、正则等全部走 Python 标准库。
- **可选依赖 pymupdf**：仅 PDF 文字层提取使用；未安装时 Markdown / 纯文本路径不受影响，PDF 路径以 `unable` 提示安装。
- **不依赖**：pdfplumber、jsonschema、bibtexparser、rdflib、reportlab。
- 脚本入口使用 `python scripts/<脚本>.py` 调用，不写死解释器绝对路径；产物路径由参数传入。
