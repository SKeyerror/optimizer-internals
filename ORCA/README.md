# ORCA Contributor 一年学习路径

> 目标：12 个月内成为 ORCA（GPORCA）的 contributor。
> 不追求成为 expert，但要达到：
> 1. **能对一般需求 / bug 独立尝试编写代码**（定位 → 复现 → 修改 → 测试 → PR）；
> 2. **熟练掌握现代查询优化器的主流技术**（Cascades 搜索、规则/变换、Join 枚举、物理属性与 Enforcer、基数估计、代价模型、子查询去相关、聚合下推、MPP 分布式优化等）；
> 3. 读过并理解支撑这些技术的**核心论文**，能把论文思想映射到 ORCA 代码。

---

## 1. 文件导航

| 文件 | 内容 |
|---|---|
| [ROADMAP.md](ROADMAP.md) | **52 周逐周计划**（带 checkbox，可直接当进度表用） |
| [PAPERS.md](PAPERS.md) | 论文清单：按主题分组、分优先级、标注对应周次和对应的 ORCA 代码 |
| [ORCA-CODE-MAP.md](ORCA-CODE-MAP.md) | ORCA 源码地图：分层结构、阅读顺序、关键类、命名约定、调试 GUC、测试工作流 |
| [templates/paper-note.md](templates/paper-note.md) | 论文笔记模板 |
| [templates/weekly-review.md](templates/weekly-review.md) | 周复盘模板 |

建议目录约定（按需创建）：

```
notes/papers/      # 每篇论文一份笔记（用 paper-note 模板）
notes/orca/        # ORCA 源码阅读笔记（按模块）
notes/weekly/      # 每周复盘
experiments/       # SQL、minidump、脚本、toy 实现
```

---

## 2. 时间预算

- 每天 2~3 个 45 min 番茄钟（下文简记为 **P**）。
- 按每周 6 天、平均 2 P 计：**基线 12 P / 周**；状态好的日子做第 3 个 P，每周可到 15~18 P。
- 全年约 **600~750 P（450~560 小时）**。ROADMAP 中每周的"必做"按 ~10~12 P 设计，"选做"留给第 3 个番茄钟。
- 第 13 / 26 / 39 / 52 周是**缓冲 + 复盘周**：补进度、写总结、不开新坑。

### 全年投入分配（大致）

| 阶段 | 周次 | 论文/理论 | ORCA 代码阅读 | 动手/贡献 | 输出（笔记） |
|---|---|---|---|---|---|
| Phase 0 环境与地图 | W1–W2 | 15% | 35% | 40% | 10% |
| Phase 1 基础 | W3–W13 | 40% | 40% | 10% | 10% |
| Phase 2 核心技术 | W14–W26 | 35% | 35% | 20% | 10% |
| Phase 3 进阶专题 | W27–W39 | 30% | 30% | 30% | 10% |
| Phase 4 贡献冲刺与前沿 | W40–W52 | 15% | 15% | 60% | 10% |

---

## 3. 每日节奏

| 日类型 | P1 | P2 | P3（可选） |
|---|---|---|---|
| 标准日（2 P） | 论文 / 理论 | ORCA 代码 / 实验 | — |
| 加强日（3 P） | 论文 / 理论 | ORCA 代码 / 实验 | 写笔记、做 ROADMAP 的"选做"、推进 PR |

几条纪律：

1. **论文和代码交替**：同一天先读论文、再看对应代码，最容易建立映射。连续读论文不超过 2 P。
2. **每个 P 开始前写一句"本 P 目标"，结束时写一句"得到了什么"**（记在当天笔记里）。
3. **每周至少一次"输出"**：一篇论文笔记或一篇源码笔记。不输出等于没读。
4. **周日 15 min 复盘**（用 weekly-review 模板），调整下周计划。落后超过 1 周就砍"选做"，不要压缩缓冲周。
5. 读论文用 Keshav 的三遍法：第一遍（10 min）看结构和结论 → 第二遍（1 P）理解主要思想和图表 → 第三遍（1~2 P，仅★★★论文）能复述算法、推导例子、想清楚 ORCA 怎么做的。

---

## 4. 贡献目标：WarehousePG

- 上游：`github.com/warehouse-pg/warehouse-pg`（PR 提交到 `main`）；本地：`/Users/zhenglong/workspace/warehouse-pg`
- ORCA 本体：`src/backend/gporca`；翻译层：`src/backend/gpopt`；GUC 定义：`src/backend/utils/misc/guc_gp.c`
- 历史参考：`greenplum-db/gpdb-archive`（只读），查老 commit 和设计演进时用

