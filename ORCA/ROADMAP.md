# 52 周逐周计划

图例：📄 论文/理论　💻 ORCA 代码阅读　🔧 动手实验/贡献　✍️ 输出　`(nP)` = 预计番茄钟数
每周"必做"约 10~12 P；标 **[选做]** 的留给第 3 个番茄钟，落后时优先砍掉。
论文详细信息见 [PAPERS.md](PAPERS.md)，代码位置见 [ORCA-CODE-MAP.md](ORCA-CODE-MAP.md)。

每周都适用的约定（方法说明见 [README §3–§5](README.md#3-每日节奏)）：

- **一个主问题**：每周开始时把本周内容压成一个可观察的问题，写进周复盘和 [学习状态](notes/learning-state.md)。
- **延迟复测**：★★★ 论文和核心模块读完的下一周，周复盘里花 10 min 合上笔记复述一次（计入复盘时间，不另占番茄钟）。
- **迁移测试**：★★★ 论文笔记必须包含一次"用它解释一个没读过的 xform / minidump"。
- **阶段基线**：W1、W14、W27、W43 各写一份不看资料的基线，对应的缓冲周重写并对照。
- **证据门槛**：缓冲周列出的证据没有齐，不进入下一阶段——先缩小范围或补前置，后续选做降级。

---

## Phase 0 · 环境与地图（W1–W2）

目标：能构建、能运行、能观察。先建立"手感"，再读理论。

> 你已有 WHPG 构建环境和 PR 经验，这两周只补 ORCA 特有的部分。

### W1 ORCA 独立构建与单测
- [ ] ✍️ **阶段基线**（在读任何本周资料之前）：不看资料、不问 Claude，画出你现在理解的"SQL → ORCA → 计划"全流程，标出每一步对应的类，存为 `notes/baseline/W01-pipeline.md`；W13 重写对照 `(0.5P)`
- [✓] 🔧 在 `src/backend/gporca` 下做 Debug 构建：`cmake -GNinja -D CMAKE_BUILD_TYPE=Debug -D CMAKE_CXX_FLAGS=-Wno-c++20-compat -H. -Bbuild.debug && ninja -C build.debug`，跑 `cd build.debug && ctest -j8`（Debug 构建会打开更多断言）`(2P)`
- [ ] 🔧 `./server/gporca_test -U CAggTest` 只跑一组；`./server/gporca_test -d ../data/dxl/minidump/<某个>.mdp` 只跑一个 minidump `(1P)`
- [ ] 💻 通读 gporca 的 `README.md`、`StyleGuide.md`、`README.format.md`；浏览目录并对照 [ORCA-CODE-MAP.md](ORCA-CODE-MAP.md) 标出各库 `(2P)`
- [ ] 💻 回看自己的 #114、#128 两个翻译层修复：它们在整条链路的哪一步？如果当时问题在 `libgpopt`，会怎么定位？`(1P)`
- [ ] 📄 Keshav《How to Read a Paper》`(0.5P)`；ORCA 论文（Soliman 2014）第一遍 `(1P)`
- [ ] 🔧 在集群上加载小规模 TPC-H（SF=1），对同一查询分别 `SET optimizer = off / on` 看 EXPLAIN 差异 `(2P)`
- [ ] ✍️ `notes/cheatsheet.md`：ORCA 构建 / 单测 / minidump 命令 `(1P)`

### W2 Memo、Trace 与编码约定
- [ ] 💻 精读 `README.memo.md`，按它的例子在自己的集群上复现一遍，读懂 `Cost Ctxts`、`PRUNED`、`Grp OptCtxts → Best Expr` `(2P)`
- [ ] 🔧 打开 trace GUC（`optimizer_print_query/plan/memo_*`、`optimizer_trace_fallback`），观察一条三表 join 的输出 `(1P)`
- [ ] 🔧 用 lldb 在 `COptimizer::PdxlnOptimize` 下断点单步走一次（`gporca_test -d` 跑 minidump 比 attach 后端方便得多）`(2P)`
- [ ] 💻 ORCA 编码约定：`GPOS_NEW/GPOS_DELETE`、`AddRef/Release`、`GPOS_ASSERT`、匈牙利命名（`mp`、`pexpr`、`pdrgp…`）与新风格（snake_case 变量、`m_` 成员）并存 `(2P)`
- [ ] 🔧 本地跑一次 `src/tools/fmt chk`（clang-format），了解 CI 的格式检查 `(0.5P)`
- [ ] 📄 Hellerstein《Architecture of a Database System》第 4 章 `(2P)`

**M0 检查**：能跑 `ctest` 和单个 minidump；能用调试器跟进优化入口；能按 README.memo.md 读懂一个 memo。

---

## Phase 1 · 基础：经典理论 + ORCA 骨架（W3–W13）

目标：理解 System R → Volcano → Cascades 的演进，并把 ORCA 的主干流程读通。

### W3 System R 与入口调用链
- [ ] 📄 Selinger 1979（★★★，三遍）`(3P)`
- [ ] 📄 快速回顾 PG `src/backend/optimizer/README` 的 join 搜索部分（你已熟悉 planner，只作对照）`(1P)`
- [ ] 💻 调用链：`CGPOptimizer` → `COptTasks` → `COptimizer`，以及 ORCA 失败时 fallback 回 planner 的路径（`gpopt/utils/COptTasks.cpp`、`CMemoryPoolPalloc` 如何把 ORCA 内存池接到 palloc 上）`(3P)`
- [ ] 💻 `gpopt/config/CConfigParamMapping.cpp`：GUC 如何变成 ORCA 的 traceflag 和 xform 开关 `(1P)`
- [ ] ✍️ 笔记：Selinger 自底向上 DP vs PG `standard_join_search` vs ORCA（先记问题，后面回答）`(1P)`

### W4 综述 + libgpos
- [ ] 📄 Chaudhuri 1998 综述（★★★）`(2P)`
- [ ] 💻 libgpos：`CMemoryPool` / `CAutoMemoryPool`、`CRefCount`、`CDynamicPtrArray`、`CHashMap`、`CBitSet` `(4P)`
- [ ] 💻 libgpos：异常机制（`GPOS_RAISE`、`GPOS_TRY/CATCH`）、`CTask`/`CWorker`、`CAutoTrace` `(2P)`
- [ ] ✍️ `notes/libgpos.md`：容器与内存所有权规则（谁 AddRef、谁 Release）`(1P)`
- [ ] **[选做]** Ioannidis 1996 综述

### W5 Volcano 与算子体系
- [ ] 📄 Graefe & McKenna 1993 Volcano Optimizer Generator（★★★）`(3P)`
- [ ] 📄 Graefe 1990 Exchange 算子（理解 Motion 的思想来源）`(1P)`
- [ ] 💻 `CExpression`、`COperator` 类层次：Logical / Physical / Scalar / Pattern；看 `CLogicalGet`、`CLogicalSelect`、`CLogicalInnerJoin`、`CPhysicalHashJoin` `(4P)`
- [ ] ✍️ 画算子分类图（logical ↔ physical 对应关系）`(1P)`

### W6 Cascades 与 Memo
- [ ] 📄 Graefe 1995 Cascades（★★★）`(3P)`
- [ ] 💻 `CMemo`、`CGroup`、`CGroupExpression`、`CGroupProxy`：插入、去重（hash / 等价判断）、group 合并 `(4P)`
- [ ] 🔧 `optimizer_print_memo_after_exploration` 输出，逐行解读一条两表 join 的 memo `(2P)`
- [ ] ✍️ 笔记：Cascades 概念 → ORCA 类名对照表 `(1P)`

### W7 Columbia 与 Job 调度
- [ ] 📄 Xu 1998 Columbia 论文（精读搜索策略与剪枝章节）`(3P)`；Shapiro 2001 上下界剪枝 `(1P)`
- [ ] 💻 `CJob` 体系：`CJobGroupExploration/Implementation/Optimization`、`CJobGroupExpression*`、`CJobTransformation`、`CScheduler`、`CJobFactory` `(4P)`
- [ ] 🔧 打开 job scheduler trace，观察 job 执行顺序 `(1P)`
- [ ] ✍️ 画每类 job 的状态机图 `(1P)`

### W8 ORCA 论文精读 + 引擎
- [ ] 📄 Soliman 2014 ORCA 论文第二、三遍（★★★）`(3P)`
- [ ] 💻 `CEngine`：search stage（`CSearchStage`）、`COptimizerConfig`、优化超时与 xform 集合配置 `(3P)`
- [ ] 💻 `COptimizationContext`、`CCostContext`：最优计划如何在 group 中被记录和提取 `(2P)`
- [ ] ✍️ 写"ORCA 端到端流程"初稿（后面持续修订）`(2P)`

### W9 DXL 与翻译层
- [ ] 💻 输入翻译：`CTranslatorQueryToDXL`（PG Query → DXL）、`CTranslatorDXLToExpr`（DXL → CExpression）`(3P)`
- [ ] 💻 输出翻译：`CTranslatorExprToDXL`、`CTranslatorDXLToPlStmt` `(2P)`
- [ ] 💻 元数据：`CMDAccessor`、MD cache、`CTranslatorRelcacheToDXL`、`IMDRelation` / `IMDType` `(2P)`
- [ ] 📄 Greenplum SIGMOD 2021 论文（了解 ORCA 所处系统）`(1P)`
- [ ] 🔧 挑一个 ORCA 会 fallback 的 SQL 特性，找到抛出 fallback 的代码位置 `(1P)`

### W10 属性框架与计划枚举
- [ ] 💻 属性体系：`CDrvdPropRelational` / `CDrvdPropPlan` / `CDrvdPropScalar`、`CReqdPropRelational` / `CReqdPropPlan`；"required 自顶向下、derived 自底向上" `(5P)`
- [ ] 📄 Waas & Galindo-Legaria 2000 计划计数与采样 `(2P)`
- [ ] 🔧 `optimizer_enumerate_plans` / `optimizer_plan_id`：强制选择第 N 个计划，观察代价差异 `(2P)`
- [ ] ✍️ 笔记：一个 hash join 从 required props 到 enforcer 插入的完整例子 `(1P)`

### W11 Minidump 与测试工作流
- [ ] 🔧 按 gporca README 生成 minidump：`optimizer_minidump = always` + `optimizer_enumerate_plans = on` + `optimizer_enable_constant_expression_evaluation = off`，再 `xmllint --format` 放入 `data/dxl/minidump/` `(1P)`
- [ ] 💻 读懂 `.mdp` 结构（Traceflags / 配置、Metadata、Query、Plan）`(1P)`
- [ ] 🔧 把 minidump 注册到测试：`server/CMakeLists.txt` 中对应测试组，或 `server/src/unittest/gpopt/minidump/CICGTest.cpp` `(1P)`
- [ ] 🔧 故意改一个代价参数让一批 minidump 失败，用 `scripts/fix_mdps.py --logFile ... --dryRun` 查看、再真正更新；最后 `git checkout` 还原 `(2P)`
- [ ] 💻 回归测试中 `*_optimizer.out` 与 planner 版本并存的组织方式（你已熟悉回归框架，重点看 ORCA 特有的部分）`(1P)`
- [ ] 📄 Moerkotte《Building Query Compilers》前两章（建立术语）`(2P)`
- [ ] ✍️ `notes/testing.md` `(1P)`

### W12 第一次读懂并改一个 Xform
- [ ] 💻 完整读 2 个简单 xform：`CXformSelect2Filter`、`CXformGet2TableScan`（pattern、`Exfp` promise、`Transform`）`(3P)`
- [ ] 🔧 本地练习：临时关闭一个 xform（`optimizer_enable_*` GUC 或 traceflag）观察计划变化；再尝试写一个"无害"的练习 xform 并注册到 `CXformFactory`（不提交）`(5P)`
- [ ] ✍️ 笔记：新增一个 xform 需要改哪些地方（checklist）`(1P)`

### W13 🟡 缓冲 + 复盘
- [ ] 补齐前 12 周欠账
- [ ] ✍️ 定稿"ORCA 端到端流程"长文（带图）
- [ ] 💻 案例学习：精读 WHPG 近期 4 个 ORCA 核心库修复的 diff 与测试——#149（`CXformGbAggWithMDQA2Join`：CTE producer 列裁剪）、#152（`CColumnFactory`：`PcrCopy` 保留 EUnused）、#154（`CExpressionPreprocessor`：过时断言）、#183（`CUtils`：ordered-set agg 崩溃），以及 a9b9f1aa1df（`CGroupExpression`：统计推导无限递归）；对每个回答"症状是什么 → 根因在哪一层 → 为什么这样改 → 测试怎么写"
- [ ] 🔧 在 WHPG issue / PR / 邮件列表中收集 5 个候选的核心库问题，作为 W24–W25 的选题池
- [ ] ✍️ 不看资料重写 W1 的阶段基线（全流程图），和 `notes/baseline/W01-pipeline.md` 并排对照，写下变化最大的三处
- [ ] 🔧 迁移测试：挑一个没跑过的 minidump，打开 memo trace，不问 Claude，解释 ORCA 为什么选了这个计划
- [ ] **证据门槛**（齐了才进 Phase 2）：① 重写后的全流程图能标出 DXL → CExpression → Memo → job → 计划提取各步的类；② 至少 3 篇 ★★★ 论文笔记完成了延迟复测；③ 至少 1 个模块做过三次对照（无 AI 基线 → Claude 辅助 → 独立版）
- [ ] **M1 自测**（不看资料回答）：
  - Volcano 与 Cascades 的核心区别？为什么 Cascades 适合 top-down + 剪枝？
  - ORCA 中 group、group expression、optimization context 三者的关系？
  - exploration / implementation / optimization 三类 job 各做什么？依赖关系如何？
  - DXL 存在的意义是什么？为什么 minidump 能离线复现问题？
  - required props 与 derived props 如何配合决定是否插入 enforcer？

---

## Phase 2 · 核心技术（W14–W26）

目标：规则/变换、Join 枚举、物理属性、MPP 分布、统计与基数、代价模型、预处理。W26 前提交第一个 PR。

### W14 Xform 框架深入
- [ ] ✍️ **阶段基线**：不看资料讲清"一个 4 表 join 在 ORCA 里怎样被枚举、选分布、插 Motion、算代价"，存为 `notes/baseline/W14-join.md`；W26 重写对照 `(0.5P)`
- [ ] 💻 `CXformFactory`、`CXform` 分类（exploration / implementation）、`CPatternLeaf/Tree/MultiLeaf`、`CXformUtils`、xform 的 promise 与去重 `(5P)`
- [ ] 📄 Ding, Narasayya, Chaudhuri《Extensible Query Optimizers in Practice》：规则与搜索框架相关章节 `(3P)`
- [ ] 🔧 统计一条 TPC-H Q5 的优化中各 xform 被触发的次数（`optimizer_print_xform` 类 trace）`(2P)`

### W15 Join 枚举理论 I
- [ ] 📄 Ono & Lohman 1990 `(1P)`；Vance & Maier 1996（子集枚举技巧）`(2P)`
- [ ] 📄 Moerkotte & Neumann 2006 DPccp（★★★）`(3P)`
- [ ] 🔧 用 Python / C++ 写 toy：DPsize、DPsub、DPccp，在链 / 星 / 团查询图上比较枚举次数 `(3P)`
- [ ] ✍️ 笔记 + toy 代码放 `experiments/join-enum/` `(1P)`

### W16 Join 枚举理论 II + ORCA 入口
- [ ] 📄 Moerkotte & Neumann 2008 DPhyp（★★★，重点：超图与非内连接）`(3P)`
- [ ] 📄 Steinbrunn 1997 启发式 / 随机化 join 排序 `(2P)`
- [ ] 💻 `CLogicalNAryJoin`、`CXformExpandNAryJoinGreedy` / `MinCard`、`CJoinOrder` 基类 `(3P)`
- [ ] **[选做]** DeHaan & Tompa 2007 top-down join 枚举

### W17 ORCA 的 Join Order
- [ ] 💻 `CJoinOrderDP`、`CJoinOrderDPv2`、`CXformExpandNAryJoinDPv2`；`optimizer_join_order`、`optimizer_join_order_threshold` 等 GUC `(5P)`
- [ ] 🔧 同一 8~10 表查询，比较 query / greedy / exhaustive / exhaustive2 下的计划、代价与优化耗时 `(2P)`
- [ ] 📄 Neumann & Radke 2018 超大 join 查询 `(2P)`
- [ ] ✍️ 笔记：ORCA DPv2 与 DPccp/DPhyp 的异同 `(1P)`

### W18 排序属性与 Enforcer
- [ ] 📄 Simmen 1996 order optimization（★★）`(2P)`；Neumann & Moerkotte 2004 `(2P)`
- [ ] 💻 `COrderSpec`、`CEnfdOrder`、Sort enforcer 插入；`CRewindabilitySpec` 及其用途（NLJ inner、Spool）`(4P)`
- [ ] 🔧 构造 ORDER BY + merge 类需求的查询，观察 sort 被放在哪里 `(1P)`

### W19 MPP 分布式优化
- [ ] 📄 Zhou, Larson, Chaiken 2010 SCOPE 分区与并行计划（★★★）`(3P)`
- [ ] 📄 Shankar 2012 SQL Server PDW 优化器 `(2P)`
- [ ] 💻 `CDistributionSpec` 家族（Hashed / Singleton / Replicated / Random / Any / Strict…）、`CEnfdDistribution`、Motion 算子（Gather / Broadcast / Redistribute）`(4P)`

### W20 分布 Hands-on
- [ ] 💻 `CPhysicalHashJoin` 如何对子节点发出多组分布请求（co-located / broadcast outer / broadcast inner…），以及 `CPhysicalJoin` 的相关逻辑 `(4P)`
- [ ] 🔧 构造不同分布键、不同表大小的 join，观察 redistribute / broadcast 的选择；调整代价参数看翻转点 `(3P)`
- [ ] 🔧 读 `CPhysicalHashAgg` / 两阶段聚合的分布需求 `(2P)`
- [ ] ✍️ 笔记：一条 join 的分布请求 → enforcer → 代价比较全过程 `(1P)`

### W21 统计与基数 I
- [ ] 📄 Ioannidis 2003 直方图历史 `(2P)`；Poosala 1996 `(2P)`；Ioannidis & Christodoulakis 1991 误差传播 `(1P)`
- [ ] 💻 `CStatistics`、`CHistogram`、`CBucket`、`CPoint`；统计信息如何从 `pg_statistic` 翻译进来 `(4P)`
- [ ] ✍️ 笔记：ORCA 直方图与 PG MCV + histogram 的对应 `(1P)`

### W22 统计与基数 II
- [ ] 📄 Leis 2015《How Good Are Query Optimizers, Really?》（★★★）`(3P)`
- [ ] 💻 `CFilterStatsProcessor`、`CJoinStatsProcessor`、`CGroupByStatsProcessor`、damping / scale factor（`CScaleFactorUtils`）、`CStatisticsConfig` `(4P)`
- [ ] 🔧 对 TPC-H / 自造相关列数据，比较 ORCA 估计行数与 `EXPLAIN ANALYZE` 实际行数，找出最离谱的一类谓词 `(3P)`

### W23 代价模型
- [ ] 💻 `CCostModelGPDB`、`CCostModelParamsGPDB`：hash join、NLJ、scan、motion、agg 的代价公式 `(5P)`
- [ ] 📄 Leis 2018《Query Optimization Through the Looking Glass》`(2P)`
- [ ] 🔧 修改一个代价参数，用 minidump 观察计划翻转 `(2P)`
- [ ] ✍️ 笔记：ORCA 代价模型的主要假设与已知局限 `(1P)`

### W24 预处理、规范化与约束推导
- [ ] 💻 `CExpressionPreprocessor` 各步骤（逐步打印前后对比）、`CNormalizer`（谓词下推）、`CPredicateUtils` `(4P)`
- [ ] 💻 约束推导：`CConstraint`、`CConstraintInterval`、`CRange`、`CPropConstraint`；等价类传播 `(3P)`
- [ ] 📄 Moerkotte 书中查询重写相关章节（按需）`(2P)`
- [ ] 🔧 从 W13 的选题池中选定首个核心库 PR 的 issue，完成复现（最好得到 minidump）`(1P)`

### W25 首个核心库 PR 冲刺
- [ ] 🔧 定位根因、修改、补充 minidump 单测和/或回归测试 `(6P)`
- [ ] 🔧 自查：`ctest` 全绿（Debug 构建）、相关回归通过、`src/tools/fmt chk` 通过；写清晰的 PR 描述（问题、根因、方案、测试）`(2P)`
- [ ] 适合首个核心库 PR 的类型：修 assert 失败或崩溃（参考 #154、#183）/ 修列引用或属性推导错误（参考 #152）/ 修某个 xform 的边界条件（参考 #149）/ 修错误的基数估计分支

### W26 🟡 缓冲 + 复盘 + 提交 PR
- [ ] 🔧 提交 PR，及时回应 review
- [ ] ✍️ 写 Phase 2 总结
- [ ] ✍️ 不看资料重写 W14 的阶段基线（4 表 join），和 `notes/baseline/W14-join.md` 对照
- [ ] **证据门槛**（齐了才进 Phase 3）：① 重写的基线能讲清 join 枚举、分布请求、enforcer 和代价各在哪个类里发生；② 第一个核心库 PR 已提交，PR 描述里的根因分析是自己写的；③ Phase 2 内至少 1 次三次对照
- [ ] **M2 自测**：
  - DPccp 为什么比 DPsub 高效？DPhyp 怎样处理 outer join 的重排限制？
  - ORCA 的 hash join 会向子节点请求哪几种分布组合？broadcast 何时胜出？
  - 基数估计误差为什么随 join 数量指数放大？ORCA 用了哪些 damping 手段？
  - `CExpressionPreprocessor` 主要做了哪些重写？为什么要在搜索之前做？
  - 画出一个 enforcer（Sort 或 Motion）被加入 memo 的时机与位置

---

## Phase 3 · 进阶专题（W27–W39）

目标：子查询、聚合、外连接、CTE、分区、窗口、索引等专题；W39 前合入 ≥ 2 个 PR。

### W27 子查询 I：经典去嵌套
- [ ] ✍️ **阶段基线**：不看资料讲清"一个带相关子查询和 `GROUP BY` 的查询在 ORCA 里经过哪些变换"，存为 `notes/baseline/W27-subquery.md`；W39 重写对照 `(0.5P)`
- [ ] 📄 Kim 1982 `(2P)`；Ganski & Wong 1987（COUNT bug）`(2P)`；Seshadri 1996 `(1P)`
- [ ] 💻 `CSubqueryHandler` 总览：标量子查询、EXISTS、IN / ANY / ALL 分别如何处理 `(4P)`
- [ ] ✍️ 笔记：COUNT bug 在 ORCA 中如何被避免 `(1P)`

### W28 子查询 II：Apply 与正交优化
- [ ] 📄 Galindo-Legaria & Joshi 2001（★★★，Apply 算子，与 ORCA 设计高度相关）`(3P)`
- [ ] 📄 Elhemali 2007 子查询执行策略 `(2P)`
- [ ] 💻 `CLogicalApply` 家族、`CXformSubqJoin2Apply`、Apply → Join 的 xform、`CDecorrelator` `(4P)`

### W29 子查询 III：任意去相关
- [ ] 📄 Neumann & Kemper 2015《Unnesting Arbitrary Queries》（★★★）`(3P)`
- [ ] 🔧 构造多层相关子查询，找出 ORCA 无法去相关（留下 SubPlan / 相关 NLJ）的 case，与 PG / DuckDB 对比 `(4P)`
- [ ] ✍️ 笔记：ORCA 去相关能力边界清单（可能成为 capstone 选题）`(1P)`
- [ ] **[选做]** Neumann 2025《Improving Unnesting of Complex Queries》

### W30 聚合优化
- [ ] 📄 Chaudhuri & Shim 1994 `(1P)`；Yan & Larson 1995 Eager / Lazy Aggregation（★★★）`(3P)`；Larson 2002 部分预聚合 `(1P)`
- [ ] 💻 `CXformSplitGbAgg`（多阶段聚合）、`CXformPushGbBelowJoin`、`CXformSplitDQA`（DISTINCT 聚合）、`CLogicalGbAgg` 的变体 `(4P)`
- [ ] 🔧 观察 `COUNT(DISTINCT)` 在不同分布下的计划形态 `(1P)`

### W31 外连接
- [ ] 📄 Galindo-Legaria & Rosenthal 1997（★★★）`(3P)`；Bhargava 1995 `(1P)`
- [ ] 💻 外连接化简（outer → inner）在预处理中的位置；`CLogicalLeftOuterJoin` 相关 xform；DPv2 对 LOJ 的支持方式 `(4P)`
- [ ] 🔧 构造 WHERE 条件使 LOJ 可化简为 inner join，验证 ORCA 是否识别 `(1P)`

### W32 CTE
- [ ] 📄 El-Helw 2015 CTE 优化（★★★，ORCA 团队论文）`(3P)`
- [ ] 💻 `CCTEInfo`、`CLogicalCTEProducer/Consumer/Anchor`、`CXformInlineCTEConsumer`、CTE 相关 GUC；producer/consumer 的分布一致性 `(4P)`
- [ ] 🔧 同一 CTE 被引用 1 次 / 多次时的 inline 决策实验 `(2P)`

### W33 分区表
- [ ] 📄 Antova 2014 分区表优化（★★★，ORCA 团队论文）`(3P)`
- [ ] 💻 `CPartInfo`、`CPartitionPropagationSpec`、`CPhysicalPartitionSelector`、DynamicScan；静态 vs 动态分区裁剪 `(5P)`
- [ ] 🔧 构造 join 触发动态分区裁剪的查询，观察 PartitionSelector 的位置 `(1P)`

### W34 窗口函数与排序类算子
- [ ] 📄 Cao 2012 窗口函数优化 `(2P)`
- [ ] 💻 `CLogicalSequenceProject`、`CPhysicalSequenceProject`、窗口的分布与排序需求；Limit / TopN 相关 `(4P)`
- [ ] 🔧 首个 PR 的后续 or 开始第二个 PR 的复现 `(3P)`

### W35 索引、Bitmap 与 Runtime Filter
- [ ] 💻 `CXformSelect2IndexGet`、`CXformSelect2BitmapBoolOp`、索引 NLJ（`...IndexGetApply` 系列 xform）`(4P)`
- [ ] 📄 Ding 2020 Bitvector-aware 优化 `(2P)`
- [ ] 🔧 第二个 PR：推进 `(3P)`

### W36–W37 第二个贡献（中等 issue）
- [ ] 🔧 选一个真正涉及优化逻辑的 issue（xform / 统计 / 属性推导），不再是翻译层 `(8P/周)`
- [ ] 🔧 流程：复现 → minidump → 定位 → 设计说明（写在 PR 或邮件列表）→ 实现 → 测试
- [ ] 🔧 每周至少 review 1 个他人的 ORCA 相关 PR（先从读懂开始，能提问即可）
- [ ] 📄 每周保留 1P 读论文（从 PAPERS.md 的"选读"中挑与当前 issue 相关的）

### W38 横向对比其他优化器
- [ ] 📄 Apache Calcite SIGMOD 2018 `(2P)`
- [ ] 📄 CockroachDB SIGMOD 2020 中的优化器部分 + 其 optgen 规则 DSL 文档 `(2P)`
- [ ] 💻 浏览 DuckDB 优化器源码（join order、unnesting）`(2P)`
- [ ] ✍️ 对比笔记：ORCA / SQL Server / Calcite / CockroachDB / DuckDB 在搜索框架、规则表达、统计上的差异 `(2P)`

### W39 🟡 缓冲 + 复盘
- [ ] 推动第二个 PR 合入
- [ ] ✍️ 写 Phase 3 总结
- [ ] ✍️ 不看资料重写 W27 的阶段基线（相关子查询 + GROUP BY），和 `notes/baseline/W27-subquery.md` 对照
- [ ] **证据门槛**（齐了才进 Phase 4）：① 重写的基线能列出 Apply 引入、去相关、聚合下推的具体 xform；② 核心库 PR 合入 ≥ 2 个；③ 能独立（不问 Claude）在 1 小时内为一个陌生 bug 生成 minidump 并定位到相关模块
- [ ] **M3 自测**：
  - ORCA 中 Apply 算子的语义？相关子查询从 SQL 到 Join 经历了哪些变换？
  - Eager aggregation 的正确性条件是什么？ORCA 何时把 GbAgg 推到 join 之下？
  - CTE producer 和多个 consumer 如何保证物理属性一致？
  - 动态分区裁剪中 PartitionSelector 为什么必须位于 join 的某一侧？
  - 你能在 1 小时内为一个陌生 bug 生成 minidump 并定位到相关 xform 吗？

---

## Phase 4 · 贡献冲刺与前沿（W40–W52）

目标：完成一个 capstone 改动；建立发现问题的能力；了解前沿方向。

### W40 优化器测试与 Bug 发现
- [ ] 📄 Gu, Soliman, Waas 2012 TAQO（ORCA 团队）`(1P)`
- [ ] 📄 Rigger 等 PQS / NoREC / TLP（SQLancer 系列）`(3P)`；Ba & Rigger QPG 2023、CERT 2024 `(2P)`
- [ ] 🔧 设计差分测试：同一查询 `optimizer = on / off` 对比结果集，以及用 TAQO 思路比较计划排序与实际耗时 `(3P)`

### W41 差分测试落地
- [ ] 🔧 写脚本批量生成 / 收集查询（TPC-DS 模板、回归测试 SQL、随机生成），自动比较结果与耗时 `(5P)`
- [ ] 🔧 分类发现的问题：结果错误 > 崩溃 / assert > 严重性能回退 > fallback；整理成 issue 候选 `(3P)`

### W42 自适应与学习型优化（开阔视野）
- [ ] 📄 Kabra & DeWitt 1998 中途重优化 `(2P)`；Deshpande 2007 AQP 综述（选读章节）`(2P)`
- [ ] 📄 Kipf 2019 Learned Cardinalities `(1P)`；Marcus 2019 Neo `(1P)`；Marcus 2021 Bao `(2P)`
- [ ] 💻 ORCA 的 hint 支持：`libgpopt/include/gpopt/hints/`（`CPlanHint`、`CJoinHint`、`CRowHint`、`CScanHint`），以及 pg_hint_plan（`gpcontrib/pg_hint_plan`）如何把 hint 传给 ORCA `(1P)`
- [ ] ✍️ 笔记：这些思想在 ORCA 中有没有落地可能（例如 Bao 式地学习选择 hint / GUC 组合；`CRowHint` 能否用来注入外部基数估计）`(1P)`

### W43 Capstone 选题与设计
- [ ] 🔧 从以下来源选题：W29 去相关边界、W22 基数误差、W41 差分测试发现、社区 issue / roadmap
- [ ] ✍️ **阶段基线**：先不查资料、不问 Claude，用 30 min 写一页设计草稿（问题、方案、风险、测试计划），存为 `notes/baseline/W43-capstone.md`；W52 对照最终合入的方案 `(0.5P)`
- [ ] 🔧 写设计说明：问题、动机（附 minidump / 查询）、方案、替代方案、影响范围、测试计划 `(6P)`
- [ ] 🔧 在社区（邮件列表 / issue）征求意见 `(1P)`
- [ ] 候选方向示例（以社区实际需求为准）：
  - 消除某类常见 fallback（按 `optimizer_trace_fallback` 统计频率）
  - 改进某类谓词的选择率估计（如 LIKE、数组 `ANY`、`IS DISTINCT FROM`、相关列）
  - 新增 / 增强一个 xform（某种谓词或聚合下推、join 化简）
  - 扩大去相关覆盖面
  - 翻译层支持新的 PostgreSQL 语法 / 节点
  - 降低优化器自身的耗时或内存（profiling 驱动）

### W44–W49 Capstone 实现
- [ ] W44 原型：最小可用实现，跑通目标查询 `(10P)`
- [ ] W45 正确性：补 minidump 单测、回归测试、边界 case（NULL、空表、分区、多分布键）`(10P)`
- [ ] W46 性能：优化时间和执行时间前后对比（TPC-H / TPC-DS）；确认没有计划回退 `(10P)`
- [ ] W47 打磨与提交 PR `(10P)`
- [ ] W48–W49 review 迭代；同时继续 review 他人 PR、处理小 issue `(10P/周)`
- [ ] 📄 每周 1P：《Extensible Query Optimizers in Practice》剩余章节 / Moerkotte 书中与 capstone 相关的章节

### W50 回馈社区
- [ ] ✍️ 把全年笔记整理成可公开的 "ORCA internals" 系列文章，或在社区 / 团队内做一次分享 `(8P)`

### W51 查漏补缺
- [ ] 回看 PAPERS.md 中未读的 ★★ 论文，挑 2~3 篇补读
- [ ] 回看 ORCA-CODE-MAP.md 中尚未读过的模块，补一份概览笔记

### W52 🟡 缓冲 + 年度复盘
- [ ] ✍️ 年度复盘：合入 PR 列表、掌握 / 未掌握的模块、论文清单完成度
- [ ] ✍️ 制定第二年计划（向 reviewer / committer 方向，或深挖某个方向成为该模块的 owner）
- [ ] ✍️ 把 W43 的设计草稿和最终合入的方案并排对照：哪些判断是对的，哪些在 review 中被改掉了，为什么
- [ ] ✍️ 把四份阶段基线和对应的重写版放在一起，写一页"这一年真正改变了什么、仍然反复出错的是什么"
- [ ] **M4 自测**：
  - 能否为一个新需求独立写出设计说明并估算工作量？
  - 能否 review 他人的 xform / 统计相关 PR 并指出潜在问题？
  - 能否向新人讲清楚 ORCA 的搜索流程、属性体系、统计与代价模型？
