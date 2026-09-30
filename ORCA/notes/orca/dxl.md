# DXL：ORCA 的数据交换格式

> 对应 [ROADMAP.md](../../ROADMAP.md) W9；代码参照 WarehousePG（`/Users/zhenglong/workspace/warehouse-pg`）。

## 1. 是什么

**DXL = Data eXchange Language**。一套 ORCA 自定义的 XML 方言，schema 定义在
[`src/backend/gporca/server/dxl.xsd`](../../../../warehouse-pg/src/backend/gporca/server/dxl.xsd)
（1747 行，命名空间 `http://greenplum.com/dxl/2010/12/`，595 处 `element`/`complexType` 定义）。

它是 ORCA 与外部世界之间**唯一**的接口协议，一份 DXL 文档最多包含三段内容：

```xml
<dxl:DXLMessage xmlns:dxl="http://greenplum.com/dxl/2010/12/">
  <dxl:Thread Id="0">
    <dxl:OptimizerConfig>...</dxl:OptimizerConfig>   <!-- GUC / traceflag / 代价参数快照 -->
    <dxl:Metadata SystemIds="0.GPDB">...</dxl:Metadata>  <!-- 表/列/类型/索引/统计信息 -->
    <dxl:Query>...</dxl:Query>                        <!-- 待优化的逻辑查询树（可选） -->
    <dxl:Plan>...</dxl:Plan>                          <!-- 输出的物理计划（可选） -->
  </dxl:Thread>
</dxl:DXLMessage>
```

内存里对应的结构是 [`CDXLNode`](../../../../warehouse-pg/src/backend/gporca/libnaucrates/include/naucrates/dxl/operators/CDXLNode.h)：
一棵通用的树/DAG，每个节点持有一个 `CDXLOperator`（说明这是什么算子）和子节点数组。

解析：Xerces-C 的 **SAX2 增量解析器**
（`libnaucrates/src/parser/CParseHandler*.cpp`），每种 DXL 元素对应一个 `CParseHandler` 子类，
边扫 XML 边建 `CDXLNode` 树，不先建整棵 DOM。

序列化 / 反序列化的统一入口是
[`CDXLUtils`](../../../../warehouse-pg/src/backend/gporca/libnaucrates/include/naucrates/dxl/CDXLUtils.h)
（源码注释："Entry point for parsing and serializing DXL documents"）。

## 2. DXL 在系统里的实际作用

```
                     DXL(Query + Metadata)                    DXL(Plan)
   PG Query  ──────────────────────────────▶   ORCA   ──────────────────────────▶  PlannedStmt
(CTranslatorQueryToDXL)                (纯 C++, 不识别任何 PG 内部结构)      (CTranslatorDXLToPlStmt)
```

DXL 不是"顺手加的中间格式"，它撑起了四件具体的事：

1. **系统解耦**：ORCA 本体（`libgpopt`/`libnaucrates`/`libgpdbcost`）不 `#include` 任何 PostgreSQL
   头文件，只认 DXL 和 `IMDProvider` 这个元数据接口。同一份 ORCA 二进制历史上被 GPDB 和
   Pivotal HAWQ 两套不同的执行引擎接过，靠的就是"你把 Query/Catalog 翻译成 DXL 喂给我，
   我把 Plan 用 DXL 吐给你"这层协议。
2. **离线可复现（这是你已经天天在用的能力）**：`optimizer_minidump = always` 生成的
   `.mdp` 就是一份完整的 DXL 快照（配置 + 元数据 + 查询）。`gporca_test -d foo.mdp`
   不需要起集群、不需要真实的表和统计信息，几百毫秒内就能重放一次优化过程。
   WarehousePG 里 `data/dxl/minidump/` 现有 **1194 个文件、共 68MB**，全部是 DXL 文本。
3. **回归测试的载体**：CI 跑几千个 minidump，本质是"重放 DXL Query → 生成 DXL Plan →
   和仓库里存的期望 DXL 逐行 diff"。`fix_mdps.py` 更新期望结果，改的也是这些 DXL 文件。
   XML 是纯文本，天然能被 `git diff`、`xmllint --format`、`dxl.xsd` 校验。
4. **调试与人工检查**：一个复杂 bug 的复现材料就是一份 `.mdp`，可以直接打开读、手工删减、
   贴进 issue，不需要额外工具还原成"人能看懂"的形式——这也是为什么它必须是文本格式，
   而不是二进制格式。

一个有意思的旁证：**SQL Server 的执行计划格式（Showplan XML）**用的是完全相同的逻辑——
带 schema 的 XML，用来做可交换、可校验、可离线分析的计划表示。选择 XML 并不是 ORCA 团队的孤例。

## 3. 换成 JSON：差在哪，实现难度怎么样

**结论先行**：技术上完全可行，JSON 能覆盖 DXL 的绝大部分需求，但会在几个具体维度上变弱，
且这些弱化点恰好是 ORCA 最依赖的能力（schema 强校验、attribute/element 语义分离、
流式解析生态）。如果是今天从零设计，JSON（配 JSON Schema）是合理选项；但对一个已经有
十几年历史、上千个手写测试夹具的系统，"换编码格式"本身的改造成本远大于收益。

### 3.1 逐项对比

