# bohrium-zzu-skills

2026 玻尔校园挑战赛（郑州大学站）10 个赛道的参赛 Skill 包索引。每个赛道一个 Skill，
都按平台运行约束设计：`/input/case.json` + `task.md` → `/app/output/result.json`，
预算 64 请求 / 900 秒 / 16384 token，无网络也能跑到底。

## 目录

| 目录 | 赛道 | challenge id | 本包核心职责 |
|---|---|---|---|
| `tracks/01_lit_parse` | 阅读01 文献文档解析 | `product-arena-08-3027d350` | 版式感知的段落/表格/公式抽取与 block 定位 |
| `tracks/02_lit_translate` | 阅读03 文献翻译 | `product-arena-01-5a9928c1` | 分段翻译 + 术语锁定 + 回译审计 + 拼装 |
| `tracks/03_lit_notes` | 阅读10 文献总结与结构化笔记 | `product-arena-06-6d8cf72b` | IMRaD 笔记卡：证据锚点、强度归属、单位对账、双版渲染 |
| `tracks/04_nanopub` | 阅读14 Nanopublication | `product-arena-09-0e8befd4` | 四件套 TriG：稳定 sha256 IRI、逐字 quote 绑定、自检 |
| `tracks/05_lit_kg` | 阅读15 文献知识图谱构建 | `product-arena-10-760d1a0b` | 别名归一、证据门禁、JSON/GraphML/CSV 导出与一跳查询 |
| `tracks/06_academic_polish` | 写作02 学术语言润色 | `product-arena-02-7051d92d` | 锁定令牌 + 词级 diff + overreach/drift 风险标注 |
| `tracks/07_citation_mgr` | 写作04 参考文献与引用管理 | `product-arena-03-429cc14d` | BibTeX/RIS 字段核验、critical/major/minor 分级、非破坏 patch |
| `tracks/08_intro_related` | 写作08 引言与相关工作写作 | `product-arena-04-4b82ad21` | outline↔草稿对账、引用完整性、缺口落地 |
| `tracks/09_methods_writer` | 写作09 方法与实验章节写作 | `product-arena-05-64faa4a2` | 六字段核对、`[[MISSING]]`/`[[CONFLICT]]` 占位、未登记数值拦截 |
| `tracks/10_latex_build` | 写作13 LaTeX 写作与编译 | `product-arena-07-865de7e4` | 静态诊断（无 TeX）+ 多轮编译与 BibTeX 编排 |

## 每个 Skill 的结构

```
SKILL.md              触发条件 / 输入契约 / 命令 / 输出契约 / 异常降级 / 边界
scripts/              纯标准库可运行的实现，重依赖均为可选
references/           字段字典、枚举、分级口径、自查清单
examples/             最小输入输出样例（EXAMPLE ONLY）
environment/resources.yaml  依赖与降级表
```

## 交付与验证口径

- 运行时只写 `/app/output`，中间产物落 `/app/work`；materials 路径按
  绝对 → case.json 目录 → `--material-dir` → cwd → `/input` 逐级解析，
  平台在 `/app/work` 跑时不会因路径判死。
- 只要写出合法 `result.json` 就 `exit 0`；`partial` / `unable` / `refused`
  都是诚实降级，成败以 `result.json.status` 判定。
- 证据 `quote` 逐字来自材料，无出处不写成事实；不伪造页码、不补造数值、
  不声称未发生的编译成功。

## 关于本仓库文本

这里的 `SKILL.md` 是各提交包的**可读镜像**，便于评审直接阅读；权威字节以
各赛道提交 zip 内文件及其清单中的 sha256 为准（本仓库文本与提交包可能存在
个别字符级差异，如空白/符号排版）。

## 许可

竞赛提交作品，仅供评审与复现阅读。
