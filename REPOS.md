# 神策（SHENCE）仓库布局与建仓清单

> 命名定稿 2026-08-28：项目群 **神策**；五仓 `shence-*`。
> 母仓库（当前工作区 Tsecbench/recon/）保留为战役档案与孵化区，代码随后迁入各仓。

## 仓库总表

| 仓库 | 项目 | 形态 | 首期内容 |
|---|---|---|---|
| `shence-docs` | 中枢文档 | 文档 | 章程（PROJECTS.md 迁入）、ADR、跨仓接口约定、验证报告归档 |
| `shence-hufu` | 虎符 P1 | DSH 插件 | 槽位状态机、工作项账本、心跳、恢复协议 |
| `shence-jintuo` | 金柝 P2 | 独立启动器+补丁 | dsh-daemon 增强版、资源告警、崩溃恢复、DSH patch overlay |
| `shence-jisi` | 集思 P3 | DSH 插件 | 派单通道（按次模型）、多模型并行收集、主 agent 自换模型门禁 |
| `shence-yebushou` | 夜不收 P4a（分支 `xiaochang` = 校场 P4b） | skill | 目标+通用经验+自积累机制；xiaochang 分支：平台适配器/私知/hint 账本/治理 |

## 各仓 README 骨架（建仓时直接贴）

### shence-docs/README.md
```
# 神策（SHENCE）
> 蜂群式 AI 渗透与 CTF 作战体系。理念：正交→独立项目（允许依赖），包含/继承→同项目分支；不干涉 AI、不教 AI 做事。
## 仓库地图
- shence-hufu  虎符：并行调度（DSH 插件）
- shence-jintuo 金柝：守护启动器（父进程+DSH 补丁）
- shence-jisi  集思：多模型思考（DSH 插件）
- shence-yebushou 夜不收：渗透 skill（分支 xiaochang=校场）
## 文档
- PROJECTS.md 章程（含依赖图）
- ADR/ 决策记录
- INTERFACES/ 跨仓接口约定（P1↔P3 软依赖契约等）
- VALIDATION/ 三个 run 的验证报告归档
```

### shence-hufu/README.md
```
# 虎符（shence-hufu）—— 并行调度
DSH 插件。给 DSH 会话提供可靠的子代理并行工作管理：
战役（campaign：工作项清单+并发上限 N+预算+完成口径）、槽位状态机
（dispatched→probing→求助→stalled→superseded→terminal）、终态去重、
stall 重派（seed 多样性）、心跳保活、调度级崩溃恢复（账本回放）。
不感知平台概念（hint 账本/平台 API 在 shence-yebushou@xiaochang）。
派单：优先经 shence-jisi 通道（按次模型）；无 jisi 回退 DSH 原生 subagent。
并发上限：N = 用户显式上限 ?? 自动推导（本地性能+模型 API 限制）。
```

### shence-jintuo/README.md
```
# 金柝（shence-jintuo）—— 守护启动器
独立父进程：启动/监督 DSH；资源监控并【只告警不代决策】向主 agent 发事件
（内存/堆/并发水位、崩溃、恢复完成）；崩溃后拉起 DSH 并恢复会话。
分工：金柝管进程级恢复（拉起+会话存在）；虎符管调度级恢复（心跳+未完成工作项）。
附带：DSH patch overlay（重启后 goal 重新武装、子代理崩溃通知父会话、内存治理、
dmesg 逃逸副本/退出证据转储——见 RECOVERY.md 复盘）。
```

### shence-jisi/README.md
```
# 集思（shence-jisi）—— 多模型思考
DSH 插件。派单通道：派活(工作描述, 模型?, 工具集?)→durable 子代理句柄；收结果。
多模型并行 = 同一工作以不同模型参数派 N 次，思路原样交回，【综合判断归主 agent】。
主 agent 自换模型：经用户同意（复用 DSH permission-presets）。
模型清单从 DSH provider 配置动态读取，不内置。
独立可运行，不依赖虎符。
```

### shence-yebushou/README.md
```
# 夜不收（shence-yebushou）—— 渗透 skill
黑盒渗透：目标定义（什么算完成）+ 通用经验 + 组织画像自积累机制
（skill 只告知"要积累"，画像数据是运行产物、存工作区本地、不随 skill 发布）。
与虎符/集思：可选受益，不写死依赖（AI/用户选择）。
分支 xiaochang = 校场：继承夜不收；CTF 目标说明、本地私知（flag 位置约定/占位 flag
纪律/题集风格）、平台适配器（tsecbench openapi+VPN 等）、hint 经济学账本、
知识治理（clean-room 托管打包+合规扫描+知识资产防火墙）。
```

## 母仓库迁移清单（recon/ → 各仓）

| 来源（Tsecbench/recon/） | 去向 |
|---|---|
| PROJECTS.md（章程） | shence-docs/ |
| HOSTED-COMPONENT-DESIGN.md、ARCH-*.md、PARALLEL-ARCHITECTURE.md | shence-docs/ADR/ |
| RECOVERY.md（含 daemon 修复记录） | shence-jintuo/docs/ |
| dsh-daemon.sh 改动、.wslconfig 说明 | shence-jintuo/ |
| HOSTED-AGENT-PROMPT.md 的调度通用部分（状态机/心跳/去重） | shence-hufu/docs/ |
| HOSTED-AGENT-PROMPT.md 的渗透/CTF 打法（35 条经验） | shence-yebushou/（xiaochang 分支再分平台/私知） |
| public-corpus/（公开资料库） | shence-yebushou@xiaochang/data/ |
| SOLUTIONS.md、DEAD-ENDS.md、各 run 报告（含题解/旗值） | shence-yebushou@xiaochang/local/（本地私知，不进托管打包） |
| HOSTED-VALIDATION-S1/XBOW-VALIDATION/CYBENCH-VALIDATION.md | shence-docs/VALIDATION/ |
| ONBOARDING.md、ACCESS.md、challenges-api.md（平台 API 知识） | shence-yebushou@xiaochang/adapters/tsecbench/ |

## 建仓后第一步（各仓顺序）

1. `shence-docs`：迁章程与验证报告，生成仓库地图。
2. `shence-hufu`：从验证期协议提炼槽位状态机与账本设计（ADR-001）。
3. `shence-jisi`：定义通道最小契约（ADR-002，被虎符引用）。
4. `shence-jintuo`：迁 dsh-daemon 增强与补丁 overlay。
5. `shence-yebushou`：拆 35 条经验 → 通用（主支）/平台与私知（xiaochang 分支）。
