# 托管模式组件设计（HOSTED-COMPONENT-DESIGN）

> 状态：设计定稿（2026-08-25 讨论结论），未动工。
> 两个决定：①**全托管**（平台沙箱运行，目标"放手"）；②**不脱离 DSH**（交付心态 = 插件 + Skill，Agent 循环机制白嫖 DSH，不写完整 Agent）。

## 一、平台契约（证据）
- 接入页面文案（BenchmarkPrepareView JS）："**大模型网关**，制作并上传 Docker 镜像，平台自动完成沙箱部署与运行。"
- 三种接入方案：①Agent 提示词接入（本地跑）②Python SDK 接入（本地跑）③**Docker 镜像上传 → 平台沙箱全托管**。
- 排行榜条目含 base_model + token_usage ⇒ LLM 调用走**平台侧大模型网关**计量，镜像内不要自带 LLM key。
- 跑分契约（sdk-samples.md）：注入 BENCHMARK_TOKEN + BENCHMARK_BASE_URL；VPN 预检 http://10.0.100.58；3 容器上限；hint 扣分；submit 幂等 duplicate。

## 二、提交单元 = Docker 镜像
```
Docker 镜像
└─ DSH headless（运行时/框架，非交付代码）
   ├─ tsecbench-plugin（唯一自研代码，两个模块）
   │   ├─ daemon 模块：VPN 预检 → challenges 轮询 → 3 槽调度 → submit/close 循环
   │   │             → 进度台账 2 分钟轮询 → 时间盒回收 → hint 经济学 → 收官对账 → 合规审计
   │   └─ solver 模块：spawn DSH 子代理（fan-out，每槽多视角）
   │                 输入 = 描述原文 + 方法论 + 公共弱口令表（托管规则密封，见 HOSTED-RULES.md）
   │                 终态 = DONE|FAIL|BLOCKED|SUPERSEDED
   └─ 知识层（Skill 形态，随包资产）
       ├─ HOSTED-AGENT-PROMPT.md（方法论，每条可溯源：公开技艺/平台公开文档/题目描述）
       ├─ 公共弱口令表（公开源重建版，与 creds-corpus 物理分离）
       └─ 公开 CVE/默认凭据语料（NVD/OSV/GitHub advisories 离线副本，防沙箱无外网）
```

## 三、形态裁决（plugin / command / skill）
- **插件 = 交付物**（可执行逻辑在插件里；DSH 提供 LLM runtime/bash/fs/session/subagent fan-out/GUI 监控）。
- **Skill = 知识层打包**（复刻 Cairn `tsec-actions/SKILL.md` 模式：平台操作流程 + 方法论，插件 spawn 子代理时注入）。
- **command/独立程序 = 不交付**（用户决定不写完整 Agent；插件核心模块保持无 DSH 专用 import 的写法，万一平台不接受镜像可降级迁移）。

## 四、未知项（首个托管轮 = 探测轮）
1. 沙箱规格（内存/CPU）：镜像跑 DSH **headless**（参考 checkout `examples/headless-agent`），GUI 只留本地开发。
2. 大模型网关形状：插件读环境变量接网关；网关是否透传 `web_search` 工具 → 决定搜索走原生工具还是 curl NVD/OSV（两条都留）。
3. 沙箱外网：可能无外网 → 知识本地化为默认路径，搜索是机会主义增益。

## 五、禁区（镜像打包红线）
- SOLUTIONS.md / DEAD-ENDS.md / creds-corpus.txt / 平台机制先验（ARCH-OBS）**一律不进镜像**。
- 镜像只含：方法论 prompt + 公开源弱口令表 + 公开 CVE 语料 + 平台公开文档知识。

## 六、与现架构的同构性
- 插件 daemon 模块 ≈ 现在"主会话=调度器"；solver 模块 ≈ 子代理。**v7.3 调度协议（2 分钟轮询、SUPERSEDED、收官对账、hint 经济学）直接搬入 daemon 模块，无需重设计。**
- 本地验证轮（run 12459/下一轮）继续用 DSH 会话模拟，作为镜像版的行为基线。


---

# 七、架构精化（2026-08-26 讨论定稿 · 两项目拆分修正）

## 分层修正：机制在项目一、策略在项目二
- **项目一（DSH 并行任务管理核心）只交付机制**：
  1. 任务队列/槽位调度/fan-out/时间盒/终态机/台账剪枝/轮询-事件双通道——全部领域无关；
  2. **通用"预算-代价台账"**：任务可挂代价动作（余额/消耗记录/策略钩子），不关心动作语义；
  3. **两个薄接口**：任务完成判定器、任务价值估计器——由分支注入，调度器只消费。
- **项目二（渗透=解空间求解）交付策略与知识**：
  - 真实渗透分支：代价动作 = 烧域名/买 0day/暴露风险/告警静默成本；完成判定 = 目标谓词（DA/外带/凭据/持久化）；价值 = 资产价值 × 风险成本。
  - CTF 分支（约束层）：代价动作 = hint（估值策略即 v7.3 F13：多旗高分题同墙 2 轮必耗、-10%<残局价值）；完成判定 = 提交 oracle（correct/awarded）；价值 = flag 分值 × 折扣。
- **flag 不作通用奖励信号**（修正）：flag 是 CTF 分支自带的 oracle，经"完成判定器/价值估计器"接口回灌；项目一不内置 flag 概念。

## 蜂群/seed 搜索模型（替代攻击图，定稿）
- **多 seed 并行**：每个 seed = 一个切入假设（web 入口/源码审计/横向/数据面……），同目标并行 N 个 seed，先破者胜（b-01 三角度、f2-05 十二角度实证）。
- **剪枝双通道**：代理内部自剪（LLM 推理否定死路，写 DEAD: 行）+ 台账剪（新 seed 开工必读，不重烧）。
- **多样性控制（关键条款）**：调度器维护"切入假设占用表"，派单时把已被 N 个 seed 占用的假设标注，新 seed 强制从剩余假设池选——防蜂群坍缩（b-02 五代代理全挤"猜 file-server 密码"的教训）。
- **不做攻击图/显式状态空间**（定稿）：预定义图与真实攻击面不匹配反而干扰；搜索发生在 LLM 推理内部，调度器只做外层 fan-out/剪枝/记分。

## 另两套题集（平台 API 实测）
- 全部 6h（21600s）、槽位平台级（set 4 实测 3，另两套首跑验证）。
- set 1 XBOW：104 题/28300 分，纯 Web，吞吐要求 ~17 题/时（本套 63 题实测 ~11.5 题/时——XBOW 无 b 系长链、题均更快，预计可覆盖，首跑先打 30 分钟定节拍再排优先级）。
- set 5 Cybench：40 题/20800 分，专业 CTF（人类 11min-24h/题），多 seed 并行是唯一杠杆；预期得分率显著低于 Web 套，先按"题难度分层 + 每题 2-3 seed"打。
