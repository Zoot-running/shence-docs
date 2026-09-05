# L2 端到端合练验证报告：虎符战役 × 集思通道（2026-09-05）

## 目标

在 dev 实例验证一场多工作项战役完整跑在 `虎符 → 集思` 通道上，无人值守，含：

- 优先级调度（tier 升序 / score 降序）；
- 并发槽位（2）与按次模型（kimi-k2.6 × 3 + glm-4.5-air × 1 混跑）；
- 失败自动 requeue 重试（最多 2 轮，seed+1）；
- 中途序列化 → restore 重放一致性（replayOk）。

## 结果（最终）

```
summary={"total":4,"open":0,"done":4,"failed":0,"blocked":0,"superseded":0,"queued":0}
replayOk=true
details=w1#1: done | w2#1: done | w3#1: done | w4#1: done
```

4/4 完成、零失败、零滞留、重放一致。

## 过程中发现并修复的两个真实缺陷

### 1. 虎符 e2e 探针调度循环只跑一波（hufu 8ba38f6）

初跑结果 `w1/w4 queued`：派单循环只在第一波执行，后续完成释放槽位后不再补派。
修复：循环改为「槽位空闲即派排队项；失败项 requeue 重试（≤2 轮）」。
复测后无 queued 残留。

### 2. 集思按次模型未解析 provider（jisi 06d23ca）

初跑结果 `w4#3 failed`（glm 连续 3 次失败），jisi_probe 复现：
`delegate glm`（显式 provider）成功、`fanout glm`（仅 model）失败。
根因：只有 model 没有 provider 的派单被送进默认 provider 路由，glm-4.5-air
被误送到 kimi 路由。注意 `ctx.subagents.start` 的第一参数是子代理注册表
provider（固定默认），LLM 路由只能经 `agentOptions.provider` 覆盖。

修复：`resolveProviderOfModel` 入纯逻辑模块 channel.ts（新增 2 个 L0 用例，
jisi L0 13→15），宿主绑定在缺 provider 时按 model 反查宿主 LLM 路由并写入
`agentOptions.provider`。

复测：`fanout glm-4.5-air → completed: PONG`，e2e 4/4 done（w4 一轮通过）。

## 覆盖的验收门

- 虎符：调度循环、并发槽位、优先级、失败重试、序列化/恢复（L0 20 + 本 e2e）。
- 集思：按次模型路由、多模型 fanout、模型清单（L0 15 + 本 e2e）。
- 金柝：本场不参与（告警/恢复由 L3 混沌测试覆盖）。

## 环境

- dev 实例：dsh-0.1.3-alpha.1，headless profile，Kimi k2.6 + 智谱 glm-4.5-air 真调。
- 生产实例零接触。
