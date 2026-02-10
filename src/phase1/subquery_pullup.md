# 子查询提升 (Subquery Pull-up)

子查询提升（Subquery Pull-up）是 PostgreSQL 优化器中最重要的逻辑优化之一。它的主要目的是将子查询（Subquery）合并到父查询（Parent Query）中，从而将原本的层次化结构“扁平化”。

## 为什么要进行子查询提升？

1.  **增加连接顺序的选择空间**: 如果保持子查询独立，优化器通常必须先优化子查询，再优化父查询，这限制了连接顺序（Join Order）。扁平化后，子查询中的表可以与父查询中的表自由结合，优化器可以探索更多的连接路径。
2.  **减少执行层级**: 减少了执行器层面的函数调用和上下文切换开销。

## 哪些子查询可以被提升？

主要处理 `FROM` 子句中的子查询（Subquery in FROM）。

1.  **简单子查询 (Simple Subqueries)**:
    *   不包含 `GROUP BY`, `HAVING`, `LIMIT`, `OFFSET`, `DISTINCT`, 集合操作（UNION/INTERSECT/EXCEPT）或窗口函数的子查询。
    *   这类子查询可以直接合并，将其 `WHERE` 条件 AND 到父查询中。

2.  **UNION ALL 子查询**:
    *   可以将 `UNION ALL` 扁平化为 Append 关系（Append Relation）。

## 特殊情况：GROUP BY 子查询的提升

在某些特定条件下，包含 `GROUP BY` 的子查询也可以被提升，但这通常涉及到更复杂的转换，如 **Aggregate Pushdown** 或 **Join Removal**。PostgreSQL 支持将某些 Outer Join 中的 Group By 子查询提升，但需要保证 Join Key 是 Unique 的。

## 源码分析

核心函数：`pull_up_subqueries` (位于 `src/backend/optimizer/prep/prepjointree.c`)

该函数递归地遍历 Query Tree 的 `jointree`，查找 `RangeTblRef` 指向的子查询。如果满足提升条件，则修改 `jointree` 结构，将子查询的 `FromExpr` 合并上来。
