# HGamey

HGamey 为外部 Agent 提供远程真机研究能力，用于系统性优化指定游戏或系统，完成可复现实验并留下可审核结果。

Agent 在自己的工具中选择方向、制定优先级和解释证据；平台提供设备、资源、任务、研究状态与结果。GOL 是可选的优化方法客户端，不使用内置模型也能组织研究流程。网页与 CLI/API 使用相同对象和操作合同。

人可以异步提供技术指导、数据、现场协助和主观评价。协助请求只阻断依赖它的方向；其余可行方向继续。没有可行工作或预算用完时记录原因并可恢复地空闲。

当前决定、待办与验证见[产品准则](https://github.com/HGamey/docs/blob/main/plans/2026-10-22-agent-platform/PRODUCT-CHARTER.md)、[todo](https://github.com/HGamey/docs/blob/main/plans/2026-10-22-agent-platform/todo.md)与[逐批记录](https://github.com/HGamey/docs/blob/main/plans/2026-10-22-agent-platform/BATCHES.md)。源码与隔离验收不等于某台设备、某个游戏或整套生产流程已可用。

---

## 优化分三层

| 层 | 改什么 | 需要什么 |
|---|---|---|
| ① 游戏代码 | 渲染流程本身：绘制顺序、后处理、分辨率策略 | 可修改、构建和运行的源码（白盒） |
| ② Vulkan 层 | 在游戏与驱动之间插一层，改写 API 调用 | 当前注入路线所需权限与设备/游戏支持，逐项核验 |
| ③ 系统参数 | DVFS、总线频率、调度、热策略、刷新率 | 对应系统接口的写权限与可核验恢复路径 |

这些是可探索方向；闭源游戏还可能通过游戏内设置和公开接口调整。可行性取决于具体设备、系统/游戏版本、权限和实现能力，不能仅凭是否 root 推断。

Android 已验证路径不外推到 Harmony。工程机、厂商官方接口、自签名应用或其他系统版本需独立核验；型号与品牌不证明注入、测量或恢复能力。

---

## 仓库分工

| 仓库 | 是什么 | 语言 |
|---|---|---|
| **[harness-ops](https://github.com/HGamey/harness-ops)** | 节点服务、网页与 CLI/API：目录、资源、持久作业、结果、研究状态与异步协助 | Python / JavaScript |
| **[game_opt_loop](https://github.com/HGamey/game_opt_loop)** | 可选方法客户端：候选生成、优化编排与已有方法工具 | Rust |
| **[phonefarm](https://github.com/HGamey/phonefarm)** | 设备与测量底座、受控动作、实验方法及范围明确的恢复证据 | Rust |
| **[refbench](https://github.com/HGamey/refbench)** | 可重复的白盒 Vulkan 负载与已知答案；机制核查不能直接外推商业游戏收益 | C++ |
| **[docs](https://github.com/HGamey/docs)** | 产品准则、合同、待办、验证记录与历史报告 | 文档 |

前序实验 [sr_loop](https://github.com/HGamey/sr_loop) 已结项：端侧超分网络的代际演化尝试，13 代记录留档。

`HGamey/knobs` 已归档，内容并入 [`phonefarm/knobs/`](https://github.com/HGamey/phonefarm/tree/main/knobs)：黑档是系统参数，灰档是 Vulkan layer 注入。

Megacity、Vulkan-Samples、AnKi 这几个白盒载体挂在 [`phonefarm/loop_v1/carriers/`](https://github.com/HGamey/phonefarm/tree/main/loop_v1/carriers) 下。

演示文稿、架构图等资料放在 [`HGamey/docs`](https://github.com/HGamey/docs)。部分链接可能需要仓库访问权限。

> HGamey/phonefarm 基于 [BH3GEI/phonefarm](https://github.com/BH3GEI/phonefarm) 协同演化。上游来源、当前配置的程序和已运行的会话版本分别核对，不能从仓库名称推断部署能力。

---

## 两侧的职责为什么要分开

外部 Agent 提出候选并预先声明判据，平台按固定请求执行和保存证据；GOL 可作为其中一种方法客户端。**看到候选结果后不能改写旧实验的及格线。**

冷机门槛、回放时长、A/B 方式、统计规则与画质预算在实验前冻结。原游戏路径是主基线，模块 ms 也可以是明确目标。命令退出、统计判定、局部恢复和整 job 资源释放分别记录；恢复未确认时保留失败/未知与原件，不宣称已回到原样。

单轮研究流程如下；每一步的实际完成范围需有证据：

```
选一个可重复的负载
   ↓  离散度自检
量基线（帧时序 / GPU 执行时长 / 带宽 / 功耗 / 温度）
   ↓
归因，冻结判定规则
   ↓
下发候选改动，A/B 交替对照
   ↓
统计检验（置换检验 / 置信区间）
   ↓  按冻结合同恢复并核验
记录恢复范围；失败或未知保留屏障
   ↓
结构化报告与证据归档
```

---

## 技术路线梳理

闭环里的候选改动从这份清单里挑，按带宽与 Tile Memory、几何与 Binning、着色器计算、分辨率重构、时序功耗与散热分成五类，每条标注量级、用什么指标验、需不需要白盒。

- 当前文档：[`HGamey/docs · MOBILE_GPU_OPT_ROUTES.md`](https://github.com/HGamey/docs/blob/main/repos/phonefarm/docs/MOBILE_GPU_OPT_ROUTES.md)
- 上游资料：[`BH3GEI/phonefarm/docs/MOBILE_GPU_OPT_ROUTES.md`](https://github.com/BH3GEI/phonefarm/blob/main/docs/MOBILE_GPU_OPT_ROUTES.md)

---

## 历史实验记录（仅适用于原记录条件）

以下保留既有报告的数字、否定结论与撤回说明。本轮只调整文档入口，没有重新实测；这些结果不证明当前部署或其他设备/版本可用。

**Vulkan 层挂进原神大世界跑通。** 客户端 7.1.0，挂只读层正常登录、进大世界、跑满一轮负载，进程无异常日志、未被杀。同一轮里试的第一条改写（LoadOp `LOAD` → `DONT_CARE`）确实命中了——改写条数、绑定次数都能自报——但主指标 frame_p95 差值 −0.51%，置换检验 p = 0.325，判为无效。这是一条有价值的否定结论：机制成立，这一条换不出性能。（phonefarm PR #31）

**停充之后功耗对上了。** 原神固定场景 30 s 并排采样，整个窗口停止充电，我们自己的 sysfs 通路与 HiSmartPerf 报出的整机功耗差 2.3%。作为对照，充电态下同一条轨两边分别报 3.086 W 和 777 W——差 250 倍。（phonefarm PR #32）

**载体的离散度落在该在的位置。** Megacity 三轮基线 frame_p95 离散度 0.987%，零热事件。横向看：refbench（白盒靶子）0.45%，Megacity 0.987%，原神（闭源真游戏）1.37%。（phonefarm PR #33）

**画质用真机帧当真值。** 用 19 张原神实拍帧算 PSNR，赛道基线模板均值 34.52 dB；演化出的最好一个算子多花 0.22 ms 换来 +3.3 dB，凭画质站进 Pareto 前沿。之前用程序化合成图案当参考图时，同一个算子的绝对值要高约 9~12 dB，而且高频分布完全不同——按合成图案排出来的画质名次，未必是真实画面的名次。（game_opt_loop evidence M2）

### 两条需要更正的旧说法

- **M1 的「6.26 W → 5.94 W」已撤回。** 采样时设备插着 USB 充电，量到的是 USB 输入轨，其中相当一部分在给电池充电并随电量漂移，跨候选比较不成立。现在功耗只有在放电态下才进入判定。
- **「原神被厂商限帧 30fps」需要加注。** 那是客户端 7.0.0、当时那套游戏内设置下的观测。7.1.0 在游戏内 60 帧设置下实测就是 60 fps，GPU 约 13.3 ms/帧。两次差异的原因还没核实，在核实之前不要把「限帧 30」当成这台设备的普遍结论。

---

## 设备

对候选效果的 A/B 归因使用同一设备与受控条件；跨设备结果需分别保留设备和条件，不能把机间差异当成候选收益。

此前设备规划包括安卓测试/调试机、鸿蒙手机 1–2 台及其他系统版本的华为设备；这是待核计划，不是现有机器或权限清单。型号、系统和是否 root 不代替具体能力验证。

当前节点目录、设备登记与采样事实分开呈现；登记不代表在线或物理核验。设备操作遵循当前租约和恢复合同，心跳失效或宿主进程退出不证明设备已恢复。

---

## 当前状态与此前方法记录

当前工作以[待办](https://github.com/HGamey/docs/blob/main/plans/2026-10-22-agent-platform/todo.md)与[逐批验证](https://github.com/HGamey/docs/blob/main/plans/2026-10-22-agent-platform/BATCHES.md)为准，代码提交、部署、隔离验收与真机验收分别记录。以下保留此前方法迁移进展，不作为当前整个平台已可用的结论：

- loop_v1 的判定口径已经全部收进 Rust（解析、归因、统计、判据、白名单、画面判据、模型交互与本地变异器），两通路对照、Vulkan-Samples 三道闸与两臂判读面、refbench/vks 报告也已收完；`autoloop` 编排也已收进 Rust（`phonefarm autoloop`），灰档层挂/摘（`gray-layer`）、灰档 A/B（`gray-ab`）与截帧比对（`shot-diff`）也已收进 Rust。剩下载体构建/路线工具、设备端 shell（按约定保持设备端）与 knobs/ 那批。
- 系统参数与画面流程改写两层的真机评测：phonefarm 的 `eval --request` 接收端已落地并真机验证——③ 系统参数端到端已出报告（首候选 REJECT），② 灰档机械链路已通、命中口径已对齐（真机实测命中 27 处 / begins 9195）。

`optimize` 总入口、agent skill 说明书、可视化前端这三个已经落地（此前方法记录见 [game_opt_loop STATUS](https://github.com/HGamey/docs/blob/main/repos/game_opt_loop/STATUS.md)）。

---

## 从已选节点开始

先确认入口 URL、node_id、执行宿主和数据根；不默认选旧后台，不自动迁移原件。网页从该节点的 `/dash/fleet` 查看目录，`/dash/games` 准备资源，`/dash/jobs` 查看任务，`/dash/results` 查看结果，`/dash/research` 记录方向与协助。`/dash/control` 需查看授权和操作员令牌，操作能力按当前会话核查。

```bash
# 在 harness-ops 工作目录使用已有节点与授权文件
export PF_NODE_URL=https://node.example.invalid
export PF_NODE_TOKEN_FILE=/private/path/node.token
node/pf-node node                          # 先读身份、宿主与数据位置
export PF_NODE_ID='确认后的 node_id'
node/pf-node jobs
node/pf-node research list direction
```

网页与 CLI 使用同一目标和对象；请求未确认时保留原 ID 查询，不改投其他节点。资源已缓存、应用安装、登录/场景就绪和实验收益各自需要证据。

### 源码构建与可选客户端示例

下面保留上游构建和 GOL 方法示例，均不是平台使用的必经步骤或真机验收。上游构建不代表当前 HGamey 节点协议；组织开发版本及配置按各仓 README 核对。需要相应 Rust/C++ 工具链，Apple Silicon 构建后重新签名。

```bash
# 上游底座源码
git clone https://github.com/BH3GEI/phonefarm && cd phonefarm
cd src && cargo build --release && cp target/release/phonefarm .. && cd ..
codesign --force --sign - ./phonefarm   # macOS 需要
./phonefarm devices                      # 枚举传输可见设备，不代表实验就绪

# 可选 GOL 方法客户端（需相应仓库访问权限）
git clone https://github.com/HGamey/game_opt_loop && cd game_opt_loop
cargo test
cargo run -- --track sr --generations 3 --children 4 --dry-run --no-llm
```
