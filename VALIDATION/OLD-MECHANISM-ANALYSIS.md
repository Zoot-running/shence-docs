# 老架构打题机制分析（源自 run 11820 时代主调度会话日志，13250 继承此机制）

> 2026-09-07 复盘。来源：生产实例会话日志
> `/home/zrn/.dsh/sessions/--mnt-d-...-Offensive-Tsecbench--/session-52aedf44.../session.jsonl.zstd`
> （48MB，841 turn / 5664 step / 5332 tool-call / **618 subagent 派发** / 3185 inbox-splice /
> 24 goal/change / 41 compaction-prune / 15 llm-retry）。
> 本档案只提取**机制**；题解与 flag 值不在此。

## 老机制全景（一句话）

**主 agent 出思路、按难度编排、continuable 子代理探索执行、结果回流入账、
经验落本地文档库循环复用、goal 轮驱动 + 上下文压缩长跑。**

## 逐条机制（全部有会话日志证据）

1. **思路是主 agent 出的**：每个 subagent prompt 的开头就是主 agent 的攻防分析——
   `【目标】题 c-01：1Panel 面板，用给定凭据登录后找漏洞（后台 RCE/文件读取/命令执行），
   最终读 /challenge/flag.txt`。子代理拿到的是"方向 + 预期路径"，不是裸题面。
2. **难度编排**：
   - 先 easy（100 分）挑最好下手的；**用 easy 题校准 flag 放置习惯**（"开 c-03 来校准
     flag 放置习惯"）；
   - 3 容器槽位满跑，`workflow` 并行流水线（"3 个子代理同时打 c-04/c-05/d-04"）；
   - 高分链条单独排（"b-01 门户渗透链 1200 分等级 S"）；
   - **hint 经济学**：按分值决策（"官方 hint 扣 10%，但 500 分题值得"）。
3. **子代理 = continuable 模式**：descriptor 证据 `mode: continuable, deepseek-v4-flash,
   label "b-02 angle3 lateral"`——子代理可续聊（保留原生上下文），带语义标签
   （题号+角度+方向）。
4. **状态快报闭环**：每解一题"✅ x 解出并入账 — 调度循环完成 — 状态快报：得分
   5890→6160（累计 25 题）"，随即 submit→close→开下一题。
5. **经验文件库（本地）**：主 agent 亲手维护并喂给子代理：
   `recon/ONBOARDING.md`（接入手册）、`SOLUTIONS.md`（解法速查，"有同系列 f1-0x 解法"）、
   `SHARED-STATE.md`（并行协作状态）。→ 这是"失败教训带回、成功积累经验"的载体。
6. **托管规则自查**：主 agent 自己从平台前端代码 + 用户协议里发现了红线——
   "不可内置针对题目的历史答题记忆…作为作弊处置"（后来 v2 的 clean-room 原则之源）。
7. **长跑支撑**：goal 轮驱动（24 次 goal/change）+ 上下文压缩（41 次 prune + 5 次
   summary）+ 会话间用户指令拼接（interactive 协同）。

## v2 与老机制的差距（改进清单）

| # | 老机制有、v2 缺 | 改进 |
|---|---|---|
| 1 | 主 agent 深度分析写进每条 prompt（方向+预期路径） | 技能：enqueue 前先写"我的分析"段 |
| 2 | 难度编排细化（easy 校准 flag 习惯 / 高分链 / hint 按分值） | 技能：编排纪律写细 |
| 3 | **continuable 子代理**（续聊保留上下文 = 磨题连续性正解） | jisi 后台模式落地（backlog 首位） |
| 4 | 每题入账状态快报闭环 | 技能：快报纪律 |
| 5 | 经验文件库喂子代理（本地 SOLUTIONS.md 等，托管红线内仅本地） | 主 agent 维护 local/ 题解库并在卡题时引用 |
| 6 | workflow 多代理编排（阶段性并行任务） | 主 agent 可用 DSH workflow（与虎符互补） |
| 7 | 上下文压缩长跑 | DSH compaction 已具备，观察 v2 长会话是否触发 |

## 立即执行（本档案落盘时同步改技能）

- 校场技能补 1/2/4/5 条（详见 git 提交）。
