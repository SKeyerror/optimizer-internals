# 查询优化论文清单

优先级：
- ★★★ **必读**：三遍精读，写完整笔记，能复述算法并映射到 ORCA 代码
- ★★ **推荐**：读两遍，写简短笔记
- ★ **选读**：按兴趣 / 当前 issue 需要阅读

"周"列对应 [ROADMAP.md](ROADMAP.md) 中的安排。论文都可通过标题在 ACM DL / VLDB / DBLP / Google Scholar 找到；多数作者主页有免费 PDF。

---

## 0. 书籍与课程（贯穿全年，按需查阅）

| 资料 | 用途 |
|---|---|
| Hellerstein, Stonebraker, Hamilton, *Architecture of a Database System*, FnT DB 2007 | 第 4 章：优化器在整个系统中的位置（W2） |
| Guido Moerkotte, *Building Query Compilers*（作者主页免费草稿） | 优化器"百科全书"：代数、重写、join 枚举、基数估计，按需查章节 |
| Ding, Narasayya, Chaudhuri, *Extensible Query Optimizers in Practice*, FnT DB 2024 | 工业级可扩展优化器（Cascades 系）的现代综述，与 ORCA 最贴近（W14、W44+） |
| CMU 15-721 *Advanced Database Systems*（Andy Pavlo）优化器相关讲次 | 视频 + 阅读清单，适合作为论文前的"导读" |
| CMU 15-799 *Query Optimization*（Spring 2025, Andy Pavlo） | 专门的查询优化课程，阅读清单可与本清单互补 |
| PostgreSQL `src/backend/optimizer/README`、Suzuki《The Internals of PostgreSQL》 | 作为 ORCA 的对照系（W3） |
| S. Keshav, *How to Read a Paper*, SIGCOMM CCR 2007 | 读论文方法（W1） |

---

## 1. 基础与综述

| ★ | 论文 | 周 | 关注点 | ORCA 映射 |
|---|---|---|---|---|
| ★★★ | Selinger et al., *Access Path Selection in a Relational Database Management System*, SIGMOD 1979 | W3 | 自底向上 DP、interesting orders、代价 = I/O + CPU、选择率默认值 | `CJoinOrderDP`；属性思想的源头 |
| ★★★ | Chaudhuri, *An Overview of Query Optimization in Relational Systems*, PODS 1998 | W4 | 搜索空间 / 代价模型 / 枚举算法三要素的全景 | 全局 |
| ★ | Ioannidis, *Query Optimization*, ACM Computing Surveys 1996 | W4 | 另一份经典综述，偏理论 | — |

## 2. 优化器框架：Volcano / Cascades / ORCA

| ★ | 论文 | 周 | 关注点 | ORCA 映射 |
|---|---|---|---|---|
| ★★★ | Graefe & McKenna, *The Volcano Optimizer Generator: Extensibility and Efficient Search*, ICDE 1993 | W5 | 规则驱动、top-down + memoization、physical property 与 enforcer | xform、`CEnfd*` |
| ★★ | Graefe, *Encapsulation of Parallelism in the Volcano Query Processing System*, SIGMOD 1990 | W5 | Exchange 算子 | Motion 算子 |
| ★★★ | Graefe, *The Cascades Framework for Query Optimization*, IEEE Data Eng. Bull. 1995 | W6 | task 栈、group / expression、rule promise、guidance | `CMemo`、`CGroup`、`CJob*`、`Exfp` |
| ★★ | Xu, *Efficiency in the Columbia Database Query Optimizer*, MS Thesis, Portland State 1998 | W7 | Cascades 的具体实现、group pruning、lower bound | `CJob*` 状态机、代价剪枝 |
| ★ | Shapiro et al., *Exploiting Upper and Lower Bounds in Top-Down Query Optimization*, IDEAS 2001 | W7 | 上下界剪枝 | `CEngine` 剪枝 |
| ★★★ | Soliman et al., *Orca: A Modular Query Optimizer Architecture for Big Data*, SIGMOD 2014 | W1, W8 | DXL、memo、并行 job 调度、多阶段优化、minidump / TAQO 测试 | **全部** |
| ★★ | Waas & Galindo-Legaria, *Counting, Enumerating, and Sampling of Execution Plans in a Cost-Based Query Optimizer*, SIGMOD 2000 | W10 | 从 memo 中计数 / 按编号取计划 | `optimizer_enumerate_plans`、`CTreeMap` |
| ★★ | Lyu et al., *Greenplum: A Hybrid Database for Transactional and Analytical Workloads*, SIGMOD 2021 | W9 | ORCA 所在系统的整体架构 | 系统背景 |

