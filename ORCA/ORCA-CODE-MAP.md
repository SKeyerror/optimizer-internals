# ORCA 源码地图

> 已对照 WarehousePG `main`（`0edf082a5e7`，2026-09）核对：文中类名均有对应头文件，GUC 均在 `guc_gp.c` 中定义。
> 本地源码：`/Users/zhenglong/workspace/warehouse-pg`。头文件在 `libxxx/include/<ns>/<模块>/`，实现在 `libxxx/src/<模块>/`。
> 代码随上游演进，找不到时用 `grep -rn` 搜索，或到 `gpdb-archive` 中查历史。

---

## 1. 整体分层

```
             PostgreSQL / Greenplum 后端 (C)
 ┌──────────────────────────────────────────────────────────┐
 │ parser → rewriter → planner()                            │
 │                       │  optimizer=on                     │
 │                       ▼                                   │
 │  src/backend/gpopt/   (C++ 胶水层，"翻译层")              │
 │   CGPOptimizer / COptTasks                               │
 │   translate/CTranslatorQueryToDXL      Query  → DXL      │
 │   translate/CTranslatorDXLToPlStmt     DXL    → PlannedStmt│
 │   translate/CTranslatorRelcacheToDXL   catalog → MD DXL  │
 │   relcache/CMDProviderRelcache, gpdbwrappers.cpp         │
 └───────────────┬──────────────────────────────────────────┘
                 │ DXL（查询 / 元数据 / 计划的中间表示）
 ┌───────────────▼──────────────────────────────────────────┐
 │  src/backend/gporca/   ORCA 本体（与 PG 解耦）            │
 │   libgpopt     优化器：算子、xform、memo、搜索、属性、翻译 │
 │   libnaucrates DXL、元数据接口、统计信息                  │
 │   libgpdbcost  GPDB 代价模型                              │
 │   libgpos      基础库：内存池、异常、容器、任务、字符串    │
 │   server/      单元测试 (gporca_test) + minidump 测试数据  │
 └──────────────────────────────────────────────────────────┘
```

ORCA 本体**不直接调用任何 PG 函数**，一切外部信息都通过 DXL 和 `CMDAccessor` 进入。这正是 minidump 能离线复现问题的原因。

---

## 2. 一条查询的生命周期（W8 要求能默写）

1. `planner()` 判断 `optimizer=on` → 进入 `CGPOptimizer`/`COptTasks`
2. `CTranslatorQueryToDXL`：PG `Query` → DXL 查询树；遇到不支持的特性抛异常 → **fallback 到 PG planner**
3. `COptimizer::PdxlnOptimize`：
   1. `CTranslatorDXLToExpr`：DXL → `CExpression`（逻辑算子树）
   2. `CExpressionPreprocessor`：规范化、谓词下推、常量折叠、子查询预处理、外连接化简、约束推导……
   3. `CEngine::Init`：把表达式插入 `CMemo`
   4. `CEngine::Optimize`：按 search stage 调度 job
      - **Exploration**：逻辑等价变换（join 交换/结合、NAry join 展开等）
      - **Implementation**：逻辑 → 物理（Get→Scan、Join→HashJoin/NLJ…）
      - **Optimization**：自顶向下传递 required props，计算代价，插入 enforcer（Sort / Motion / Spool），每个 `COptimizationContext` 选出最优
   5. 从 memo 提取最优计划（`CExpression` 物理树）
   6. `CTranslatorExprToDXL`：物理树 → DXL 计划
4. `CTranslatorDXLToPlStmt`：DXL 计划 → `PlannedStmt` → executor

---

## 3. 目录与关键类（按阅读顺序）

### 3.1 libgpos（W4）

| 主题 | 关键类 / 宏 |
|---|---|
| 内存 | `CMemoryPool`、`CAutoMemoryPool`、`GPOS_NEW(mp)`、`GPOS_DELETE`、`GPOS_NEW_ARRAY` |
| 引用计数 | `CRefCount`（`AddRef()` / `Release()`）、`CAutoRef` |
| 容器 | `CDynamicPtrArray`、`CHashMap`、`CHashSet`、`CList`、`CSyncHashtable`、`CBitSet` |
| 异常 | `CException`、`GPOS_RAISE`、`GPOS_TRY / GPOS_CATCH_EX / GPOS_CATCH_END` |
| 任务 | `CTask`、`CWorker`、`CWorkerPoolManager` |
| 调试 | `GPOS_ASSERT`、`CAutoTrace`、`IOstream`、`OsPrint()`、`DbgPrintMixin`（Debug 构建下提供 `DbgStr()`，见 5.2） |

