# 外连接消除与无用连接移除 (Outer Join Elimination & Join Removal)

在经过了表达式级别的折叠与扁平化之后，PostgreSQL 优化器会尝试通过约束分析，尽可能精简查询树中的连接节点（Join Nodes）。精简的连接树不但执行效率高，同时也给了后续的动态规划/遗传算法更多排列组合的选择。

这一步主要包含了两个阶段的优化：
1. **外连接降级/消除 (reduce_outer_joins)**
2. **无用连接移除 (remove_useless_joins)**

## 1. 外连接降级的原理与意义 (`reduce_outer_joins`)

**源码位置**：`src/backend/optimizer/prep/prepjointree.c`
**调用时机**：它是由 `subquery_planner` 函数中调用的。

外连接（Outer Join）通常会强制优化器遵循特定的连接顺序，因为 Nullable 一侧必须在关联后才能保留。`reduce_outer_joins` 的目的就是检查是否存在某些强有力的 `WHERE` 条件，使得原本的 Outer Join 可以退化为 Inner Join 甚至是 Anti Join。

### 1.1 两遍扫描机制 (Pass 1 & Pass 2)
为了减少不必要的递归，`reduce_outer_joins` 使用两阶段处理：
*   **Pass 1 (`reduce_outer_joins_pass1`)**: 收集查询树底层的 `Relids`（涉及了哪些基础表），并快速标记哪些子树中**真正包含外连接**（`contains_outer`）。如果没有外连接，直接跳过整个复杂的分析。
*   **Pass 2 (`reduce_outer_joins_pass2`)**: 进入实际的检查和转换逻辑。

### 1.2 Outer 到 Inner 的降级
其核心依靠提取过滤条件中的**严格约束 (Strict Quals)**。
利用 `find_nonnullable_rels` 扫描 `WHERE` 子句。如果 `WHERE` 强制要求 Nullable Side 的某个引用列不能为空（比如 `LEFT JOIN B ON ... WHERE B.id > 10`，当 B 补充 NULL 时，`NULL > 10` 为 NULL/False 被淘汰），那么：
*   **LEFT JOIN** 可以安全地转换为 **INNER JOIN**。
*   **RIGHT JOIN** 可以安全地转换为 **LEFT JOIN** 然后内部翻转，或者直接转为 **INNER JOIN**。
*   **FULL JOIN** 可以根据两边约束的情况降级为 LEFT、RIGHT 或是 INNER JOIN。

### 1.3 Outer 到 Anti 的转化
外连接不仅可以被降阶为内连接，有时还可以变成反连接。
利用 `find_forced_null_vars` 检查 `WHERE` 是否有显式要求变元为 NULL 的约束（例如 `LEFT JOIN B ON ... WHERE B.id IS NULL`）。
如果是这样，那么这个 Join 明确就是用来找“匹配不上的行”，代码中会将该 `JOIN_LEFT` 转换为 **`JOIN_ANTI`**。Anti Join 在执行器阶段具有更早熔断的高效特性。

## 2. 无用连接移除 (`remove_useless_joins`)

**源码位置**：`src/backend/optimizer/plan/analyzejoins.c`
**调用时机**：它发生得晚一些，是在 `grouping_planner` -> `query_planner` 的起始阶段，即准备构建底层路径（Scan / Join Paths）时进行。

有时候我们连接了一张表，只是为了取个数据，但后来发现（由于视图展开、ORM 生成的过度复杂的 SQL 等原因）这张表输出的列根本没有任何用处，那我们能直接移除这个 JOIN 吗？
如果这是一个 `LEFT JOIN`，并且满足某些严格的等价条件，就能做到。

### 2.1 移除的苛刻条件 (`join_is_removable`)
要移除一个左连接的右表，必须同时满足以下苛刻条件：
1.  **没有列输出**：Nullable Side（被连接的右表）的任何字段（包括 PlaceholderVars) 都**没有**被上层的 `TargetList`、`HAVING`、`ORDER BY` 引用（检查 `attr_needed` 集合）。
2.  **不改变左表的基数 (Cardinality)**：这个最为关键。不能因为移除右表，导致原本左表匹配出多行的结果减少，或者原本膨胀的行数变少。因此右表的关联必须是**最多匹配一行**。
    * 优化器会调用 `rel_is_distinct_for`。
    * 根据连接条件，判断右表在这些条件限制下是否呈现**唯一性 (Unique)**。
    * 这通常依赖于数据库中建立的**唯一索引 (Unique Index)** 或 **主外键约束 (Foreign Key)**，且连接符必须是可合并连接的等值运算符。

### 2.2 全局状态清理 (`remove_leftjoinrel_from_query`)
如果这个无用连接成功被认定为可以移除，这绝不仅仅是在查询树里删掉一个节点这么简单。
因为此时规划器已经构建了诸如 `RestrictInfo`、`EquivalenceClass`（等价类）甚至是各种 `PlaceHolderVar`。规划器必须倒查并清理由于移除了该表（及其 Relid，甚至其特有的 Outer Join Relid `ojrelid`）而受到关联的所有辅助结构，确保后续基于 DP（动态规划）产生的子集连接搜索不会引用到幽灵节点。