仓库里已有的、应在 Phase 0 读完的官方文档：

| 文档 | 内容 |
|---|---|
| `src/backend/gporca/README.md` | ORCA 独立构建（cmake + ninja）、`ctest`、`gporca_test -U / -d`、新增 minidump、`fix_mdps.py` 更新期望 |
| `src/backend/gporca/README.memo.md` | 如何逐行读 memo dump（group、cost context、`Grp OptCtxts`、`Best Expr`） |
| `src/backend/gporca/StyleGuide.md`、`README.format.md`、`README.tidy.md` | 代码风格、clang-format（`src/tools/fmt chk`）、clang-tidy（`src/tools/tidy`），CI 会检查 |
| `CONTRIBUTING.md` | 必须附带回归测试；大改动先上邮件列表讨论；PR 需至少一个 +1 且没有 -1 |

## 5. 起点校准

从 WarehousePG 的 git 历史看，你已经有一批合入的 PR，其中与 ORCA 相关的集中在**翻译层**（`src/backend/gpopt/translate`，如 #114、#128），其余在 planner / executor / cdb 侧（DQA、SubLink 上拉、分布键上拉等）。所以本计划中的"成为 contributor"具体指：

> **从"修翻译层和 planner 侧的问题"走到"能改 ORCA 核心库"**：`libgpopt`（xform、属性、预处理、搜索）、`libnaucrates`（统计、DXL、元数据）、`libgpdbcost`（代价模型）。

据此做了几处调整：
- Phase 0 压缩：构建环境、回归测试、PR 流程你都已熟悉，把时间留给 ORCA 独立构建、单测、memo 阅读和 ORCA 编码约定。
- PostgreSQL planner 的内容只做对照，不再单独学。
- 里程碑中的"第一个 PR"改为"**第一个改动 ORCA 核心库的 PR**"。
- 学习时以 WHPG 近期的 ORCA 修复 PR 作为案例（如 #149、#152、#154、#183），读它们的 diff 和测试。

---

## 6. 里程碑

| 里程碑 | 时间 | 可检验的标准 |
|---|---|---|
| **M0** ORCA 环境就绪 | W2 末 | 能独立构建 ORCA 并跑 `ctest`；能用 `gporca_test -d` 跑单个 minidump；能打开 memo / plan trace 并按 README.memo.md 读懂 |
| **M1** 看懂骨架 | W13 末 | 能不看代码画出"SQL → Query → DXL → CExpression → Memo → 搜索 → 最优计划 → DXL → PlannedStmt"全流程；能解释 Cascades 的 job 调度 |
| **M2** 第一个核心库 PR | W26 末 | 掌握 xform、join 枚举、属性/enforcer、统计、代价模型；**提交第一个改动 `libgpopt` / `libnaucrates` / `libgpdbcost` 的 PR**（附 minidump 单测） |
| **M3** 独立修核心 bug | W39 末 | 掌握子查询、聚合、外连接、CTE、分区等专题；**核心库 PR 合入 ≥ 2 个**，其中至少 1 个改动的是优化逻辑（xform / 统计 / 属性推导） |
| **M4** ORCA contributor | W52 末 | 完成一个中等规模的 capstone（设计 → 实现 → 单测 + 回归 → 性能验证 → 合入）；定期 review 他人的 ORCA PR；整理出自己的 ORCA 内部机制笔记系列 |

每个阶段结束的自测题见 [ROADMAP.md](ROADMAP.md) 中各缓冲周。

---

## 7. 前置技能自查

开始前花 10 分钟自查，薄弱项在 Phase 0 / Phase 1 的"选做"里补：

- [ ] C++：类继承与虚函数、模板、RAII、手动内存管理（ORCA 不用 STL 和智能指针，而是自带内存池和引用计数）
- [ ] 调试：gdb / lldb 断点、条件断点、打印复杂对象；`perf` 或 Instruments 基本使用
- [ ] 关系代数与 SQL 语义：NULL 三值逻辑、outer / semi / anti join、GROUP BY、窗口函数
- [ ] PostgreSQL 基本架构：parser → rewriter → planner → executor；EXPLAIN / EXPLAIN ANALYZE
- [ ] Git：rebase、cherry-pick、bisect（bisect 在追 ORCA 回归问题时非常有用）
- [ ] 构建环境：ORCA 本体可在 macOS 上独立构建和跑单测（`README.macOS.bash` 装好 cmake、xerces-c 即可），日常读代码、调 xform 用这个就够；需要完整集群做 `EXPLAIN ANALYZE` 或跑回归时，再用 `dockerfiles/Dockerfile.el8/el9` 或 Linux VM