**所有权规则**（最常见的 bug 来源）：谁 `GPOS_NEW` 或拿到返回值，谁负责 `Release()`；把对象放进容器 / 传给构造函数通常意味着转移一份引用，若自己还要继续用，先 `AddRef()`。读每个函数时留意注释里对引用的说明。

### 3.2 libgpopt/operators（W5）

- 基类：`COperator` → `CLogical` / `CPhysical` / `CScalar` / `CPattern`
- 表达式树：`CExpression`（operator + children + 派生属性缓存）
- 常见逻辑算子：`CLogicalGet`、`CLogicalSelect`、`CLogicalProject`、`CLogicalInnerJoin`、`CLogicalLeftOuterJoin`、`CLogicalNAryJoin`、`CLogicalGbAgg`、`CLogicalApply` 家族、`CLogicalCTE*`、`CLogicalSequenceProject`、`CLogicalLimit`、`CLogicalDynamicGet`
- 常见物理算子：`CPhysicalTableScan`、`CPhysicalFilter`、`CPhysicalHashJoin`（及 Inner/LeftOuter/LeftSemi/LeftAnti 变体）、`CPhysicalNLJoin`、`CPhysicalHashAgg`、`CPhysicalStreamAgg`、`CPhysicalSort`、`CPhysicalMotion*`（Gather / Broadcast / HashDistribute / RandomDistribute）、`CPhysicalSpool`、`CPhysicalPartitionSelector`
- 标量：`CScalarIdent`、`CScalarConst`、`CScalarCmp`、`CScalarBoolOp`、`CScalarSubquery*`
- 工具：`CPredicateUtils`、`CUtils`、`CExpressionPreprocessor`、`CNormalizer`、`CDecorrelator`、`CSubqueryHandler`

### 3.3 libgpopt/search（W6–W7）

| 类 | 作用 |
|---|---|
| `CMemo` | 所有 group 的容器；负责插入、去重、group 合并、计划提取 |
| `CGroup` / `CGroupProxy` | 等价类；通过 proxy 加锁访问 |
| `CGroupExpression` | group 中的一个"算子 + 子 group 列表" |
| `CJob` / `CJobFactory` / `CScheduler` | job 抽象、创建、调度（支持并行） |
| `CJobGroupExploration` / `Implementation` / `Optimization` | group 级 job |
| `CJobGroupExpressionExploration` / `Implementation` / `Optimization` | group expression 级 job |
| `CJobTransformation` | 在一个 group expression 上应用一个 xform |
| `CSearchStage` | 分阶段搜索（每阶段一组 xform + 超时 / 代价阈值） |

### 3.4 libgpopt/engine、optimizer、hints（W8, W42）

- `CEngine`：搜索主控；`COptimizer`：对外入口；`COptimizerConfig`：配置（`CEnumeratorConfig`、`CStatisticsConfig`、`CCTEConfig`、`CHint`…）
- `hints/`：`CPlanHint`、`CJoinHint`、`CJoinTypeHint`、`CRowHint`、`CScanHint`——pg_hint_plan 的 hint 在 `COptTasks.cpp` 中被翻译成这些对象传给 ORCA

### 3.5 libgpopt/base：属性体系（W10, W18–W20）

| 类别 | 类 |
|---|---|
| 派生属性 | `CDrvdPropRelational`（输出列、键、FD、约束、分区信息…）、`CDrvdPropPlan`、`CDrvdPropScalar` |
| 需求属性 | `CReqdPropRelational`、`CReqdPropPlan` |
| 优化上下文 | `COptimizationContext`、`CCostContext` |
| 排序 | `COrderSpec`、`CEnfdOrder` |
| 分布 | `CDistributionSpec` 及 `…Hashed` / `…Singleton` / `…Replicated` / `…Random` / `…Any` / `…Strict*` 等，`CEnfdDistribution` |
| 回绕 | `CRewindabilitySpec`、`CEnfdRewindability` |
| 分区传播 | `CPartitionPropagationSpec`、`CPartInfo` |
| 列 | `CColRef`、`CColRefSet`、`CColumnFactory` |
| 约束 | `CConstraint`、`CConstraintInterval`、`CConstraintConjunction`、`CRange`、`CPropConstraint` |
| 其他 | `CKeyCollection`、`CFunctionalDependency`、`CCTEInfo` |

### 3.6 libgpopt/xforms（W12, W14 以后各专题）

