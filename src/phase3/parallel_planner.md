# 并行执行计划生成 (Parallel Query Planning)

自从 PostgreSQL 9.6 划时代地引入并行查询架构以来，这个体系经历了数十个大版本的迭代，如今在 PostgreSQL 18.1 中已经发展成了一个极为庞大、健壮且无缝融入代价优化器（CBO）的深水区模块。

并行查询的魔力在于：当面对亿级别的海量数据集扫表或重型聚合时，规划器可以果断打破此前单进程单核运算的桎梏，派遣多个后台工作进程（Background Workers）分进合击，最后再由发起查询的主进程（Leader Process）进行数据汇总。

在这个章节中，我们将深深扎入源码底层的并行抽象网格，搞清楚“谁在下派任务”、“怎么汇总结果”以及“惊艳的并行 Join 是怎么实现的”。

---

## 1. 并行的基石抽象：Partial Path 与 Gather

所有的并行计划，在优化器里的表现形式无非是两个全新出现的构件。

### 1.1 部分路径 (Partial Path)
如果你之前读过我们的“单表扫描”章节必然记得：在 `set_plain_rel_pathlist` 中，系统不但会添加一条 `SeqScan` 路径，还会调用 `create_plain_partial_paths` 添加一条被称作 `Partial Path` 的分身。

**什么是 Partial？**
Partial 的物理意义是：**“这条路只要你把它扔给一个独立 Worker 去跑，它只会吐出现有总数据量的一个局部切片（Slice）”**。
为了实现这个互不干涉的切片，底层执行器在扫表时会依赖一块动态共享内存（DSM, Dynamic Shared Memory）里维护的原子游标（Block Counter）。多个 Worker 齐头并进地拿原子操作去抢页（Page），谁抢到了谁读，这导致输出的数据是绝对无序且切分的。
在规划阶段（Planner），它通过设置 `path->parallel_workers` 大于零，并且将此路的预期代价值 `/ workers` 比例折算进行入库。

### 1.2 并行安全 (Parallel Safe) 的铁律
不是所有的一切都可以拉起并行！优化器在任何一个关系被标记为 `consider_parallel = true` 前，都会启动一场极度严苛的 `is_parallel_safe()` 地毯式搜查。
1. **函数/操作符检验**：如果你在 `WHERE` 或者 `TargetList` 里面引用了一个被标记为 `PARALLEL UNSAFE` 的用户自定义函数（UDF，比如带外发网络请求、或者直接写表的函数），那么整个这一脉络将永久拉黑并行探索。
2. **写事务隔离**：绝不可以在含有未提交的写缓冲数据的临时表（Temporary Tables）和游标结果集上生成并行作业，因为 Worker 进程是不允许看到 Leader 的私有本地内存更改的。

---

## 2. 收集节点：Gather 与 Gather Merge

光有各个 Worker 在底下苦哈哈地算出局部结果（Partial Path）是不够的。这些零散的支流必须要被收束回主进程（Leader Process）形成唯一的返回流。
优化器会通过不断往上推进调用 **`generate_useful_gather_paths()`** 强行为底部的 Partial 树冠盖上一个叫 `Gather Node` 的盖子。

有了 Gathering，一条原本残缺的 `Partial Path` 在顶层看来就重新变成了一条完整的正常 `Path` 参与总路线的角逐。按照是否保序，PG 提供了两大收束神器：

### 2.1 粗犷提速的无序统合 (Gather)
最经典的普通 Gather 算子。
Leader 创建多个物理队列池并启动协程。各个 Worker 算完数据就往共享内存的消息队列（Tuple Queue）里无脑塞。Leader 轮询拿到什么数据就立刻吐回给客户端。这是对性能损耗最低的收集手段，但它把所有的底层排列完全打乱了。

### 2.2 优雅归并的保序收集 (Gather Merge)
**这是一个极为考究代价取舍的东西。**
假设你底下一个 Worker 在走并行 B-Tree 索引扫描（其实在扫描中依然是按页分配，只是保证了单 Worker 自己读到的局部区块是有序的）。Leader 在收到这 N 个流时，如果不加思索地套用 Gather，出来的总记录集就是一堆交错穿插乱序的数据。