## 3. Join 枚举

| ★ | 论文 | 周 | 关注点 | ORCA 映射 |
|---|---|---|---|---|
| ★★ | Ono & Lohman, *Measuring the Complexity of Join Enumeration in Query Optimization*, VLDB 1990 | W15 | 链 / 星 / 团查询图的搜索空间大小 | — |
| ★★ | Vance & Maier, *Rapid Bushy Join-Order Optimization with Cartesian Products*, SIGMOD 1996 | W15 | 位集子集枚举技巧 | `CJoinOrderDP` |
| ★★★ | Moerkotte & Neumann, *Analysis of Two Existing and One New Dynamic Programming Algorithm for the Generation of Optimal Bushy Join Trees without Cross Products*, VLDB 2006（DPccp） | W15 | csg-cmp 对枚举，DPsize/DPsub/DPccp 复杂度对比 | `CJoinOrderDPv2` |
| ★★★ | Moerkotte & Neumann, *Dynamic Programming Strikes Back*, SIGMOD 2008（DPhyp） | W16 | 超图、复杂谓词、非内连接 | NAry join + LOJ 处理 |
| ★★ | Steinbrunn, Moerkotte, Kemper, *Heuristic and Randomized Optimization for the Join Ordering Problem*, VLDB Journal 1997 | W16 | 贪心、GOO、KBZ、随机化 | `CJoinOrderGreedy/MinCard` |
| ★ | DeHaan & Tompa, *Optimal Top-Down Join Enumeration*, SIGMOD 2007 | W16 | top-down 枚举与 Cascades 的结合 | — |
| ★ | Fender & Moerkotte 等关于 top-down 分区枚举（MinCut 系列）的论文, ICDE 2011–2013 | W16 | 同上，更高效的 top-down 枚举 | — |
| ★★ | Neumann & Radke, *Adaptive Optimization of Very Large Join Queries*, SIGMOD 2018 | W17 | 按查询规模自适应选择枚举策略 | `optimizer_join_order_threshold` 的思想 |
| ★ | Moerkotte, Fender, Eich, *On the Correct and Complete Enumeration of the Core Search Space*, SIGMOD 2013 | W31 | 非内连接重排的正确性 | LOJ 重排 |

## 4. 物理属性与 MPP 分布式优化

| ★ | 论文 | 周 | 关注点 | ORCA 映射 |
|---|---|---|---|---|
| ★★ | Simmen, Shekita, Malkemus, *Fundamental Techniques for Order Optimization*, SIGMOD 1996 | W18 | 基于 FD / 等价类的 order 化简 | `COrderSpec`、`CFunctionalDependency` |
| ★ | Neumann & Moerkotte, *An Efficient Framework for Order Optimization*, ICDE 2004 | W18 | 有限状态机表示 order | — |
| ★★★ | Zhou, Larson, Chaiken, *Incorporating Partitioning and Parallel Plans into the SCOPE Optimizer*, ICDE 2010 | W19 | 分区属性、分布需求推导、exchange 作为 enforcer | `CDistributionSpec*`、Motion |
| ★★ | Shankar et al., *Query Optimization in Microsoft SQL Server PDW*, SIGMOD 2012 | W19 | MPP 中 data movement 的代价化枚举 | Motion 代价 |
| ★ | Kossmann, *The State of the Art in Distributed Query Processing*, ACM Computing Surveys 2000 | W19 | 分布式查询处理综述 | — |

## 5. 基数估计与统计

