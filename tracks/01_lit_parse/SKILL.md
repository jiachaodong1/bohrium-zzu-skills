---
name: bohr-lit-parse
description: Parse scientific PDFs into structured blocks, section tree, tables and formulas with page-level evidence locators. Use when reading or extracting a research paper PDF, correcting two-column order, reconstructing table cells, or locating formulas, captions and references.
---

# 阅读01｜文献文档解析（bohr-lit-parse）

把一份科研 PDF 解析成**可复核的结构化结果**：block 级定位、章节层级树、表格行列归属、
公式与特殊字符保留，每条关键结论可指回原文页码与区块。不理解科学含义，只做版式与
结构还原。

## 1. 适用条件

- 输入：学术论文 / 综述 / 预印本 PDF（单栏、双栏为主；多栏按最近侧归并）。
- 任务：提取标题、摘要、章节、正文段落、图表题注、表格、公式、参考文献；修正串栏、
  错行、跨页续接；为下游翻译、问答、笔记、知识图谱提供带定位的素材。
- 不适用：图片型扫描件的高质量 OCR 还原（本技能仅做降级标记与可选 OCR）；
  公式的科学含义解读；非 PDF 文档解析。

## 2. 输入与输出

**输入**（平台运行时）：`/input/case.json`，结构见 `references/output_contract.md`：

```json
{ "case_id": "case-001", "task": {"hint": "解析要求"},
  "files": ["paper.pdf"], "options": {"max_pages": null} }
```

**输出**（落在 `/app/output/`）：

| 产物 | 内容 |
|-|-|
| `result.json` | 交付包：status、主结果、evidence、uncertainty、limits、change_log |
| `blocks.jsonl` | 每页 block 级记录：block_id、bbox、page、column、type、续接标记 |
| `parse.json` | 结构化主结果（章节树、表格、公式、题注、覆盖率统计） |

## 3. 调用方式

```bash
# 标准入口（平台：case.json 与材料在 /input，结果写 /app/output）
python scripts/parse_pdf.py --input /input/case.json --output /app/output

# 校验交付包：契约字段 + evidence 逐字回溯
python scripts/validate_result.py --result /app/output/result.json \
  --input /input/case.json

# 本地调试：直接指定 PDF 与页数预算
python scripts/parse_pdf.py --pdf paper.pdf --output out/ --max-pages 20
```

脚本不写死解释器绝对路径；依赖探测见 `environment/resources.yaml`。

## 4. 解析流程

1. **装载案例**：读 case.json，校验 `files` 清单；材料按“案例目录 → /input”顺序定位。
2. **打开 PDF**：PyMuPDF 打开；检测口令加密、损坏、零页；记录 sha256。
3. **block 抽取**：`get_text("dict")` 取 span 级文本、字号、flags、基线原点；
   空白归一化但不改写字符。
4. **页眉页脚剔除**：顶部 7.5% / 底部 7.5% 边带 + 跨页重复文本双重判定；
   保留记录（`type=header/footer`）但不进正文阅读顺序。
5. **栏位与阅读顺序**：按 block x0 聚类判定单/双栏；通栏 block（宽 ≥60% 页宽）优先，
   再左栏纵向、右栏纵向；相邻 block 做语义断裂探针，标记跨页/跨栏续接。
6. **类型与层级**：编号正则 + 字号阈值判定标题级别；题注、公式、参考文献分类；
   构造章节层级树。
7. **表格重建**：优先 pdfplumber；缺失时降级 fitz 矢量线条网格，按中心落入行列归属，
   记录 colspan 推断；无框线表不猜归属。
8. **公式与符号**：Unicode 原文保留，不转写；span flags 与基线偏移标注上/下标；
   希腊字母原样保留。
9. **扫描件降级**：有图无文字层的页面标记 `ocr_required`；装了 pytesseract 则 OCR
   并标注“错字需复核”；未装则写入 uncertainty，不伪造文字。
10. **证据回溯**：候选 quote 经空白归一化后必须在抽取全文中逐字命中才进 `evidence`；
    未命中转入 `uncertainty`。

## 5. 异常与降级

所有异常都落盘为合法 `result.json`，不抛栈中断；status 四值语义见
`references/output_contract.md`。

### 5.1 输入缺失

- **情形**：case.json 本身不存在，或 `files` 登记的材料在案例目录与 `/input` 下均未找到。
- **status**：`unable`。
- **status_reason**：写明缺失路径与原因；人工接管位置：确认 `/input` 下材料已挂载、
  文件名与 `files` 清单一致后重跑。
- **产物**：`result.json` 含空 `primary_results` 与定位到“locate”步骤的 change_log。

### 5.2 字段缺失

- **情形**：case.json 是合法 JSON，但顶层不是对象，或缺非空 `files` 数组。
- **status**：`unable`。
- **status_reason**：写明缺哪个字段；人工接管位置：按 `references/output_contract.md`
  补齐案例结构后重跑。
- **注意**：`case_id` 缺失不阻断，记为 `case-unknown`。

### 5.3 材料不可读或加密

- **情形**：PDF 打开异常、零页、需要口令（`needs_pass`），或解析引擎（fitz）不可用。
- **status**：`unable`。
- **status_reason**：写明异常类型（加密/损坏/引擎缺失）；人工接管位置：提供解密口令、
  未损坏未加密的 PDF，或在运行环境安装 PyMuPDF。
- **降级**：不尝试口令爆破；不把损坏页内容补猜进结果。

### 5.4 超预算

- **情形**：页数超过 `max_pages`（案例 options 或 `--max-pages`）；或运行中接近
  900 秒 / 64 次请求预算。
- **status**：`partial`。
- **status_reason**：写明总页数与已解析页数；未覆盖页面写入 `limits`；
  人工接管位置：分批解析剩余页面，或在预算内调高页数上限。
- **落盘策略**：先存已解析的 `blocks.jsonl` / `parse.json` / `result.json`，再增量补全，
  禁止一次吃满预算后无产物。

### 5.5 合规拒答（refused）

- **情形**：任务要求伪造数据、编造文献结论或绕过合规边界。
- **status**：`refused`；`status_reason` 说明触发的合规边界，不执行被拒动作。

## 6. 自检方法

交付前运行：

```bash
python scripts/validate_result.py --result /app/output/result.json --input /input/case.json
```

校验项：契约字段齐全、status 合法、evidence locator 完整、quote 逐字回溯命中、
blocks.jsonl 结构合法。人工对照 `references/checklist.md` 复核版式与边界项。

## 7. 边界与限制

- 双栏为主；三栏及以上按最近侧归并，可能误判。
- 表格：无线条表、跨页续表合并、复杂 rowspan 展开未完整实现；推断的 colspan 需复核。
- 公式：保留符号与上下标标记，不转 LaTeX、不做结构语义解析。
- 扫描件：无 OCR 引擎时只标 `unreadable`，不输出猜测文本。
- 非英文文献的断词、连字符换行未做词典级还原。

## 8. 依赖与降级行为

| 依赖 | 必需性 | 缺失行为 |
|-|-|-|
| PyMuPDF (fitz) | 必需 | 无法解析，status=`unable`，提示安装引擎 |
| pdfplumber | 可选 | 表格降级 fitz 矢量线条网格；`source` 字段记录路径 |
| pytesseract | 可选 | 文字层缺失页面标 `partial` + uncertainty，不 OCR |
| jsonschema | 可选 | 校验由 `validate_result.py` 标准库实现，不依赖该包 |
