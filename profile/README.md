# HGamey

把手机游戏的性能优化做成一套能自己跑完的 harness。

用户只需要提一种任务：**「把 \<游戏\> 在 \<手机\> 上优化一下」**。之后的事情由 harness 自己决定——先摸清这台设备和这个游戏当前是什么状况，判断该开哪几条优化路线，让大模型提出候选改动，在真机上做对照实验，用统计结果决定哪些留下、哪些回退。

harness 本身就是产品。我们交付的不是某一个优化算子，而是这条从任务到结论的流水线。

---

## 优化分三层

| 层 | 改什么 | 需要什么 |
|---|---|---|
| ① 游戏代码 | 渲染流程本身：绘制顺序、后处理、分辨率策略 | 源码（白盒） |
| ② Vulkan 层 | 在游戏与驱动之间插一层，改写 API 调用 | **不需要源码，但需要 root** |
| ③ 系统参数 | DVFS、总线频率、调度、热策略、刷新率 | 需要 root |

白盒游戏三层都能用，黑盒商用游戏只剩 ② 和 ③。② 和 ③ 的区别只在 ② 不需要游戏源码，**要 root 这一条两者一样**。

所以国内鸿蒙商用机上 ①②③ **一层都改不了**——没有 root，只能测。几条可能的出路：工程机、厂商官方性能接口、开发者模式下自签名应用，以及海外版华为手机（EMUI，基于安卓改的系统，麒麟芯片，能 root）——在它上面可以测 ② 和 ③ 的改动在麒麟上到底有多少效果，国内鸿蒙机只能做对照。这几条都还没有核实过，先记在这里。

---

## 仓库分工

