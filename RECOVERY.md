# 会话休眠事故复盘与恢复手册（2026-08-27 00:37 OOM）

## 事故链（run 12941 收尾期，证据在 DSH logs/）

1. **2026-08-27 00:37:09** `logs/daemon.log`：`web exited code=137`（SIGKILL）。
   **修正（mem.log 逐条核对后）**：死时 RSS 仅 ~3.2GB（00:36:39 读数 3245388 KB），远未到 6GB 堆上限，daemon 的 4.5GB 预警（4.5GB 以上才打 WARN）**从未在事故窗口触发**；web.log 里那些 5.1-5.6GB 的 WARN 是更早时段（08-26 06:37-07:13）的旧进程记录。code=137 是 Linux OOM killer（WSL 宿主内存压力）对最大进程的直接击杀，不是 Node 堆耗尽。
   内存轨迹：23:56-23:59 RSS 2.6→3.3GB（重复报告风暴 + goal 轮连发时段），00:30-00:36 在 2.9-3.4GB 振荡，峰值 3.47GB（00:31:39）后被杀。盘上会话日志仅 312MB、session_projcache 2MB——**RAM 工作集 ≈ 落盘数据的 10 倍**，大头是主会话活上下文 + 3 个在跑代理的 LLM 上下文/工具输出 + 事件深拷贝暂存，不是"驻留的子会话"。
2. daemon 3 秒后重启 web（pid 12995），**会话落盘完好**（重启 2 分钟后 RSS 已回到 620MB，冷读机制工作正常）。
3. 重启后三个在途求解代理（xben-092/101/081c）的运行时被杀：最终轮从未执行完 → **没有 finish 通知事件**。它们现在是 `ready`（可恢复、非终态）。
4. goal-round-driver 源码（packages/goal/goal-round-driver/src/index.ts L418-421）：**驱动加载既有 agent 时一律 disarm**（"never inherits hidden automatic authority from an earlier producer instance"）→ 重启后 goal activation=disarmed → 自动轮次（27-40）不再触发。
5. 结论：goal 轮不是定时器，是**事件驱动**（agent idle + goal armed 才续轮）。子代理 finish 通知是事件，但子代理没执行完就没有事件；goal 被 disarm 连兜底也没了。nohup watcher 不是 harness 托管任务，进程死了无任何通知。

## 恢复协议（主会话恢复后第一组动作，按序）

1. `curl /api/v1/runs/<id>/status`（Bearer）→ 拿权威分/状态；run 若已 timeout 直接进入归档。
2. `list_agents` → `ready` 的求解代理用 `send_message` 恢复（会话持久化支持续跑）——前提 run 未结束。
3. `get_goal` → 若 `phase=active` 且 `activation=disarmed` 且目标未完 → `update_goal(resume)` 重新武装自动轮。
4. 新 run 的 watcher **必须用 harness 后台任务**（bash run_in_background=true），禁止 nohup：
   - 心跳模式：循环 90s 必退出并打印状态行 → 每次 settle 都是一条唤醒事件，主会话靠它保持清醒；
   - 事件模式（可选叠加）：命中 FLAG/HINT_REQ 即退出。

## 子会话归档现状（代码核查结论，2026-08-28）

- **归档机制已存在且工作正常**，无需新造：
  1. session 持久化：事件流 write-behind 落盘（300+ 会话 = 312MB，`~/.dsh/sessions/`），restart 后 2 分钟冷读恢复全部状态；
  2. `session/disposed` 时释放内存：core/session 源码 "fiber unload tears the session + agent down"，子代理跑完（ready 态）即仅存于存储、用 `send_message` 时冷读调入；
  3. session-projection-cache（`session_projcache.json`，2MB）——投影检查点落盘 + cold-read ladder（缓存行 → persistence readFrom 尾 → registry restore），恢复不靠全量读盘。
- **本次内存问题是 live 工作集，不是"驻留的子会话"**：死时 RAM 3.2GB ≈ 落盘 10 倍；大头 = 主会话活上下文 + 3 个在跑代理的 LLM 上下文与工具输出 + 事件快照深拷贝。子会话侧可做的只是水位控制（见下）。

## 根因级修复建议（DSH 层面，待实施）

1. **goal-round-driver 重启恢复**：加载既有 agent 时对 `phase=active` 的 goal 恢复 armed（或发 `goal/driver-loaded` 事件重新武装），而不是无条件 disarm。当前行为直接掐断长任务自动续跑。
2. **内存/OOM 治理**（按优先级）：
   a. WSL 层：`.wslconfig` 提高 memory 上限（或给 dsh 专用 distro 配额）——本次 SIGKILL 来自宿主 OOM killer，Node 本身没爆；
   b. V8 层：`--max-old-space-size` 从 6144 降到 ~4096，逼 V8 提前 GC（RSS 上限与 WSL 配额错开）；
   c. daemon 层：预警线 4.5GB 降到 3.0GB，且命中即优雅重启（先 SIGTERM 让 flush 走完）——现在只打印 WARN 不动作；
   d. idle 子会话水位：idle 超时（如 10min）强制 dispose 进冷存，live 会话数上限 + LRU（当前 idle agent 保持 loaded，无 TTL）。
3. **在途子代理崩溃安全**：web 重启后，非终态子会话应向父会话投递 `child interrupted` 通知（今天静默变 `ready`，零事件）——这才是"子会话事件能唤醒"缺失的半环。
4. **会话外 watchdog**（不依赖 web 进程）：daemon 加 tick 或 systemd timer，轮询 progress.log；命中 FLAG 直接 curl 平台 submit/close/start（无需 LLM），把"待派单"写成 ticket 文件；主会话醒来读 ticket 批量派单。

## 行为准则变更

- 收官期不再轻信"没有事件=一切正常"：每隔 N 分钟心跳必须回来；心跳断了 = 进程死了，按本协议第 1 步立刻恢复。
- 时间敏感的 run（6h 窗口）期间，主会话绝不以"等待事件"为唯一收尾状态：每轮都挂一个新心跳后台任务。
