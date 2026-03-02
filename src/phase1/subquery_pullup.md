# 子查询提升 (Subquery Pull-up)

子查询提升（Subquery Pull-up）是 PostgreSQL 优化器在预处理早期阶段的重要逻辑重写机制。其主要目的是**消除不必要的查询嵌套层级，将查询树尽可能的“拍平”**。一旦子查询和主查询成功合并，优化器在排列多表连接顺序 (Join Order) 时就能拥有更大的搜索空间，同时也能减少执行器在运行时的执行层级和开销。

---

## 1. 为什么需要子查询提升？

如果不进行子查询提升，位于 `FROM` 或 `WHERE` 子句中的嵌套子查询在最终生成的执行树中会保留为独立的查询节点（如 `SubqueryScan` 节点或 `InitPlan/SubPlan` 组件）。

这种隔离会带来以下两个问题：
*   **连接重排受限**：优化器无法将主查询中的表与子查询内部的表混合在一起进行 Join Order 优化，从而限制了寻找更优连接顺序的可能性。
*   **运行时开销增加**：在实际执行时，执行引擎可能会在不同层级上下文中进行多次调用，或者只能将子查询作为一个黑盒独立执行。

为了解决这个问题，PostgreSQL 的规划器在其 `src/backend/optimizer/prep/prepjointree.c` 中，主要通过两套机制来处理和提升子查询。

---

## 2. 处理 SubLink：`pull_up_sublinks`

在 PostgreSQL 的内部实现中，`SubLink` 通常指的是位于 `WHERE` 或是 `JOIN/ON` 过滤条件中的子查询。在 `subquery_planner` 启动时，优化器首先会调用 `pull_up_sublinks()` 来处理这些结构。

该逻辑依赖于递归函数 `pull_up_sublinks_qual_recurse`，它会遍历条件表达式树，针对不同类型的 `SubLink` 进行转化：

### 2.1 转化 `ANY` 子链接 (`convert_ANY_sublink_to_join`)
遇到类似 `WHERE x.id = ANY (SELECT id FROM y)`（常用的 `IN (子查询)`结构）时：
*   优化器会将其从 `WHERE` 条件中提取出来，并转化为能够与上层父查询合并的 **`JOIN_SEMI` (半连接)**。
*   如果其内部是一个包含常数的列表，例如 `IN (1, 2, 3)`，系统也会在这个阶段调用 `convert_VALUES_to_ANY` 尝试进行就地化简。

### 2.2 转化 `EXISTS` 子链接 (`convert_EXISTS_sublink_to_join`)
遇到 `WHERE EXISTS (SELECT 1 FROM y WHERE x.id = y.id)` 的结构时：
*   处理逻辑与 `ANY` 类似，将其转化为 **`JOIN_SEMI` (半连接)**。
*   如果原始条件包含 `NOT` 否定修饰（如 `WHERE NOT EXISTS ( ... )`），优化器将其转化为 **`JOIN_ANTI` (反连接)**。这使得优化器在后续的物理计划评估中，能够选择如 Hash Anti-Join 等高效的算子来进行排他性过滤。

---

## 3. 合并 Subquery：`pull_up_subqueries`

有别于作为条件表达式存在的 `SubLink`，正规的 `Subquery` 通常指的是位于 `FROM` 子句中、作为数据源使用的子查询（例如 `SELECT * FROM a JOIN (SELECT * FROM b) c ON ...`）。
当处理完 `SubLink` 后，规划器会调用 `pull_up_subqueries()`，尝试将 `FROM` 列表中的子查询结构展开并融进主查询。

### 3.1 简单子查询 (`is_simple_subquery` / `pull_up_simple_subquery`)
满足以下条件的会被判定为可以提升的“简单（Simple）”子查询：
*   不包含 `GROUP BY` 和 `HAVING` 子句。
*   不带有 `LIMIT` 和 `OFFSET`。
*   没有使用 `UNION`、`INTERSECT`、`EXCEPT` 等集合运算。
*   选择列表中不包含聚合函数或窗口函数。

在确认是简单子查询后，`pull_up_simple_subquery()` 会执行以下操作：
1. 将该子查询中的 `FromExpr` 节点包含的对象，合并到主查询所在层级的 `FromExpr` 树中。
2. 剥除该子查询外层的包装节点。
3. 调用 `pullup_replace_vars` 函数，处理和替换关联的变量（如果涉及到 Grouping Set，还需要将其装入 `PlaceHolderVar` 来维持语义）。

### 3.2 提升简单的 UNION ALL (`pull_up_simple_union_all`)
对于类似 `FROM (SELECT x FROM a UNION ALL SELECT x FROM b)` 的结构：
只要确认它不带有 `ORDER BY`，没有 `LIMIT`，并且是单纯的 `UNION ALL`（不需要去重），优化器就会调用 `is_simple_union_all`。
它会将这个 `UNION ALL` 树转换为平铺的追加关系列表 (`AppendRelInfo`)。这样就可以将集合操作展平为底层的 `Append` 节点，避免不必要的子查询资源开销。

### 3.3 处理 LATERAL 引用
在引入了 `LATERAL` 关键字后，提升逻辑变得更加复杂，因为它涉及对外层查询的变量引用。
如果在 `pull_up_subquery` 的过程中存在 `LATERAL` 产生的跨层引用，系统会通过调用诸如 `IncrementVarSublevelsUp` 等逻辑来修正 `Var` 变量的层级深度（Sublevel）。并且通过 `CombineRangeTables` 合并底层 RangeTable (范围表) 中的依赖信息，确保即使是这种带有侧向引用的子查询也能在语义不变的前提下安全“拍平”。
