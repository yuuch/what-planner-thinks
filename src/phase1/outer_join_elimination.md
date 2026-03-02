# 外连接消除与连接移除 (Outer Join Elimination & Join Removal)

当查询树完成了表达式的重写和规范化之后，PostgreSQL 优化器会尝试通过逻辑推导和约束条件，精简查询树中不必要的表连接（Join）节点。
一棵被修剪和化简过的查询树不仅提高了后续执行的效率，也极大地减少了动态规划或遗传算法在生成 Join Order 时需要枚举的组合空间。

相关的优化逻辑主要分为两个阶段：
1. **外连接消除 (reduce_outer_joins)**
2. **连接移除 (remove_useless_joins)**

---

## 1. 外连接消除 (`reduce_outer_joins`)

**源码位置**：`src/backend/optimizer/prep/prepjointree.c`
**执行时机**：由查询的主计划函数 `subquery_planner` 调度。

外连接（Outer Join）相较于内连接（Inner Join），会限制优化器对表连接顺序的自由排列组合。`reduce_outer_joins` 的主要任务是遍历查询条件（如 `WHERE`），检查是否存在能够使外连接降级（由于 NULL 值过滤条件）并转化为安全且更高效的 Inner Join 或 Anti Join (反连接) 的情况。

### 1.1 两次遍历策略 (Pass 1 & Pass 2)
为了提升效率，`reduce_outer_joins` 使用了两轮过滤法：
*   **粗筛阶段 (Pass 1: `reduce_outer_joins_pass1`)**: 遍历提取底层节点的 `Relids`（涉及的基础表），并标记查询中是否存在真正的外连接（设置标志位 `contains_outer`）。如果没有外连接存在，则立即返回，省去后续的处理开销。
*   **重写阶段 (Pass 2: `reduce_outer_joins_pass2`)**: 仅对确认含有外连接的查询部分，进行严密的逻辑推演和类型转化。

### 1.2 外连接向内连接的转化 (Outer to Inner)
将外连接降级的核心在于寻找表达式中存在的**严苛约束条件 (Strict Quals)**。
优化器会调用 `find_nonnullable_rels` 在条件子句中搜寻：是否某个条件要求原本由于外连接产生的 Null 值端，必须是个非空合法实体。
例如，执行 `LEFT JOIN B ON ... WHERE B.id > 10` 时。如果在执行左连接后 B 表的行以 NULL 补充，而在条件评估阶段遇到 `NULL > 10` 会被判定为 False 而过滤掉这些行。那么这个保留了左侧不匹配记录的“左连接”其实和内连接没有任何区别。

面对这种依赖性质：
*   **LEFT JOIN** 会被转化为 **INNER JOIN**。
*   **RIGHT JOIN** 则可能转为 **LEFT JOIN**（同时翻转关联左右侧）或转为 **INNER JOIN**。
*   **FULL JOIN** 会根据左右侧各自的约束条件情况，降级成 LEFT JOIN、RIGHT JOIN，或直接化简为 INNER JOIN。

### 1.3 外连接向反连接的转化 (Outer to Anti)
除了转化为内连接，外连接在某些情况也可以转为反连接。
通过调用 `find_forced_null_vars`，优化器会检查查询条件中是否明确要求连接生成的特定列必须为 NULL（常见的例子是 `LEFT JOIN B ON ... WHERE B.id IS NULL`）。
在这种情境下，由于逻辑实际上是为了寻找无法在右表找到匹配记录的行，系统会将该 `JOIN_LEFT` 转换为 **`JOIN_ANTI` (反连接)**。在物理执行层面，Anti Join 在找到第一个匹配项时就可以停止搜索，相比保留左连接具有显著的性能优势。

---

## 2. 无用连接的移除 (`remove_useless_joins`)

**源码位置**：`src/backend/optimizer/plan/analyzejoins.c`
**执行时机**：由 `query_planner` 的起始阶段进行调用，发生在生成扫描路径和连接路径的前夕。

在复杂的应用或基于视图的查询中，常常包含对当前结果没有贡献的表关联。此时，优化器可以识别并彻底移除某些表连接（通常针对 `LEFT JOIN` 的右侧表）。

### 2.1 移除的判断依据 (`join_is_removable`)
要安全地将 `LEFT JOIN` 中的右侧表移除，必须同时满足以下严苛的条件验证：
1.  **没有在上层提供输出列**：被连接的右表中的列，没有在更高层的 `TargetList` (选择目标集)、`HAVING` 等子句中被引用（优化器会通过 `attr_needed` 属性检查）。
2.  **不改变基数 (Cardinality)**：移除右表不能影响结果集行数或数据唯一性。也就是说，右表能够提供的连接匹配项，对于左表的每一行，**必须且只能匹配出最多一行**，否则会引发由于交叉连接产生的行数变动。
    * 优化器会通过 `rel_is_distinct_for` 等逻辑。
    * 检查连接条件是否能保证右表记录的**唯一性 (Unique)**。
    * 这通常依赖于数据库里的**唯一索引 (Unique Index)** 或**主外键约束 (Foreign Key)**，且这些关联必须使用等值操作符（Mergejoinable Equality Operators）。

### 2.2 状态清理与重组 (`remove_leftjoinrel_from_query`)
如果一段关系满足了被抛弃的条件，优化器不仅要在查询树结点上剥除该分支，还需要清理那些由此连带创建的衍生结构。
在判定废弃前，规划器已经为其建立了例如约束信息 `RestrictInfo`、等价类 `EquivalenceClass`，甚至还有部分 `PlaceHolderVar`。
在确认移除该表及其相关的 `Relid`（以及包含的 Outer Join Relid `ojrelid`）后，规划器必须进行全覆盖的清算操作，把涉及这些表的关联辅助结构全部删去，以免这些废弃的数据状态影响接下来的动态规划寻优操作。
