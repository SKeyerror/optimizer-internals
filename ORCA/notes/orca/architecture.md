# ORCA 架构图：模块与代码映射

> 图源：Soliman et al., *Orca: A Modular Query Optimizer Architecture for Big Data*, SIGMOD 2014
> （对应 [ROADMAP.md](../../ROADMAP.md) W1、W8；后续各模块详细阅读见 W4–W23）。
> 代码路径均在 WarehousePG（`/Users/zhenglong/workspace/warehouse-pg/src/backend/gporca`）。

```
             DXL Query ──▶                    ◀── DXL Plan
 ┌──────────────────────────────────────────────────────────────┐
 │ Orca                                                          │
 │  ┌───────────────┐        ┌──────────────────────────────┐   │
 │  │    Search     │◀──────▶│  Optimizer Tools         │   │
 │  │  (CEngine)    │        │  Operators                   │   │
 │  │      ▲▼       │        │  Property Enforcement        │   │
 │  │ Job Scheduler │        │  Transformations              │   │
 │  │ (CScheduler,  │        │  Card. Estimation             │   │
 │  │  CJob*)       │        │  Cost Model                   │   │
 │  │      ▲▼       │        │  MD Cache                     │   │
 │  │    Memo       │        └──────────────────────────────┘   │
 │  │ (CMemo/CGroup)│                                            │
 │  └───────────────┘                                            │
 └──────────────────────────────────────────────────────────────┘
 ┌──────────────────────────────────────────────────────────────┐
 │ GPOS: File I/O │ Exception Handling │ Concurrency │ Memory   │
 └──────────────────────────────────────────────────────────────┘
                              OS
```

## 1. 搜索引擎三件套（左侧，互相双向调用）

| 图中框 | 作用 | 代码 |
|---|---|---|
| **Search** | 顶层驱动：把优化过程切成若干 `CSearchStage`（不同阶段用不同 xform 集合/超时），决定何时收敛、提取最终计划 | `CEngine`（`libgpopt/include/gpopt/engine/CEngine.h`） |
| **Job Scheduler** | 把"探索一个 group""对一个 group expression 应用一个 xform""为一个 group 求最优物理计划"拆成 `CJob`，用状态机（`CJobStateMachine`）驱动，支持多线程并行 | `CScheduler`、`CJobFactory`、`CJobGroup*`、`CJobGroupExpression*`、`CJobTransformation`（`libgpopt/include/gpopt/search/`） |
| **Memo** | 动态规划表：`CGroup` 是逻辑等价类，`CGroupExpression` 是组内一个"算子+子 group"节点，去重和最优解都记在这里 | `CMemo`、`CGroup`、`CGroupExpression`、`CGroupProxy`（同目录） |

三者的双向箭头：Search 让 Job Scheduler 跑 job → job 读写 Memo → Search 轮询 Memo 判断是否收敛。

## 2. Optimizer Tools（右侧，被 job 调用的工具库，不控制搜索流程本身）

