# HGamey

移动游戏性能优化的 agent harness。

目标是把「改一版网络 → 部署 → 真机测一轮」这个原本靠人工、按天计的循环，压缩成一条自动跑的闭环：**大模型提结构 → 真机秒筛 → 短训 → Pareto 归档 → 下一代**。真机是唯一裁判——跑得慢的候选当场淘汰，不看纸面 FLOPs。

---

## 两个仓库

| 仓库 | 是什么 | 语言 |
|---|---|---|
| **[phonefarm](https://github.com/HGamey/phonefarm)** | 设备侧基建。驱动 Android(adb) 与 OpenHarmony(hdc) 真机，提供延迟标尺、画面采集、遥测、并行调度、保活巡检 | Rust |
| **[sr_loop](https://github.com/HGamey/sr_loop)** | 端侧超分网络的自进化环。基因组、模型生成、TFLite 导出、短训、Pareto 归档、代际推进 | Python |

分工很清楚：**物理能力全在 phonefarm，算法循环全在 sr_loop**。sr_loop 不直接碰设备，它通过 `phonefarm` 这个独立 CLI 拿真机延迟和游戏画面。

---

## 已经验证到哪一步

2026-09-09 在红魔 NX809J（骁龙 8 Elite Gen 5 / Adreno 840）上跑完 13 代：

- 累计最优 PSNR 序列 Spearman **rho = 0.966, p = 7.7e-8** —— 统计显著的正向演进，不是随机游走
- 最优个体 `gen10-6`：**2.967 ms / 39.862 dB**，比 Bicubic 基线 +2.576 dB
- 第 10 代出现真机 SLOW 否决（4.151 ms）—— 真机标尺确实在否决候选

各代模型权重、训练数据集 A 与原始采集帧均已发布在 [sr_loop releases](https://github.com/HGamey/sr_loop/releases)（`v1.0-gate4`，数据集与采集帧因体积分片打包）；也可按下方步骤本地采集重建。

---

## 上手

### 你需要什么

- 一台 macOS 或 Linux 机器（Apple Silicon 需要额外一步 codesign，README 里有说明）
- 一台 **已 root 的** Android 手机，或一块 OpenHarmony 开发板。锁频和读功耗都要 root
- Rust 工具链、Python 3.12
- 一个大模型 API key（变异步骤要用，glm-5.3-flash 就够）

### 第一步：把设备侧跑起来

```bash
git clone https://github.com/HGamey/phonefarm && cd phonefarm
cp secrets.env.example secrets.env        # 填 key
cd src && cargo build --release && cp target/release/phonefarm .. && cd ..
codesign --force --sign - ./phonefarm     # Apple Silicon 必须，否则静默卡死在 _dyld_start
./phonefarm devices                       # 看设备认到没有
```

`phonefarm` 会按 `ADB_BIN` → 仓库内 `platform-tools/` → `PATH` → 系统 SDK 常见目录的顺序找 adb。

### 第二步：确认真机标尺可信

这是整条链路的地基。标尺不准，后面所有结论都是假的。

```bash
./tools/tflite/fetch.sh                   # 拉官方 nightly 的 benchmark_model（含 GPU Delegate）
./phonefarm bench --serial <ID> --model model.tflite --runs 3 --json
```

每轮的动作：等冷机（<40°C）→ 锁 CPU performance + GPU 档位 → 跑 benchmark_model 走 GPU Delegate → 立刻解锁。
判定条件是全图 GPU（有 CPU 回退就算失败）、3 轮离散度 ≤5%、GPU 内核时延中位数 ≤4.0 ms。退出码 0/1/2 = PASS/FAIL/ERROR。

### 第三步：采集画面、造数据集

```bash
# 人先手动把游戏进到大世界，然后：
./phonefarm capture --serial <ID> --out ../sr_loop/data/capture_A --frames 150 --no-shutdown --json
# 已经在大世界时用 --ready-only，只确认状态、不走位
```

采集会做 HUD 排除、相位相关对齐、Bicubic PSNR 熔断。[sr_loop releases](https://github.com/HGamey/sr_loop/releases) 的 `v1.0-gate4` 已经打包了数据集 A 与原始采集帧（体积较大，分片压缩），可以直接下载复用；这里的步骤是本地从头采集复现的路径。

### 第四步：跑演化环

```bash
git clone https://github.com/HGamey/sr_loop && cd sr_loop
uv venv --python 3.12 ~/.venvs/srloop && uv pip install --python ~/.venvs/srloop/bin/python -r requirements.txt

~/.venvs/srloop/bin/python -m sr_loop.gen0 --out models/gen0                       # Gate 0: 起点网络
~/.venvs/srloop/bin/python -m sr_loop.gate0 --serial <ID>                          # Gate 0: 标尺验收
~/.venvs/srloop/bin/python -m sr_loop.gate2 --capture data/capture_A --data data/A --steps 2000 --out runs/gate2
~/.venvs/srloop/bin/python -m sr_loop.evolve --serial <ID> --data data/A --out runs/evo --generations 20 --children 6 --steps 2000
~/.venvs/srloop/bin/python -m sr_loop.analyze --archive runs/evo                   # Spearman 结论
```

---

## 一代里发生了什么

```
大模型提 6 个新结构
      ↓  硬边界卡住超预算的（参数量 / FLOPs）
   导出 TFLite
      ↓  推到真机测速
   慢的直接淘汰（SLOW / FALLBACK / UNSTABLE）
      ↓  活下来的统一短训 2000 步
   跑 PSNR，计分
      ↓  指纹去重后进 Pareto 归档
   gen_N.json
```

跑满 N 代之后 `analyze` 做 Spearman 秩相关，回答一个问题：**这些代之间的提升是真的趋势，还是随机波动。**

---

## 设计上的几条硬规矩

- **真机是唯一裁判。** 秒筛在物理设备上做，不靠估算。
- **测量必须确定性。** agent 负责把设备开到可测状态，测量本身不经过模型。
- **证据落盘。** 每道门产出 `report.json`，结论可追溯到具体那一次运行。
- **不预估时间。** 用进程存活轮询判断完成，不写死等待秒数。

---

## 想改点什么

- 换别的任务（不止超分）：动 `sr_loop/` 下的基因组定义和评分函数，设备侧不用改
- 换设备平台：phonefarm 的 adb/hdc 后端是插拔式的，上层执行回路完全复用
- 加控制维度：功耗和内存的采集通道已经在遥测里（68 项指标），接进评分即可

Issue 和 PR 都开着。
