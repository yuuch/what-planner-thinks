# 单表扫描路径生成 (Access Paths)

在 PostgreSQL 查询优化的漫长旅程中，一旦查询树（Query Tree）经过了逻辑重写、子查询提升、表达式扁平化以及外连接消除等预处理阶段，规划器（Planner）就正式跨入了核心的**代价优化阶段（Cost-Based Optimization, CBO）**。

代价优化的第一步，也是构建整棵物理执行计划树的基石，就是为查询中涉及的每一个基础表（Base Relation）计算出所有可行的**单表扫描路径（Single Relation Access Paths）**。在这个环节，无论是顺序全表扫描、各种变体的索引扫描，还是并行的扫描方式，都会被事无巨细地生成出来，供后续的路径打分和连接搜索使用。

本章节将结合 PostgreSQL 18.1 的源码（主要集中在 `src/backend/optimizer/path/allpaths.c` 和 `indxpath.c`），带你深度潜入单表扫描路径的生成机制与数据结构。

---

## 1. 核心数据结构与入口：`make_one_rel`

生成基础路径的整个过程由核心大管家函数 `make_one_rel()`（位于 `allpaths.c`）统筹。顾名思义，它的最终目标是把查询里的所有关系（Relations）“合成”为一个整体（One Rel），代表最终的查询结果。

在此之前，首先要做的就是武装每一个基表。

### 1.1 两个关键阶段的拆分
在 `make_one_rel` 中，对基表的处理被极其考究地拆分为了两大步：
1. **尺寸推断 (`set_base_rel_sizes`)**：先估算出行数（Rows）、宽度（Width）等统计学信息。
2. **路径生成 (`set_base_rel_pathlists`)**：基于前面算出的尺寸信息，真刀真枪地生成多种底层数据访问的物理路径（Path）。

**为什么要严格分两步走？**
因为并行查询和参数化路径的生成极度依赖于表的大小估算。
举个例子，某个表的单表估算数据量如果太小，优化器可能就会直接将其标记为**不需要并行（Cancel consider_parallel）**。如果一边估算一边生成，可能会导致前面生成的路径错失获取准确统计信息或是其它表横向对比信息的机会。

### 1.2 `RelOptInfo`：关系信息的集大成者
在进入生成环节时，每个基表、连接结果甚至子查询，都会被包装成一个非常庞大且极为核心的结构体——`RelOptInfo`（Relation Optimization Info，定义于 `nodes/pathnodes.h`）。

在路径生成阶段，`RelOptInfo` 有几个极为关键的字段：
*   **`rows` / `reltarget`**: 代表估算的基表输出行数与列宽，决定了扫描的代价。
*   **`baserestrictinfo`**: 也就是大家最熟悉的 `WHERE` 过滤条件。在逻辑优化阶段，它们已经被分解成了隐含着 AND 语意的约束信息列表集（`RestrictInfo` 的 List）。
*   **`pathlist`**: 这是一个链表（List）。在这一阶段，由 `set_base_rel_pathlists` 各种手段**疯狂塞入的扫描路径（Path）都会挂载在此**。
*   **`cheapest_startup_path` / `cheapest_total_path`**: 在尝试了所有可能的 Path 后，优化器会选拔出**启动代价最小**和**总体运行代价最小**的两名“状元”，缓存于此，方便快速检索。

---

## 2. 顺序全表扫描 (Sequential Scan) 与并行化

顺序扫描可以说是任何一棵基表的“保底策略”。即使你没有任何索引，哪怕代价再高，数据库也一定能通过从外存中一页一页读取全表来得到正确结果。

在 `allpaths.c` 的 `set_plain_rel_pathlist` 里：

### 2.1 兜底路径：`create_seqscan_path`
优化器首先无条件地通过 `add_path(rel, create_seqscan_path(root, rel, required_outer, 0))` 塞入一个 `SeqScan`（对于普通关系表）。
顺序扫描的物理含义是沿着堆表（Heap Table）的底层分页逻辑，从第一块（Block 0）老老实实地扫到最后一块。

*   **代价估算要素**：顺序扫描的代价几乎只由两件事决定——这表有多少个块（`rel->pages`）决定了连续 I/O 时间；这表能匹配多少条记录（`rel->rows` / CPU filtering）决定了 CPU 处理条件判断的时间。

