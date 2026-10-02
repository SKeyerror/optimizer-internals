# 学习状态

> 跨会话的唯一事实源。每次开始学习（或和 Claude 开新会话）先读这里；每次结束把变化写回来。
> 只写事实和证据位置，不写心情日记。Claude 的记忆不是事实源，这个文件才是。

- 状态版本：v1
- 更新时间：2026-10-02
- 计划起点：2026-09-29（W1 周一）

## 1. 当前位置

- 当前周：**W1**（Phase 0 · ORCA 独立构建与单测）
- 本周主问题：我能否在 Linux 和 macOS 上独立构建 ORCA Debug 版、跑通 `ctest`，并单独跑一个 minidump？
- 当前阶段目标：M0（W2 末）—— 能跑 `ctest` 和单个 minidump；能用调试器跟进优化入口；能按 README.memo.md 读懂一个 memo

## 2. 已完成

| 日期 | 内容 | 证据位置 | 状态 |
|---|---|---|---|
| 2026-09-29 | 一年学习计划（README / ROADMAP / PAPERS / ORCA-CODE-MAP） | 本仓库 | 完成 |
| 2026-09-29 | 在 neo（Linux, GCC 13.3）上做 ORCA Debug 构建，`gporca_test` 编译失败：`consteval` 是 C++20 关键字，`-Werror=c++20-compat` | warehouse-pg 分支 `fix/consteval-cxx20-compat`（已推到 fork，**未开 PR**）；ROADMAP W1 构建项已勾选（采用 `-Wno-c++20-compat` 方案） | 构建完成，**`ctest` 通过 / 失败数未记录** |
| 2026-09-30 | DXL 笔记（W9 内容，提前读） | [dxl.md](dxl.md) | 即时表现，**Claude 主导撰写**，未做关屏复测 |
| 2026-09-30 | ORCA 架构图模块映射（W1 / W8 内容） | [architecture.md](architecture.md) | 即时表现，**Claude 主导撰写**，未做关屏复测 |

> 注意：dxl.md 和 architecture.md 目前只到"知道"这一层——内容是和 Claude 讨论后由 Claude 整理的。按 README §5 的规则，它们需要一次关屏复测才算自己的。

## 3. 基线样本

| 样本 | 题目 | 位置 | 日期 | 对照于 |
|---|---|---|---|---|
| Phase 0–1 基线 | 不看资料画出"SQL → ORCA → 计划"全流程，标出每一步对应的类 | `notes/baseline/W01-pipeline.md` | **未写** | W13 |

## 4. 错误、疑问与待补

| 内容 | 来源 | 下一步 |
|---|---|---|
| `CMDAccessor` / `IMDProvider` 怎样把 relcache 信息翻译成 DXL 的 `<dxl:Relation>` / `<dxl:Type>`？ | dxl.md §4 | W9 `CTranslatorRelcacheToDXL` |
| Metadata 中 `Mdid`（如 `6.65536.1.0`）的编码规则？ | dxl.md §4 | W9 |
| `.mdp` 中 `TraceFlags` 数值如何映射回 GUC / xform 开关？ | dxl.md §4 | W3 `CConfigParamMapping.cpp` |

## 5. 已验证有效的方法

- （暂无。记录格式：方法 — 在什么条件下有效 — 证据）

## 6. 待验证的假设

- macOS 上 ORCA 独立构建（brew 的 cmake + xerces-c）足够日常读代码和调 xform，只有 `EXPLAIN ANALYZE` / 回归测试才需要 neo —— W1 内验证

## 7. 下一项最小任务

1. **写 Phase 0–1 基线**（15–25 min，不看任何资料、不问 Claude）：画出现在理解的 ORCA 全流程，存为 `notes/baseline/W01-pipeline.md`
2. 把 neo 上 `ctest -j8 --output-on-failure` 的结果（通过数 / 失败用例）记到本文件第 2 节
3. `./server/gporca_test -U CAggTest` 和 `-d` 跑一个 minidump
4. 决定 `fix/consteval-cxx20-compat` 是否开 PR

## 8. 开新会话时给 Claude 的提示

```text
先读 ORCA/notes/learning-state.md。用不超过 6 条复述：当前周、本周主问题、已完成和证据位置、
未完成项、错误与疑问、下一项最小任务；指出文件里的冲突或缺口，不要补写文件里没有的事实。
然后只围绕"下一项最小任务"工作：需要我先作答的，先让我作答再反馈，不要直接给完整答案。
结束时输出可以贴回本文件的更新：完成了什么、证据位置、新增错误/疑问、下一项最小任务。
```
