# 后置优化与物理算子拼接 (Post-Optimization & Upper Planner)

如果说查询规划的“上半场”（严格来说在代码里叫 Lower Planner，即底层的 Join 树构建）解决的是“数据去哪里找、按什么顺序拼接最快”的问题，那么规划机制的“下半场”——**Upper Planner (高层优化器)**，解决的则是“拼完了数据怎么满足用户最终的语义控制”的问题。

当你的 `FROM` 和 `WHERE` 连接网络被 `query_planner` 函数结算为一颗拥有最优代价值的 `RelOptInfo`（例如 `join_rel_level[N]`）返回时，优化器会从 `query_planner` 函数退出来，回到其广袤的母调度函数：**`grouping_planner`**（位于 `src/backend/optimizer/plan/planner.c`）。

在 `grouping_planner` 中，系统会为这棵刚刚建成的基底数据树逐层裹上“包装”，处理包括且不仅限于：`GROUP BY` (聚合), `WINDOW FUNC` (窗口函数), `ORDER BY` (排序), `DISTINCT` (去重) 以及 `LIMIT/OFFSET` (截断) 等高级操作。

结合 PostgreSQL 18.1 的最新代码，我们将深入这条“物理算子包装流水线”。

---

## 1. 入口统筹：Upper Rel 与 `grouping_planner`

与底层基表（Base Rel）或连接（Join Rel）不同，PG 抽象出了一种名为 **Upper Rel (上层关系)** 的 `RelOptInfo`，通过 `UPPERREL_GROUP_AGG`、`UPPERREL_WINDOW`、`UPPERREL_ORDERD` 这种专有的 `UpperRelationKind` 分级不断往顶层堆叠。

在 `grouping_planner` 中，标准的装配链（Pipeline）是极其严苛有序的。每当装配完一层，就会生成一个涵盖了前面所有计算结果的新 `RelOptInfo`：
1. 底层扫描与联接 (`query_planner` 完成)。
2. （如有）追加分组与聚合 `UPPERREL_GROUP_AGG`。
3. （如有）追加窗口函数计算 `UPPERREL_WINDOW`。
4. （如有）追加 `SELECT DISTINCT` 去重。
5. （如有）追加 `ORDER BY` 排序 `UPPERREL_ORDERD`。
6. （如有）追加 `LIMIT / OFFSET` 截断。

下面重点剖析这套外置神兵利器的核心模块。

---

## 2. 聚集规划 (Aggregation & Grouping)

在面对包含聚合函数（`SUM()`, `AVG()`）或声明了 `GROUP BY` 的查询时，系统会跳入 `create_ordinary_grouping_paths()` （普通分组路径构建）去生成聚合方案。

PG 支持三种硬核的物理聚合算子：
### 2.1 基础聚合 (Plain Aggregate)
应对的是诸如 `SELECT count(*) FROM table`，没有 `GROUP BY` 的查询。
此时系统只需在顶端挂载一个单行的聚集器。输入流来了之后，它只会在内存中累加内部状态值（Internal State），流结束后输出唯一的一条合并行记录。启动代价小，逻辑极其精简。

### 2.2 排序归并聚合 (GroupAggregate / Sorted Aggregation)
要求输入的数据流**必须已经按照 `GROUP BY` 列强行排好序**。
*   **机制**：类似于一个流式游标。当读入的值和上一条的值在 Group 键上一样时，累加聚合状态；一旦读到新的 Group 键值不同的数据（越界），立即抛出刚刚结算完的那个聚合组结果，并重置内部计数器重新开始。
*   **优缺点**：内存占用可谓是极致的省（无论表有几百 T，内存消耗也几乎不变）。但代价是：如果你底层吐出来的数据压根没有 `pathkeys`（比如是用 HashJoin 做首选计划出来的无序堆栈），GroupAggregate 就会惨叫着要求在底下先强压一个昂贵的 `Sort Node`（排序算子）再去 Agg。

### 2.3 哈希聚合与落盘 (HashAggregate & Disk Spill)
不需要预先排序。
*   **机制**：上来就在内存（`work_mem`）里修筑一个庞大的 Hash Table。扫入的数据通过计算 Group 键的 Hash 值强行扔入不同的 Bucket 里并累加状态。扫完全表后，再将 Hash 表里的结果一口气倾倒（Egest）出栈。
*   **重大进化 (PG 13+)**：在老版本的 PG 中，如果 Group 的类别数量异常庞大，Hash 表把内存活生生撑爆会导致直接 OOM 报错。后期的 PG 加入了 **HashAgg Disk Spill (落盘机制)**。这意味着优化器在调 `cost_agg` 进行代价估测时，一旦发现不同组数（Number of Groups，通过 `ndistinct` 估算）超过内存，它也不会简单粗暴抛弃哈希聚合，而是会引入多批次落盘合并的额外磁盘 I/O 代价折算惩罚。
*   **Mixed Aggregate**：如果 `GROUP BY` 有多组展开逻辑（如 `GROUPING SETS`, `CUBE`, `ROLLUP`），PG 会利用专有的 `GroupingSetsPath` 同时衍生混合 Hash 与 Sorted 的交叉策略。通常是对 ROLLUP 等能构成线性前缀依赖的使用 Sort 算子一次切出多个聚合层次（优美地降维），对平级的无序组合使用 Hash 并行构建。

