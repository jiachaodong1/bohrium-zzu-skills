---
name: bohr-latex-build
description: "Diagnose and compile LaTeX: detect missing packages, undefined refs/citations, encoding/Unicode, float/overfull and unresolved bibliography from log/aux/bbl/blg/tex; emit an auto/manual fix list and repeatable BibTeX build. Use when compiling .tex, fixing LaTeX errors, or preparing a submission PDF."
---

# 10 LaTeX 写作与编译（latex_build）

把一份 LaTeX 工程诊断为可复核的结果：先静态解析已有 `.log/.aux/.toc/.bbl/.blg`
与 `.tex` 源码（**无需 TeX 二进制**），定位六类常见编译问题；工具链存在时再按
"多轮编译 + BibTeX"真实构建，抓 exit code、耗时与最终日志，输出按
"可自动收敛 / 需人工决策"分组的修复清单。**静态检查通过不等于编译通过；没有
PDF 就不声称编译成功。**

本包聚焦"编译得通、改得对、交得可用、复现得了"这一最小切片；公式排版美化、
期刊模板专项适配、正文写作润色只在边界内提供辅助。

## 1. 触发条件

满足任一即按本 Skill 流程执行：

- 要求编译 `.tex` 工程、产出提交用 PDF，或排查编译报错；
- 要求定位缺宏包、未定义引用/文献、编码/Unicode、浮动体溢出、参考文献未解析等问题；
- 要求把编译轮次、引擎、宏包依赖记录成可复现构建；
- 无 TeX 环境下先给出可执行的静态诊断，并如实标注未编译。

## 2. 输入契约

输入为一个 `case.json`（字段详见 `references/output_contract.md` 与
`examples/input_example.json`）：

- `inputs.project`：LaTeX 工程目录（相对 case.json 所在目录）；
- `inputs.tex`（可选）：主 `.tex` 文件；缺省时按 `main.tex / manuscript.tex /
  paper.tex / article.tex` 顺序自动查找；
- `static_only`（可选）：为 `true` 时跳过编译，只走静态诊断。

## 3. 执行流程

```bash
# 完整流程：静态诊断 + 真实编译（有工具链时）
python scripts/run_pipeline.py --case case.json --out out/

# 仅静态诊断（无 TeX 环境或离线复核）
python scripts/run_pipeline.py --case case.json --out out/ --static-only

# 指定材料根目录（默认 case.json 所在目录）
python scripts/run_pipeline.py --case case.json --out out/ --material-dir /app/work

# 单独调用诊断器
python scripts/diagnose_latex.py --project . --tex main.tex --out out/
python scripts/diagnose_latex.py --log sample.log --project . --out out/

# 单独调用构建器
python scripts/build_latex.py --project . --tex main.tex --out out/ --timeout 240
```

步骤说明：

1. **装载与校验**：读取 `case.json`；核对工程/源码路径存在、UTF-8 可解码、未加密。
2. **静态诊断**：解析 `.aux`（`\citation`/`\bibcite`/`\newlabel`）、`.bbl`、`.blg`、
   `.toc` 与 `.log`；对 `.tex` 做括号/环境配对、label/ref、citekey/`.bib`、素材文件
   一致性检查。
3. **交叉核对**：aux 引用但 `.bbl` 未解析 → 判为文献工具未跑或条目缺失；
   ref 指向的 label 缺失 → 判为源码补标签。
4. **真实构建（可选）**：探测 `latexmk/xelatex/pdflatex`；无 latexmk 时按
   "引擎 → BibTeX（aux 含 citation 时）→ 引擎 ×2"编排；每轮带超时保护。
5. **复核落盘**：解析最终日志重新归类；产出 `diagnosis.json`、`fix_list.md`、
   `build_report.json`、`result.json`。

## 4. 问题分类与处置口径

六类信号的典型报文、根因、修复动作与验证方法见 `references/error_taxonomy.md`；
类别枚举与 `issues` 字段结构见 `references/output_contract.md` §2.1/§5。

- **auto**：少跑一轮 LaTeX、漏跑 BibTeX/Biber 等可确定性收敛的问题；脚本给出
  轮次动作与验证信号。
