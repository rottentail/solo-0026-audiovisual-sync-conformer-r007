诊断完成。仓库未做任何业务代码改动（`git status` 干净，临时工具已删除），全部复现产物在 `/tmp/repro/` 下的隔离数据目录中。

# 数据目录写入故障导致任务卡在"处理中"的诊断报告

## 一、结论摘要

服务把每个生命周期状态**先落盘再继续执行**(`queued→detecting→ready`、`queued→rendering→committing→completed`)。当某一次状态落盘因数据目录写入故障失败时，请求以通用的 `500 internal_error` 报错返回，但磁盘上保留的是**上一个已成功落盘的非终态记录**。重启后这条记录**不会被任何恢复逻辑接管**——启动路径（`internal/storage/repository.go:19`）只做 `MkdirAll`，没有扫描/对账非终态记录的代码——它从此永久停留在"处理中"。同标识符重试走的是幂等回放分支，直接原样返回这条卡住的记录，**不会重新处理**。

## 二、复现方法（可重复）

- 用 Go 1.23.4 构建未修改的 `cmd/syncservice`，监听 `127.0.0.1:1809x`，每个场景使用独立的全新数据目录（`/tmp/repro/dataA`…`dataE`)。
- 媒体：通过生产代码 `media.CalibrationPair()` 生成校准 WAV/Y4M（音频事件 500ms、画面事件 375ms、期望偏移 +125ms)。
- 故障注入：一个轮询监视器在目标记录文件（如 `analyses/sync-a1.json`）首次出现（即 `queued` 已落盘）后，立即 `chmod 0555` 对应命名空间目录，使**下一次**状态落盘的 `rename`/`create` 以 `EACCES` 失败。这与磁盘满（ENOSPC)、只读重挂载（EROFS）走完全相同的错误路径（`internal/storage/atomic.go:13,34` → 包装为普通 error → `internal/domain/errors.go:38` 映射为 500)。
- 对照场景：C（故障发生在首条记录落盘之前）、D（提交冲突但失败记录可写）、E（直接 SIGKILL 模拟进程崩溃）。

## 三、关键证据

### 场景 A：分析中途写故障（dataA)

| 步骤 | 结果 |
|---|---|
| `POST …/sync-analyses`（故障注入中） | **HTTP 500** `{"code":"internal_error","message":"unexpected service failure"}` |
| `GET /v1/sync-analyses/sync-a1` | 200,`state:"detecting"`,`confidence:0`，无 `offset_ms` |
| 修复目录权限 + **重启服务**后再 GET | **仍是 `detecting`**,`updated_at` 停留在 `15:51:13.928257403Z` |
| 相同请求重试（同 id 同策略） | **HTTP 200 原样返回该 `detecting` 记录**,`updated_at` 未变（未发生任何写入） |
| 同 id 不同策略重试 | 409 `analysis_conflict` |
| 换**新** id `sync-a2` | 201 `ready`,`offset_ms:125` —— take 本身完全可用 |
| 对卡住的分析创建 conform | 409 `analysis_not_ready` |

### 场景 B：审片包生成中途写故障（dataB)

- `POST …/conforms` → **500**；记录停在 `queued`;`GET …/bundle` → 409 `conform_not_ready`。
- ** forensic 痕迹**：目录下遗留 `conf-b1.json.tmp-1210303807`(305 字节），内容正是服务**试图写入但未能 rename 到位的 `rendering` 记录**——`writeFileAtomic` 的 `defer os.Remove` 在只读目录上同样失败（`internal/storage/atomic.go:18`)，孤儿暂存文件永久残留。
- 重启后仍为 `queued`；同 id 重试 → 200 原样返回；不同上限 → 409 `conform_conflict`；换新 id `conf-b2` → 201 `completed`，包校验通过：`bundle.zip` 实际 SHA-256 与字节数 = 记录值（`24a9b72b…` / 283819),ZIP 含 `audio-aligned.wav`、`video-conformed.y4m`、`conform-receipt.json` 三项，回执 `applied_offset_ms:125`。

### 场景 C（对照）：故障发生在首条记录落盘**之前**

500 之后 `GET` 是 **404 `analysis_not_found`**；目录修复后用**同一标识符**重试 → 201 `ready`。**标识符是否报废，取决于 `queued` 记录是否已落盘**——这正解释了"重试有时有效、有时不重新处理"。

