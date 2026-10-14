# GPU-GATE-001 / T001 GPU 锁机制风险分析 (P0 审计)

**日期**: 2026-10-13
**任务**: 分析 `gpu_lock.py` + `infra/gpu_lock/client.py` 的 acquire / release / reconcile / lease, 找出可绕过 / 不可靠点

---

## 1. gpu_lock 设计概要

代码位置: `hanflux/AI-Video-Factory` 仓库 `infra/gpu_lock/client.py` (22,592 B, 600 行) + `scripts/gpu_lock.py` (CLI 适配, 6,131 B)

### 锁协议

- **原子获取**: `os.makedirs(<lock_dir>)` 抛 `FileExistsError` (POSIX + SMB mount 都支持)
- **token**: `secrets.token_urlsafe(32)` 32 字符, 写入 `lock_dir/token.json`
- **心跳**: 写 `heartbeat/<lock_id>.json`, 默认 30s
- **lease**: 300s, 超时视为 stale
- **二次验证** (reconcile): PID alive + port listening + nvidia-smi compute-apps

## 2. acquire() 流程 (client.py:341-473, 108 行)

1. NAS 可达性检查 (fail-closed) — 不可达就 raise GpuLockError
2. `_ensure_dirs()` 创建目录
3. 循环:
   - `_load_queue_entry(queue_entry_id)` — 验证 queue entry 存在
   - `_atomic_acquire_dir(self.lock_dir)` — 原子 mkdir
   - 成功: 写 lock.json + token.json + heartbeat + 更新 queue entry + history event
   - **失败**: 读 lock.json → **如果 holder.heartbeat_at 超 stale_threshold** → `_safe_reconcile(holder)` (自动清)
   - 否则: 等待 deadline → raise GpuLockBusy

## 3. release() 流程 (client.py:475-547, 59 行)

```python
def release(self, lock_token, *, status="completed", reason=None):
    if not _nas_reachable: raise fail-closed
    rec = self._load_lock()  # 读最新 lock.json
    if rec is None: return False
    if rec.lock_token != lock_token: raise GpuLockNotOwner  # 校验 token
    # 1. 更新 queue entry
    self._atomic_write_json(queue_dir / f"{qid}.json", entry.to_dict())
    # 2. 写 release snapshot
    self._atomic_write_json(snapshots_dir / f"released-{int(now)}.json", rec.to_dict())
    # 3. unlink lock.json + token.json
    for p in (self.lock_json, self.token_json):
        try: p.unlink()
        except FileNotFoundError: pass
    # 4. rmdir lock_dir
    try: self.lock_dir.rmdir()
    except (FileNotFoundError, OSError): pass
    # 5. unlink heartbeat
    try: hb.unlink()
    except FileNotFoundError: pass
    # 6. 写 history event
    self._record_history("release", ...)
```

## 4. reconcile() 流程 (client.py:485-547, 63 行)

```python
def reconcile(self, *, force=False):
    if not _nas_reachable: return {...nas_reachable: False}
    rec = self._load_lock()
    if rec is None: return {lock: None}
    is_stale = self._now() - rec.heartbeat_at > self.stale_threshold_sec
    secondary_verify = self._secondary_verify(rec)
    # 只有 stale + force + all secondary_verify True → 自动清
    if is_stale and (force or all(secondary_verify.values())):
        if all(secondary_verify.values()):
            # 标记 STALE_NEEDS_REVIEW, 更新 queue, 写 history, NOTIFY feishu
            ...
    return out
```

## 5. _secondary_verify() (client.py:549-580, 31 行)

```python
def _secondary_verify(self, rec):
    return {
        "pid_alive": _pid_alive(rec.pid_hint),  # os.kill(pid, 0) on local host
        "port_listening": _port_listening(rec.executor_host or "127.0.0.1", rec.port_hint)
                          if rec.port_hint else False,
        "gpu_process_present": <nvidia-smi --query-compute-apps=pid... match rec.pid_hint>
    }
```

## 6. 已知风险 (按严重程度)

