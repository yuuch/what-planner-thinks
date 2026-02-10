# 选择率与代价模型 (Selectivity & Cost Model)

代价模型（Cost Model）是基于代价的优化器（CBO）的灵魂。PostgreSQL 将时间作为代价的统一单位。

## 成本单位

PostgreSQL 在 `postgresql.conf` 中定义了一系列基础成本参数：

*   `seq_page_cost` (默认 1.0): 顺序扫描一个磁盘页面的代价（基准）。
*   `random_page_cost` (默认 4.0): 随机访问一个磁盘页面的代价。
*   `cpu_tuple_cost`: 处理一行元组的 CPU 代价。
*   `cpu_index_tuple_cost`: 处理索引元组的 CPU 代价。
*   `cpu_operator_cost`: 执行一个操作符或函数的代价。

总代价 = I/O 代价 + CPU 代价。

## 统计信息 (Statistics)

为了估算代价，必须先估算**行数**（Cardinality）。而行数的估算依赖于统计信息。
统计信息存储在 `pg_statistic` 系统表中（用户可以通过 `pg_stats` 视图查看）。

主要统计项：
*   `null_frac`: NULL 值的比例。
*   `avg_width`: 列的平均宽度（字节）。
*   `n_distinct`: 不同值的数量（如果是负数，表示占总行数的比例）。
*   `most_common_vals` (MCV): 最常出现的值列表。
*   `most_common_freqs`: MCV 对应的频率。
*   `histogram_bounds`: 直方图边界（用于非 MCV 的数据分布）。
*   `correlation`: 物理行序与逻辑值的相关性（影响 Index Scan 代价）。

## 选择率 (Selectivity)

选择率是一个 0 到 1 之间的概率值，表示经过条件过滤后剩余行数的比例。

`Rows = Total_Rows * Selectivity`

### 常用估算函数 (`src/backend/utils/adt/selfuncs.c`)

*   `eqsel`: 等值条件 (`=`) 的选择率。优先查 MCV，没有则用 `1 / n_distinct`。
*   `scalarltsel` / `scalargtsel`: 范围条件 (`<`, `>`) 的选择率。主要使用直方图。
*   `clauselist_selectivity`: 组合多个条件的选择率。假设列之间相互独立（Independence Assumption），通常是连乘 `P(A) * P(B)`。但 PG 也有扩展统计信息（Extended Statistics）来处理列相关性。

## 代价计算示例：Seq Scan

`Cost = (disk_pages * seq_page_cost) + (rows * cpu_tuple_cost) + (rows * filter_cost)`

*   优化器会先估算表的大小（pages）和行数（rows）。
*   如果有过滤条件，还需要加上计算过滤表达式的 CPU 开销。
