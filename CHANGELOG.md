# CHANGELOG（评审后修订）

2026-11-27 外部评审（DeepSeek，两轮）后对 10 个包做的修订。仓库内 SKILL.md 为评审前镜像，
以下改动以各赛道提交 zip 内文件为准。

## 07_citation_mgr

- 默认改为离线：只用案例参考元数据或本地一致性检查，`verification_method=offline`、`network_policy=offline_default`。
- 新增 `--allow-online`（默认关）、`--online-max`（默认 3）、`--timeout` 默认由 10s 降到 4s；
  最坏在线耗时 3×4s=12s，失败即回落本地。
- `primary_results` 新增 `network_policy` / `online_attempts` / `online_max`。

## 05_lit_kg

- `primary_results` 新增 `counts`：`input_entities` / `valid_entities` / `input_relations` /
  `accepted_relations` / `rejected_relations` / `deferred_relations`。
- 新增 `rejected_relations[]`：每条带 `reject_reason`、原样回显的 `input_quote`、`locator`。
- SKILL 写死状态阈值：`valid_entities>=1 且 accepted>=1` → success；`accepted==0` → partial；
  `valid_entities==0` → unable；恒等式 `input = accepted + rejected + deferred`。

## 04_nanopub

- `resolve_source` 增 `case_dir → cwd → /input` 存在性回退（原先只认 case 目录）。

## 10_latex_build

- SKILL 增两条硬规则：状态不得因静态分支产物齐全而升为 success；静态分支
  `primary_results` 十项字段必填（compile_status / engine / engine_detection /
  diagnosis_method / passes / issues / counts / static_lint / pdf / pages）。

## 校验结果

- 静态闸门 issue=0；10 包实跑 exit=0；引文逐字命中 14/14、14/14、5/5、10/10。
- 10 个 bundle 已重打，`GITHUB_URL.txt` 指向本仓库对应 `tracks/<dir>`。
