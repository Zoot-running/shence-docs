# ADR-001 虎符调度核心契约（状态机/账本/战役）

- 状态：已接受（2026-09-05）
- 仓库：shence-hufu
- 关联：ADR-002 集思通道（派单执行面）

## 背景

验证期（三个 run）的调度协议 v7.3 是对话内口头约定：槽位纪律、stall 重派、重复报告去重、心跳保活、崩溃恢复全靠主会话自觉执行，无法复用、无法测试。虎符把它固化为可独立验证的纯逻辑 + 少量宿主端口。

## 决策

### 1. 工作项状态机（src/state-machine.ts）
```
queued --dispatch--> dispatched --help--> help
dispatched/help --progress--> 不变
dispatched/help --stall--> stalled
stalled/dispatched/help --supersede--> superseded（终态，旧 seed）
dispatched/help/stalled --terminal--> done|failed|blocked
superseded --terminal(旧seed迟到)--> superseded（吸收）
stalled/failed/blocked/superseded --requeue--> queued（seed+1）
```
- 非法转移抛 InvalidTransitionError。
- isActive（占用槽位）= dispatched/help/stalled；isTerminal 含 superseded。

### 2. 工作项账本（src/ledger.ts）
- 追加日志：dispatch/progress/help/stall/supersede/terminal/requeue（带 seed 与时间戳）。
- **去重规则**：事件折叠时旧 seed 事件一律吸收——superseded 后的迟到 terminal 不改变新 seed 状态。
- 恢复 = 账本重放（HufuLedger.restore / HufuCampaign.restore）。

### 3. 战役编排（src/campaign.ts）
- 队列排序：priority.tier 升序 → score 降序（验证期的难度升序→分值降序）。
- 槽位：N = 用户显式上限 ?? 自动推导（自动推导是宿主绑定层的职责，核心只接受 N）。
- stall 检测：dispatched/help 超过 stallAfterMs 无 progress → stall →（首次）supersede+requeue（seed+1）+ 中断旧 seed。
- 心跳：heartbeatMs 必退出（宿主绑定层以 harness 后台任务实现，结算即唤醒）。
- 完成判定：全部终态或预算耗尽。
- 注入端口：now()/dispatch/interrupt——宿主绑定（集思通道或 DSH 原生 subagent）在插件层实现。

## 后果

- 调度逻辑可单测（L0 20 项全绿）、可重放（崩溃恢复）。
- hint 经济学、平台 API、交卷口径均不在此——它们属 shence-yebushou@ctf（平台特化）。
- 派单执行面经软依赖走集思通道；无集思时回退 DSH 原生 subagent。

## 替代方案

- A. 把调度器做成独立进程（不寄生 DSH）——与"DSH 插件"定位冲突，且失去 send_message/会话恢复等原生能力，否决；
- B. 状态机并入平台适配器——平台特化混入通用调度，违背正交原则，否决。
