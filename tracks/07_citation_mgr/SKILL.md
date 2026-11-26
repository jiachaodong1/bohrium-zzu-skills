---
name: bohr-citation-mgr
description: "Verify BibTeX/RIS entries against Crossref or supplied metadata: check DOI/title/author/year/volume/pages, grade critical/major/minor conflicts with field-level before/after, emit reports and a non-destructive patch. Use when auditing citations, checking metadata errors, or fixing a bibliography."
---

# 07 参考文献与引用管理（citation_mgr）

把一份文献库（BibTeX / RIS）核验为可复核的结果：逐条解析字段，对照 Crossref
在线元数据或案例提供的参考元数据做逐字段比对，按 critical / major / minor 分级
冲突，给出字段级 before/after 与证据来源，交付人读报告、机器记录、unified diff
patch 与修正副本；**原文献库全程只读，绝不覆盖**。

本包聚焦"对得上"（来源核验与一致性检查）这一最小切片；引用插入定位、编号连锁
更新、去重归并、格式转换只在边界内提供辅助，不实现完整参考文献管理系统。

## 1. 触发条件

满足任一即按本 Skill 流程执行：

- 要求核验文献库条目真实性、核对 DOI / 年份 / 卷期 / 页码 / 作者 / 标题；
- 要求检出元数据错配、作者顺序异常、页码偏差，并给出可采纳的修正；
- 要求对 BibTeX/RIS 做字段审计、输出可复核报告与 patch；
- 要求在离线条件下做本地一致性检查，并如实标注未对外核验的条目。

## 2. 输入契约

输入为一个 `case.json`（字段详见 `references/output_contract.md` 与
`examples/input_example.json`）：

- `inputs.bibliography`：`.bib` / `.ris` 文献库路径；相对路径按 绝对路径 → case.json 目录 →
  `--material-dir` → cwd → `/input` 顺序解析；
- `reference_metadata`（可选）：按条目 key 给出参考元数据（title / author / year /
  volume / pages / doi / source），作为离线比对基准；也可用
  `inputs.reference_metadata_file` 指向外部基准 JSON（同顺序解析）；
- 无 `reference_metadata` 且网络可用时，按 DOI 在线查询 Crossref；两者皆无则降级为
  本地一致性检查，条目标 `unverifiable`。

## 3. 执行流程

```bash
# 1) 解析（独立自检：BibTeX/RIS -> JSON）
python scripts/parse_refs.py --input refs.bib --out out/parsed.json

# 2) 核验 + 分级 + 报告 + patch
python scripts/verify_refs.py --case case.json --out out/

# 离线模式（不访问网络；verification_method=offline）
python scripts/verify_refs.py --case case.json --out out/ --no-network

# 预算截断：单次最多核验 N 条，其余记入 deferred_keys
python scripts/verify_refs.py --case case.json --out out/ --max-entries 50

# 显式指定材料基目录（缺省为 case.json 所在目录）
python scripts/verify_refs.py --case case.json --out out/ --material-dir /path/to/materials
```

在线查询为可选增强，不强制依赖外网：无网络或查询失败时自动退化为离线字段
一致性校验，`verification_method=offline`，退出码仍为 0。

步骤说明：

1. **装载**：读取 `case.json`；核对文献库路径存在、可解码、未加密。
2. **解析**：自研标准库解析器处理嵌套花括号、跨行字段、引号转义、`#` 拼接；
   bibtexparser 未安装时不影响运行。
3. **取参考基准**：优先案例提供的 `reference_metadata`；其次 Crossref 在线解析；
   在线失败或关闭时降级本地一致性检查，并在报告中写明 `verification_method`。
4. **逐字段比对**：year / volume / author / pages / title / DOI 格式，按
   `references/conflict_grading.md` 分级，每条冲突记录 before/after/证据来源。
5. **生成 patch**：仅对有参考基准且 after 非空的字段生成 `fixed.bib` 副本与
   `refs.patch` unified diff；离线无基准只标注、不编造修正值。
