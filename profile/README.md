# HGamey

移动端图形渲染能效与画质自主优化实验台（Agent Harness）。

**Harness 既是评估性能差异的标准框架，也是自主优化演进的闭环回路。** 目标是将「量测基线 → 定位瓶颈 → 提出优化改动 → 对照复测 → 判定保留或回退」这一流程，沉淀为自动化、可重复运行的闭环。以真机物理实测为客观裁决依据——判定规则在测试前固化冻结，改动若无统计学显著收益则自动回退，保持现场整洁。

---

## 📌 移动 GPU 优化技术路线

闭环中的优化改动均基于移动 GPU 架构特征规划：

> 🔒 组织成员：[`HGamey/phonefarm/docs/MOBILE_GPU_OPT_ROUTES.md`](https://github.com/HGamey/phonefarm/blob/main/docs/MOBILE_GPU_OPT_ROUTES.md)
> 🌐 公开镜像：[`BH3GEI/phonefarm/docs/MOBILE_GPU_OPT_ROUTES.md`](https://github.com/BH3GEI/phonefarm/blob/main/docs/MOBILE_GPU_OPT_ROUTES.md)

路线梳理涵盖移动端 TBDR 架构与共享内存带宽特性，分为五大方向：**带宽与 Tile Memory、几何与 Binning、着色器计算、分辨率重构与上采样、时序功耗与散热**。每项技术路线均明确其测量指标与验证条件。

已落盘的关键实测参考：

| 结论 | 说明与实测依据 |
|---|---|
| **带宽瓶颈需依场景实测验证** | 锁定 DDR/LLCC 下限到高频档位后，帧时间稳定性可显著改善，但 GPU 活跃占比未必同向变化，说明瓶颈特征随渲染负载而异 |
| **黑盒应用上限受限帧策略约束** | 在固定帧率封顶（如 30/60 fps）且 GPU 未满载场景中，降低渲染工作量主要转换为整机功耗与发热的优化，而非帧率跃升 |
| **端到端管线考量开销** | 模型或算子的评估需结合系统显存往返、驱动同步与后处理全链路，零拷贝显存挂载可显著压降额外搬运损耗 |

---

## 仓库矩阵与能力分工

| 仓库 | 定位与核心职责 | 主要语言 |
|---|---|---|
| **[phonefarm](https://github.com/BH3GEI/phonefarm)** | 自动化物理 Harness 底座。支持 Android (adb) 与 OpenHarmony (hdc) 设备：硬件时序与功耗遥测、系统旋钮控制、统计显著性裁决及现场快照还原 | Rust |
| **[game_opt_loop](https://github.com/HGamey/game_opt_loop)** 🔒 | 渲染能效与画质自主优化 Harness。覆盖分辨率重构、时序插帧、着色器极简化与自适应锐化多赛道，集成静态安全防爆门禁、大模型变异与 Pareto 优胜归档 | Rust |
| **[refbench](https://github.com/HGamey/refbench)** 🔒 | 白盒基准靶场。提供确定性 Vulkan 渲染管线负载与后处理挂接插槽，瓶颈特征可定向配置，为 Harness 提供已知基准环境 | C++ |
| **[knobs](https://github.com/HGamey/knobs)** 🔒 | 系统与驱动调节层。提供黑盒系统旋钮（调度/频率/热限制）与灰盒 Vulkan Layer 注入能力，实现优化策略的可插拔应用与还原 | C++ |
| **[sr_loop](https://github.com/HGamey/sr_loop)** | 前序端侧轻量神经网络演化实验。探索模型生成、轻量短训与真机标尺代际筛选流程 | Python |

**系统协作分工**：
- **phonefarm** 承担物理执行、硬件遥测与测试环境控制。
- **game_opt_loop** 作为算法演化 Harness，自主编排生成算子并调度 phonefarm 评测。
- **refbench** 与 **knobs** 分别提供确定性基准靶场与系统调优接口。

> phonefarm 上游公开仓库为 [BH3GEI/phonefarm](https://github.com/BH3GEI/phonefarm)；组织内通过私有仓库承载协同开发分支。

---

## 闭环验证流程

系统标准单轮评测回路如下：

```
选定可重复测试负载
      ↓  离散度检验与漂移自检
   采集物理基线（帧时序 / 硬件执行时长 / 总线与功耗遥测）
      ↓
   瓶颈归因与判定规则冻结
      ↓
   下发候选改动并执行 A/B 交替对照
      ↓
   统计学显著性检验（置换检验 / 置信区间）
      ↓  未达标或劣化则自动回退
   设备现场快照逐项校验与恢复
      ↓
   产出结构化评测报告与证据归档
```

---

## 核心设计原则

- **物理实测为准**：核心指标以真机物理遥测为准，不依赖经验推断。
- **规则预先冻结**：评测判定门限在测试运行前冻结，避免后验偏差。
- **确定性与可复现**：环境状态受控，测试报告与数据链路具备可复现性。
- **现场无残留**：系统修改具备原子恢复机制，运行前后状态校验一致。

---

## 快速上手

### 环境需求

- macOS 或 Linux 宿主工作站（Apple Silicon 设备需执行构建代码签名）
- 具备测试权限的移动设备（Android 支持 adb / OpenHarmony 支持 hdc）
- Rust 与 C++ 编译环境

### 运行验证

```bash
# 1. 构建 phonefarm 基础底座
git clone https://github.com/BH3GEI/phonefarm && cd phonefarm
cd src && cargo build --release && cp target/release/phonefarm .. && cd ..
codesign --force --sign - ./phonefarm     # macOS 环境签名
./phonefarm devices                       # 检查设备连接

# 2. 运行 game_opt_loop 算法闭环
git clone https://github.com/HGamey/game_opt_loop && cd game_opt_loop
cargo test                                # 运行防爆门禁与架构单测
cargo run -- --track frame_gen --dry-run  # 离线运行演化验证
```

