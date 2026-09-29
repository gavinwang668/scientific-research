# ModelScope 下载工作流

本文档定义段 3（ModelScope 下载）的完整流程、缓存约定与失败降级策略。

## 触发条件

- 段 1（显式路径）与段 2（本地探测）均未命中；
- **且** `task_context.allow_download == true` 或 `execution_flags.autonomous_mode == true`；
- 交互式确认（非 autonomous_mode 时）：向用户展示推断的 `repo_id` 与目标缓存目录，等待确认后继续。

## Repo ID 推断

优先级从高到低：

1. `task_context.modelscope_repo_id` 显式提供 → 直接使用。
2. 从 primitives 的 `spec.md` / `usage.md` 中提取（若返回内容包含 `modelscope.cn/datasets/<org>/<name>` 或 `repo_id: <...>` 等模式）。
3. 从 `metadata.json` 的 `provider` / `source` / `provenance` 字段提取。
4. 默认约定：`OneScience/<dataset_name>`（组织名固定为 `OneScience`，与 [marketplace.json](file:///e:/works/zhognkeshuguang/data_management/oneskills-work/.claude-plugin/marketplace.json) 一致）。

推断结果必须记录到 `observation.source_resolution.download_details.repo_id_source`（`explicit | spec_extract | metadata_extract | default_convention`）。

## 缓存目录约定

固定为：`~/.onescience/datasets/<dataset_name>/raw/`

- **命中判定采用"递归可见文件"口径**：目录存在且含至少一个非隐藏文件（路径各段均不以 `.` 开头）→ 视为已下载，跳过下载直接使用（幂等，`method=cache_hit`）。
- 仅含隐藏残留（`.lock/`、`._____temp/`、`.mdl`/`.msc` 元数据等 SDK 记账产物）→ **视为未下载**，走正常下载流程；隐藏残留不会造成假命中（与 `resolve_source.py::_dir_ok` 同一口径）。
- 若目录存在但为空 → 删除后重新下载。
- 下载过程使用**临时目录**（`~/.onescience/datasets/<dataset_name>/.raw.tmp.<pid>/`），完成后移动到最终位置，避免半完成状态污染缓存。
- **失败残留自动清理**：三级路径全部失败后，删除临时目录并调用 `_cleanup_residue`——若数据集缓存根下无 populated cache 且无任何可见文件，则整目录移除残留骨架，保证下次运行能重新触发下载而不是被残留骗成"本地已有数据"。

## 下载执行

### 首选路径：ModelScope Python SDK

```python
from modelscope.hub.snapshot_download import snapshot_download
local_path = snapshot_download(
    repo_id=repo_id,
    repo_type='dataset',
    cache_dir=str(cache_root),   # ~/.onescience/datasets/<name>/
    revision=revision,           # 可选，来自 task_context
)
```

依赖：`pip install modelscope`。SDK 未安装时抛出可诊断异常并降级到 CLI 路径。

### 降级路径：ModelScope CLI

```bash
modelscope download --dataset <repo_id> --repo-type dataset --local_dir <cache_dir>
# 模型仓库则用 --model <repo_id> --repo-type model
```

依赖：`modelscope` CLI 在 `PATH` 中。注意 CLI v1.37+ **没有 `--repo-id` 参数**，repo 通过 `--dataset`/`--model` 传入。

### 二级降级：Git LFS

若 SDK 与 CLI 均不可用，且 `repo_id` 能推断出 ModelScope 网页地址：

```bash
git lfs install
git clone https://www.modelscope.cn/datasets/<repo_id>.git <cache_dir>
```

此路径仅在 `task_context.allow_git_lfs_fallback == true` 时启用（默认 false，避免大文件不受控下载）。

## 认证与凭据

- 公开数据集：无需凭据。
- 私有/受限数据集：需要 `MODELSCOPE_API_TOKEN` 环境变量或 `~/.modelscope/credentials`。
- 本技能**不管理凭据**，仅在下载失败且错误信息含 `401/403/authentication` 时，在 `blocked_details` 中提示用户配置凭据。

## 失败降级

失败分类由 `download_modelscope.py::_classify_error` 产出，**细分 reason 直接透传为
`blocked_reason`**（不再统一为 `download_failed`）。分类优先级：认证 → 网络 → 磁盘 →
校验和 → 仓库不存在 → 兜底。**网络信号优先于 repo 措辞**：DNS/断网 traceback 中常混有
"repo ... not exist" 字样，若先匹配 repo 规则会把可重试的网络故障误判为不可重试的
`repo_not_found`（已修复的实测缺陷）。

| 失败类型 | `blocked_reason` | 行为 |
|---|---|---|
| SDK 与 CLI 均未安装 | `download_failed` | `blocked_details` 提示 `pip install modelscope` |
| 网络故障（DNS 解析失败 / 连接拒绝 / 超时 / max retries） | `network_error` | **可重试**：SDK 最多 3 次、指数退避（1s/4s/16s）；仍失败降级 CLI，再降级 git-lfs（需显式开启） |
| Repo 不存在（404 / not exist(s) / does not exist） | `repo_not_found` | **不可重试**：SDK 立即短路，仍尝试 CLI 与 git-lfs 各一次；`next_recommendation` 建议核对 repo_id 或手工提供 source_dir |
| 认证失败（401/403/auth/token） | `authentication_failed` | 不可重试路径同上；`blocked_details` 提示配置 `MODELSCOPE_API_TOKEN` |
| 磁盘空间不足（space/quota/disk） | `insufficient_disk_space` | 不可重试路径同上 |
| 文件损坏（checksum/hash/corrupt） | `checksum_mismatch` | 可重试（与 network_error 同预算）；下载先落临时目录，失败即整体废弃重来，无需单独"删缓存"步骤 |
| 下载"成功"但缓存内无可见文件 | `download_failed` | 显式报 `download produced empty cache`，防止空仓库/元数据-only 静默假成功 |
| 其他未知错误 | `download_failed` | 兜底分类 |

任何失败路径统一执行：删除临时目录 → `_cleanup_residue` 清理隐藏残留骨架（见"缓存目录约定"）→ 抛出携带细分 reason 的 `DownloadError`。

## 幂等与并发

- 同一 `dataset_name` 的下载**必须幂等**：缓存命中即跳过。
- 并发保护：下载前尝试独占创建 `~/.onescience/datasets/<name>/.download.lock`；已被占用时轮询等待最多 30 分钟，超时后 `status=blocked`。

## 观测输出

`observation.source_resolution.download_details` 必须包含：

```yaml
download_details:
  repo_id: <...>
  repo_id_source: explicit | spec_extract | metadata_extract | default_convention
  repo_type: dataset | model
  cache_dir: <绝对路径>
  method: sdk | cli | git_lfs | cache_hit
  bytes_downloaded: <int>      # 仅统计可见文件（隐藏元数据不计入）
  duration_seconds: <float>
  files_count: <int>           # 同上，仅可见文件
  retries: <int>
```