- 框架：`CXform`、`CXformExploration`、`CXformImplementation`、`CXformFactory`（注册所有 xform）、`CXformUtils`、`CXformContext`、`CXformResult`
- 一个 xform 的三要素：**pattern**（匹配什么形状）、**`Exfp` promise**（是否值得尝试）、**`Transform`**（产出等价表达式）
- 按专题索引：

| 专题 | 代表 xform / 类 |
|---|---|
| 入门 | `CXformSelect2Filter`、`CXformGet2TableScan`、`CXformProject2ComputeScalar` |
| Join | `CXformInnerJoinCommutativity`、`CXformJoinAssociativity`、`CXformJoinSwap`、`CXformExpandNAryJoin*`（Greedy / MinCard / DP / DPv2）、`CXformImplementInnerJoin`（inner join → hash / NL join；旧的 `InnerJoin2HashJoin/NLJoin` 已删除，`CXform.h` 中留有 `____removed` 占位）、`CJoinOrder*` |
| 聚合 | `CXformGbAgg2HashAgg`、`CXformGbAgg2StreamAgg`、`CXformSplitGbAgg`、`CXformSplitDQA`、`CXformGbAggWithMDQA2Join`、`CXformPushGbBelowJoin`、`CXformCollapseGbAgg`、`CXformSimplifyGbAgg` |
| 子查询 | `CXformSubqJoin2Apply`、`CXformSubqNAryJoin2Apply`、`CXformSelect2Apply`、`CXformProject2Apply`、`CXformInnerApply2InnerJoin`、`CXformLeftSemiApply2LeftSemiJoin`、`CXformImplement*CorrelatedApply`、`CXformSubqueryUnnest` |
| 半连接 / 反连接 | `CXformLeftSemiJoin2HashJoin`、`CXformLeftSemiJoin2InnerJoin`、`CXformLeftAntiSemiJoin2HashJoin`、`CXformLeftAntiSemiJoinNotIn2HashJoinNotIn`、`CXform*SemiJoin*Swap` |
| 外连接 | `CXformLeftOuterJoin2HashJoin`、`CXformRightOuterJoin2HashJoin`、`CXformSimplifyLeftOuterJoin`、`CXformPushDownLeftOuterJoin`、`CXformLeftOuter2InnerUnionAllLeftAntiSemiJoin`、`CXformExpandFullOuterJoin` |
| CTE | `CXformInlineCTEConsumer`、`CXformInlineCTEConsumerUnderSelect`、`CXformCTEAnchor2Sequence`、`CXformImplementCTEProducer/Consumer` |
| 索引 | `CXformSelect2IndexGet`、`CXformSelect2BitmapBoolOp`、`CXformJoin2IndexApply`、`CXformJoin2IndexGetApply`、`CXformJoin2BitmapIndexGetApply`、`CXformLimit2IndexGet`、`CXformMinMax2IndexGet` |

`CXformFactory.cpp` 中当前注册了约 150 个 xform（`grep -c 'Add(GPOS_NEW'`）。

### 3.7 libnaucrates（W9, W21–W22）

- DXL：`CDXLNode`、`CDXLOperator` 家族、`CParseHandler*`（Xerces SAX 解析）、`CDXLUtils`
- 元数据：`CMDAccessor`、`IMDProvider`、`IMDRelation`、`IMDType`、`IMDIndex`、`IMDScalarOp`、`CMDIdGPDB`
- 统计：`CStatistics`、`CHistogram`、`CBucket`、`CPoint`、`CStatsPredUtils`、`CFilterStatsProcessor`、`CJoinStatsProcessor`、`CGroupByStatsProcessor`、`CScaleFactorUtils`

### 3.8 libgpdbcost（W23）

- `CCostModelGPDB`（各物理算子的代价函数）、`CCostModelParamsGPDB`（可调参数）

### 3.9 翻译层 src/backend/gpopt（W3, W9）

> 你之前的 ORCA 修复（#114、#128）都在这一层。

- `CGPOptimizer.cpp`、`utils/COptTasks.cpp`：入口、错误处理、fallback、hint 翻译
- `utils/CMemoryPoolPalloc*.cpp`：ORCA 内存池在后端中基于 palloc 实现
- `config/CConfigParamMapping.cpp`：GUC → ORCA traceflag / xform 开关的映射
- `relcache/CMDProviderRelcache.cpp`：元数据提供者
- `translate/CTranslatorQueryToDXL.cpp`、`CTranslatorScalarToDXL.cpp`、`CTranslatorDXLToPlStmt.cpp`、`CTranslatorDXLToScalar.cpp`、`CTranslatorRelcacheToDXL.cpp`
- `gpdbwrappers.cpp`：ORCA 调用 PG 函数的唯一出口（带异常转换）
- 相关 GUC 定义：`src/backend/utils/misc/guc_gp.c`（`grep -n optimizer_ guc_gp.c`）

