# 神策测试方案（TESTING.md）

> 状态：方案定稿 2026-09-05。前提：用户脱手（hands-off），开发与测试全程由本会话自治执行；最高要求——**过程不能让生产会话崩溃**。
> 核心手段：**双实例拓扑**——开发测试全部发生在独立的第二个 DSH 实例中，生产实例零接触。

## 0. 环境事实（方案依据）

- 生产实例：`/mnt/d/Software/WSLSoftware/Agents/deepseek-harness`，端口 3080，DSH_HOME=`/home/zrn/.dsh`，**落后上游 2917 提交**。
- `dsh web --port <port>` 支持端口覆盖（`packages/bundle/web-app/cordis.patch.yml`: `ctx.webStartup.port ?? 3080`）；`port 0` = 系统自动分配。
- DSH_HOME 环境变量隔离 sessions/storages/profiles（settings.yaml、会话落盘均在 DSH_HOME 下）。
- 已有 headless 通道：`dsh --profile headless "<任务>"` —— 一次任务、打印结果、退出（开发测试的主要自治驱动面）。
- 插件装载：profile 机制（`~/.dsh/profiles/<name>/` + `dsh plugin --profile <name> add <pkg>`）。

## 1. 双实例拓扑（崩溃隔离的物理基础）

```
生产实例（本会话所在，绝不改动）         开发测试实例（所有项目工作发生地）
  checkout: deepseek-harness               checkout: deepseek-harness-dev（最新版 clone）
  端口 3080                                端口 3081（或 --port 0 自动分配）
  DSH_HOME=/home/zrn/.dsh                  DSH_HOME=/home/zrn/.dsh-dev
  堆 4GB（daemon 管控）                    堆 2GB（独立 dev daemon 管控）
  本会话是它的居民 → 动它会害死我        它崩溃只影响它自己，我读日志查错
```

隔离清单（逐条验证）：
1. 独立 checkout（git clone origin/main 最新版）；
2. 独立 DSH_HOME（会话、settings、storages、profiles 全隔离）；
3. 独立端口与独立 daemon（各自的堆上限、日志、pid 文件）；
4. 共享的只有：宿主内存（12GB WSL 配额）与 DeepSeek API 额度。

## 2. 开发测试实例搭建（步骤固定，可脚本化）

1. `git clone git@github.com:deepseek-ai/deepseek-harness.git deepseek-harness-dev`（最新版）；
2. `pnpm install && pnpm build`（monorepo 一次构建）；
3. `export DSH_HOME=/home/zrn/.dsh-dev`，首次启动生成干净 home；
4. 写 dev 专用 daemon（拷贝金柝增强前的 dsh-daemon.sh 改三处：堆 2048MB、`--port 3081`、日志目录指向 dev checkout；DSH_HOME 注入环境）；
5. 冒烟：`dsh --profile headless "print 1"` 通过 + web 3081 可访问；
6. **行为差异核对（关键前置）**：验证期发现的三个问题在最新版是否已修复——
   a. goal-round-driver 重启后无条件 disarm；
   b. 子代理中断无父会话通知；
   c. 内存治理/预警线。
   核对结果决定金柝补丁 overlay 的最终清单（上游已修的 → 只做回归测试；未修的 → 落 patch）。

## 3. 自治测试回路（脱手协议）

- 一切测试通过**脚本**执行：`bash shence-junji/scripts/run-test.sh <level> <case>` 风格入口，产出结构化结果文件（exit code + JSON 报告）。
- 驱动方式：本会话用 bash 后台任务调 dev 实例 headless CLI；每次调用一个受控实验。
- 崩溃不传染：dev 进程死了 → 后台任务退出 → 我读 dev 日志/结果文件定责；生产实例不感知。
- 长测试保活：沿用验证期心跳模式（90s 必退出后台任务）。
- 里程碑（仅这些点需要用户）: 搭建完成 / 每仓验收通过 / 最终切换。中间过程零人工。

## 4. 测试金字塔

| 层 | 名称 | 环境 | 内容 | 执行频率 |
|---|---|---|---|---|
| L0 | 单元测试 | 各仓 vitest | 状态机转移表、账本幂等、hint 记账、经验格式解析、通道参数校验 | 每次提交 |
| L1 | 集成测试 | dev 实例 + **平台 mock + LLM 桩** | 插件在真实 DSH 树装载；平台 mock 回放 run 13250 归档的完整交互序列（score_events 全量已保存）；LLM 桩用固定响应 | 每次提交 |
| L2 | 端到端 | dev 实例 + **真实 LLM**（最便宜模型）+ 平台 mock | 迷你战役（3-5 工作项）跑通派单→求解→回传→账本全链 | 每仓验收 |
| L3 | 混沌测试 | dev 实例（随便杀） | 故意 kill dev web / 注入内存压力 → 验证金柝拉起+恢复、虎符账本回放、**生产实例全程无恙** | 每仓验收 |
| L4 | 真实验收 | 真实 tsecbench（用户 token） | 一次 6h run，预算内 ≥95% 且零人工干预；clean-room 扫描通过 | 最终一次 |

- L1 的平台 mock：yebushou@xiaochang 的适配器配套夹具，录制自三个 run 的真实交互（题单/start/submit/close 响应全有）。
- L2/L4 消耗真实 DeepSeek API 额度：默认用最便宜模型、每用例 token 上限、花费记入 dev 日志；**建议每日预算上限（待用户确认，默认 ¥X/天）**。
- L3 是"保证我不能崩"的直接验证：把 dev 实例杀掉 N 次，断言生产会话零影响。

## 5. 验收门（每仓，逐仓执行）

1. L0 全绿（`pnpm test`）；
2. L1 插件在 dev 实例挂载冒烟通过；
3. L2 迷你战役端到端通过；
4. L3 混沌 5 轮通过（含生产实例无感断言）；
5. 打 tag + 更新 shence-junji 验收记录。

顺序：集思（通道契约）→ 虎符（状态机）→ 金柝（混沌自测）→ 夜不收（经验格式 + 适配器 + 治理扫描）。

## 6. 最终迁移（全部通过后）

1. 生产 checkout `git pull` 到与 dev 相同版本并重建；
2. 生产 profile `dsh plugin add` 安装四个插件；
3. 生产实例冒烟（不碰既有会话），确认无回归后切工作流。

## 7. 待用户确认（2 项）

1. L2/L4 真实 LLM 测试的**每日费用上限**（无上限则仅 L1 桩测试，L2 缩减为最小用例）；
2. L4 真实验收是否要做（消耗一次真实 6h run 与额度；可延后到项目全部完成时）。
