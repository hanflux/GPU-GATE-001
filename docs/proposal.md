# GPU-GATE-001 / T001 本地统一调度网关初步方案 (P0 提案)

**日期**: 2026-10-13
**任务**: 评估将权威调度入口迁移到 4090 PC 本地的可行性
**状态**: P0 提案, **不实现**, 等 Owner 授权进 P1

---

## 1. 为什么需要本地网关

### 当前痛点 (P0 审计发现)

1. **E 盘 9200 ComfyUI 不经任何锁** — 任何 ssh 4090 的人直接 POST
2. **Mac mini /tmp/avf* 临时脚本** 都 ssh + curl, **不调 gpu_lock.py**
3. **gpu_lock.py 自 commit 218ce02 (Issue #4) 以来, AVF-016/017 任务一个都没用过** (P0 实测 `queue_len: 33` 全是 `completed` 历史, 不含 AVF-016)
4. **T0006 (Qwen 8198)** 独立 side instance, **不感知 NAS lock**
5. **T0007 / AVF-MONA-001** 多个项目共享 4090 GPU, 互不协调

### 现有 gateway 候选 (Mac mini 端)

- `/Users/oc/.hermes/avf/avf_mona_bridge.py` (已实现 5 能力: status/preflight/submit/job-status/collect) — **Draft, 不使用** (Owner PR #25/#26)

## 2. 4090 PC 本地做网关的 5 大优势

| 优势 | 证据 |
|---|---|
| **本地回环快** | HTTP localhost < 1ms, 不用走 LAN |
| **少 ssh 跳数** | 现有 ssh+curl 链路 2 跳 (Mac→4090→9200), 网关 1 跳 (Mac→4090-Gateway) |
| **完整进程可见性** | nvidia-smi / Windows process / 9200 listener 都在同一 host, 二次验证更准 |
| **Windows-native 工具链** | nssm 服务 / Task Scheduler / 防火墙 / Windows ACL / 事件日志, 适合做权威调度 |
| **GPU 物理机即调度机** | 锁状态 + 显存 + 进程 PID 全部本地, 不会跨主机假阴性 |

## 3. 网关候选设计 (3 选项)

### 选项 A: Windows HTTP 服务 (Flask/FastAPI + nssm)

```
Mac mini POST http://192.168.50.202:9201/gateway/submit
  → 4090 gateway service (nssm)
  → 内部: gpu_lock acquire → preflight → POST 127.0.0.1:9200/prompt
  → 返回: prompt_id, job_id
```

**优势**:
- 唯一暴露端口 9201 (9200 仍可直连, 但建议防火墙禁外网访问)
- nssm 自动重启, watchdog 容易集成
- 完整 request log 留档

**劣势**:
- Python 进程常驻 (~200 MB RAM)
- 依赖 Windows Python 3.13 (E 盘 0.34 用的 embed)
- 需要 Windows 防火墙配置 (开 9201 + 关 9200 外网)

### 选项 B: ComfyUI API proxy (改 main.py)

直接在 ComfyUI 0.34 启动时注入 API hook, 拦截 /prompt 调用走锁.

**优势**: 不增加新进程, 复用 ComfyUI 进程

**劣势**:
- 改 ComfyUI 核心 = **违反 Owner 严令 "不修改 E 盘 ComfyUI 0.34"**
- ComfyUI 升级会丢修改

### 选项 C: 现有 9200 + 集成 gpu_lock (扩展 main.py)

加 wrapper 启动脚本: `python main.py --gpu-gateway-port 9201` 或在 main.py 启动时调 `gpu_lock.acquire()`.

**优势**: 启动时强制锁

**劣势**:
- 仍改 E 盘 0.34 (违反严令)
- 单点失败: 9200 挂 = 网关挂

## 4. 推荐: 选项 A (HTTP 服务 + nssm)

理由: 不改 E 盘 0.34, 独立进程可独立审计, 用现有 nssm 工具链.

### 4.1 架构

```
┌──────────┐     HTTP/JSON     ┌──────────────────┐
│ Mac mini │ ─────────────────→ │ 4090 gateway     │
│  + others│                   │ (nssm 9201)      │
└──────────┘                   │  - authn         │
                               │  - gpu_lock      │
                               │  - preflight     │
                               │  - log           │
                               └────────┬─────────┘
                                        │ HTTP/JSON
                                        ▼
                               ┌──────────────────┐
                               │ ComfyUI 9200     │
                               │ (existing)       │
                               └──────────────────┘
```

### 4.2 关键能力 (继承自 avf_mona_bridge.py)

1. `status` — GPU/queue/lock 实时查询
2. `preflight` — 资源 + workflow 静态检查 + dry-run
3. `submit` — 加锁 + 转发到 9200 + dmon 启动
4. `job-status` — prompt_id 跟踪
5. `collect` — 样片 + 历史路径
6. `cancel` — 定向 /interrupt by prompt_id

### 4.3 安全门 (不放松)

- 12 GiB RAM 提交门
- 23,564 MiB VRAM 运行限 (23 GiB)
- 1s dmon 真实采样
- 已方 UUID /interrupt
- history 终态
- AVF-016 余量过紧不批同配置 (Owner 严令)
- 4090 RAM 44.58 GiB / VRAM 22.26 GiB free (P0 实测)

### 4.4 不绕过 GPU 锁 (强制)

- 所有 /prompt 必须先 `gpu_lock.py --nas-root <real_nas> acquire <qid>`
- 9200 仍 LISTEN 0.0.0.0 (不改), 但建议在 Windows 防火墙**限制 9200 源 IP** (仅 127.0.0.1 允许)
- 这样: 只有 4090 本地进程能直 POST, 外部必须走 9201 网关

### 4.5 跨主机协调 (4 个 writer 共享 GPU)

- T0006 side instance (8198): 注册到 gpu_lock (端口 8198 作为 pid_hint)
- T0007 staging: 启动前必须先 acquire
- AVF-MONA-001 (隔离 0.39): acquire 端口 0 (CPU)
- AVF-016 生产 (9200): acquire 端口 9200

**所有 acquire 的 port_hint 写实际 ComfyUI 监听端口**, `_secondary_verify` 才能验证正确.

## 5. 可行性评估 (✅ 满足 P0 列出的所有项)

| Owner 要求 | 选项 A 满足 |
|---|---|
| 不修改 E 盘 ComfyUI 0.34 | ✅ 独立 9201 端口, 9200 不动 |
| 不重启生产 | ✅ 新增 nssm 服务, 9200 继续跑 |
| 不强行新锁协议 | ✅ 复用 `scripts/gpu_lock.py` |
| 不破坏其他项目 writer | ✅ 9201 是新加, 旧路径全保留 |
| PR #25/#26 保留 Draft | ✅ 选项 A 是新实现, 不基于 PR #25/#26 |
| 4 个 writer 协调 | ✅ 全部走 gpu_lock |
| 单 job_id 唯一 | ✅ 网关强制 job_id |
| 不能绕 NAS lock 直 POST | ✅ 9200 防火墙限制 + 网关强制 acquire |

## 6. 实施步骤 (供 P1-P3, 本 P0 不做)

### P1: 最小网关原型 (1 周)
- [ ] 在 4090 写 `C:\gpu-gateway\gateway.py` (Flask, port 9201)
- [ ] 集成 `infra/gpu_lock/client.py` (复制或 pip install)
- [ ] nssm 注册 `GpuGateway` 服务, StartType=Auto
- [ ] 6 个能力 (status/preflight/submit/job-status/collect/cancel)
- [ ] 本地 temp dir 测 acquire/release/cancel 完整流

### P2: 集成 + 防火墙 (1 周)
- [ ] Windows 防火墙: 9200 仅 127.0.0.1
- [ ] 9201 接受 Mac mini IP 段 (192.168.50.x)
- [ ] T0006 启动器集成: setup 时 acquire 8198, stop 时 release
- [ ] AVF-016 现有 /tmp/avf* 脚本迁移到 9201

### P3: 全网切换 + 退役旧入口 (1 周)
- [ ] Mac mini 默认走 9201
- [ ] 9200 防火墙彻底锁
- [ ] 旧 ssh+curl 路径废弃
- [ ] Mona 隔离 0.39 也走 9201
- [ ] dashboard / history 终态报告

## 7. 风险 (P1 进前需 Owner 确认)

1. **修改 Windows 防火墙** — Owner 严令 "不擅自", 需授权
2. **nssm 新服务** — 跟现有 ComfyUI nssm 并存, 资源消耗 ~200 MB RAM
3. **4090 PC 维护窗口** — 网关需 reload 时的 1-2s downtime
4. **Mona 隔离 0.39 改造** — controller 内部集成网关, 不破坏 CPU admission

## 8. 不在本方案内 (留给后续 P4+)

- GPU 多租户公平调度 (FIFO priority 已支持, weighted fair queue 待 P4)
- 远程训练任务 (LoRA/QLoRA) — Owner 严令 "不启动 A2 LoRA 训练", 等商业决策矩阵
- 商业训练矩阵 (FLUX.1-dev vs Schnell vs BFL Pro) — 零 GPU, 另列
- LTX-2.5 原生音频 (AVF-017) — 已在 AVF-017 路径, 不冲突

## 9. 关键参考

- 已有 AVF-MONA-001 bridge 草案: `hanflux/AI-Video-Factory` PR #25/#26 (Draft)
- 已有 gpu_lock: `hanflux/AI-Video-Factory` `scripts/gpu_lock.py` + `infra/gpu_lock/`
- T0006 side instance contract: `E:\T0006-bootstrap\t0006_*_side_instance.ps1` (12 KB)
- 4090 PC 资源 (P0 实测): C 295 GB / E 38 GB / VRAM 22.26 GiB free / 9200 LISTEN

