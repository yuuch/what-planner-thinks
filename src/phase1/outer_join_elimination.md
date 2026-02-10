# 外连接消除 (Outer Join Elimination)

外连接消除（Outer Join Elimination）是指在不改变查询语义的前提下，将 Outer Join（Left/Right/Full Join）转换为 Inner Join，或者完全移除该 Join。

## 转换原理

### 1. Outer Join 转换为 Inner Join

当 Outer Join 的“空值生成端”（Null-generating side）受到一个“严格”（Strict）的过滤条件限制时，Outer Join 等价于 Inner Join。

*   **严格条件**: 指当输入为 NULL 时，表达式结果为 NULL 或 False 的条件。例如 `WHERE t2.col > 5`。
*   **场景**: `SELECT * FROM t1 LEFT JOIN t2 ON t1.id = t2.id WHERE t2.col > 5;`
    *   在 Left Join 中，如果 t2 匹配不上，`t2.col` 会是 NULL。
    *   `NULL > 5` 为 NULL（即 False）。
    *   因此，所有 t2 补 NULL 的行都会被 WHERE 过滤掉。
    *   查询等价于 `SELECT * FROM t1 JOIN t2 ...`。

转换的好处是 Inner Join 的连接顺序更灵活（可以交换左右表），而 Outer Join 的顺序通常是固定的。

### 2. Join 移除 (Join Removal)

如果连接的结果没有被使用，且连接操作不影响行数（Cardinality），则可以移除 Join。

*   **场景**: `SELECT t1.x FROM t1 LEFT JOIN t2 ON t1.id = t2.id;`
    *   如果我们只查询 `t1` 的列。
    *   且 `t2.id` 是唯一的（Unique），或者说 `t1.id` 参照 `t2.id` 的外键存在。
    *   Left Join 保证了 `t1` 的每一行至少出现一次。
    *   如果 `t2` 端匹配唯一，则 `t1` 的行不会膨胀。
    *   此时可以移除 `t2` 的连接。

## 源码分析

核心函数：`reduce_outer_joins` (位于 `src/backend/optimizer/prep/prepjointree.c`)

该函数在 `subquery_planner` 早期被调用。它通过分析 `WHERE` 子句和 `JOIN/ON` 子句中的约束条件（Qualifiers），判断是否满足“严格性”要求。

*   **Full Join**: 可以先降级为 Left 或 Right，再进一步降级为 Inner。
*   **Left Join**: 如果右表（Nullable side）有强过滤条件，降级为 Inner。