### R1: release 旧 A 校验后 → B 取得新锁 → 旧 A 继续删 B 的 TOCTOU ⚠ 严重

**代码** (release L495-528):
```python
rec = self._load_lock()       # L495 读
if rec.lock_token != lock_token: raise  # L499 校验
# ... 跳过 unlink, 写 snapshot
for p in (self.lock_json, self.token_json):  # L518
    try: p.unlink()  # 旧 A 继续 unlink
```

**场景**:
- t0: A 拿到锁, token_a
- t1: A 进入 release(), `_load_lock()` 读到 lock.json (含 token_a), token 校验通过
- t2: A 在 L518 之前被 OS 挂起 (例如 GC pause, IO wait)
- t3: 300s 后 lease 过期, B 拿新锁, lock_dir 重生, lock.json (含 token_b) + token.json (含 token_b)
- t4: A 恢复执行, **继续 unlink(self.lock_json)** — **删除的是 B 的 lock.json!**
- t5: A 继续 rmdir(lock_dir) — **删除 B 的 lock_dir!**
- t6: B 状态丢失, 但 B 进程仍在跑 GPU, 无人知晓

**修复手段** (P0 报告不修):
- 在 unlink 之前再 `_load_lock()` 一次, 验证 token 还匹配
- 或用文件锁 (flock) 包裹 unlink + rmdir
- 或在 lock_dir 内写 `release_intent` 标记, B acquire 时检查

### R2: release rmdir 失败 → 半清理状态 ⚠ 中

**代码** (release L521-524):
```python
try: self.lock_dir.rmdir()
except (FileNotFoundError, OSError): pass
```

**场景**:
- t0: A unlink lock.json + token.json 成功
- t1: A rmdir 失败 (例如目录非空, SMB 网络中断, OS 缓存滞后)
- t2: A 继续 unlink heartbeat 成功
- t3: lock_dir 还在, 但 lock.json + token.json 已删
- t4: B acquire 调用 `_atomic_acquire_dir(lock_dir)` → **FileExistsError** (目录还在) → False
- t5: B 走 _safe_reconcile 路径, 验证 holder 失败, **不清理**
- t6: B 等到 deadline → raise GpuLockBusy
- t7: 锁状态完全卡死, 只有 `reconcile(force=True)` 才能清

**修复手段** (P0 报告不修):
- rmdir 失败时回滚 unlink (重写 lock.json + token.json)
- 或用 `shutil.rmtree(ignore_errors=True)` 强制清

### R3: acquire() 内 _safe_reconcile() 误清活锁 ⚠ 中

**代码** (acquire L447-456):
```python
current = self._read_json(self.lock_json)
if current:
    try:
        holder = LockRecord.from_dict(current)
        if self._now() - holder.heartbeat_at > self.stale_threshold_sec:
            if self._safe_reconcile(holder):  # 无 force 也清
                self._record_history("stale_cleared", ...)
                continue
    except Exception: pass
```

