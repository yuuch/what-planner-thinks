# 并行执行计划生成 (Parallel Query Planning)

自从 PostgreSQL 9.6 划时代地引入并行查询架构以来，如今在 PostgreSQL 18.1 中已经发展成了一个极为庞大、健壮且无缝融入代价优化器（CBO）的深水区模块。

并行查询的魔力在于：当面对亿级别的海量数据集扫表或重型聚合时，规划器可以果断打破此前单进程单核运算的桎梏，派遣多个后台工作进程（Background Workers）分进合击，最后再由发起查询的主进程（Leader Process）进行数据汇总。

在这个章节中，我们将深深扎入源码底层的并行网格，搞清楚“谁在下派任务”、“怎么汇总结果”以及“惊艳的并行 Join 是怎么实现的”。

---

## 1. 并行的基石抽象：Partial Path 与 Gather

所有的并行计划，在优化器里的表现形式无非是两个全新出现的构件。

### 1.1 部分路径 (Partial Path)
如果你之前读过“单表扫描”章节必然记得：在 `set_plain_rel_pathlist` 中，系统不但会添加一条 `SeqScan` 路径，还会调用 `create_plain_partial_paths` 添加一条被称作 `Partial Path` 的分身。

**什么是 Partial？**
Partial 的物理意义是：**“这条路只要扔给一个独立 Worker 去跑，它只会吐出现有总数据量的一个局部切片（Slice）”**。
为了实现互不干涉的切片，底层执行器在扫表时会依赖动态共享内存（DSM）里维护的原子游标（Block Counter）。多个 Worker 拿原子操作去抢页，谁抢到了谁读，导致输出的数据是无序且切分的。
在规划阶段（Planner），它通过设置并行进程数大于零，并且将此路的预期代价值按比例折算入库。

### 1.2 并行安全 (Parallel Safe) 的铁律
不是所有的一切都可以拉起并行！优化器在任何一个关系被标记为 `consider_parallel = true` 前，都会启动一场极度严苛的 `is_parallel_safe()` 地毯式搜查。
1. **函数检验**：如果在 `WHERE` 或者 `TargetList` 里面引用了一个被标记为 `PARALLEL UNSAFE` 的用户自定义函数（比如直接写表的函数），那么整个脉络将永久拉黑并行。
2. **写事务隔离**：绝不可以在含有未提交缓冲数据的临时表（Temporary Tables）上生成并行作业，因为 Worker 进程不允许看到 Leader 的私有本地内存更改。

---

## 2. 收集节点：Gather 与 Gather Merge

光有各个 Worker 在底下苦哈哈地算出局部结果（Partial Path）是不够的。这些支流必须要被收束回主进程（Leader Process）形成唯一的流。
优化器会通过不断往上推进调用 **`generate_useful_gather_paths()`** 强行为底部的 Partial 树冠盖上一个叫 `Gather Node` 的盖子。

有了 Gathering，一条残缺的 `Partial Path` 在顶层看来就重新变成条完整的正常路参与总角逐。按照是否保序，PG 提供了两大收束神器：

### 2.1 粗犷提速的无序统合 (Gather)
最经典的 Gather 算子。
Leader 创建多个队列池并启动协程。各个 Worker 算完数据就往共享内存的消息队列里无脑塞。Leader 轮询拿到什么数据就立刻吐给客户端。这是对性能损耗最低的手段，但它把所有的底层排列完全打乱了。

### 2.2 优雅的保序收集 (Gather Merge)
**这是一个极为考究代价取舍的东西。**
假设你底下一个 Worker 在走并行 B-Tree 索引扫描（保证了单 Worker 自己读到的局部区块是有序的）。Leader 在收到这 N 个流时，如果不加思索地套用 Gather，出来的总集就是一堆交错乱序的数据。

但在某些场景下，上层非常需要这个 `ORDER BY` 排序。
此时 PG 会盖上一个 `Gather Merge`！
Gather Merge 节点自带一个 **堆排优先队列 (Min-Heap / Priority Queue)**。它死死盯住 N 个 Worker 发来的管线头，不断抓取极小值向上抛出，从而完美确保了 N 个已排序局部流在合成时，**丝毫不破坏总体的有序性！**

---

## 3. 并行连接 (Parallel Joins) 的底层黑科技

如果并行只能做全表扫，遇到 Join 依旧回退到串行那就如鸡肋。PostgreSQL 的王炸在于针对三种不同的物理 Join 设计了出神入化的并行执行版本。

### 3.1 Parallel Hash Join (并行哈希连接)
普通的 Hash Join，如果让 N 个进程并行，很可能会有 10 个进程在内存里各自建了一个一样的外表 Hash Table，完全浪费内存，甚至 OOM。

在 PG 中，通过了史诗级的同步：**基于 DSM 的 Shared Hash Table（共享哈希表）**。
1. **建表 (Build Phase)**: Leader 在 DSM 中开辟好小表的坑位。所有的 Worker 一拥而上读取内表，然后共同把 Tuple 塞进这唯一的大 Hash 桶里！
2. **锁步栅栏 (Synchronization Barrier)**: 用一个全局信号，强制等所有 Worker 把建表拼完。
3. **嗅探 (Probe Phase)**: 放开栅栏！所有 Worker 扭头去对外表（Outer Table）开启无序分块扫。扫出一条就跑去大表碰哈希。彻底把 Join 计算跑满多核 CPU。

### 3.2 Parallel Nested Loop (并行嵌套循环)
这个看起来非常粗暴但往往有奇效（尤其当使用**参数化路径 Parameterized Path**）时。
* **外表切片**：Outer Table 必须是一个 Partial 数据流状态（多个 Worker 在抢外表数据）。
* **内表广播**：Inner Table 是串行的 Full 扫描。这意味着每一个 Worker 每次抢到一行外表数据后，都要拿着这行数据去对完整的内表进行一次（通常是 Index Scan 的）扫描。这类似于大数据的 Broadcast Join 思想。

---

## 4. 并行查询的代价模型打压 (Cost Punishments)

你或许会问：既然并行这么天下无敌，为什么 PG 大多数查询还是单机的？
优化器里存在两个著名的估算打压：**Setup Cost** 与 **Communication Cost**。

1. **`parallel_setup_cost` (启动成本高昂)**
   拉起后台进程是一个结结实实的操作系统 `fork()`。优化器会在计划里硬砸下默认高达 `1000.0` 代价的初始启动惩罚，这意味着只有执行时间可能需要大几秒的沉重查询，才“配得上”并发。
2. **`parallel_tuple_cost` (通讯细微阻塞)**
   将数据从 Worker 写入环形缓冲，并在主进程抽出来。这是一笔不可忽略的 IPC 收发代价。若查询结果有千万级交由 Leader 吐出，这笔惩罚加起来极为震撼。系统可能会因此妥协：“还是干脆单核扫完，免得在内存里互相等锁阻塞”。

通过对上述所有精妙代数模型估算并在最后层交火比拼。在 `grouping_planner` 收尾时刻，如果是自带 Gather 的这棵庞然大物路径总体成本比串行确实大幅胜出，那么恭喜，这张 SQL 稿件，即将驱动整个服务器的 CPU 集群矩阵疯狂咆哮了。