6. **落盘**：`report.json`、`report.md`、`fixed.bib`、`refs.patch`、`result.json`。

## 4. 分级与比对口径

- 级别定义、字段判定口径、回写纪律：`references/conflict_grading.md`。
- 输出字段、状态语义、产物清单：`references/output_contract.md`。
- 条目状态：`pass` / `critical` / `major` / `minor` / `error` / `unverifiable`。

## 5. 异常与降级

只有 **case.json 本身缺失或不可解析** 才整体判 `unable`；其余材料缺失、不可读、
DOI 解析失败均降级为 `partial`，登记进 `uncertainty` 并继续产出已有结果，不静默
重试、不伪造核验结果。

| 情形 | 判定 | status | 原因码 | 处置与人工接管位置 |
|-|-|-|-|-|
| **输入缺失/不可解析** | `case.json` 不存在，或不是合法 JSON | `unable` | `input_file_missing` / `invalid_record` | 写出仅含状态的 `result.json`；人工补齐可读的 `/input/case.json` 后重跑 |
| **文献库/基准元数据缺失** | `inputs.bibliography` 未给、文件按 case 目录 / `--material-dir` / cwd / `/input` 顺序查找后仍不存在；或 `reference_metadata_file` 缺失 | 整体 `partial` | `missing_material` | 缺材料登记进 `uncertainty`；人工提供可读 `.bib`/`.ris` 或基准元数据后重跑；离线无基准不产出 after 值 |
| **材料不可读或加密** | 文献库 UTF-8 解码失败、二进制/加密内容（NUL 占比异常），或元数据 JSON 不可解析 | 整体 `partial` | `material_unreadable` | 不产出 after 值；人工解密或替换为可读 `.bib`/`.ris` 后重跑 |
| **字段缺失** | 条目缺 `title` / `author` / `year` 任一必填字段 | 条目 `error`；整体 `partial` | `missing_field` | 坏条目不参与字段比对，记入 `uncertainty`；人工补齐字段后重跑；禁止用推测值补全 |
| **DOI 解析失败** | 在线查询 HTTP 404 / 超时 / 连接错误 | 条目 `unverifiable`；整体 `partial` | `doi_unresolved` | 按 `local-consistency` 继续，不在报告中声称已在线核验；人工核实 DOI/元数据 |
| **超预算** | 达到 `--max-entries` 上限，或响应/时间预算紧张 | `partial` | 延迟条目记入 `deferred_keys` | 先落盘当前批次报告与 patch；延迟条目登记在案，预算内续跑核验，不一次性吃满额度 |

**退出码约定**：只要写出合法的 `result.json`，退出码即为 **0**——包括 `partial` 与
`unable`；非零退出仅保留给根本无法产出合法结果的情形（如输出目录不可写、argparse
参数缺失被拒绝）。评审以 `result.json` 内的 `status` 与 `uncertainty` 为准，不凭退出码
判运行失败。

补充：相对材料路径解析顺序为 绝对路径 → case.json 所在目录 → `--material-dir` →
cwd → `/input`；`--material-dir` 缺省取 case.json 所在目录。触发安全/合规边界（如
要求伪造 DOI 元数据或编造核验分数）→ `refused`，交人工合规审查。

## 6. 输出与自查

- 产物清单与字段契约：`references/output_contract.md`。
- 交付前逐项核对：`references/checklist.md`。
- 自检要点：`result.json` 字段齐全；离线运行 `verification_method=offline`；
  原库运行前后 sha256 一致；`fixed.bib` 与 `refs.patch` 为独立副本。

## 7. 边界

- 仅覆盖 BibTeX 与 RIS；EndNote / CNKI / NoteExpress 导出需先转换格式。
- 在线核验仅接 Crossref；中文期刊 CNKI/万方覆盖不足，未命中时不判错。
- 不做正文论断句与文献主题的论断级匹配；不自动处理正文编号连锁更新与引用插入。
- 作者仅按姓氏归一比对；同名不同人、姓名拼写歧义需人工复核。