| ★ | 论文 | 周 | 关注点 | ORCA 映射 |
|---|---|---|---|---|
| ★★ | Ioannidis, *The History of Histograms (abridged)*, VLDB 2003 | W21 | 直方图类型的演进 | `CHistogram`、`CBucket` |
| ★★ | Poosala, Haas, Ioannidis, Shekita, *Improved Histograms for Selectivity Estimation of Range Predicates*, SIGMOD 1996 | W21 | 等深 / V-optimal / MaxDiff | 同上 |
| ★★ | Ioannidis & Christodoulakis, *On the Propagation of Errors in the Size of Join Results*, SIGMOD 1991 | W21 | 误差随 join 数指数放大 | damping 的动机 |
| ★★★ | Leis et al., *How Good Are Query Optimizers, Really?*, VLDB 2015 | W22 | JOB 基准；基数估计才是主要矛盾 | `C*StatsProcessor` |
| ★★ | Leis et al., *Query Optimization Through the Looking Glass, and What We Found Running the Join Order Benchmark*, VLDB Journal 2018 | W23 | 代价模型、计划空间、基数的相对影响 | `CCostModelGPDB` |
| ★ | Stillger, Lohman, Markl, Kandil, *LEO – DB2's LEarning Optimizer*, VLDB 2001 | W22 | 反馈式统计修正 | — |
| ★ | Ilyas et al., *CORDS: Automatic Discovery of Correlations and Soft Functional Dependencies*, SIGMOD 2004 | W22 | 相关列发现 | 相关列选择率 |
| ★ | Leis et al., *Cardinality Estimation Done Right: Index-Based Join Sampling*, CIDR 2017 | W22 | 基于采样的 join 基数估计 | — |
| ★ | Flajolet et al., *HyperLogLog*, AofA 2007 | W22 | NDV 估计 | NDV 相关 |

## 6. 子查询与去相关

| ★ | 论文 | 周 | 关注点 | ORCA 映射 |
|---|---|---|---|---|
| ★★ | Kim, *On Optimizing an SQL-like Nested Query*, TODS 1982 | W27 | 嵌套查询分类与改写 | `CSubqueryHandler` |
| ★★ | Ganski & Wong, *Optimization of Nested SQL Queries Revisited*, SIGMOD 1987 | W27 | COUNT bug | 标量子查询 + outer join 处理 |
| ★ | Seshadri, Pirahesh, Leung, *Complex Query Decorrelation*, ICDE 1996 | W27 | magic decorrelation | — |
| ★★★ | Galindo-Legaria & Joshi, *Orthogonal Optimization of Subqueries and Aggregation*, SIGMOD 2001 | W28 | Apply 算子、将去相关拆成正交规则 | `CLogicalApply*`、`CDecorrelator` |
| ★★ | Elhemali, Galindo-Legaria, Grabs, Joshi, *Execution Strategies for SQL Subqueries*, SIGMOD 2007 | W28 | 子查询执行策略的代价化选择 | Apply → Join / NLJ |
| ★★★ | Neumann & Kemper, *Unnesting Arbitrary Queries*, BTW 2015 | W29 | 依赖 join 的通用去相关 | ORCA 去相关的能力边界 |
| ★ | Neumann, *Improving Unnesting of Complex Queries*, BTW 2025 | W29 | 上文的改进 | — |

## 7. 聚合、外连接、CTE、分区、窗口等专题

| ★ | 论文 | 周 | 关注点 | ORCA 映射 |
|---|---|---|---|---|
| ★ | Chaudhuri & Shim, *Including Group-By in Query Optimization*, VLDB 1994 | W30 | group-by 与 join 交换 | — |
| ★★★ | Yan & Larson, *Eager Aggregation and Lazy Aggregation*, VLDB 1995 | W30 | 聚合下推的正确性条件 | `CXformPushGbBelowJoin` |
| ★★ | Larson, *Data Reduction by Partial Preaggregation*, ICDE 2002 | W30 | 部分预聚合 | `CXformSplitGbAgg`（多阶段聚合） |
| ★★★ | Galindo-Legaria & Rosenthal, *Outerjoin Simplification and Reordering for Query Optimization*, TODS 1997 | W31 | outer → inner 化简、结合律条件 | 预处理中的 LOJ 化简 |
| ★ | Bhargava, Goel, Iyer, *Hypergraph Based Reorderings of Outer Join Queries with Complex Predicates*, SIGMOD 1995 | W31 | 外连接重排 | — |
| ★★★ | El-Helw et al., *Optimization of Common Table Expressions in MPP Database Systems*, VLDB 2015 | W32 | CTE producer / consumer、inline 决策 | `CLogicalCTE*` |
| ★★★ | Antova et al., *Optimizing Queries over Partitioned Tables in MPP Systems*, SIGMOD 2014 | W33 | PartitionSelector、动态分区裁剪 | `CPhysicalPartitionSelector` |
| ★★ | Cao et al., *Optimization of Analytic Window Functions*, VLDB 2012 | W34 | 窗口排序 / 分区需求的合并 | `CLogicalSequenceProject` |
| ★★ | Ding et al., *Bitvector-aware Query Optimization for Decision Support Queries*, SIGMOD 2020 | W35 | runtime filter 感知的优化 | runtime filter 相关 |

