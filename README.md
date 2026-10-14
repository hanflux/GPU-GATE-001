# GPU-GATE-001

GPU 调度现状审计 + 未来统一调度网关设计 (独立仓库, 不影响 hanflux/AI-Video-Factory 主线).

## 范围

- **P0** (本次): 只读审计 Mac mini / 4090 PC / NAS 之间的任务提交链路, 区分 writer, 分析锁风险, 评估 4090 本地网关可行性
- **P1+** (未来): 实施 4090 本地调度网关 (9201 + nssm + 集成 gpu_lock)

## 文档

- [`docs/topology.md`](docs/topology.md) — 真实运行拓扑
- [`docs/writers.md`](docs/writers.md) — GPU writer 清单 (A 真运行 / B 代码 / C 脚本 / D 未知)
- [`docs/lock-risks.md`](docs/lock-risks.md) — 锁机制风险 (R1-R6 + B1-B2)
- [`docs/proposal.md`](docs/proposal.md) — 4090 本地统一调度网关初步方案
- [`docs/P0_REPORT.md`](docs/P0_REPORT.md) — P0 审计总报告 (仓库链接/交付/风险/下一阶段)

## 约束

- 不修复、不重启、不修改 E 盘 ComfyUI 0.34
- 不合并任何已有 PR (AI-Video-Factory PR #25/#26 保留 Draft)
- 不提交 GPU 推理任务
- 不清理、重建或修改生产锁目录
- 不强制卸载 SMB、不杀生产进程
- 不把推断当作已验证事实

## 状态

- P0 完成: 2026-10-13
- 报告: `docs/P0_REPORT.md`
- 下一步: 等 Owner 授权进 P1 (建立 4090 本地网关原型)
