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
| [notes/learning-state.md](notes/learning-state.md) | **学习状态**：当前位置、证据、错误、下一项最小任务。跨会话的唯一事实源，每次开始学习（或和 Claude 开新会话）先读它 |
| [templates/paper-note.md](templates/paper-note.md) | 论文笔记模板（含阅读前基线、延迟复测、迁移测试） |
| [templates/module-note.md](templates/module-note.md) | 源码模块笔记模板（同样带证据点） |
| [templates/weekly-review.md](templates/weekly-review.md) | 周复盘模板（主问题、四项检查、容量、中断恢复、只改一个变量） |

建议目录约定（按需创建）：

```
notes/learning-state.md  # 学习状态（唯一事实源）
notes/                   # ORCA 源码阅读笔记（按模块，直接放在 notes/ 下，如 dxl.md、architecture.md）
notes/papers/            # 每篇论文一份笔记（用 paper-note 模板）
notes/baseline/          # 阶段基线样本：每个 Phase 开头不看资料写的版本，缓冲周拿来对照
notes/weekly/            # 每周复盘
experiments/             # SQL、minidump、脚本、toy 实现
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

### 3.1 四档容量

计划不只认一种理想状态。同一条主线准备四个档位，按当天的真实容量选，而不是按计划惩罚自己：

| 档位 | 何时用 | 做什么 |
|---|---|---|
| **加强日（3 P）** | 时间、精力都充足 | P1 论文 / 理论，P2 ORCA 代码 / 实验，P3 写笔记、做"选做"、推进 PR |
| **标准日（2 P）** | 常态 | P1 论文 / 理论，P2 ORCA 代码 / 实验 |
| **压缩日（10–20 min）** | 加班、注意力被切碎 | 只做"回忆 → 输出 → 记录"：合上资料写下上次学到的 3 点，把一个疑问写进学习状态，写下下次打开文件的第一步 |
| **恢复日（0 P）** | 生病、失眠、家事、紧急 bug | 不学新内容。最多把下次要读的论文 / 文件打开放好。**不算失败，也不补做** |

写成"如果 → 那么"，替自己省掉临场纠结：

```
如果有 ≥ 90 分钟且精力完整  → 标准日 / 加强日，按 ROADMAP 推进
如果只有 10–20 分钟           → 压缩日：回忆 3 点 + 记一个疑问 + 写下一步
如果连续两天睡眠不足或身体不适 → 恢复日：不学新内容，不做"补课"计划
```

### 3.2 一个番茄钟的默认结构（45 min）

| 时长 | 动作 | ORCA 场景中的例子 |
|---|---|---|
| 5 min | **回忆**：不看资料写下上次记得的 3 点 | 合上笔记写出 `CJobGroupOptimization` 的几个状态 |
| 15 min | **输入**：只处理一小段材料 | 论文一节 / 一个类 / 一个 xform 的 `Transform` |
| 15 min | **输出**：留下一个看得见的结果 | 手写伪代码、画状态图、跑一个 minidump 并解释计划、写一个单测 |
| 5 min | **对照**：拿原文 / 代码检查输出，标出错处 | 我画的状态转移和 `CJobStateMachine` 实际定义差在哪 |
| 5 min | **安排**：把下一步写进 [学习状态](notes/learning-state.md) | "下次先读 `CJobTransformation::EevtTransform`" |

不是每个 P 都要严格切分，但"回忆"和"输出"这两步不能省：**只读不输出，留下的只是熟悉感。**

### 3.3 纪律

1. **论文和代码交替**：同一天先读论文、再看对应代码，最容易建立映射。连续读论文不超过 2 P。
2. **每周只带一个主问题**：在周复盘里写成可观察的句子，例如"我能否不看代码，讲清 `CXformSplitGbAgg` 什么时候把聚合拆成两阶段？"，而不是"本周学习聚合"。
3. **每周至少一次输出**：一篇论文笔记、一篇模块笔记或一个实验。笔记本身也是作品：写给"三个月后的自己"这个读者，能在不打开对话记录的情况下看懂。
4. **周日 20 min 复盘**（用 [weekly-review 模板](templates/weekly-review.md)）：先看证据，再解释原因，**下周只改一个变量**。
5. 读论文用 Keshav 的三遍法：第一遍（10 min）看结构和结论 → 第二遍（1 P）理解主要思想和图表 → 第三遍（1~2 P，仅 ★★★ 论文）能复述算法、推导例子、想清楚 ORCA 怎么做的。

### 3.4 中断后怎么回来

一年里一定会有加班、出差、线上问题打断计划。中断不是失败，把它解释成"我不行"才会让它扩大。回来时只写四行（记在当周复盘里）：

```
发生了什么事实：（不做人格判断）
哪个条件变了：时间 / 精力 / 难度 / 环境 / 责任
24 小时内的最小回归动作：（通常是一个压缩日的动作）
明确不补做什么：
```

规则：**不熬夜还债，不重写整个计划**。落后 < 1 周就砍"选做"；落后 ≥ 1 周用最近的缓冲周吸收，并在 ROADMAP 里把后续"选做"降级。同一个阻力连续出现 3 周，说明计划本身该改（任务太大、时段不对、前置知识缺失），而不是要求自己更用力。

---

## 4. 学习闭环：三种"会"与四个证据点

### 4.1 三种"会"

| 状态 | 含义 | ORCA 中的例子 |
|---|---|---|
| **知道** | 读得懂，看到答案觉得熟悉 | 读完 `CXformPushGbBelowJoin`，觉得"原来如此" |
| **能做** | 不看资料，在相似条件下独立完成一次 | 合上代码写出它的 pattern、promise 和正确性条件；改一个 xform 让某个 minidump 的计划按预期变化 |
| **能交付** | 在真实约束下完成，并让别人能用、能质疑 | 一个带 minidump 单测和回归测试、经过 review 合入的 PR |

本计划的里程碑都是按"能做"和"能交付"写的。读完、看懂、和 Claude 讨论清楚，都只到"知道"。

### 4.2 四个证据点

同一项能力，至少留下这四份样本，并分清楚每一份里谁提供了帮助：

| 证据点 | 目的 | ORCA 中怎么做 |
|---|---|---|
| **基线** | 学之前（或问 Claude 之前）的真实起点 | 每个 Phase 开头不看资料写一份 [阶段基线](#43-阶段基线)；读一个模块前先写 3 行"我现在以为它是怎么工作的" |
| **即时表现** | 刚学完 / 刚讨论完能做到什么 | 论文 / 模块笔记本身 |
| **延迟保持** | 隔 3–7 天，去掉提示后还剩什么 | ★★★ 论文和核心模块：下周复盘时花 10 min，合上笔记复述算法或画出调用关系，再对照笔记标出遗忘的部分 |
| **迁移** | 换一个没见过的场景还能不能用 | 用刚学的框架解释一个**没读过的** xform 或 minidump 计划；用论文里的算法手算一个 ORCA 测试里的查询 |

只看即时表现最容易高估自己；只看最终 PR 又看不出自己是怎么从不会走到会的。

### 4.3 阶段基线

每个 Phase 的第一周不看任何资料、不问 Claude，花 15–25 min 写一份基线，存进 `notes/baseline/`；该 Phase 的缓冲周拿同一个题目再写一遍，两份并排对照：

| 阶段 | 基线题目（开头写，缓冲周重写） | 写于 | 对照于 |
|---|---|---|---|
| Phase 0–1 | 画出你现在理解的"SQL → ORCA → 计划"全流程，标出每一步对应的类 | W1 | W13 |
| Phase 2 | 讲清：一个 4 表 join 在 ORCA 里怎样被枚举、选分布、插 Motion、算代价 | W14 | W26 |
| Phase 3 | 讲清：一个带相关子查询和 `GROUP BY` 的查询在 ORCA 里经过哪些变换 | W27 | W39 |
| Phase 4 | 为 capstone 写一页设计草稿（问题、方案、风险、测试计划） | W43 | W52 |

基线写得难看是正常的。它的作用是防止三个月后的自己只凭感觉回忆"以前好像更差"。

---

## 5. 与 AI（Claude）协作的规则

这个计划本身就是和 Claude 一起做的，所以最大的风险是：**能力留在了对话记录里，没有留在自己身上**。几条规则：

1. **先留下自己的版本**：问 Claude 解释一个类 / 一篇论文之前，先写 3 行——我现在的理解是什么、最不确定的地方是什么、什么结果能证明我学会了。
2. **分工清楚**：

   | Claude 适合做 | 必须自己做 |
   |---|---|
   | 定位代码、解释调用链、核对类名 / GUC 是否存在 | 论文笔记的"核心思想""手算例子"两节 |
   | 出反例、出平行题（"换成 LOJ 还成立吗？"） | 设计说明和 PR 的第一版 |
   | 审阅笔记：只指出最影响理解的 3 个问题 | 判断一个改动对不对、要不要提交 |
   | 生成实验用的 SQL / minidump 场景 | 解释为什么一个计划变化是合理的（review 时会被追问） |

3. **关屏复测**：一次讨论结束后，关掉对话，独立复述 / 画图 / 写代码。离开对话就讲不下去的部分，就是还没学会的部分，写进学习状态的"错误与待补"。
4. **三次对照**：每个 Phase 至少挑一个模块做一次——无 AI 基线 → Claude 辅助版 → 3–7 天后的独立版：

   | 结果 | 解读 |
   |---|---|
   | 辅助版和独立版都变好 | Claude 提供了有效的脚手架，能力留下来了 |
   | 辅助版变好，独立版退步 | 关键理解被外包了，下次减少直接解释，多让 Claude 提问 |
   | 读得更快，但笔记错误变多 | 用速度换了"速度债"，放慢 |

5. **会话开始和结束**：开新会话先让 Claude 读 [学习状态](notes/learning-state.md) 并复述当前位置；结束时把"完成了什么、证据在哪、新增的错误 / 疑问、下一项最小任务"写回学习状态。Claude 的记忆不是事实源，这个文件才是。

---

## 6. 贡献目标：WarehousePG

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

## 7. 起点校准

从 WarehousePG 的 git 历史看，你已经有一批合入的 PR，其中与 ORCA 相关的集中在**翻译层**（`src/backend/gpopt/translate`，如 #114、#128），其余在 planner / executor / cdb 侧（DQA、SubLink 上拉、分布键上拉等）。所以本计划中的"成为 contributor"具体指：

> **从"修翻译层和 planner 侧的问题"走到"能改 ORCA 核心库"**：`libgpopt`（xform、属性、预处理、搜索）、`libnaucrates`（统计、DXL、元数据）、`libgpdbcost`（代价模型）。

据此做了几处调整：
- Phase 0 压缩：构建环境、回归测试、PR 流程你都已熟悉，把时间留给 ORCA 独立构建、单测、memo 阅读和 ORCA 编码约定。
- PostgreSQL planner 的内容只做对照，不再单独学。
- 里程碑中的"第一个 PR"改为"**第一个改动 ORCA 核心库的 PR**"。
- 学习时以 WHPG 近期的 ORCA 修复 PR 作为案例（如 #149、#152、#154、#183），读它们的 diff 和测试。

---

## 8. 里程碑

| 里程碑 | 时间 | 可检验的标准 |
|---|---|---|
| **M0** ORCA 环境就绪 | W2 末 | 能独立构建 ORCA 并跑 `ctest`；能用 `gporca_test -d` 跑单个 minidump；能打开 memo / plan trace 并按 README.memo.md 读懂 |
| **M1** 看懂骨架 | W13 末 | 能不看代码画出"SQL → Query → DXL → CExpression → Memo → 搜索 → 最优计划 → DXL → PlannedStmt"全流程；能解释 Cascades 的 job 调度 |
| **M2** 第一个核心库 PR | W26 末 | 掌握 xform、join 枚举、属性/enforcer、统计、代价模型；**提交第一个改动 `libgpopt` / `libnaucrates` / `libgpdbcost` 的 PR**（附 minidump 单测） |
| **M3** 独立修核心 bug | W39 末 | 掌握子查询、聚合、外连接、CTE、分区等专题；**核心库 PR 合入 ≥ 2 个**，其中至少 1 个改动的是优化逻辑（xform / 统计 / 属性推导） |
| **M4** ORCA contributor | W52 末 | 完成一个中等规模的 capstone（设计 → 实现 → 单测 + 回归 → 性能验证 → 合入）；定期 review 他人的 ORCA PR；整理出自己的 ORCA 内部机制笔记系列 |

每个阶段结束的自测题和**证据门槛**见 [ROADMAP.md](ROADMAP.md) 中各缓冲周。

里程碑是门槛，不是日期：缓冲周的门槛没有证据，就不要自动进入下一阶段——先缩小范围、补前置知识，或把下一阶段的"选做"降级，而不是在没站稳的地方继续加码。

---

## 9. 前置技能自查

开始前花 10 分钟自查，薄弱项在 Phase 0 / Phase 1 的"选做"里补：

- [ ] C++：类继承与虚函数、模板、RAII、手动内存管理（ORCA 不用 STL 和智能指针，而是自带内存池和引用计数）
- [ ] 调试：gdb / lldb 断点、条件断点、打印复杂对象；`perf` 或 Instruments 基本使用
- [ ] 关系代数与 SQL 语义：NULL 三值逻辑、outer / semi / anti join、GROUP BY、窗口函数
- [ ] PostgreSQL 基本架构：parser → rewriter → planner → executor；EXPLAIN / EXPLAIN ANALYZE
- [ ] Git：rebase、cherry-pick、bisect（bisect 在追 ORCA 回归问题时非常有用）
- [ ] 构建环境：ORCA 本体可在 macOS 上独立构建和跑单测（`README.macOS.bash` 装好 cmake、xerces-c 即可），日常读代码、调 xform 用这个就够；需要完整集群做 `EXPLAIN ANALYZE` 或跑回归时，再用 `dockerfiles/Dockerfile.el8/el9` 或 Linux VM
