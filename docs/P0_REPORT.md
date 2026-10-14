# GPU-GATE-001 / T001 — P0 审计报告

**日期**: 2026-10-13
**任务**: GPU 调度现状审计 (P0 只读)
**范围**: 不修复不重启不提交 GPU 任务

## 仓库

- **hanflux/GPU-GATE-001** (新建): https://github.com/hanflux/GPU-GATE-001

## 4 份交付文档

- `docs/topology.md` — 真实运行拓扑 (物理 / ComfyUI 实例 / 资源 / NAS / 提交链路)
- `docs/writers.md` — GPU writer 清单 (A 真运行 / B 代码 / C 脚本 / D 未知)
- `docs/lock-risks.md` — 锁机制风险 (R1-R6 + B1-B2 8 项)
- `docs/proposal.md` — 4090 本地统一调度网关初步方案 (选项 A: HTTP + nssm 9201)

## 关键已验证事实

### 1. 4090 PC 唯一运行 ComfyUI

- **PID 32332** (E:\ComfyUI_windows_portable, ComfyUI 0.34, Python 3.13.12)
- 启动 2026-10-11 18:40, 端口 **9200 LISTEN 0.0.0.0**
- nssm 服务 `ComfyUI` Status=Running
- 工作目录 E:\ComfyUI_windows_portable\ComfyUI\
- 输出 E:\comfyui-output\
- **无 auth, 无 rate-limit, 不参与 GPU 锁**

### 2. 4090 PC 资源 (P0 实测)

- RTX 4090 24,564 MiB total, **22,261 MiB free**, util 0%
- C: 295.2 GB free, E: **38.1 GB free** (E 盘生产 ComfyUI, 不易扩展)
- 0 个其他 Python 进程, nvidia-smi driver 610.88

### 3. 其他 ComfyUI 实例 (未运行)

- **E:\ComfyUI_T0006_Qwen2511** (Qwen-Image-Edit-2511, 完整 ComfyUI + 启动器 8198)
- **E:\ComfyUI_T0007_FluxKlein4B** (FLUX.2 Klein 4B, 仅有 workflow JSON, 无 launcher)
- **C:\...\AVF-MONA-001\runtime-validation\comfy-v0.39.0-...** (Mona 隔离 0.39, 已结束 CPU_ONLY 验证)

### 4. T0006 side instance contract

- 端口 8198 (强制)
- 严格 PID 身份验证: PID + CreationDate + ExecutablePath + CommandLine
- 启动器: `E:\T0006-bootstrap\t0006_*_side_instance.ps1` (12-5-2 KB)
- **与 NAS lock 无关** (T0006 自己的契约)

### 5. Mona controllers (CPU-only)

- `cpu-contract-controller.py` (6.8 KB) — 测 /prompt 和 /interrupt 的 CPU 端到端
- `first-round-controller.py` (14.7 KB) — 严格 CPU admission, **禁止身份 prompt POST** (`IDENTITY_SUBMISSION_BLOCKED_BEFORE_POST` L164)

### 6. Mac mini 提交入口

- `/Users/oc/.hermes/avf/avf_mona_bridge.py` (Draft, **不使用** per Owner)
- `/tmp/avf*workflow*.py` (30+ 历史一次性脚本, ssh+curl 9200, **不经 NAS lock**)

### 7. NAS 锁目录

- 4 子目录 (heartbeat/history/queue/snapshots), **无 .lock**
- mtime Oct 13 22:53 (今天触动过)
- queue_len 33 全 `completed` 历史, **不含 AVF-016/017**
- SMB mount 在, 但 P0 期间曾 readdir 阻塞 15-30 min, **根因未确认**

## 未验证事项 (P0 范围外)

| 项 | 原因 |
|---|---|
| AVF-016 master_identity_v1_manifest.json 在 NAS 是否最新 | stat rc=1 No such file (NAS 端不存? 或 Owner 报告时存在过) |
| E:\ComfyUI_H3_V0002_3 是否含历史 launcher | Owner 严令 "不重启不修改 E 盘", 不可启动 |
| T0007 launcher 在哪 | t0007_staging\ 只有 workflow JSON, 没找到 setup 脚本 |
| NAS readdir 阻塞根因 | 推测 SMB cache 同步, **未证实** (Owner 6151264721 明确要求不许 ROOT_CAUSE_CONFIRMED) |
| 4090 ssh 用户名单 | 不在 P0 范围, 不查 |
| /tmp/avf* 临时脚本完整 30+ 个数 | 抽查 ~5, 全部 ssh+curl 模式 |

## 风险清单 (lock-risks.md 详)

| ID | 风险 | 严重 | 来源 |
|---|---|---|---|
| **B1** | 4090 /prompt 不经任何锁 | **严重** | main.py 0.0.0.0:9200 无 auth |
| **B2** | /tmp/avf* 不经 NAS lock | **严重** | 现有所有 Hermes 任务 |
| **R1** | release TOCTOU 跨 owner 删 | 严重 | 进程挂起 + lease 过期 |
| **R5** | NAS 卡死无限挂起 | 严重 | SMB mount 故障 |
| **R2** | release rmdir 失败留半状态 | 中 | SMB 中断 / 磁盘满 |
| **R3** | acquire 内 _safe_reconcile 误清 | 中 | 单主机 PID 短暂消失 |
| **R4** | token 在 argv + stdout | 低 | 设计选择 |
| **R6** | heartbeat 轮询拖 NAS | 低 | 高并发 acquire |

## 下一阶段建议 (供 P1)

1. **P1 优先**: 建立 4090 本地统一调度网关 (选项 A: HTTP 9201 + nssm, 复用 gpu_lock)
2. **P1 同时**: 修 R1 (release token 再校验) + R5 (readdir probe + 总超时)
3. **P2**: Windows 防火墙限制 9200 仅 127.0.0.1, T0006 启动器集成 gpu_lock
4. **P3**: 全网切换 9201, 退役 ssh+curl 直 POST
5. **P4+**: 多租户公平调度 + LoRA 训练 (等商业决策矩阵) + LTX-2.5 音频 (AVF-017)

## 状态

- **GPU 任务**: 0 (P0 只读, 没碰 4090)
- **E 盘 ComfyUI 0.34**: 不动
- **NAS lock 文件**: 不动
- **其他 writer**: 不重启
- **PR #25/#26**: 保留 Draft, 不动
- **新建 PR**: 0 (本 P0 不发 PR, 只在独立项目仓库)
- **新建 commit**: 仅 `docs/` 4 份 + 1 份 README
- **停止在 P0**, 不自主启动 P1-P3