### 2.2 并行化扫描：`create_plain_partial_paths`
在 PostgreSQL 支持了查询并行化后，SeqScan 迎来了一个重量级的变体——**Partial Sequential Scan**。
紧接着生成单纯的 SeqScan 后，优化器会判断：
```c
if (rel->consider_parallel && required_outer == NULL)
    create_plain_partial_paths(root, rel);
```
在 `create_plain_partial_paths` 的内部，系统调用 `compute_parallel_worker()` 会根据该表的大小，套用一系列数学对数阈值来计算**该表值得分配多少个后台自动工作进程（Background Workers）**来齐头并进地扫描。
算出的 worker 数量会被赋值，并且调用 `add_partial_path` 添加一个带 `parallel_workers` 属性的分支。
*注意：这里的 partial (单方面/部分) 指的仅是单个 worker 只会扫描其中的一部分块（PG 的并行扫描机制通过在内部共享一个 Block 计数器来抢块，而不是静态切片），最后必须在顶层被 `Gather` 节点统一汇总。*

---

## 3. 索引扫描 (Index Paths) 深度剖析

只有全表扫描当然是不够的。对于大多数 OLTP 事务型查询和具备高选择性（High Selectivity，指只能捞出表中极少数特定行）的查询，利用 B-Tree 等索引能将查询耗时呈指数级地降低。

接下来，`set_plain_rel_pathlist` 会调用 `indxpath.c` 里的终极重头戏——`create_index_paths()`。这个函数的工作深度极高，它会尝试穷尽所有的单索引甚至多索引组合（Bitmap Scan）方案。

### 3.1 `Index Scan`：最传统的单索引回表策略
当通过 `build_index_paths` 检索到 `WHERE` 里面的约束（比如 `id = 5`）能够利用某个存在于系统的 `IndexOptInfo`（索引优化信息）时，将产生一条标准的 `Index Scan Path`。

**执行流约束**：
对于一个普通索引扫描，即使它在聚簇树的叶节点上瞬间定位了 `id = 5` 的数据条目所在地，获得了 `TID`（Tuple ID），它依旧不得不产生一次**回表（Heap Fetch）动作**，跳回表的数据块把整条记录抽上来，然后再判断这条记录上的其它没有索引的非主键约束（例如 `name = 'pg'`) 是否满足条件。

### 3.2 `Index Only Scan`：VM 护航的最佳拍档
这也是在 `build_index_paths` 中自动衍生探索的一种路径。
很多时候由于查询写得很棒，你要取出的列，在那个索引本身涵盖的结构里就已经全部存在了（比如 `SELECT id FROM table WHERE id = 5`）。在这种极端的完美情境下（又名 Covering Index），优化器其实**不需要也不应该**回堆表的数据块（Heap Page）。

**致命局限与 Visibility Map**：
然而由于 PostgreSQL 是基于经典的多版本并发控制（MVCC）并把系统表自身作为版本维护载体，仅仅提取索引键出来并不能断定这行数据在当前事务快照里是不是**已经过期的（Dead Tuple）**，或者是不是还处于对当前事务不可见的在途事务中——因为行可见性（`xmin`、`xmax`）等藏在原始数据行的 Header 里，索引条目上没有！
为了促成不用回表的 `Index Only Scan`，PG 引入了 **VM（Visibility Map，可见性映射文件）**。
在 `create_index_paths` 评估代价时，会参考目标表对应数据块在 VM 里的位标志：只有当该块的所有数据被标为 `all-visible`（该块没有在跑并发脏写事务，全局确定可见），优化器才会确认能避开 Heap Fetch。否则哪怕列都在索引里，也会被按回表比例加大代价，甚至直接回退到普通的 Index Scan。

### 3.3 `Bitmap Index Scan` 与 `Bitmap Heap Scan`
如果你的 WHERE 里不是一个等值，而是范围，甚至还有 `OR` 分离的多个字段索引用法怎么办（比如 `WHERE age = 20 OR salary > 8000`）？

依靠单一的 `IndexScan` 会因为疯狂读跳不同索引来提取而带来极度混乱的随机 I/O （Random I/O）。于是优化器会构建两阶段联合战法——**位图扫描（Bitmap Scan）**。
在 `indxpath.c` 里相关的函数 `generate_bitmap_or_paths` 处理。

*   **阶段 1: `Bitmap Index Scan`**：先不对表数据做任何触碰，仅仅在不同的索引文件中分别扫荡。如果匹配了某一行，就在内存里建立的一个虚拟位图（通常是物理 Page 级别的）把对应该行的位（Bit）涂成全变位 `1`。
*   如果针对两个索引的查询是 `AND` 或 `OR` 关系，由于得到的仅仅是纯碎的内存位图，在内存中直接进行**集合按位与、或操作（BitmapAnd / BitmapOr）**的速度是非常极致的，合并非常优雅。
*   **阶段 2: `Bitmap Heap Scan`**：拿到最后融合好的一张硕大 Bitmap 后，拿着这张图去访问表数据。因为是依着物理块序列号线性向后推进读取 `1` 的地方，此时对磁盘带来的完全是优雅降级的**近乎顺序连续 IO (Sequential IO)**，从而消灭了直接回表带来的大量碎片化磁盘寻道。这是一项非常有智慧的做法。

