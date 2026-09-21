# HGamey

移动游戏性能评估与优化的 agent harness。

**harness 既是评估性能 gap 的框架，也是优化性能的 loop。** 目标是把「量基线 → 找瓶颈 → 提改动 → 对照复量 → 判定保留或回退」这个原本靠人工、按天计的循环，压缩成一条无人值守跑完的闭环。真机是唯一裁判——判定规则在看到候选数据**之前**落盘冻结，改善不达标就自动回退，不留痕迹。

NN（端侧超分 / 插帧）是这个闭环里的**一类候选改动**，不是主线。它和其它每一条候选一样，走同一套基线→对照→判定。

---

## 两个仓库

| 仓库 | 是什么 | 语言 |
|---|---|---|
| **[phonefarm](https://github.com/BH3GEI/phonefarm)** | 闭环本体。驱动 Android(adb) 与 OpenHarmony(hdc) 真机：帧时序采集、GPU 归因、系统旋钮、统计判定、证据归档；以及延迟标尺、画面采集、并行调度、保活巡检 | Rust |
| **[sr_loop](https://github.com/HGamey/sr_loop)** | NN 候选的自进化环。基因组、模型生成、TFLite 导出、短训、Pareto 归档、代际推进 | Python |

分工：**物理能力全在 phonefarm，算法循环全在 sr_loop**。sr_loop 不直接碰设备，它通过 `phonefarm` 这个独立 CLI 拿真机延迟和游戏画面。

> phonefarm 上游公开仓库是 [BH3GEI/phonefarm](https://github.com/BH3GEI/phonefarm)（原创作者与著作权方）。组织内另有一份私有开发仓库 `HGamey/phonefarm`，仅成员可见，用于承载私有改动。下面的命令都用公开上游。

---

## 先读这个

**[移动 GPU 优化技术路线梳理](https://github.com/BH3GEI/phonefarm/blob/main/docs/MOBILE_GPU_OPT_ROUTES.md)** —— 闭环里的候选改动从这份清单里挑。

按移动 GPU 的三条结构性差异（TBDR、多一遍 binning、带宽与热受限且共享 LPDDR）分五类：带宽与 tile memory、几何与 binning、shader、分辨率与上采样、时序功耗热。每条标注移动端特有性、量级、以及**用闭环里的哪个指标验**——不能被这套闭环验证的路线不进候选。

---

## 已经验证到哪一步

### 闭环本身成立（2026-09-16，红魔 NX809J / Android 16 / Adreno 840v2）

对一个可重复的游戏负载（原神定点匀速转视角）跑通完整一轮，五条判据全部实测通过：

| 判据 | 结果 |
|---|---|
| 负载可重复 | `frame_p95` 离散度 **1.63%**（阈值 5%），无系统漂移 |
| 归因成立 | 5 轮结论一致：**限帧器封顶 @30fps，GPU 只用掉 64.5%** |
| 统计显著 | p95 改善 **−0.586 ms（−1.61%）**，d = −2.44，95%CI [−0.959, −0.213] 不含 0，精确置换检验 **p = 0.0079** |
| 不留痕 | 进出设备快照 38 行逐行相等 |
| 可离线回放 | 21 轮 trace + report.json 全部重算**字节一致** |

被验证的改动是把 DDR/LLCC 总线下限钉到硬件上限（3187 → 5333 MHz，+67% 带宽）。一个反直觉的结果：帧时间三项指标都显著改善，但 `gpu_active_mean` **不显著**（p = 0.42）——该负载下 GPU 并没有被带宽饿着。**「带宽不够」不能当默认假设，要一条条量。**

详见 [`loop_v1/README.md`](https://github.com/BH3GEI/phonefarm/blob/main/loop_v1/README.md)。

### NN 候选线（2026-09-09，13 代）

- 累计最优 PSNR 序列 Spearman **rho = 0.966, p = 7.7e-8** —— 统计显著的正向演进，不是随机游走
- 最优个体 `gen10-6`：**2.967 ms / 39.862 dB**，比 Bicubic 基线 +2.576 dB
- 第 10 代出现真机 SLOW 否决（4.151 ms）—— 真机标尺确实在否决候选

**同时要记账的两个数**：`phonefarm bench` 区分 `gpu`（只含模型算子，2.967 ms）与 `invoke`（含张量拷贝与同步）两个口径，后者比前者高约 **7 ms 且与网络结构无关**，540×960 → 1080×1920 每帧拷贝 **31 MB**。端到端约 10 ms，吃掉 60fps 帧预算的 60%。

这 7 ms 的性质尚未定论——它可能相当部分是测量工装的产物（`benchmark_model` 从 CPU 内存喂 fp32 张量）。真接进引擎可绑 GL SSBO 省掉往返、换 fp16 后 31 MB 降到 8 MB 量级。**这个对照实验决定 NN 支线的去留，在它落地前说「NN 行」或「NN 不行」都缺证据。**

各代模型权重、训练数据集 A 与原始采集帧已发布在 [sr_loop releases](https://github.com/HGamey/sr_loop/releases)（`v1.0-gate4`，数据集与采集帧因体积分片打包）。

---

## 上手

### 你需要什么

- 一台 macOS 或 Linux 机器（Apple Silicon 需要额外一步 codesign，README 里有说明）
- 一台 **已 root 的** Android 手机，或一块 OpenHarmony 开发板。锁频、读 ftrace、读功耗都要 root
- Rust 工具链、Python 3.12
- 一个大模型 API key（NN 变异步骤才要用，glm-5.3-flash 就够）

### 第一步：把设备侧跑起来

```bash
git clone https://github.com/BH3GEI/phonefarm && cd phonefarm
cp secrets.env.example secrets.env        # 填 key
cd src && cargo build --release && cp target/release/phonefarm .. && cd ..
codesign --force --sign - ./phonefarm     # Apple Silicon 必须，否则静默卡死在 _dyld_start
./phonefarm devices                       # 看设备认到没有
```

`phonefarm` 会按 `ADB_BIN` → 仓库内 `platform-tools/` → `PATH` → 系统 SDK 常见目录的顺序找 adb。

### 第二步：跑一轮性能优化闭环

这是产品主线。人先手动把游戏进到可重复的起始状态，然后：

```bash
cd loop_v1
adb push tools/device_snapshot.sh tools/ftrace_capture.sh tools/knob_ddr_boost.sh /data/local/tmp/
adb shell chmod 755 /data/local/tmp/{device_snapshot,ftrace_capture,knob_ddr_boost}.sh

adb shell "su -c 'sh /data/local/tmp/device_snapshot.sh'" > runs/snap_before.txt

# 交错跑改动臂与对照臂，让残余漂移对两臂等量影响
for i in 1 2 3 4 5; do
  adb shell "su -c 'sh /data/local/tmp/knob_ddr_boost.sh apply'"
  bash tools/run_once.sh knob$i runs/knob$i
  adb shell "su -c 'sh /data/local/tmp/knob_ddr_boost.sh restore'"
  bash tools/run_once.sh ctrl$i runs/ctrl$i
done

python3 tools/report.py --baseline 'runs/ctrl*' --knob 'runs/knob*' \
    --snap-before runs/snap_before.txt --snap-after runs/snap_final.txt > runs/report.json
bash tools/replay_test.sh runs
```

两条纪律写死在流程里：判定规则在看到候选数据前冻结；负载必须在光照稳定窗口内跑完（昼夜循环会把离散度从 1.37% 推到 5.35%，而且是单向漂移）。

### 第三步（可选，NN 候选线）：真机标尺 → 采集 → 演化

```bash
./tools/tflite/fetch.sh                   # 拉官方 nightly 的 benchmark_model（含 GPU Delegate）
./phonefarm bench --serial <ID> --model model.tflite --runs 3 --json
# 判定: 全图 GPU（有 CPU 回退就算失败）、3 轮离散度 ≤5%、GPU 内核时延中位数 ≤4.0 ms。退出码 0/1/2 = PASS/FAIL/ERROR

# 人先手动把游戏进到大世界，然后采集造数据集：
./phonefarm capture --serial <ID> --out ../sr_loop/data/capture_A --frames 150 --no-shutdown --json
# 已经在大世界时用 --ready-only，只确认状态、不走位

git clone https://github.com/HGamey/sr_loop && cd sr_loop
uv venv --python 3.12 ~/.venvs/srloop && uv pip install --python ~/.venvs/srloop/bin/python -r requirements.txt

~/.venvs/srloop/bin/python -m sr_loop.gen0 --out models/gen0
~/.venvs/srloop/bin/python -m sr_loop.gate0 --serial <ID>
~/.venvs/srloop/bin/python -m sr_loop.gate2 --capture data/capture_A --data data/A --steps 2000 --out runs/gate2
~/.venvs/srloop/bin/python -m sr_loop.evolve --serial <ID> --data data/A --out runs/evo --generations 20 --children 6 --steps 2000
~/.venvs/srloop/bin/python -m sr_loop.analyze --archive runs/evo
```

数据集 A 与原始采集帧也可以直接从 [sr_loop releases](https://github.com/HGamey/sr_loop/releases) 的 `v1.0-gate4` 下载复用，跳过采集步骤。

---

## 一轮闭环里发生了什么

```
选一个可重复的负载
      ↓  离散度 + 漂移自检；不达标不往下走
   量基线（ftrace + kgsl：帧节奏 / GPU 执行时长 / 带宽投票 / 热降频）
      ↓
   归因（这一帧花在哪类开销上）
      ↓  判定规则在此处落盘冻结
   改一个旋钮 → 交错跑对照臂
      ↓
   精确置换检验 + 置换反演 CI
      ↓  不显著 / 方向不对 → 回退
   设备快照逐行 diff，确认不留痕
      ↓
   report.json 归档，可离线字节复现
```

NN 候选走的是同一条路，只是「改一个旋钮」换成「换一个网络结构」。

---

## 设计上的几条硬规矩

- **真机是唯一裁判。** 秒筛与判定都在物理设备上做，不靠估算。
- **判定规则先于数据冻结。** 先写死「改善多少才算数」，再去看候选跑出什么。
- **测量必须确定性。** agent 负责把设备开到可测状态，测量本身不经过模型；下游解析全是纯函数，同一份 trace 每次重算逐字节一致。
- **不留痕。** 所有 sysfs 写入退出前恢复，进出快照逐行相等才算这一轮成立。
- **证据落盘。** 每一轮产出 `report.json`，结论可追溯到具体那一次运行的原始 trace。
- **不预估时间。** 用进程存活轮询判断完成，不写死等待秒数。

---

## 当前的观测边界

诚实说明，这决定了哪些优化路线现在就能验：

| 能看见 | 看不见 |
|---|---|
| 帧节奏（p50/p95/p99/mean）、抖动 | **render pass / draw call 级归因** |
| GPU 执行时长与占比 | 硬件计数器：实测带宽、shader busy、tile 数、overdraw、纹理 cache miss |
| 总线投票与余量 | 顶点 / binning 的独立开销 |
| 队列等待、热降频事件、CPU/GPU 频率、结温 | |
| TFLite 算子级 GPU 内核延迟 | |

测帧手段三条路都实测排除：`SurfaceFlinger --latency` 在 Android 16 已失效；`gfxinfo` 测不到走 SurfaceView 的原神；Perfetto 的 GPU producer 在本机驱动未注册。最终走 raw ftrace 的 kgsl 事件。

**含义**：第三方游戏（黑盒）下只能做评估与系统级旋钮，render pass / shader / draw call 一个都改不了。想验证路线清单里的大多数条目，需要一个白盒靶子（引擎 demo 工程）。

---

## 想改点什么

- 加一类新的优化候选：在 `loop_v1/tools/` 下加一个旋钮脚本，报告与判定流程完全复用
- 换别的 NN 任务（不止超分）：动 `sr_loop/` 下的基因组定义和评分函数，设备侧不用改
- 换设备平台：phonefarm 的 adb/hdc 后端是插拔式的，上层执行回路完全复用；鸿蒙侧需要把 ftrace + kgsl 这一层换成 hdc + RenderService/hidumper，上层判据逻辑可平移
- 加控制维度：功耗和内存的采集通道已经在遥测里（68 项指标），接进判定即可

Issue 和 PR 都开着。
