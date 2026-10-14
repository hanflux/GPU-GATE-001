# GPU-GATE-001 / T001 真实运行拓扑 (P0 审计)

**仓库**: hanflux/GPU-GATE-001
**任务**: GPU 调度现状审计 (P0 只读)
**日期**: 2026-10-13
**范围**: Mac mini / 4090 PC / NAS 提交链路盘点, 不修复不重启不提交 GPU 任务

---

## 1. 物理拓扑

| 节点 | 角色 | 关键地址 | 凭证 |
|---|---|---|---|
| **Mac mini** (Hermes default) | 调度主控 | Tailscale 100.x, 局域网 192.168.50.x | `oc` user |
| **4090 PC** | GPU worker | Tailscale 100.x, 局域网 **192.168.50.202** | `scott` ssh key |
| **OpenClaw NAS** (192.168.50.201) | 锁目录 + 归档 | SMB `//openclaw@192.168.50.201/workspace-ai` | system Keychain |
| **Tailscale** | 远程桥 | (未用, 当前走 LAN) | - |

> 注: Owner 多次纠正 4090 PC IP 是 `192.168.50.202` (不是 .201, 那是 NAS); 任何写 .201 的脚本都应修正.

## 2. ComfyUI 实例 (4090 PC)

实际命令: `Get-Process python -ErrorAction SilentlyContinue` + `Get-NetTCPConnection -LocalPort 9200 -State Listen`

| 实例 | PID | 端口 | 启动 | 工作目录 | 状态 |
|---|---|---|---|---|---|
| **E:\ComfyUI_windows_portable** (生产, 0.34) | 32332 | **9200** | 2026-10-11 18:40:27 | E:\ComfyUI_windows_portable\ComfyUI\ | **Running** ✓ |
| E:\ComfyUI_T0006_Qwen2511 | (无) | (无) | - | - | **代码存在, 未启动** |
| E:\ComfyUI_T0007_FluxKlein4B | (无) | (无) | - | - | **代码存在, 未启动** |
| C:\Users\scott\AVF-MONA-001\runtime-validation\comfy-v0.39.0-... | (无) | (无) | 2026-10-13 CPU 验证后已结束 | C:\Users\scott\AVF-MONA-001\runtime-validation\comfy-v0.39.0-62e14b51-ab32-4d9b-bbaf-ffe8d19cda97\ComfyUI_windows_portable\ | **Stopped** (CPU_ONLY) |

> **证据**:
> - PID 32332 命令行: `"E:\ComfyUI_windows_portable\python_embeded\python.exe" -u E:\ComfyUI_windows_portable\ComfyUI\main.py --listen 0.0.0.0 --port 9200 --output-directory E:\comfyui-output`
> - 端口 9200 唯一 LISTEN 进程 PID 32332
> - nssm 服务: `ComfyUI` Status=Running, StartType=Manual

## 3. 4090 PC 服务/任务清单 (Task Scheduler)

| 任务 | 状态 | 备注 |
|---|---|---|
| `ComfyUI` (nssm) | Running | 唯一运行 ComfyUI |
| `AVF_INT8_*` (12 个) | Ready | 早 AVF INT8 量化任务, 不跑 |
| `AVF_008_*` (2 个) | Ready | 早 AVF 008 任务 |
| `LTX*Download` (3 个) | Ready | LTX 模型下载任务 |
| `cua-driver-serve` | Ready | cua-driver 背景服务 |
| `CITIC_FT_ActiveEs*` (3 个) | Running | 富途证券 (跟 AVF 无关) |
| `360ZipUpdater`, `Adobe*` | Ready | 系统/第三方 |

> **没有专门的 GPU 调度网关 service** — 提交靠 Mac mini ssh + curl 9200.

## 4. 4090 PC 资源

| 项 | 值 |
|---|---|
| GPU | RTX 4090, 24,564 MiB total, **22,261 MiB free** (Driver 610.88, util 0%) |
| C: | 295.2 GB free of 1906 GB |
| E: | **38.1 GB free of 931 GB** ⚠ (E 盘生产 ComfyUI, 不易扩展) |
| RAM available | 44.58 GiB |
| 9200 端口 | LISTEN (PID 32332) |
| 其他端口 (8188/8189/8190/9201/9202/9203) | 未 listen |

## 5. NAS 锁目录 (SMB)

> **状态**: 当前 (2026-10-13 22:53 后) readdir 可用, 但历史曾阻塞 (15-30 min), 根因未确认.

| 路径 | 内容 |
|---|---|
| `/Users/oc/nas-workspace/infra/gpu-lock/4090/` | 4 个子目录 (heartbeat, history, queue, snapshots), **无 .lock 文件** |
| `/Users/oc/nas-workspace/AI-Video-Factory/V0004/evidence/AVF-016/` | a1/manifest/master/reference/two_pose (V1 master 等) |
| OpenClaw NAS 192.168.50.201 | TCP 445 OPEN, 4 SMB share active (workspace-ai ×2, home, video, download) |

> **证据命令**:
> - `mount -t smbfs` → `//openclaw@192.168.50.201/workspace-ai on /Users/oc/nas-workspace (smbfs, nodev, nosuid, mounted by oc)`
> - `ls -la /Users/oc/nas-workspace/infra/gpu-lock/4090/` → rc=0, 4 个子目录, mtime `Oct 13 22:53`

## 6. 提交链路

### 当前实际生产路径 (AVF-016/017)
1. Mac mini: `python3` 生成 workflow JSON (`/tmp/avf016_*.py`)
2. Mac mini: `scp` 传到 4090 (E:\tmp or 输出目录)
3. Mac mini: `ssh scott@192.168.50.202` + `curl -X POST http://127.0.0.1:9200/prompt -d @wf.json`
4. 4090: ComfyUI 0.34 处理 (无锁, 无 auth, 无 rate-limit)
5. 4090: 输出到 `E:\comfyui-output\`
6. Mac mini: `scp` 取回 + 视觉验收 + 写 NAS AVF-016 dir

### 旁路 (Mac mini 自己控制)
- `/Users/oc/.hermes/avf/avf_mona_bridge.py` (Draft, **不使用** per Owner)

### 旁路 (其他项目)
- T0006 Qwen-Image-Edit-2511: `E:\T0006-bootstrap\t0006_*_side_instance.ps1` → 启动独立 ComfyUI 端口 8198, 跟 NAS lock 无关
- T0007 FluxKlein 4B: 仅有 workflow JSON (`E:\t0007_staging\w_*.json`), 无 launcher
- AVF-MONA-001 Mona 隔离 0.39 (CPU only): `first-round-controller.py` 明确禁止身份 prompt POST (`IDENTITY_SUBMISSION_BLOCKED_BEFORE_POST`)

