# ADR-002 集思派单通道最小契约

- 状态：已接受（2026-09-05）
- 仓库：shence-jisi
- 关联：shence-hufu（软依赖本契约）；DSH 原生 subagent 工具（回退路径）

## 背景

验证期（三个 run）暴露两个需求缺口：
1. 派单绑死单一模型——困难题想换更强模型/多模型并行没有抓手（DSH 原生 subagent 工具的 `agentOptions.model` 是配置级默认，不是按次指定）；
2. 多模型并出的思路需要原样交回主 agent 综合（不教 AI 做事）。

## 决策

集思提供**通道原语**，全部围绕"按次指定模型"展开：

### 1. delegate —— 派活
```
delegate(work, opts?) → ChildRef
  work: {
    prompt: string        # 自包含工作描述（唯一必填）
    workdir?: string      # 可选工作目录
    tools?: string[]      # 可选工具过滤（缺省继承）
  }
  opts: {
    model?: string        # 按次指定模型（缺省=默认模型）
    provider?: string     # 可选 provider 覆盖（缺省按 model 反查宿主路由）
    reasoningEffort?: string  # 按次思考强度（off/low/high/max 等，adapter 自有语义；
                              # 2026-09-05 增补：经 agentOptions.reasoningEffort 透传）
    background?: boolean  # 后台执行（默认 true，durable 子代理）
  }
  ChildRef: { id: string }  # durable 子代理 id（可 send_message 续聊）
```

> 增补说明（2026-09-05，v1.1）：模型调度不止"换模型"，还包括**思考强度**维度
> （flash/pro 之分由 model 承载；effort 由 reasoningEffort 承载）。宿主绑定把
> reasoningEffort 写入 `agentOptions.reasoningEffort`；llm-openai-compat 按路由
> 的 thinking 配置把 effort id 映射为提供方私有 wire 参数（如 Zhipu `thinking`）。

### 2. collect —— 收结果
```
collect(ref) → Report
  Report: {
    status: 'completed' | 'failed' | 'blocked'
    text: string          # 子代理最终答复，原样
  }
```

### 3. fanout —— 多模型并行（便捷封装）
```
fanout(work, models: string[]) → Report[]
# 同一 work 以不同 model 各派一次，全部 settle 后原样返回；
# 不做综合、不排序、不合并——综合判断归主 agent。
```

### 4. listModels —— 模型清单
```
listModels() → { id: string; provider: string }[]
# 从 DSH provider 配置动态读取，不内置模型清单。
```

### 5. 主 agent 自换模型（独立于通道）
```
switchMainModel(model) — 需用户同意门禁（DSH permission-presets）
# 实现依赖宿主能力；若宿主不支持，本原语降级为"提示主 agent 目标模型"。
```

### 事件（可观测性）
`jisi/dispatched`（work 摘要 + model + ref）、`jisi/settled`（ref + status）——供虎符/审计订阅。

## 后果

- 虎符派单优先经集思通道（按次模型）；无集思时回退 DSH 原生 subagent 工具（功能降级、行为不变）。
- 通道不感知"槽位/队列/战役"——那是虎符的策略层；通道只做"把活发给谁、用什么模型、何时交回"。
- 无自动综合：fanout 返回的是 N 份原样报告。

## 替代方案

- A. 扩展 DSH 原生 subagent 工具的 schema 加 model 参数（改上游）——耦合上游评审周期，否决；
- B. 通道做成 DSH 共享原语、两项目平级消费——按用户决定，通道归属集思（P3 独立运行），虎符软依赖；
- C. 在集思内置综合策略（打分/投票）——违背"不教 AI 做事"原则，否决。