| 图中框 | 作用 | 代码 |
|---|---|---|
| **Operators** | 算子类层次，是每个 `CGroupExpression` 包的那个"算子"本身。基类 `COperator` 分四支：`CLogical`（代数算子，`CLogicalInnerJoin`/`CLogicalGbAgg`/`CLogicalGet`…）、`CPhysical`（可执行物理算子，`CPhysicalHashJoin`/`CPhysicalSort`/`CPhysicalMotion*`）、`CScalar`（标量表达式/谓词）、`CPattern`（xform 模式匹配用的占位算子，如 `CPatternLeaf`） | `libgpopt/include/gpopt/operators/` |
| **Property Enforcement** | 判断物理算子是否已天然满足所需的排序/分布/回绕/分区传播属性，不满足则插入 enforcer（Sort/Motion/Spool/PartitionSelector）。每个 `CPhysical` 算子实现 `EpetOrder()`/`EpetDistribution()`/`EpetRewindability()`/`EpetPartitionPropagation()`，配合 `CEnfd*`（"要求 + 严格程度"）判断 | `libgpopt/include/gpopt/base/CEnfd*.h`、`CPhysical.h` |
| **Transformations** | xform 框架：`CXform` 基类 + `CXformFactory` 注册表 + 约 150 个具体 xform，分 exploration（逻辑→逻辑）和 implementation（逻辑→物理）。`CJobTransformation` 执行"对一个 group expression 跑一个 xform" | `libgpopt/include/gpopt/xforms/` |
| **Card. Estimation** | 基数/统计推导：`CStatistics`（按列存 `CHistogram`/`CBucket`）+ 按算子类型区分的 `C*StatsProcessor`（`CFilterStatsProcessor`、`CJoinStatsProcessor` 家族、`CGroupByStatsProcessor`…），派生新逻辑表达式统计时组合子节点统计 | `libnaucrates/include/naucrates/statistics/`（不在 `libgpopt`，因为要与 DXL 翻译共享） |
| **Cost Model** | 把已知基数的物理 group expression 换算成代价数字。接口 `ICostModel`，具体实现 `CCostModelGPDB` + 可调参数 `CCostModelParamsGPDB`（每种物理算子一条代价公式），由 `CJobGroupExpressionOptimization` 在算 `CCostContext` 时调用 | 接口在 `libgpopt/include/gpopt/cost/`；实现在独立的 `libgpdbcost/include/gpdbcost/` |
| **MD Cache** | 元数据缓存：`CMDAccessor` 是每次查询的门面，背后是进程级、引用计数的 `CMDCache`（缓存 `IMDRelation`/`IMDType`/`IMDIndex` 等 `IMDCacheObject`，按 `CMDKey`/mdid 索引），避免同一张表元数据被重复解析；数据来源是 DXL Metadata（见 [dxl.md](dxl.md)） | `libgpopt/include/gpopt/mdcache/`；底层通用缓存机制由 GPOS 的 `CCache` 提供 |

## 3. GPOS（平台无关基础库，与"优化"语义无关）

| 图中框 | 作用 | 代码 |
|---|---|---|
| **File I/O** | 文件读写抽象；minidump 落盘走这层 | `libgpos/include/gpos/io/`（`CFileReader`/`CFileWriter`/`COstream` 家族） |
| **Exception Handling** | `CException`、`GPOS_TRY/GPOS_CATCH_EX`。**`CMiniDumper` 接口本身定义在这里**——"异常发生时自动生成 minidump"是异常处理机制的一部分，不是上层业务代码另加的 | `libgpos/include/gpos/error/` |
| **Concurrency Control** | 并发原语（`CSyncHashtable`、`CSyncList`、`CSyncPool`）+ 任务调度（`CTask`/`CWorker`/`CWorkerPoolManager`），是 Job Scheduler 能多线程并行跑 job 的基础 | `libgpos/include/gpos/common/`、`libgpos/include/gpos/task/` |
| **Memory Manager** | 池化内存分配器；所有 `GPOS_NEW(mp)` 最终都走这里 | `libgpos/include/gpos/memory/`（`CMemoryPool`/`CMemoryPoolManager`/`CAutoMemoryPool`） |

## 4. OS

真正的操作系统。整个设计里**只有 GPOS 直接碰系统调用**（文件、线程、内存），`libgpopt`/`libnaucrates`/`libgpdbcost` 都只通过 GPOS 的抽象访问底层资源——这也是 ORCA 能被移植到不同宿主系统（GPDB、当年的 HAWQ）而不改动优化逻辑本身的原因。

## 5. 和 ROADMAP 的对应关系

| 模块 | 周次 |
|---|---|
| GPOS（内存/容器/异常/任务） | W4 |
| Operators | W5 |
| Memo | W6 |
| Job Scheduler | W7 |
| Search（CEngine）、MD Cache、DXL | W8–W9 |
| Property Enforcement | W10, W18–W20 |
| Transformations（xform 框架） | W12, W14 |
| Cost Model | W23 |
| Card. Estimation | W21–W22 |

---

延伸阅读：[dxl.md](dxl.md)（DXL 数据交换格式，MD Cache 的数据来源）