| 维度 | DXL（XML + XSD） | JSON（+ JSON Schema） | 说明 |
|---|---|---|---|
| **结构校验** | XSD：类型、基数（`minOccurs`/`maxOccurs`）、枚举值、命名空间隔离，工具链成熟（`xmllint --noout --schema`） | JSON Schema 今天也成熟，但 ORCA 设计时（约 2010~2012 年）JSON Schema 还很不成熟 | 实现难度：改造期一致（都要重写/复用 schema），但当年二选一时 XSD 明显更现成 |
| **attribute vs element** | 有明确区分：定长标量字段用 attribute（`Mdid="6.65536.1.0"`），嵌套结构用 element；DXL 里大量"一堆元数据字段 + 一段嵌套子树"正好吃这个语义 | 只有 key-value，没有区分；要么全塞成扁平字段，要么自己发明"标量字段 vs 子对象"的约定 | JSON 不是做不到，只是要额外的约定去补上这层区分，team 内一致性要靠文档/review 维持 |
| **命名空间** | `xmlns:dxl=...`，避免和宿主系统里其他 XML（比如 pg_hint_plan 的配置）混淆 | 没有对应机制，只能靠字段前缀或外层包一层 `{"dxl": {...}}` | 影响不大，因为 DXL 本来就是独立文件，冲突场景少 |
| **流式/增量解析** | Xerces SAX2，边解析边建对象，适合处理体积较大的 minidump（含完整统计信息） | 主流 JSON 库（RapidJSON、simdjson 等）也有流式 API，但 DOM-first 的用法更普遍，SAX 风格的生态明显更薄 | 实现难度：要达到同等的"边解析边建 `CDXLNode` 树"效果，JSON 这边选型和实现工作量不小于当前的 Xerces `CParseHandler` 体系 |
| **人类可读 / 可 diff** | 是（`.mdp` 本身就是给人读、给 `git diff` 用的测试夹具） | 同样可以，且更紧凑（没有闭合标签的冗余） | JSON 在这一项上**更占优** |
| **体积 / 解析速度** | XML 标签冗余，1194 个 minidump 共 68MB，单文件可能到几百 KB | 通常比等价 XML 小 30%~50%，解析也更快 | JSON 在这一项上**更占优**，但 ORCA 的瓶颈从来不是"解析 DXL 花多久"，而是优化搜索本身，这个优势基本不影响真实场景 |
| **改造实现难度** | — | 需要：①重写 `dxl.xsd` 为 JSON Schema；②重写 `libnaucrates/src/parser/` 整套 `CParseHandler`（数十个类，按 DXL 元素一一对应）为 JSON 解析回调；③重写 `CDXLUtils` 的序列化侧；④迁移全部 1194 个 minidump（自动转换 + 人工抽查）；⑤所有引用 `.mdp` 路径和内容的测试基础设施（`fix_mdps.py`、`CMakeLists.txt` 里的注册列表）都要跟着改 | 这是一次贯穿 `libnaucrates` 解析层、CI 脚本、上千个测试夹具的**系统性重写**，不是加个新 parser 就完事 |

### 3.2 其他候选格式，为什么也没被选

- **Protobuf / Cap'n Proto / FlatBuffers**（二进制、schema 驱动、强类型代码生成）：
  体积和解析速度是三者里最好的，但**不可读**——minidump 就没法直接打开、手工删减、贴进 issue 了，
  等于砍掉了"离线调试 + 人工检查"这条主要用途。而且引入代码生成会让 `CDXLOperator` 这类
  手写的类层次和生成代码之间多一层同步负担。
- **S-expression / Lisp 风格文本**：这其实是**已经在用的替代方案**——PostgreSQL 自己
  序列化 `Plan`/`Query` 树（`nodeToString`/`stringToNode`，`src/backend/nodes/outfuncs.c`）
  用的就是这种带括号的文本格式，主要用于 parallel worker 间传递计划。
  ORCA 没有直接复用它，因为：① ORCA 的数据模型（`CExpression`/`CDXLNode`）和 PG 的 `Node` 树
  不是同构的，翻译工作量不会减少；② PG 的这套格式没有独立 schema、没有命名空间，
  不满足"多系统共享同一份协议"的解耦目标；③ 缺少 XSD 那样的显式结构校验。
- **纯手写二进制格式**：省掉了通用序列化库的开销，但要自己解决版本兼容、可读性、
  校验这三件 XML/Protobuf 都替你解决好的事，性价比最低。

### 3.3 一句话总结

DXL 真正值钱的不是"XML"这个编码选择本身，而是背后的架构决定：
**用一份带独立 schema 的序列化格置，把优化器和宿主系统的内存结构彻底切开，
换来跨系统复用、离线可复现、和可回归测试**。这个决定换成 JSON 也成立，
换成 Protobuf 就不成立（丢了可读性），换成手写二进制更不成立。
ORCA 选 XML 是 2010 年代初的工具链现实使然；今天重新设计大概率会选 JSON，
但对现在的 ORCA 来说，迁移成本（重写解析层 + 迁移 1194 个测试夹具）远大于收益，
所以维持现状是理性的。

## 4. 延伸问题（留着以后回答）

- [ ] `CMDAccessor` 和 `IMDProvider` 具体怎样把 relcache 查出来的目录信息翻译成 DXL 的
      `<dxl:Relation>`/`<dxl:Type>`？（W9 `CTranslatorRelcacheToDXL`）
- [ ] Metadata 部分的 `Mdid`（如 `6.65536.1.0`）编码规则是什么？
- [ ] 一份 `.mdp` 里的 `TraceFlags`（如 `101013,102001,...`）如何映射回具体的 GUC/xform 开关？
