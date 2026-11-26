---
name: bohr-nanopub-build
description: Turn a reading card's claim-evidence-source records into a four-graph Nanopub TriG with stable sha256 IRIs, verbatim quote binding and self-checking validation; use when exporting machine-readable FAIR claims, auditing provenance, or producing RDF/TriG nanopublications.
---

# 04 Nanopublication

把阅读卡中的“主张—证据—出处”转成可机器复核的四件套 Nanopublication。核心不是把摘要改成 RDF，
而是**每条断言都能回到原文的一个位置**：页码/段落/图表定位 + 逐字片段 + 论文作者/年份/期刊溯源链。

## 何时触发

- 需要把论文结论拆成一条条可独立判断真假的断言，并绑定最小充分证据；
- 需要导出 RDF/TriG 形式的 FAIR 断言结构，供下游知识图谱复用；
- 需要审计断言是否有出处、溯源链是否完整、引用片段是否逐字来自原文。

## 输入

一份 `case.json`（结构见 `examples/input_example.json` 与 `references/output_contract.md`）：

- `paper`：标题、作者、期刊、年份、DOI、来源文件；
- `claims`：每条断言含 `subject` / `relation` / `object` / `qualifier` / `evidence`
  （`page`、`block`、`figure`、`table`、`quote`、`kind`）/ `evidence_kind` / `confidence`。

断言内容由调用方阅读卡提供；本 Skill 不硬编码任何论文结论。

## 流程

1. **结构校验**：检查论文元数据与每条断言的必填字段；词表外关系标 warning，不静默改写。
2. **证据核验**：加载来源原文，按页码优先、整篇兜底，对 `quote` 做空白归一化子串匹配；
   未命中的断言隔离出断言图，不写成已核验结论。
3. **IRI 生成**：按 `references/trig_schema.md` 的稳定哈希规则生成断言 IRI 与 head IRI；
   同内容复现同 IRI，改一字则变；时间戳不进入哈希。
4. **四件套序列化**：生成 `assertion` / `provenance` / `publication_info` / `pubinfo` 四张命名图。
5. **自检**：跑四类强制校验（语法、必需 prov、无出处三元组、pubinfo 三图引用），可选逐字 quote 复核。

## 命令

```bash
# 生成四件套与 result.json
python scripts/nanopub_build.py --input <case.json> --output <out/>

# 校验（--source 提供原文时做逐字片段复核）
python scripts/nanopub_validate.py --trig <out/nanopub.trig> \
    --json <out/nanopub.json> --source <source.md> --out <out/validation-report.json>
```

## 输出

| 产物 | 说明 |
|-|-|
| `result.json` | 对齐 `references/output_contract.md`：status、evidence、uncertainty、limits、change_log、artifacts |
| `nanopub.trig` | 四命名图 TriG |
| `nanopub.json` | 结构化断言与溯源链 |
| `validation.json` | 隔离断言与校验问题 |

## 异常与降级

| 情形 | 判定与 status | 处置 | 人工接管位置 |
|-|-|-|-|
| **输入缺失**（case.json 不存在或不可解析） | `unable` | 不生成断言图；`status_reason` 记录缺失文件名；`result.json` 落盘为空结果壳 | 提供可读的 `/input/case.json`，核对 JSON 语法后重跑；接管点：`inputs.files` 为空的记录条目 |
| **字段缺失**（断言缺主体/关系/客体/页码/片段） | 对应断言隔离；整体 `partial`（全部缺失则 `unable`） | 缺字段断言进入 `primary_results.quarantined`，写明字段与原因；不补猜字段 | 按 `quarantined[].field` 补全阅读卡字段后重跑；接管点：隔离清单逐条核对 |
| **材料不可读或加密**（PDF 加密、扫描版无文本层、原文无法访问） | 无法核验的断言标 `unreadable`；全部不可核验则 `unable` | 不抽取 OCR 图像文字；quote 核验跳过并写入 `uncertainty`；不编造片段 | 先由 OCR/解密流程提供可读文本层，或补充纯文本原文；接管点：`uncertainty[].item` 对应来源条目 |
| **超预算**（请求/时间/token 接近上限） | 已核验部分落盘为 `partial`，未核验项隔离 | 增量落盘：先写 `result.json` 与已生成 TriG，不再追求补全；`change_log` 记录截断位置 | 预算恢复后按 `quarantined` 清单继续；接管点：`change_log` 最后一条截断记录 |

异常状态码固定：`success` / `partial` / `unable` / `refused`；触发伪造数据、编造证据等
合规边界时用 `refused`，并在 `status_reason` 写明边界。

## 硬规则

- 一条断言只表达一个主张；复合句先拆成多条，不整段塞进一个字段。
- 相关性不写成因果：`correlates_with` / `associated_with` 不得升格为 `causes`。
- 否定结果保留：用 `does_not_support` / `no_effect_on`，不抹平为肯定支持。
- 限定词（may / suggest / in this model）留在 `qualifier`，不抹平为确定结论。
- `publication_info` 不冒用论文作者身份：生成者记为 `dct:creator`，论文作者另记为 `boh:paperAuthor`。
- 没有出处的断言不得进入断言图；每条断言必须沿 provenance 回到来源论文与定位。

## 自检与复核

- 四类校验：语法结构、必需 prov 三元组、无出处三元组检测、pubinfo 三图引用完整性。
- 抽查证据：`python scripts/spotcheck_quotes.py --result <out/result.json> --source <source.md>`，
  逐条在源文件中按页码定位，空白归一化后逐字检索 `quote` 并打印上下文。
- 复现性：同输入两次运行比对 head IRI 与断言 IRI；改动一个字符后重跑确认 IRI 变化。
- 失败样例在自测包 `selftest/cases_bad/`（留存于提交目录、非运行依赖），每类校验对应一个可复现失败样本。

## 边界与限制

- 关系谓词受控词表见 `references/relations.vocab.md`；词表外只标 warning。
- quote 比对为空白归一化子串匹配；跨页断裂、OCR 错字不在解决范围。
- 自研校验器不做完整 RDF 蕴含推理；rdflib 可用时追加解析，覆盖差异见 README。
- 扫描版/加密 PDF、不可访问原文如实标注，不补猜证据。