- **manual**：宏包选型、源码改写、表格/公式宽度与浮动体取舍；脚本只给定位、
  候选方案与验证方法，不替作者改语义。

## 5. 异常与降级

四类异常均给出明确 `status`、原因码与人工接管位置；降级路径先落盘已有成果，
不静默重试、不伪造编译结果。

| 情形 | 判定 | status | 原因码 | 处置与人工接管位置 |
|-|-|-|-|-|
| **输入缺失** | `case.json` 不存在或不可解析 | `unable` | `input_file_missing` / `invalid_record` | 写出仅含状态的 `result.json`；人工补齐可读的 `/input/case.json` 后重跑。这是唯一允许判 `unable` 的路径 |
| **字段缺失** | 未提供 `inputs.project` / `inputs.tex`，或主文件路径指向不存在的文件，但仍有 `.log/.aux/.bbl` 等辅助材料 | `partial` | `material_missing`（登记进 `uncertainty`） | 对可用辅助文件照常输出静态诊断与修复清单；人工补齐工程目录或主 `.tex` 路径后重跑，复核源码级问题 |
| **材料不可读或加密** | 源码 UTF-8 解码失败、NUL 占比异常（二进制/加密内容） | `partial` | `material_unreadable`（登记进 `uncertainty`） | 不产出任何修复后源码；按已有辅助材料诊断，人工解密或替换为可读 UTF-8 源码后重跑 |
| **超预算** | 达到轮次/条目上限，或响应/时间预算紧张 | `partial` | `over_budget` | 先落盘当前诊断与已有产物；延迟项登记在 `uncertainty`，预算内续跑，不一次吃满额度 |

**无 TeX 工具链时的 `unable_to_compile` 语义（本赛道额外约定）**：

- 探测顺序：PATH（`shutil.which`）→ 常见安装目录回退（TeX Live / MiKTeX /
  `/usr/bin`、`/usr/local/bin`）；均未命中，或显式 `--static-only`，则
  `compile_status=unable_to_compile`、`engine=null`、`pdf=null`、`pages=-1`。
- 此时**禁止**声称编译成功、禁止给出 PDF 路径；静态诊断照常执行并落盘。
- `unable_to_compile` 是编译环节语义，写入 `uncertainty` 与 `build_report.json`，
  **不把整体 status 判为 unable**；整体为 `partial`，`status_reason` 写明
  "未真实编译，结论未经编译验证"。
- 接管位置：安装 TeX Live 或 MiKTeX 并将引擎加入 PATH，或把工程迁移到有工具链
  的环境重跑；已有 `diagnosis.json` 作为迁移前检查保留。
- **无引擎场景的判据**：静态诊断（`.log/.aux/.bbl` 等解析）照常给出修复清单，
  整体 `status=partial`、`compile_status=unable_to_compile`，这不是运行失败；
  绝不伪造编译成功、绝不给出不存在的 PDF 路径。

**Exit code 约定**：只要写出合法 `result.json`，退出码即为 `0`（含 `partial` /
`unable` / `unable_to_compile`）；非零退出仅用于根本产不出合法 `result.json` 的
不可恢复故障。编译结论以 `status` / `compile_status` 为准，不以退出码判断。

补充：触发安全/合规边界（如要求伪造编译结果、编造 PDF 产物或 benchmark 分数）
→ `refused`，交人工合规审查。

## 6. 输出与自查

- 产物清单与字段契约：`references/output_contract.md`。
- 交付前逐项核对：`references/checklist.md`。
- 最小可编译模板（article + 中文支持 + bib 示例）：`references/template/`。
- 自检要点：`result.json` 字段齐全；未编译时 `compile_status=unable_to_compile`；
  源码运行前后哈希一致；`.bib` 原件不被覆盖。

## 7. 边界

- 仅解析 `.log/.aux/.toc/.bbl/.blg` 与 `.tex/.bib`；Biber 的 `.bbl` 语义按
  BibTeX 格式近似处理。
- 不自动改写作者正文来换取版面干净；overfull、浮动体与模板版式取舍需人工决策。
- 未覆盖专有期刊/学位论文 `.cls` 的版式专项校验；模板必填项核对需结合案例要求。
- 无工具链环境下不产出 PDF，页数与最终排版无法核验，结论不越界。
