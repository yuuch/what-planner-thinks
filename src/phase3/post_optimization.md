# 聚合与排序优化 (Post-processing)

在生成了连接路径（Join Paths）之后，优化器还需要处理 `GROUP BY`, `AGGREGATE`, `ORDER BY`, `DISTINCT`, `LIMIT` 等子句。这一阶段通常称为 upper planner 阶段。

## 聚合 (Aggregation)

PostgreSQL 支持多种聚合策略：

1.  **Plain Aggregate**: 没有 GROUP BY，只有聚合函数（如 `SELECT count(*) FROM t`）。
2.  **Sorted Aggregation (GroupAggregate)**:
    *   要求输入数据按 GROUP BY 键排序。
    *   流式处理，内存占用低。
3.  **Hash Aggregation (HashAggregate)**:
    *   在内存中构建哈希表。
    *   通常比排序聚合快，但如果内存不足（超过 `work_mem`），会溢出到磁盘（Disk Spill）。

优化器会根据输入路径的 PathKeys（是否已排序）和统计信息（组数估算）来决定使用哪种策略。

## 排序 (Sorting)

如果在之前的步骤中（Scan/Join）没有产生满足 `ORDER BY` 要求的顺序，优化器必须在顶层添加一个 `Sort` 节点。

*   **Incremental Sort**: 如果数据已经按 `(x)` 排序，但需要按 `(x, y)` 排序，PG 可以使用增量排序，只在 `x` 相同的组内对 `y` 排序。

## 其他操作

*   **LockRows**: 处理 `SELECT FOR UPDATE / SHARE`。通常在顶层添加 `LockRows` 节点。
*   **Limit/Offset**: 添加 `Limit` 节点。如果存在 Limit，优化器可能会倾向于使用 Startup Cost 更低的路径（如 Index Scan 而非 Seq Scan）。
*   **Window Functions**: 类似于聚合，但需要特定的窗口排序。
*   **Set Operations**: UNION, INTERSECT, EXCEPT。

## 源码入口

*   `grouping_planner`: 负责处理这些非 Join 的操作。
*   `create_grouping_paths`: 生成聚合路径。
