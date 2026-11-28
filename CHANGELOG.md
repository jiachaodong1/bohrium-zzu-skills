# CHANGELOG

## 2026-11-27 第二轮：paper-reader 移植（02 / 03 / 01）

把成熟本地 skill paper-reader 的三个可复用判据移植进比赛包（不整包搬入：其 69 文件 / 507KB
破平台 50 文件上限，且 jinja2 + Swiss Grid HTML 输出不匹配 result.json 契约）。仓库内 SKILL.md
仍为可读镜像，字节权威以各赛道 zip 内文件与 PACKAGE_INDEX 的 sha256 为准。

### 02_lit_translate

- `audit_translation.py` 新增三条硬错（探针由七类扩至十类）：
  `pseudo_translation`（译文与原文逐字相同，照抄冒充翻译）、`placeholder_left`（残留 TODO/待译标记）、
  `no_cjk_output`（无汉字输出）。
- 新增失败用例 `selftest/case/case_pseudo/`：实测 `error=2`、`status=partial`、`--strict` 下 `exit=1`。

### 03_lit_notes

- `notes_schema.py` 证据强度三态扩为四态，新增 `unreadable_scope`（图表内文字不在文本层）；
  `validate_notes.py` 该态落在事实位即报 `E-STRENGTH` 并提示移入 uncertainty。
- 新增 `scripts/spotcheck.py`：NFKC + 空白归一，按页/整篇定位引文，确定性抽样 ≥30% 且 ≥2 条；
  `quote_scope=figure_image` 记 `out_of_text_layer`；`--strict` 硬缺失 `exit=1`。
- 用例：`note_good.json` 12/12 located；`note_bad_quote.json`（`12-week`→`12-weeks`）
  11 located + 1 not_in_source、`exit=1`。

### 01_lit_parse

- `parse_pdf.py` 增加 PDF 魔数前置校验：`.pdf` 文件头不是 `%PDF-` 即 `status=unable`，
  `uncertainty` 记 `magic_bytes_mismatch`，`--strict` 之外退出码仍为 0（诚实失败而非崩溃）；
  应对出版社直链返回 200+HTML 的常见坑，杜绝按文件名猜测内容。
- 用例：`selftest/_mb/`（`<html>403</html>` 改名 `fake.pdf`）→ unable 且 reason 回显读到字节；
  正常 `case.json` 保持 success / blocks=29 / evidence=14。

### 校验

- 静态闸门 issue=0；10 包实跑 exit=0；引文命中 01 14/14、03 14/14、06 5/5、10 10/10；
  10 个 bundle 已重打。

## 2026-11-27 首轮：外部评审（DeepSeek，两轮）后修订

- 07：默认离线，新增 `--allow-online` / `--online-max` / `--timeout=4`，输出 `network_policy`。
- 05：新增 `counts` 六元组与 `rejected_relations[]`（带 reject_reason 与原样 input_quote），
  状态阈值钉在 `valid_entities==0 → unable`，恒等式 `input = accepted + rejected + deferred`。
- 04：`resolve_source` 增 `case_dir → cwd → /input` 回退。
- 10：状态不得因静态分支产物齐全而升 success；静态分支十项字段必填。

校验：issue=0、10 包 exit=0、引文 14/14、14/14、5/5、10/10，bundle 已重打。
