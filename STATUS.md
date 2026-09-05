# 神策验收状态（2026-09-05）

## 验收门矩阵

| 项目 | 仓库 | L0 单元 | L1 集成 | L3 混沌 | v1 状态 |
|---|---|---|---|---|---|
| 集思 | shence-jisi | ✅ 13 | ✅ 实测（按次模型/多模型并行/模型清单，Kimi+智谱真调） | — | 收官 |
| 虎符 | shence-hufu | ✅ 20 | ✅ 实测（双模型迷你战役，done/failed 双路径） | — | 收官 |
| 金柝 | shence-jintuo | —（bash） | ✅ 告警插件实测（模型经 jintuo_alerts 读告警） | ✅ 3/3（杀进程 ~44-51s 自动恢复+告警/证据记录） | 收官 |
| 夜不收 | shence-yebushou | ✅ 15（main 画像 6 + ctf 适配器/hint 账本/治理 9） | ✅ skill 实测（模型调用并正确执行技能） | — | 收官 |

## 环境

- dev 实例：`/mnt/d/Software/WSLSoftware/Agents/deepseek-harness-dev`（最新版 dsh-0.1.3-alpha.1，DSH_HOME=/home/zrn/.dsh-dev，端口 3081，**由金柝守护**）。
- 模型：DeepSeek（key 待用户补）、Kimi k2.6、智谱 glm-4.5-air/4.6（llm-openai-compat 适配器，实测通过）。
- 生产实例零接触：本会话运行于生产 3080，全部开发测试均在 dev 实例完成。

## 遗留（v1.1，各仓 README 已记录）

- 集思：后台 continuable 服务端收结果；主 agent 自换模型；fanout 供应商限流重试。
- 虎符：宿主级 interrupt；心跳后台任务自动续挂；账本自动持久化。
- 金柝：goal-rearm.patch 待应用到 dev checkout 并回归（上游仍 PRESENT）；child-interrupted-notice 补丁设计。
- 夜不收：平台适配器接真实 run 的 L2 合练；治理扫描接入托管打包流水线。

## 下一步（按 TESTING.md）

1. 跨项目 L2 端到端合练（虎符+集思+金柝：一场多工作项战役跑在 dev 实例，全程无人值守）。
2. L4 真实验收（可选，需用户提供 tsecbench 平台 token 与一次真实 run）。
3. 最终迁移：生产实例升级到同版本 + 安装四插件。
