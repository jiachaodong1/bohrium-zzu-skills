# bohrium-zzu-skills

Bohrium Campus Challenge（郑州大学站 / Zhengzhou University station）十个赛道的 Skill 参赛作品集。

每个赛道是一套可被 Agent 调用的 Skill：`SKILL.md` 定义触发条件、输入输出契约、流程、异常降级与边界；配套脚本、schema、examples 与自测证据以 ARM bundle 形式随平台提交（逐字节清单见 [PACKAGE_INDEX.md](PACKAGE_INDEX.md)）。

## 赛道

| # | 赛道 | challenge | Skill |
|-|-|-|-|
| 01 | 阅读01｜文献文档解析 | `product-arena-08-3027d350` | [tracks/01_lit_parse](tracks/01_lit_parse) |
| 02 | 阅读03｜文献翻译 | `product-arena-01-5a9928c1` | [tracks/02_lit_translate](tracks/02_lit_translate) |
| 03 | 阅读10｜文献总结与结构化笔记 | `product-arena-06-6d8cf72b` | [tracks/03_lit_notes](tracks/03_lit_notes) |
| 04 | 阅读14｜Nanopublication | `product-arena-09-0e8befd4` | [tracks/04_nanopub](tracks/04_nanopub) |
| 05 | 阅读15｜文献知识图谱构建 | `product-arena-10-760d1a0b` | [tracks/05_lit_kg](tracks/05_lit_kg) |
| 06 | 写作02｜学术语言润色 | `product-arena-02-7051d92d` | [tracks/06_academic_polish](tracks/06_academic_polish) |
| 07 | 写作04｜参考文献与引用管理 | `product-arena-03-429cc14d` | [tracks/07_citation_mgr](tracks/07_citation_mgr) |
| 08 | 写作08｜引言与相关工作写作 | `product-arena-04-4b82ad21` | [tracks/08_intro_related](tracks/08_intro_related) |
| 09 | 写作09｜方法与实验章节写作 | `product-arena-05-64faa4a2` | [tracks/09_methods_writer](tracks/09_methods_writer) |
| 10 | 写作13｜LaTeX 写作与编译 | `product-arena-07-865de7e4` | [tracks/10_latex_build](tracks/10_latex_build) |

## 共同设计

- **统一结果契约**：`result.json` 含 `case_id / task_type / status / status_reason / inputs / primary_results / evidence[] / uncertainty[] / limits[] / change_log[] / artifacts[]`；`status` 四值 `success | partial | unable | refused` 语义固定。
- **证据可回溯**：每条 evidence 带 locator（page / block / table / file+line）与逐字 quote；quote 经空白归一化后必须能在源文件中精确命中。
- **异常分级**：材料级问题一律降级 `partial` 并登记 uncertainty，只有输入本身缺失 / 不可解析才 `unable`；合法 result.json 落盘即退出码 0。
- **依赖可降级**：核心走标准库 + PyMuPDF（可选），pdfplumber / jsonschema / bibtexparser / rdflib / pytesseract 缺失均有明确降级路径。

## 验证

十个包均在纯净系统 Python 上按其自身 `SKILL.md` 声明的命令逐条实跑：十个全部 exit=0 并产出字段齐全的 `result.json`；对 evidence 引文做了源文件逐字回溯抽查（01 14/14、03 14/14、06 5/5、10 10/10）。

平台三个正式用例不公开，全部自测均为 SELF-MADE 样例，仓库与包内不含任何官方结果、官方分数或预制答案。

## 许可

内容用于本次赛事提交与评审；引用请注明来源仓库。
