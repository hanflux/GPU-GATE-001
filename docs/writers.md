# GPU-GATE-001 / T001 GPU writer 清单 (P0 审计)

**日期**: 2026-10-13
**任务**: 区分真实运行 writer / 代码存在 writer / 未知 writer

---

## 1. 三类 writer 分类

| 类型 | 定义 | 数量 |
|---|---|---|
| **A. 真实运行 writer** | 进程在跑, 能 /prompt | 1 (E 盘 0.34, PID 32332) |
| **B. 代码存在, 未运行** | 完整 ComfyUI 安装, 但无进程 | 3 (T0006/T0007/AVF-MONA-001) |
| **C. 仅脚本/Controller** | 启动器或 Controller, 不带 ComfyUI 主体 | 4 (T0006 setup/start/stop + Mona first-round) |
| **D. 未知** | 仅凭代码痕迹, 不能确认能力 | 0 (本次审计没发现) |

## 2. A 类: 真实运行 writer

### A.1 `E:\ComfyUI_windows_portable\ComfyUI\main.py` (生产)

| 字段 | 值 |
|---|---|
| PID | 32332 |
| 端口 | **9200** (LISTEN, 0.0.0.0) |
| 启动 | 2026-10-11 18:40:27 |
| RAM | 3623 MB |
| ComfyUI 版本 | 0.34.0 (系统_stats 报告) |
| Python | 3.13.12 |
| NSSM 服务 | `ComfyUI` Status=Running, StartType=Manual |
| 工作目录 | E:\ComfyUI_windows_portable\ComfyUI\ |
| 输出目录 | E:\comfyui-output\ |
| 鉴权 | **无** (HTTP unauthenticated) |
| 速率限制 | **无** |
| GPU 锁 | **不参与** (没集成 NAS lock) |

> **证据**:
> - `Get-Process python | Where-Object Path-like *ComfyUI*` → PID 32332
> - `Get-NetTCPConnection -LocalPort 9200 -State Listen` → PID 32332
> - `Get-Service ComfyUI` → Status=Running
> - 命令行: `"E:\ComfyUI_windows_portable\python_embeded\python.exe" -u ...\main.py --listen 0.0.0.0 --port 9200 --output-directory E:\comfyui-output`

## 3. B 类: 代码存在, 未运行

### B.1 `E:\ComfyUI_T0006_Qwen2511\ComfyUI\`

| 字段 | 值 |
|---|---|
| 来源项目 | T0006 (Qwen-Image-Edit-2511, Apache-2.0) |
| 完整 ComfyUI | 已在 E 盘 |
| 启动端口 | 8198 (强制) |
| 启动器 | `E:\T0006-bootstrap\t0006_setup_side_instance.ps1` (12,364 B) |
| 状态 | 进程未启动, venv `E:\ComfyUI_T0006_Qwen2511\venv\Scripts\python.exe` 存在 |

> **证据**:
> - `Get-ChildItem C:\Users\scott -Recurse -Filter python.exe` → 含此路径
> - setup 脚本硬编码: `if ($Port -ne 8198) { throw "Only port 8198 is supported." }`

### B.2 `E:\ComfyUI_T0007_FluxKlein4B\ComfyUI\`