**劣势**：Bitmap Heap Scan 的强行物理排序意味着其生成的 Record 流毫无“内在的键值排序特征”。假如上层还有一个 `ORDER BY age`，那它将无法复用索引原生的有序性，导致上一层强行追加显式 `Sort` 节点。

---

## 4. 路径组合的魔法：参数化路径 (Parameterized Paths)

普通扫描只需要评估自己的 `baserestrictinfo`。然而 PostgreSQL 在规划层面十分早熟地考虑到了 `JOIN` 的情境。特别是大名鼎鼎的**嵌套循环连接（Nestloop Join）**。

试想这样一个场景：`A JOIN B ON A.val = B.id`。
在这里，假如 B 是内表（被反复驱动扫描的那张表），如果只给 B 提前留一条无参扫描路径，那么无论 A 给什么，B 都得全扫，代价巨大！但如果能把 `A.val` 作为一个**传递在途的变量（Parameter）**硬塞给 B，并在 B 的底层进行索引前置判断呢？

*   在 PG 中，这被称为**参数化路径 (Parameterized Path)**。
*   `PG` 优化器引入了专门的机制：`ParamPathInfo`（参数化路径信息）。在 `create_index_paths` 时，优化器不仅仅只生成针对 B 自己过滤的一套独立路径；它还会“未雨绸缪”地去扫一眼连接缓存区，看 B 跟别人连接时能不能借助从别人那传过来的等值条件利用 B 自己的索引！
*   如果有，PG 会生成附带 `lateral_relids`（依赖哪些外部基表）标记的特殊索引扫描，它具有更微弱甚至为 0 的单独执行可行性，但这将会是外层 NestLoop 构建极低代价算法的绝杀神兵。LATERAL 语法的基石也是借由这一机制发扬光大的。

---

## 5. 路径的残忍角斗场：Pruning 与 `add_path`

经过上面一番狂轰滥炸式的推演探索，无论是谁，都会为哪怕一个简单的表构建出几十种奇形怪状的 Path（SeqScan, IndexScan使用索引 X, 使用索引 Y, Bitmap组合，并行...）。

为了防止随后引发指数级别的内存溢出和连接组合爆炸（Path Explosion），优化器必须随时修建多余的枝叶——**路径剪枝（Path Pruning）**。

所有的扫描路径都不会直接插入 `RelOptInfo->pathlist` 的尾巴，而是必须经过唯一的“角斗场保安”门卫函数： **`add_path(RelOptInfo *parent_rel, Path *new_path)`**。

`add_path` 内蕴含了一套非常精彩的淘汰机制，遵循类似于帕累托最优（Pareto Optimality）的法则。
当新生成的 Path 来试图登记入册时，它必须和 `pathlist` 现存的老家伙们逐一比较：
1.  **代数对比（Cost Fudge）**：如果新路径的 Startup 代价（`startup_cost`，流式吐出第一行需要的初始代价，有利于 Limit 处理）和 Total 代价（`total_cost`）都一致地高于某个老路径，那从纯纸面实力来看，新路径就是个废物。（PG 会引入大约 `1e-10` 级别的 `fuzz` 模糊乘数来应对浮点数舍入不确定性误差）。
2.  **排序特征免死金牌 (Pathkeys) **：这是 PG Planner 非常精妙的一击。如果新老两套路老代价小，新代价巨高，但老的是杂乱无章出来的结果，而新路径（比如恰好使用了按键值 B-Tree 索引）附带上了与上一层节点所需的 `ORDER BY` 契合的输出天然排序特征（`pathkeys`），那么新路径就**不能被单纯按价抛弃**！因为它由于免去了上层做外置大数据量排序（明文的显式 Sort 节点）而变相立功，此时它会与老路径被**全数并存保留**入名录。

最终，淘汰了那些各个维度（代价无优势、排列无收益）都一无是处的方案后，留存在 `rel->pathlist` 上的就是浓缩了最优解潜力、千锤百炼下来的基础扫描规划路径了。它们摩拳擦掌，即将在接下来的 Join 阶段互相重叠连结。