---

## 4. 命名约定（匈牙利前缀）

ORCA 老代码使用匈牙利命名；`StyleGuide.md` 规定新代码变量用 snake_case、成员加 `m_`，但为避免大规模改名，类名、文件名和大部分函数名仍保留旧风格。两种风格会长期并存，读代码时先"解码"：

| 前缀 | 含义 | 例子 |
|---|---|---|
| `mp` | memory pool（极少数旧代码仍为 `pmp`） | `CMemoryPool *mp` |
| `pexpr` | `CExpression *` | `pexprOuter` |
| `pop` | `COperator *` | `popJoin` |
| `pgexpr` | `CGroupExpression *` | |
| `pcr` / `pcrs` | `CColRef *` / `CColRefSet *` | `pcrsOutput` |
| `pdrgp…` | dynamic array of pointers | `pdrgpexpr`、`pdrgpcr` |
| `pdxln` | `CDXLNode *` | |
| `pds` / `pos` / `prs` | distribution / order / rewindability spec | |
| `prpp` / `prprel` | required plan / relational props | |
| `pdp…` | derived props | `pdprel` |
| `pstats` | `IStatistics *` | |
| `ul` / `f` / `d` | ULONG / BOOL / DOUBLE（旧风格） | |
| `m_` | 成员变量 | `m_pmemo` |
| `C` / `I` | 类 / 接口 | `CMemo`、`IMDRelation` |
| `E…` | 枚举 | `EOperatorId`、`EXformId` |

函数名前缀通常表示返回类型：`PexprXxx()` 返回表达式，`FXxx()` 返回 BOOL，`UlXxx()` 返回 ULONG，`PdsXxx()` 返回分布 spec。

---

## 5. 调试与观察

### 5.1 常用 trace GUC（需 `SET client_min_messages = log;`）

```sql
SET optimizer = on;
SET optimizer_trace_fallback = on;                    -- 打印 fallback 原因
SET optimizer_print_query = on;                       -- 预处理前后的逻辑树
SET optimizer_print_plan = on;                        -- 最终物理计划
SET optimizer_print_memo_after_exploration = on;
SET optimizer_print_memo_after_implementation = on;
SET optimizer_print_memo_after_optimization = on;
SET optimizer_print_optimization_context = on;        -- 配合 memo 看每个 group 的最优选择（见 README.memo.md）
SET optimizer_print_xform = on;                       -- xform 输入输出
SET optimizer_print_xform_results = on;
SET optimizer_print_job_scheduler = on;
SET optimizer_print_expression_properties = on;
SET optimizer_print_group_properties = on;
SET optimizer_print_optimization_stats = on;          -- 优化耗时、memo 大小等
SET optimizer_print_missing_stats = on;
SET optimizer_enumerate_plans = on;                   -- 配合 optimizer_plan_id 取第 N 个计划
SET optimizer_minidump = always;                      -- 生成 minidump
SET optimizer_join_order = exhaustive2;               -- query / greedy / exhaustive / exhaustive2（默认）
```

> 以上 GUC 均已在 WHPG `guc_gp.c` 中核实。另有约 50 个 `optimizer_enable_*` GUC 可开关特定 xform / 算子，做对比实验很好用：
> `grep -o '"optimizer_enable_[a-z_]*"' src/backend/utils/misc/guc_gp.c`。

### 5.2 调试器

- **优先离线调试**：`lldb -- ./server/gporca_test -d ../data/dxl/minidump/Foo.mdp`，不用 attach 后端进程，可反复运行
- 常用断点：`COptimizer::PdxlnOptimize`、`CEngine::Optimize`、`CExpressionPreprocessor::PexprPreprocess`、具体 xform 的 `Transform`、`gpos::CException::Raise`（抓 fallback / 报错源头）
- 打印结构：Debug 构建下，继承了 `DbgPrintMixin` 的类（`CExpression`、`CMemo`、`CGroup`、`CGroupExpression`、`COptimizationContext`、`CColRef`、`CStatistics`、`CHistogram` 等）提供 `DbgStr()`：
  ```
  (lldb) settings set escape-non-printables false
  (lldb) p pexpr->DbgStr()
  ```
