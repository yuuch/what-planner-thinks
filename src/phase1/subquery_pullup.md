# 子查询提升 (Subquery Pull-up)

子查询提升（Subquery Pull-up）是 PostgreSQL 优化器中最为核心的逻辑重写优化机制之一。它发生在规划器的预处理早期阶段。其根本目标是**“打破查询边界，扁平化查询树”**，使得父子查询能够融合在一起，从而大幅度扩展 Join Order（连接顺序）的搜索空间，并且减少执行层的嵌套开销。

## 1. 背景与目标：为何要提升？

如果不进行提升，嵌套在 `FROM` 或者是 `WHERE` 子句中的子查询，在执行树上就会变成单独的 `SubqueryScan` 节点或是 `SubPlan` 操作（InitPlan/SubPlan）。
这限制了优化器的能力：
*   **黑盒隔离**：父查询的表无法与子查询内部的表混合进行 Join Order 规划。
*   **运行时开销**：执行引擎在运算时需要频繁穿越执行计划的不同层级进行上下文切换或独立求解。

因此，PostgreSQL 规划器在 `src/backend/optimizer/prep/prepjointree.c` 中，专门设计了两大核心路径来拆解各类“提升”。

## 2. SubLink 提升：`pull_up_sublinks`

`SubLink` 通常指的是出现在 `WHERE` 或者 `JOIN/ON` 过滤条件中的子查询结构。
优化器在 `subquery_planner` 一开始就会调用 `pull_up_sublinks()`。

其核心递归函数 `pull_up_sublinks_qual_recurse` 会遍历条件，并根据 `SubLink` 的类型采取不同的提升策略。这依赖于几个关键转换函数：

### 2.1 `ANY` SubLink (`convert_ANY_sublink_to_join`)
当遇到类似 `WHERE x.id = ANY (SELECT id FROM y)`（即 `IN (subquery)`）时：
*   优化器会将其从 `WHERE` 条件中剥离出来，转化成与父查询的一张表的 **`JOIN_SEMI` (半连接)**。
*   对于常数数组 `IN (1, 2, 3)`，也会在此阶段通过 `convert_VALUES_to_ANY` 做尝试折叠化简。

### 2.2 `EXISTS` SubLink (`convert_EXISTS_sublink_to_join`)
当遇到 `WHERE EXISTS (SELECT 1 FROM y WHERE x.id = y.id)` 时：
*   本质与 `ANY` 类似，这会被转化为一个 **`JOIN_SEMI` (半连接)**。
*   但是它进一步处理了带有 `NOT` 修饰的情况：如果是 `WHERE NOT EXISTS ( ... )`，在 PG 中会被直接转化为 **`JOIN_ANTI` (反连接)**。这就允许优化器使用 Hash Anti-Join 等高效物理算子直接过滤数据。

## 3. Subquery 提升：`pull_up_subqueries`

区别于 `SubLink`，一般的 `Subquery` 主要指代的是出现在 `FROM` 子句里面的表源定义（如 `SELECT * FROM a JOIN (SELECT * FROM b) c ON ...`）。
当处理完 `SubLink` 之后，`planner` 接着调用 `pull_up_subqueries()`，尝试将 FROM 下拉取出来的子查询给融会贯通。

### 3.1 简单子查询 (`is_simple_subquery` / `pull_up_simple_subquery`)
怎样才叫一个“简单（Simple）”的子查询？
*   它不能包含 `GROUP BY`、`HAVING`。
*   不能包含 `LIMIT` / `OFFSET`。
*   不能包含集合操作（`UNION`、`INTERSECT`、`EXCEPT`）。
*   不能包含顶层聚合函数或是窗口函数。

如果是简单子查询，`pull_up_simple_subquery()` 将大显身手：
1. 它将子查询的 `FromExpr` 的儿子全部并入到当前层级的 `FromExpr` 树里。
2. 剥除该子查询外层包装。
3. 利用 `pullup_replace_vars` 构建或替换那些需要保持上下文独立性的变量引用（如果在 Grouping Set 里，有时还要为其包装一层 `PlaceHolderVar`）。

### 3.2 简单 UNION ALL 的扁平化 (`pull_up_simple_union_all`)
对于 `FROM (SELECT x FROM a UNION ALL SELECT x FROM b)` 这样的集合操作：
如果没有任何排序 `ORDER BY`、没有 `LIMIT`，并且是 `UNION ALL` （不用去重），优化器会调用 `is_simple_union_all`。
它会把这颗 `UNION ALL` 子树彻底压平，转化为一系列的追加关系 (`AppendRelInfo`)。这就将集合操作融化成底层扫描层级的简单 `Append`，而不用执行繁重的 Subquery 拼接。

### 3.3 LATERAL 子查询的修补
在 PostgreSQL 支持了 `LATERAL` 之后，提取时还涉及 `LATERAL` 跨层依赖参数。
如果发生 `pull_up_subquery` 且原本有 `LATERAL` 引用，代码会通过 `IncrementVarSublevelsUp` 等复杂的方法调整 Vars 的层级深度（Sublevel），并借由 `CombineRangeTables` 重组合并底层的 RangeTable 中记录被依赖的状态。这一机制对关联子查询能够被最终压平至关重要。
