# UE5_Entity_Stress_Optimizer
基于 Unreal Engine 5 的海量实体极限性能压测与底层调度系统

> **演示视频**: ![演示视屏截图](images/game_screenshots.jpg)

https://www.bilibili.com/video/BV1ZGLK6pEVZ/

## 📌 项目简介
本项目是一个针对 UE5 海量同屏实体（400+）引发的 Tick 阻塞、$O(N^2)$ 物理重叠风暴及内存 GC 卡顿问题，而自研的极致轻量化 3D 动作游戏测试底盘。
通过运用软件工程思想，从内存复用、逻辑调度、物理降维三个核心层面进行了深度优化，在受限硬件下实现了极高的性能跃升。

## 🚀 核心优化策略

### 1. 内存复用与生命周期接管 (Object Pool Subsystem)
针对高频射弹与杂兵生成导致的内存碎片与 GC (垃圾回收) 引发的帧率毛刺，基于 `UGameInstanceSubsystem` 设计了全局对象池。
* **策略**: 拦截原生 `Destroy()` 调用，转为 `Recycle()` 逻辑。
* **实现**: 提取了关键的重置接口，在对象压入/弹出池时，清理残余状态与物理速度，彻底消除野指针隐患，极大降低了引擎的 GC 压力。

### 2. 逻辑层分级调度 (Graceful Degradation & Tick LOD)
抛弃了高消耗的全局 NavMesh，运用 3D 向量数学针对远距离实体实施轻量级平移计算。
* **Tick LOD 降频**: 设计了基于视距的 `Tick LOD` 调度器。
* **算法**: 通过帧切片与整数取模算法，动态切换 `SetActorTickInterval`，实现屏幕外实体的阶梯式平滑降频，显著突破 CPU 算力瓶颈。

### 3. 物理碰撞矩阵降维 ($O(N^2)$ 阻断)
* **策略**: 重新设计 Collision Profile 规则与碰撞过滤策略。
* **优化**: 将海量杂兵交互规则降级为 `Ignore`，并配合物理移动时的 Sweep 检测保障健壮性，斩断了大规模重叠带来的 CPU 物理线程满载开销。

## 📊 性能验证与遥测
自研轻量级性能探针，利用内存缓冲机制异步导出 CSV 压测数据。
* **压测基准**: 开启完整碰撞、血量计算与移动逻辑。
* **优化成果**: 在同屏 400 实体规模下，稳态帧率由 **10 FPS** 跃升至稳定 **45-50 FPS**。

*![实时帧率分析](images/FPS_analyze.jpg)

## 📁 核心源码展示 (本项目仅展示核心机制逻辑)
* `ObjectPoolManager`：内存复用管理系统。
* `TickSchedulerSubsystem`：实体的动态分级调度器。
* `BTTask_PrepareCharge`：基于 C++ 与黑板数据的高速冲锋行为树节点。

![Boss冲锋行为树逻辑](images/BT_Boss.jpg)
核心机制：C++ 与黑板数据驱动的混合 AI 架构
摒弃了纯蓝图连线造成的“面条代码 (Spaghetti Code)”。将底层的距离运算、向量推导与物理组件接管交由 C++ 节点 (BTTask_PrepareCharge) 处理；而蓝图层仅作为状态机控制流 (Selector/Sequence) 与黑板数据 (Blackboard) 的可视化配置面板。实现了程序逻辑与策划层面的完美解耦，极大提升了 AI 迭代效率。

![GameMode 刷怪计时器](images/BP_SpawnBoss.jpg)
核心机制：基于 Timer 的无阻塞生命周期控制
在 GameMode 层面统筹宏观游戏节奏。利用引擎底层的定时器 (Timer) 替代全局 Tick 进行低开销的挂起与唤醒，结合安全的实体唯一性校验，确保全局生成管线的高效流转与内存防溢出。

![血条广播与击杀掉落](images/BP_BossHealthBar.jpg)
核心机制：基于多播委托 (Multicast Delegate) 的异步事件通信
严格遵循 UI 性能规范，坚决摒弃 Event Tick 轮询读值。血条刷新、飘字反馈与战利品掉落逻辑，全部通过动态绑定底层的 OnHealthChanged C++ 委托来实现。仅在状态发生突变时由事件驱动触发 (Event-Driven)，彻底消除了 UI 蓝图与业务逻辑层的无效 Tick 开销。