- 必须在集群上调试时：先 `SELECT pg_backend_pid();`，再 attach 到该进程

---

## 6. 测试工作流

### 6.1 构建与运行单测（在 `src/backend/gporca` 下）

```bash
cmake -GNinja -D CMAKE_BUILD_TYPE=Debug -H. -Bbuild.debug   # Debug 构建断言更全
ninja -C build.debug
cd build.debug
ctest -j8 --output-on-failure
ctest -j8 --rerun-failed --output-on-failure
./server/gporca_test -U CAggTest                            # 只跑一个测试类
./server/gporca_test -d ../data/dxl/minidump/TVFRandom.mdp  # 只跑一个 minidump
```

测试类定义在 `server/src/unittest/`（按 `gpopt/{base,engine,operators,search,xforms,minidump…}` 分目录），在 `server/CMakeLists.txt` 中用 `add_orca_test(...)` 注册；minidump 类测试（如 `CICGTest`、`CAggTest`、`CSubqueryTest`、`CCTETest`、`CJoinOrderDPTest`）各自列出要跑的 `.mdp`。

### 6.2 新增 minidump 测试

```sql
SET client_min_messages = 'log';
SET optimizer = on;
SET optimizer_enumerate_plans = on;
SET optimizer_minidump = always;
SET optimizer_enable_constant_expression_evaluation = off;
-- 执行查询，minidump 写入 $COORDINATOR_DATA_DIRECTORY/minidumps/
```

```bash
xmllint --format $COORDINATOR_DATA_DIRECTORY/minidumps/Minidump_xxx.mdp > data/dxl/minidump/MyTest.mdp
# 然后在 server/CMakeLists.txt 对应测试组（或 CICGTest.cpp）中加入 MyTest
```

`.mdp` 是 XML：包含 traceflags / 配置、完整元数据（表、类型、算子、统计信息）、DXL 查询、ORCA 产出的 DXL 计划。当前 `data/dxl/minidump/` 下约有 1200 个，是很好的学习素材：挑与当前专题同名的 minidump 跑一遍、看计划。

### 6.3 更新期望计划

改动导致已有 minidump 的计划变化时：

```bash
ctest -j8 --rerun-failed --output-on-failure | tee /tmp/failures.out
../scripts/fix_mdps.py --logFile /tmp/failures.out --dryRun   # 先看
../scripts/fix_mdps.py --logFile /tmp/failures.out            # 再覆盖（先 commit，便于看 diff）
```

逐个确认计划变化是合理的——这一步是 review 时最容易被追问的地方。

### 6.4 提交前

- `src/tools/fmt chk`（clang-format，CI 强制）；`src/tools/tidy`（clang-tidy，见 `README.tidy.md`）
- 回归：`make installcheck` 系列；ORCA 相关期望文件多以 `*_optimizer.out` 与 planner 版本并存
- CONTRIBUTING 要求所有修复都附回归测试；ORCA 改动最好 minidump 单测 + SQL 回归两者都有

### 6.5 一个标准的 bug 修复流程

```
复现 SQL → 生成 minidump → gporca_test -d 离线复现 → trace / lldb 定位到 xform、stats、属性推导或 translator
→ 修改 → 新增 minidump 单测（必要时 fix_mdps.py 更新已有期望）→ 补 SQL 回归 → ctest + installcheck + fmt chk
→ PR（问题 / 根因 / 方案 / 测试）
```

参考案例（WHPG 近期的核心库修复，建议逐个读 diff）：#149 `CXformGbAggWithMDQA2Join`、#152 `CColumnFactory`、#154 `CExpressionPreprocessor`、#183 `CUtils`、a9b9f1aa1df `CGroupExpression`、dc47771f112 `CGroup`。

---

## 7. 常用检索命令

```bash
# 找到一个 xform 的所有相关位置（定义、注册、GUC 开关）
grep -rn "CXformPushGbBelowJoin" src/backend/gporca src/backend/gpopt

# 查看所有 xform 的注册
grep -n "Add(GPOS_NEW" src/backend/gporca/libgpopt/src/xforms/CXformFactory.cpp

# 查看一个算子 ID 在哪里被 switch 处理
grep -rn "EopLogicalGbAgg" src/backend/gporca/libgpopt | head -50

# 查 fallback 原因的抛出点
grep -rn "ExmiQuery2DXLUnsupportedFeature" src/backend/gpopt

# 追某段代码的历史设计
git log --follow -p -- src/backend/gporca/libgpopt/src/xforms/CXformSplitGbAgg.cpp
```
