# 并行执行计划生成 (Parallel Query Planning)

从 PostgreSQL 9.6 开始，引入了并行查询功能。优化器需要决定是否生成并行计划，以及如何分配 workers。

## 核心概念

1.  **Parallel Safe**: 只有当查询中使用的所有函数和操作符都是“并行安全”的，才允许并行执行。
2.  **Partial Path**: 并行 workers 执行的部分工作路径。例如 `Parallel Seq Scan`。
3.  **Gather Node**: 负责收集 workers 的结果并汇总的节点。
    *   `Gather`: 无序收集。
    *   `Gather Merge`: 按顺序收集（保持下层路径的排序）。

## 规划流程

1.  **生成 Partial Paths**:
    *   在扫描基表时，除了生成常规路径，还会尝试生成 `Parallel Seq Scan` 等 Partial Paths。
2.  **Join 中的并行**:
    *   如果输入路径是 Partial 的，PG 支持 **Parallel Hash Join** (Shared Hash Table) 和 **Parallel Nested Loop**。
3.  **最终汇总**:
    *   在 `standard_planner` 的后期，检查是否有名为 `Gather` 的最优路径。
    *   如果有，优化器会根据配置（`max_parallel_workers_per_gather`）和表大小决定启动多少个 workers。

## 代价模型调整

并行计划的代价 = (Total Cost / Workers) + Communication Cost。
优化器会权衡并行带来的加速与进程间通信/启动的开销。