| 仓库 | 是什么 | 语言 |
|---|---|---|
| **[game_opt_loop](https://github.com/HGamey/game_opt_loop)** 🔒 | 入口与决策层。接任务、摸底、决定开哪几路、组织大模型出方案、维护优胜档案、出总报告 | Rust |
| **[phonefarm](https://github.com/BH3GEI/phonefarm)** | 底座。设备连接（adb / hdc）、应用改动、测量、统计判定、现场还原 | Rust |
| **[refbench](https://github.com/HGamey/refbench)** 🔒 | 白盒测试载体。确定性 Vulkan 负载，每条优化路线一个开关，瓶颈类型可定向构造 | C++ |
| **[sr_loop](https://github.com/HGamey/sr_loop)** | 前序实验，已结项。端侧超分网络的代际演化尝试，13 代记录留档 | Python |

`HGamey/knobs` 已归档，内容并入 [`phonefarm/knobs/`](https://github.com/BH3GEI/phonefarm/tree/main/knobs)：黑档是系统参数，灰档是 Vulkan layer 注入。

Megacity、Vulkan-Samples、AnKi 这几个白盒载体挂在 [`phonefarm/loop_v1/carriers/`](https://github.com/BH3GEI/phonefarm/tree/main/loop_v1/carriers) 下。

> phonefarm 组织内另有私有协同分支；对外以 [BH3GEI/phonefarm](https://github.com/BH3GEI/phonefarm) 为准。

---

## 两侧的职责为什么要分开

game_opt_loop 出方案，phonefarm 出判定。**出方案的一方改不了判定规则。**

判定规则在看到候选数据之前就落盘冻结：冷机门槛、回放时长、A/B 交替方式、显著性阈值。跑完之后不合格就自动回退，设备现场逐项校验回到原样。这条分界是为了防止一件很容易发生的事——调参的人顺手把及格线也一起调了。

单轮回路大致是这样：

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
   ↓  不达标就回退
现场快照逐项校验
   ↓
结构化报告与证据归档
```

---

## 技术路线梳理

闭环里的候选改动从这份清单里挑，按带宽与 Tile Memory、几何与 Binning、着色器计算、分辨率重构、时序功耗与散热分成五类，每条标注量级、用什么指标验、需不需要白盒。

- 🔒 组织成员：[`HGamey/phonefarm/docs/MOBILE_GPU_OPT_ROUTES.md`](https://github.com/HGamey/phonefarm/blob/main/docs/MOBILE_GPU_OPT_ROUTES.md)
- 🌐 公开镜像：[`BH3GEI/phonefarm/docs/MOBILE_GPU_OPT_ROUTES.md`](https://github.com/BH3GEI/phonefarm/blob/main/docs/MOBILE_GPU_OPT_ROUTES.md)

---

## 已经量到的一些东西

下面每条都有对应的仓库证据，读数字之前先看条件。

**Vulkan 层挂进原神大世界跑通。** 客户端 7.1.0，挂只读层正常登录、进大世界、跑满一轮负载，进程无异常日志、未被杀。同一轮里试的第一条改写（LoadOp `LOAD` → `DONT_CARE`）确实命中了——改写条数、绑定次数都能自报——但主指标 frame_p95 差值 −0.51%，置换检验 p = 0.325，判为无效。这是一条有价值的否定结论：机制成立，这一条换不出性能。（phonefarm PR #31）

**停充之后功耗对上了。** 原神固定场景 30 s 并排采样，整个窗口停止充电，我们自己的 sysfs 通路与 HiSmartPerf 报出的整机功耗差 2.3%。作为对照，充电态下同一条轨两边分别报 3.086 W 和 777 W——差 250 倍。（phonefarm PR #32）

**载体的离散度落在该在的位置。** Megacity 三轮基线 frame_p95 离散度 0.987%，零热事件。横向看：refbench（白盒靶子）0.45%，Megacity 0.987%，原神（闭源真游戏）1.37%。（phonefarm PR #33）

**画质用真机帧当真值。** 用 19 张原神实拍帧算 PSNR，赛道基线模板均值 34.52 dB；演化出的最好一个算子多花 0.22 ms 换来 +3.3 dB，凭画质站进 Pareto 前沿。之前用程序化合成图案当参考图时，同一个算子的绝对值要高约 9~12 dB，而且高频分布完全不同——按合成图案排出来的画质名次，未必是真实画面的名次。（game_opt_loop evidence M2）

### 两条需要更正的旧说法

- **M1 的「6.26 W → 5.94 W」已撤回。** 采样时设备插着 USB 充电，量到的是 USB 输入轨，其中相当一部分在给电池充电并随电量漂移，跨候选比较不成立。现在功耗只有在放电态下才进入判定。
- **「原神被厂商限帧 30fps」需要加注。** 那是客户端 7.0.0、当时那套游戏内设置下的观测。7.1.0 在游戏内 60 帧设置下实测就是 60 fps，GPU 约 13.3 ms/帧。两次差异的原因还没核实，在核实之前不要把「限帧 30」当成这台设备的普遍结论。

---

## 设备

同一个候选的 A/B 两臂必须跑在同一台手机上——跨机比较拿不到可比的数字。

下面是**计划**，不是现有的机器清单：

- **安卓旗舰机**：一台跑测试，一台留着调试。要并行跑多条线，得是同型号的多台——不同型号之间的数字本来就不可比。
- **鸿蒙手机 1–2 台**：做对照。没有 root，只能测。
- **海外版华为手机几台**：EMUI，基于安卓改的系统，麒麟芯片，能 root。用它在麒麟上测 ② Vulkan 层和 ③ 系统参数的改动到底有多少效果——国内鸿蒙商用机改不了，只能拿来对照。

设备是共享的，用 `devlock` 抢锁排队，带心跳，跑完释放。

---

## 正在做

- loop_v1 的判定口径已经全部收进 Rust（解析、归因、统计、判据、白名单、画面判据、模型交互与本地变异器），两通路对照、Vulkan-Samples 三道闸与两臂判读面、refbench/vks 报告也已收完；`autoloop` 编排也已收进 Rust（`phonefarm autoloop`）。剩下载体构建/路线工具、设备端 shell（按约定保持设备端）与 knobs/ 那批。
- 系统参数与画面流程改写两层的真机评测：phonefarm 的 `eval --request` 接收端已落地并真机验证——③ 系统参数端到端已出报告（首候选 REJECT），② 灰档机械链路已通、命中口径已对齐（真机实测命中 27 处 / begins 9195）。

`optimize` 总入口、agent skill 说明书、可视化前端这三个已经落地（详见 [game_opt_loop](https://github.com/HGamey/game_opt_loop) 🔒 的 STATUS.md）。

---

## 快速上手

需要一台 macOS 或 Linux 工作站、一台有调试权限的手机（Android 走 adb，OpenHarmony 走 hdc），以及 Rust 与 C++ 编译环境。Apple Silicon 上构建完要对二进制做代码签名。

```bash
# 底座
git clone https://github.com/BH3GEI/phonefarm && cd phonefarm
cd src && cargo build --release && cp target/release/phonefarm .. && cd ..
codesign --force --sign - ./phonefarm   # macOS 需要
./phonefarm devices                      # 看看设备连上没有

# 决策层（🔒 私有仓库，组织成员可访问）
git clone https://github.com/HGamey/game_opt_loop && cd game_opt_loop
cargo test
cargo run -- --track sr --generations 3 --children 4 --dry-run --no-llm
```