## 8. 其他系统（横向对比，W38）

| ★ | 论文 | 关注点 |
|---|---|---|
| ★★ | Begoli et al., *Apache Calcite*, SIGMOD 2018 | Volcano 风格规则 + trait（与 ORCA 属性体系对比） |
| ★★ | Taft et al., *CockroachDB: The Resilient Geo-Distributed SQL Database*, SIGMOD 2020 | Cascades 风格优化器 + optgen DSL |
| ★ | Armbrust et al., *Spark SQL*, SIGMOD 2015 | Catalyst 规则框架 |
| ★ | Chang et al., *HAWQ: A Massively Parallel Processing SQL Engine in Hadoop*, SIGMOD 2014 | 另一个使用 ORCA 的系统 |

## 9. 优化器测试（W40）

| ★ | 论文 | 关注点 |
|---|---|---|
| ★★ | Gu, Soliman, Waas, *Testing the Accuracy of Query Optimizers*, DBTest 2012（TAQO） | 计划排序与实际耗时的相关性（ORCA 团队） |
| ★★ | Rigger & Su, *Testing Database Engines via Pivoted Query Synthesis*, OSDI 2020（PQS） | 逻辑 bug 发现 |
| ★★ | Rigger & Su, *Detecting Optimization Bugs in Database Engines via Non-Optimizing Reference Engine Construction*, ESEC/FSE 2020（NoREC） | 专门针对优化 bug |
| ★★ | Rigger & Su, *Finding Bugs in Database Systems via Query Partitioning*, OOPSLA 2020（TLP） | 三值逻辑分区 |
| ★ | Ba & Rigger, *Testing Database Engines via Query Plan Guidance*, ICSE 2023（QPG） | 以计划多样性引导测试 |
| ★ | Ba & Rigger, *CERT: Finding Performance Issues in Database Systems Through the Lens of Cardinality Estimation*, ICSE 2024 | 基数估计导致的性能问题 |

## 10. 前沿：自适应与学习型优化（W42，开阔视野）

| ★ | 论文 | 关注点 |
|---|---|---|
| ★★ | Kabra & DeWitt, *Efficient Mid-Query Re-Optimization of Sub-Optimal Query Execution Plans*, SIGMOD 1998 | 运行时重优化 |
| ★ | Deshpande, Ives, Raman, *Adaptive Query Processing*, FnT DB 2007 | AQP 综述 |
| ★★ | Kipf et al., *Learned Cardinalities: Estimating Correlated Joins with Deep Learning*, CIDR 2019 | 学习型基数估计 |
| ★ | Marcus et al., *Neo: A Learned Query Optimizer*, VLDB 2019 | 端到端学习型优化器 |
| ★★ | Marcus et al., *Bao: Making Learned Query Optimization Practical*, SIGMOD 2021 | 学习选择 hint 组合，最容易落到现有优化器 |

---

## 统计

- ★★★ 必读：15 篇 —— 全年核心，约 50~60 P
- ★★ 推荐：约 30 篇 —— 约 60 P
- ★ 选读：按需

## 阅读进度

在每篇论文笔记的文件名中使用统一格式，例如 `notes/papers/1995-graefe-cascades.md`，便于检索与统计完成度。