**问题**: `_safe_reconcile()` 在 `all(secondary_verify)==True` 时清锁. **独立 probe 确认**: 旧实现 (commit 218ce02 之前) 曾误清活锁 (Owner 评论 6151264721). **当前实现没改** (PR #27 也没碰这个).

**触发场景**:
- PID 短暂消失 (重启 service, 进程被 kill 但 lock 还没清) + heartbeat 过期 + 二次验证全 True → 清
- 反例: Mac mini 验证 4090 PID 永远返回 false (跨主机), 所以**跨主机 acquire 永远走不到 stale_clear 分支** — 实际是**安全 side effect**.

**单主机风险**: 同主机多进程 (Mona 隔离 0.39) acquire() 时, 若对方 PID 短暂消失 + 二次验证通过, 会自动清.

### R4: token 在 argv 和 stdout JSON 中 ⚠ 低

**代码** (scripts/gpu_lock.py:144 `release` / 151 `heartbeat`):
- `--token <lock_token>` 是 CLI 参数, 任何 logger 捕获 argv 都看得到
- acquire 返回的 stdout JSON 完整含 `lock_token` 字段

**修复状态** (PR #27 Draft):
- 删除了 `print(f"LOCK_TOKEN={...}", file=sys.stderr)` 一处重复 stderr 输出
- **但 argv `--token` 仍含 token, stdout JSON 仍含 token** — 仅是**减少一处**, 不能算"不再泄漏"

### R5: NAS SMB mount 卡死时无防护 ⚠ 严重

**代码** (acquire/release L341+):
```python
if not _nas_reachable(self.nas_root): raise GpuLockError("NAS unreachable - fail-closed")
```

**实测** (本次审计):
- 22:53 之前 NAS readdir 阻塞 15-30 min, 实际证明 SMB mount 可卡死
- 卡死期间任何 enqueue / status / acquire / release 都 hang (因为依赖 readdir)
- 只能 `kill -9` 进程退出

**当前无防护**: acquire/release 调用前不预先探测 readdir, 直接进入文件操作, 卡死

### R6: heartbeat 默认 30s 间隔 + 5s 死循环 retry ⚠ 低

**代码** (acquire L467-471):
```python
if self._now() < deadline:
    time.sleep(min(2.0, max(0.5, deadline - self._now())))
    continue
```

**场景**: A 持锁, B 等到 deadline 前 sleep 0.5-2s, 然后再 retry. **这本身就是轮询模式**, 没用信号量 — 大量并发 acquire 会卡 NAS.

## 7. 可绕过性 (最严重, 但不在 gpu_lock 范围内)

### B1: 4090 ComfyUI HTTP /prompt 不经任何锁 ⚠ 严重

**证据**: `main.py --listen 0.0.0.0 --port 9200` + 无 auth + 无 rate-limit.

任何能 ssh 192.168.50.202 的人, 直接 `curl -X POST http://127.0.0.1:9200/prompt -d @wf.json` 就能跑 GPU 任务, **完全不经过 NAS lock**.

**历史实测**: 整个 AVF-016 历史 (8-10 月) 都是用这个路径直接 POST. gpu_lock 实际上**没有被使用过** (P0 状态: `lock: null`, `queue_len: 33` 全是历史 `completed` 状态, 不含 AVF-016 任何 entry).

### B2: Mac mini /tmp/avf* 临时脚本不经锁 ⚠ 严重

`/tmp/avf*workflow*.py` (30+ 个) 都直接 `scp` + `ssh + curl`, **不调 `gpu_lock.py` acquire**.

## 8. 风险清单 (一句话总结)

| ID | 风险 | 严重 | 触发条件 | 修复 |
|---|---|---|---|---|
| R1 | release TOCTOU (跨 owner 删) | 严重 | 进程挂起 + lease 过期 + B 拿新锁 | release 内再 load 验证 |
| R2 | release rmdir 失败留半状态 | 中 | SMB 中断 / 磁盘满 | 回滚 unlink 或 rmtree |
| R3 | acquire 内 _safe_reconcile 误清 | 中 | 单主机 PID 短暂消失 + 二次验证通过 | 加 `--no-auto-reconcile` flag |
| R4 | token 在 argv + stdout | 低 | 任何 logger 捕获 | 不修, 设计选择 |
| R5 | NAS 卡死时无限挂起 | 严重 | SMB mount 故障 | 加 readdir probe + 总超时 |
| R6 | heartbeat 轮询拖 NAS | 低 | 高并发 acquire | 用条件变量 / event |
| B1 | 4090 /prompt 不经锁 | 严重 | ssh 4090 + curl | **必须建立网关** |
| B2 | /tmp/avf* 临时脚本不经锁 | 严重 | Hermes 现有所有任务 | **必须统一网关** |

## 9. 修复优先级建议 (供 P1 参考, 本 P0 不修)

1. **B1 + B2** (严重 + 根本): 建立 4090 本地统一调度网关, 强制所有 /prompt 走网关
2. **R5** (严重): 加 readdir probe + 总超时
3. **R1** (严重): release 内 token 再校验
4. **R2** (中): release rmdir 失败回滚
5. **R3** (中): 加 `--no-auto-reconcile` flag
6. **R4 + R6** (低): 当前可接受