---

## 3. 窗口函数 (Window Functions)

在聚合之后，就是窗口函数的战场 `UPPERREL_WINDOW`（位于 `planner.c: create_window_paths`）。

当你用到了 `SUM(x) OVER (PARTITION BY a ORDER BY b)` 这种强大的跨窗函数时，必须明白：
1. 窗口函数实质上是一种变种的 GroupAgg，它**极端渴求**输入序列必须按 `a` 和 `b` 严格有序（`pathkeys` 的要求简直苛刻到了极致）。
2. **多窗口合并优化**：如果一个 SQL 里有五六个窗口函数，有的 `PARTITION BY a ORDER BY b`，有的则是 `PARTITION BY c`，PG 会在 `group_window_funcs` 操作里尝试把 `OVER` 子句诉求完全一致的合并在一次物理节点轮询里执行掉。不同规则的必须被切割成独立执行栈。
3. 优化器会重新评估底下这堆由 `GROUP BY` 输出出来的路（Path）。假如下面的路正好自带按 `a, b` 排好的 `pathkeys`，那就省去了二次排序，直接加上窗口算子。如果没有？那就硬着头皮加上显式的 Sort Node。

---

## 4. 排序的神器进阶：增量排序 (Incremental Sort)

这是近期 PostgreSQL 带来的真正惊艳业界的一大特性。
所有的排序需求（无论源于 `ORDER BY a, b` 还是别的）归根结底都要通过 `create_ordered_paths` 往上塞。对于最棘手的 `Sort` 我们以前只有全表堆排序。

**试想场景**：
假如你的查询需要 `ORDER BY a, b`。由于你在 `a` 上有一个出色的 B-Tree 索引，底层的 Index Scan 替你捞上来的数据**确实已经按 `a` 排好序了，只是 `b` 还是乱序的**。也就是所谓的“提供了部分前缀排序（Prefix Sorted）”。
在没有 Incremental Sort 之前，PG 会长叹一声：“反正 `pathkeys` 对不上 `(a,b)` 组合的完美一致，底下的排序全当废纸！全量装进内存来强制 Sort！”。

**Incremental Sort 机制**：
现在，由 `pathnode.c` 中相关的逻辑负责捕获 `Prefix Sorted` 特征。当检测到已存在 `a` 的前缀有序性时，优化器能够插入一条轻量级的 `Incremental Sort` 路径。
在执行时，只要探测到前一个读取的 `a` 的值没变，它就把 `b` 暂时吃进缓存；一旦 `a` 变了，这意味着前一个组所有的 `a` 全部到齐了，就立即针对前面那个小缓存栈里的区区几条数据对 `b` 进行**极低代价的微型内存 Sort 并输出。** 这个技术直接将 `ORDER BY a, b` LIMIT 查询的延迟在很多长尾场景下削减了数量级倍。

---

## 5. 截断 (Limit / Offset) 对底层引擎计划的倒逼

如果你以为 `LIMIT 1` 只是最后在最外层加一个类似 `head -n 1` 的截断算子，那就大错特错了。
`LIMIT` 具备极为强烈的**计划倒置影响力 (Plan Inversion Power)**！它存在于 `create_limit_path()` 之中。

当你的整棵查询树带着几十万总代价值（Total Cost），经过了一路淘汰走到最顶的 `grouping_planner` 遇到了 `LIMIT 10` 时：
PG 的代价估算器（`cost_limit` 函数）不仅修改上层代价值，甚至会去翻阅之前在 `RelOptInfo->pathlist` 遗留的所有未决执行路径。

之前说过，优化器会为一个组合保留两条路：`cheapest_startup_path` (启动极快但总体最慢) 和 `cheapest_total_path` (启动很慢需要等待加载但一旦动起来总体全扫完最快)。
如果是没有 LIMIT 的普通提取：`Hash Join` 由于扫全表建哈希表，`total_cost` 极低，必定压倒 `NestLoop`。
但在有了 LIMIT 之后：优化器**只需要拿到十行记录** (tuples needed = 10，基于 `rel->rows` 做了乘积比估值降维)。`Hash Join` 因为需要先把两张表各自的几百万全缓存下来才能吐出第一行，其 `startup_cost` 令人发指。而看似非常笨拙的 `NestLoop + Index Scan` 却能在 0 缓冲的情况下瞬间拿到第一行、第十行的结果并直接终止循环。

在 `LIMIT` 的神来之笔下，优化器会给拥有最优 `startup_cost` 的路径重新打上高倍数的惩罚削减分数，成功颠覆由系统默认的总价计算主导下的 HashJoin 王权，切换到那些可能一辈子都不会被选用的 `Fast-Start` 参数化嵌套循环和纯净索引路径（Index Scan with order）。

整个 `post_optimization` 的 Upper 处理环节看似简单地串联收尾，实则是处处存在与底层状态联动拉扯的精妙算计！
