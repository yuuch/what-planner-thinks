# 多表连接顺序 (Join Ordering)

当查询涉及多个表时，决定表的连接顺序（Join Order）是优化器最核心、最耗时的任务。连接顺序的选择对性能影响巨大。

## 动态规划 (Dynamic Programming)

PostgreSQL 默认使用标准的动态规划算法（System R 风格）来寻找最优连接树。

### 算法步骤

1.  **Level 1**: 已经计算好所有单表（Base Relations）的最优访问路径。
2.  **Level 2**: 考虑所有可能的两表连接（Rel A + Rel B）。计算各种连接方式（NestLoop, Hash, Merge）的代价，保留最优路径。
3.  **Level 3**: 将 Level 2 的结果与第三个表连接（(AB) + C）。
4.  **...**
5.  **Level N**: 得到包含所有表的最优路径。

### 搜索空间限制

理论上，N 个表的连接顺序是 N! (factorial)。为了避免搜索空间爆炸，PG 默认只考虑 **Left-deep Tree**（左深树）和 **Bushy Tree**（灌木丛树/多支树）。
*   早期的优化器倾向于左深树（右操作数必须是基表），但现代 PG 也会探索 Bushy Tree（允许两个 Join 的结果再进行 Join）。

### 核心函数

*   `make_one_rel`: 构建最终的关系。
*   `make_rel_from_join_list`: 根据 `join_info_list` 指导连接过程。
*   `standard_join_search`: 标准的 DP 实现。
*   `join_search_one_level`: 计算某一层级的连接。

## Join Rel 构建

对于每一对需要尝试连接的关系（RelOptInfo），优化器会调用 `make_join_rel` -> `populate_join_path` 来生成 `NestLoop`, `MergeJoin`, `HashJoin` 等具体路径，并根据代价选优。