但在某些场景下，上层非常需要这个 `ORDER BY` 排序（比如正好遭遇了 Merge Join 或者外层有极耗时的全局大排序限制）。
此时 PG 会给它盖上一个 `Gather Merge`！
Gather Merge 节点自带一个 **堆排优先队列 (Min-Heap / Priority Queue)**。它同时死死盯住 N 个 Worker 发来的管线头部数据记录，不断抓取最小（排序最优）的一个向上抛出，从而完美确保了 N 个已排序局部流在合成 1 条全球流时，**丝毫不破坏总体的有序性（Pathkeys 完美保存）！**

---

## 3. 并行连接 (Parallel Joins) 的底层黑科技

如果并行只能做全表扫，遇到多表连接依旧回退到串行，那等于鸡肋。PostgreSQL 的王炸在于针对三种不同的物理 Join 设计了出神入化的并行执行版本。

### 3.1 Parallel Hash Join (并行哈希连接)
普通的 Hash Join，如果让 N 个进程并行，很可能发生 10 个进程在内存里各自都建了一个一样的外表 Hash Table，完全浪费内存，还可能爆 OOM。

在 PG 的实现中，通过了史诗级的动态底层同步：**基于 DSM 的 Shared Hash Table（共享哈希表）**。
1. **建表阶段 (Build Phase)**: Leader 在 DSM 中开辟好内存坑位。所有的 Worker 一拥而上读取内部小表（Inner Table），每个人扫多少块算多少数据，然后用原子操作共同把 Tuple 塞进这唯一一个共享巨大的 Hash 桶里！这种利用多核 CPU 加速建立散列表的过程非常恐怖。
2. **锁步栅栏 (Synchronization Barrier)**: 用一个全局信号栅栏，强制等所有 Worker 都把建表这一步拼完了。
3. **嗅探阶段 (Probe Phase)**: 放开栅栏！所有 Worker 扭头去对外表（Outer Table）开启互相切片的无序分块扫。扫出一条就跑去 DSM 的中间大表里碰哈希，碰上就合成输出。彻底把 Join 计算在多核 CPU 跑满了 100% 的并行度。

### 3.2 Parallel Nested Loop (并行嵌套循环)
这个看起来非常粗暴但往往有奇效（尤其当使用前面提到的**参数化路径 Parameterized Path**）时。
*   **外表切片**：Outer Table 必须是一个产生 Partial 数据流的状态（也就是多个 Worker 在各自切块抢这几百万条外表数据）。
*   **内表广播**：Inner Table 是串行的 Full 扫描。这意味着每一个 Worker 每次抢到一行外表数据后，都要拿着这行数据作为上下文，去对完整的内部大表进行一次（通常是 Index Scan 的）完整扫描。各个 Worker 之间互不干扰。这类似于 MPP 大数据引擎里的 Broadcast Join 分发思想。

---

## 4. 并行查询的代价模型与现实打压 (Cost Punishments)

你或许会问：既然并行这么天下无敌，为什么 PG 大多数查询还是单进程执行的？
优化器里存在两个非常著名的“反并行强力惩罚”估算：**Setup Cost** 与 **Communication Cost**。

1. **`parallel_setup_cost` (启动成本高昂)**
   拉起后台进程（Backend Process）并不是通过操作系统轻量级线程。PG 由于历史架构，这是一个结结实实的进程级 `fork()`（并且带有海量的锁库与共享内存地址映射重建）。优化器会在计划里硬生生砸下默认高达 `1000.0` 代价的初始启动惩罚，这意味着只有执行时间可能需要秒级的查询，才“配得上”去等待拉起并行的开销。
2. **`parallel_tuple_cost` (管线通讯极细微的阻塞)**
   将一行计算好的 Tuple 从 Worker 封包写入 DSM 的环形缓冲环，然后在主进程又把它解包抽出来。这是一笔显著不可忽略的 IPC 收发代价。若查询在 Join 给出了千万级海量的最终结果要交由 Leader 吐出，这笔通讯惩罚加起来极为震撼。系统甚至可能会因此倒逼出一个：“我们还是干脆用原版单核自己安安静静扫完，还不用在内存里互相等锁阻塞” 的最优妥协。

通过对上述所有的精妙代数模型估算并在最后层通过 `add_path` 交火比拼。在 `grouping_planner` 收尾时刻，如果是自带 Gather 的这棵庞然大物路径总体成本比串行确实依然大幅胜出，
那么，恭喜，你写下的这张 SQL 稿件，即将驱动整个服务器那密密麻麻的 CPU 集群矩阵疯狂咆哮了。