| 字段 | 值 |
|---|---|
| 来源项目 | T0007 (FLUX.2 Klein 4B 蒸馏/基础) |
| 启动端口 | (无 setup script) |
| 启动器 | **缺失** (`E:\t0007_staging\` 只有 workflow JSON) |
| 状态 | 进程未启动, venv `E:\ComfyUI_T0007_FluxKlein4B\venv\Scripts\python.exe` 存在 |
| 关联 workflow | `w_t2i.json`, `w_base.json`, `w_distilled.json` (96-109 KB each) |

### B.3 `C:\Users\scott\AVF-MONA-001\runtime-validation\comfy-v0.39.0-...`

| 字段 | 值 |
|---|---|
| 来源 | Mona (AVF-MONA-001 隔离验证) |
| ComfyUI 版本 | 0.39.0 (独立下载, sha256 `3dca347842ed1c...`) |
| Python | 3.13.14 |
| Torch | 2.14.0+cu130 |
| 工作目录 | `C:\Users\scott\AVF-MONA-001\runtime-validation\comfy-v0.39.0-62e14b51-...` |
| 状态 | 进程已结束 (CPU_ONLY 验证后) |
| 启动端口 | 0 (CPU 模式, http_service_started: false) |

> **证据**: `runtime-status.json` 字段 `phase: STOPPED_CPU_VALIDATION`, `gpu: false`, `inference: false`

## 4. C 类: 仅脚本/Controller (启动器, 不带 ComfyUI 主体)

### C.1 `E:\T0006-bootstrap\t0006_setup_side_instance.ps1` (12,364 B)

- 设置 E:\ComfyUI_T0006_Qwen2511 为 side root
- 下载模型 (Qwen-Image-Edit-2511 31 GB, 4 模型)
- 启动端口 8198
- 包含 `Assert-ProductionHealthy` 守护 (启动前/后 + 停止时, 检查 8188 健康)
- 包含 `Assert-NoReparse` 路径检查 (防 junction)
- 包含 SHA256SUMS.txt 验证

### C.2 `E:\T0006-bootstrap\t0006_start_side_instance.ps1` (5,817 B)

- 启动 ComfyUI 8198, 写 PID 记录到 `runtime\t0006-comfyui.pid`

### C.3 `E:\T0006-bootstrap\t0006_stop_side_instance.ps1` (2,043 B)

- 严格 PID 身份验证: PID + CreationDate + ExecutablePath + CommandLine + main.py + listen + port
- Kill + WaitForExit 10s
- 移除 PID 记录
- 始终断言生产 8188 健康

### C.4 `C:\Users\scott\AVF-MONA-001\identity-round-1\a69a6db1-...`

- `cpu-contract-controller.py` (6,833 B) — 测 `/prompt` 和 `/interrupt` 的 CPU 端到端
- `first-round-controller.py` (14,741 B) — **CPU admission only**, 禁止身份 prompt POST (`IDENTITY_SUBMISSION_BLOCKED_BEFORE_POST` L164)
- 严格: 拒绝 `{'prompt':{},'prompt_id':owned_uuid}` 之外任何 POST
- 任何外部 cancel 必须是 owned_uuid

## 5. 提交入口 (Mac mini 端)

| 路径 | 状态 | 备注 |
|---|---|---|
| `/Users/oc/.hermes/avf/avf_mona_bridge.py` | Draft, **不使用** (per Owner PR #25/#26 保留) | Mac mini CLI 适配层 |
| `/tmp/avf*workflow*.py` (30+ 个) | 历史, 一次性脚本 | ssh + curl 9200, 不经 NAS lock |
| `/tmp/avf*_wf.json` | 同上 workflow 文件 | 由 /tmp/avf*.py 生成 |
| AVF-MONA-001 cpu-contract / first-round | 隔离 0.39 用, 已结束 | 不通过生产 9200 |

## 6. 未知 / 需进一步审计 (未在本轮验证)

| 项 | 状态 | 阻断原因 |
|---|---|---|
| E:\ComfyUI_H3_V0002_3 | 目录存在, 未审计 | Owner 严令 "不重启不修改 E 盘生产", 不可启动 |
| E:\t0007_staging\w_*.json 的 launcher 是否有 | 不知道 | T0007 setup/start script 不在 E:\t0007_staging 目录 |
| SynologyDrive / ollama_models | 第三方工具, 跟 AVF 无关 | - |
| 其他用户 (非 scott) 是否有 ssh 4090 | 不知道 | 不在 P0 范围 |

## 7. 关键结论 (用于 docs/lock-risks.md)

1. **E 盘生产 ComfyUI 0.34 (9200) 是唯一真运行 writer**, 但**不参与任何 GPU 锁** — 这就是 Owner 提到的 "缺覆盖全部现有 GPU writer 的权威独占准入" 风险.
2. **T0006 有自己的 side instance contract** (端口 8198 + 严格 PID 验证), **不通过 NAS lock 协调** — 跟 AVF 互不感知, 理论可同时启动两个 ComfyUI 进程, 共享同一 GPU.
3. **T0007 / AVF-MONA-001 都已停止**, 不构成现役 writer.
4. **Mona first-round-controller 严格 CPU admission**, 不会发身份 prompt POST — 但能发 cancel/interrupt 测试.
5. **任何能 ssh 192.168.50.202 + 知道 9200 的人都能直 POST**, **不经过 NAS lock** — 这是当前最严重的可绕过性.