### 场景 D（对照）：提交期失败但失败记录可写

预置非空 `conforms/assets/conf-d1/` 占据提交目标 → `POST` 返回 **422 `bundle_commit_failed`**，记录到达**终态 `failed`**，不暴露 bundle(409)。同 id 重试 → 200 返回该 failed 记录（同样不重跑）。注意：这条 failed 记录里仍带有 `bundle_digest`/`bundle_bytes`，但磁盘上**并不存在对应 bundle**——记录中的摘要无法核验。

### 场景 E（对照）：SIGKILL 模拟崩溃

记录停在 `rendering`；重启后依旧 `rendering`；同 id 重试 → 200 原样返回。**崩溃与写入故障的重启语义完全相同**。

## 四、重启后这种状态意味着什么

磁盘上的非终态记录（`queued`/`detecting`/`rendering`/`committing`）= **一段被中断的在途工作的残骸**：它证明某个请求曾被受理、且至少越过了第一次状态落盘，但永远不会再推进。原因：

1. 状态机只允许从非终态向前迁移（`internal/domain/state.go:23,38`)，而**唯一驱动迁移的代码在创建型工作流内部**(`internal/service/service.go:104,148,235,283`)。
2. 创建入口先做幂等查找：记录已存在且参数一致 → 直接返回现存记录（`repository.go:172`、`repository.go:237` → `service.go:100,213`)；参数不一致 → 409 冲突。**没有任何路径会重新进入状态机**。
3. 启动时不做崩溃恢复，也没有 DELETE 路由——该标识符被永久"焊死"。

另外，客户端看到的 500 是通用 `internal_error`（存储错误不是 `domain.Error`，被 `errors.go:38` 统一掩盖），响应本身不透露是哪一步落盘失败；真正的现场只有磁盘上的记录状态 + 可能存在的 `.tmp-*` 孤儿文件。

## 五、交付证据判定

**仍可靠（可作为交付证据）**:
- **录音与画面**:`takes/<id>/audio.wav`、`picture.y4m`、`record.json`。它们在故障发生前已通过"暂存目录 + 原子 rename"整体发布，实测两份负载的 SHA-256 与 `record.json` 中摘要**完全一致**（场景 A、B 均验证 MATCH)。这是后续重跑的可靠输入。
- **故障前后已到达终态的产物**：如 `ready` 的 `sync-a2`/`sync-b1`、`completed` 的 `conf-b2` 及其 bundle——后者摘要与字节数经独立重算核验一致，回执链（take digest → analysis offset → applied samples）完整。

**不可作为交付证据**:
- 卡住的非终态记录（`sync-a1` 的 `detecting`、`conf-b1` 的 `queued`、`conf-e1` 的 `rendering`)：不含任何测量值或已提交输出；按规范只有 `ready`/`failed`、`completed`/`failed` 才是有意义的终态。
- 非终态（甚至 D 中 failed）记录里的 `bundle_digest`/`bundle_bytes`：磁盘上没有对应 bundle，属于**无法核验的声明**。
- 孤儿暂存文件（`conf-b1.json.tmp-*`)、`tmp/` 残留、陈旧 assets 目录：未提交的草稿。

**恢复后的完整交付集** = 原 take（录音+画面+记录） + 新标识符的 `ready` 分析 + 新标识符的 `completed` conform 及其 bundle。卡住的旧标识符应标记为废弃并排除。

## 六、影响范围

- **按标识符隔离**：同一 take 可新建分析、同一 ready 分析可新建 conform（实测均成功）；其他 take/分析/任务互不影响（按 id 加锁、目录命名空间隔离）。无数据污染扩散。
- **无已提交数据丢失**：损失仅限于在途工作本身 + 该标识符永久不可用；轮询客户端会对该 id 永远看到"处理中"，且重试返回 200 旧记录，容易误判为"仍在排队"。
- **运维残留**：孤儿 `.tmp-*` 文件无清理机制，会随故障次数累积。
- 本轮未修复业务代码；如需后续处理，方向（供下轮决策）是启动对账（把超时非终态记录标记为 `failed`)、创建路径区分"重放"与"重跑"、以及孤儿暂存文件回收。
