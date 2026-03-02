# 并行执行计划生成 (Parallel Query Planning)

PostgreSQL 在 9.6 版本引入了并行查询架构，截至 18.1 版本，该架构已深度整合至代价优化器（CBO）中，成为处理分析型查询的重要手段。

并行查询的核心思想是：在处理大规模数据扫描或复杂的聚合操作时，利用多个后台工作进程（Background Workers）分担计算任务，最终由发起查询的主进程（Leader Process）进行数据整合。

本章将探讨规划器中与并行执行相关的设定，涉及部分路径的生成、数据的收集汇总以及并行连接机制。

---

## 1. 并行的基础构件：Partial Path 与 Gather

在优化器中，生成并行计划的基础在于两种特殊的算子抽象：部分路径和收集节点。

### 1.1 部分路径 (Partial Path)
在“单表扫描路径生成”阶段，`set_plain_rel_pathlist` 在生成 `SeqScan` 路径的同时，也会调用 `create_plain_partial_paths` 生成一条 `Partial Path`（部分路径）。

**特点**：
部分路径表示该路径分支如果由某个独立的工作进程执行，它只会输出整体数据的一部分（Slice）。
为了实现互不干扰的切片，底层执行引擎通过动态共享内存（DSM）中的共享计数器（Block Counter）协调扫描进度。由于各个 Worker 竞争性地获取数据块进行处理，导致部分路径输出的数据通常是无序的。
在生成计划阶段，系统设定并行的进程数量，并按比例计算和调整这条路径对应的成本。

### 1.2 并行安全性检查 (Parallel Safe)
在优化器评估某个表关系是否可以采用并行策略（标记 `consider_parallel = true`）前，必须通过 `is_parallel_safe()` 进行严格的约束条件检查。
1. **函数校验**：如果查询中的 `WHERE` 或 `TargetList` 子句引用了被标记为 `PARALLEL UNSAFE` 的用户自定义函数（如修改表状态的函数），那么优化器将禁用并行路径。
2. **读写隔离**：对于包含未提交数据的临时表（Temporary Tables），系统不会生成并行作业，因为后台子进程无法访问 Leader 进程的私有内存变更。

---

## 2. 收集节点：Gather 与 Gather Merge

由多个 Worker 独立运算出的局部数据（Partial Path）需要汇总至主进程（Leader Process），形成统一的数据流。
优化器通过调用 **`generate_useful_gather_paths()`**，在部分路径树的上方添加 `Gather Node`（收集节点）。

添加了 `Gather` 节点后，原本局部的路径变为了完整的扫描路径。根据对数据序列的保留要求，PostgreSQL 提供了两种收集策略：

### 2.1 基础的记录收集 (Gather)
最常见的无序收集算子。
Leader 进程启动多个工作进程，工作进程完成计算后将结果发送至基于共享内存的消息队列。Leader 以轮询方式获取数据。这种方式的开销相对较低，但收集的数据不再保持底层工作进程可能自带的处理顺序。

### 2.2 保持排序的收集 (Gather Merge)
**该策略涉及代价的权衡。**
如果在下方执行的 Worker 使用了如并行 B树索引扫描（单 Worker 读取的局部数据是有序的），Leader 在合并这 N 个结果流时，若是普通的 Gather，总结果集会变为无序。

当外层查询（如 `ORDER BY` 等操作）需要维持数据的有序性时，优化器会考虑使用 `Gather Merge`。
Gather Merge 节点维护了一个堆结构优先队列 (Min-Heap / Priority Queue)。它监视 N 个 Worker 传回的数据流，持续抽取最小值合并，从而在整合多路局域排序数据时，保证了**总体结果集的严格顺序**。

---

## 3. 并行连接 (Parallel Joins) 机制

不仅是全表扫描可以并行，PostgreSQL 针对表连接操作同样设计了并行执行方案。

### 3.1 Parallel Hash Join (并行哈希连接)
如果采用普通的 Hash Join，分配 N 个进程将导致各自在本地构建并维系外表的哈希表，不仅浪费内存，且无法协同。

在 并行执行方案中，系统使用基于 DSM 的 Shared Hash Table（共享哈希表）。
1. **构建阶段 (Build Phase)**: Leader 在 DSM 中分配空间。所有的 Worker 同时扫描内表，并将记录共同填充至同一张共享哈希桶中。
2. **同步屏障 (Synchronization Barrier)**: 系统使用全局屏障信号，确保所有 Worker 等待共享哈希表完全建立完毕。
3. **嗅探阶段 (Probe Phase)**: 屏障解除后，所有 Worker 独立对外表（Outer Table）获取分块（执行部分扫描），并行使用哈希表进行探测与计算。

### 3.2 Parallel Nested Loop (并行嵌套循环)
这种模式结构较为简单，但在配合参数化路径（Parameterized Path）时可能发挥重要作用。
* **外表配置**：外部表 (Outer Table) 必须是一个部分数据流（Partial）状态，即由多个 Worker 共同竞争分担外表的扫描数据。
* **内表配置**：内部表 (Inner Table) 是完整的串行扫描表现。这意味着每一个 Worker 每次处理一行外表数据后，都要去扫描完整的内表（通常是携带参数的 Index Scan 检索）。这与分布式计算结构中的广播关联（Broadcast Join）思想相似。

---

## 4. 并行查询的代价惩罚权重 (Cost Punishments)

尽管理论上并行执行能缩短响应时间，但 PostgreSQL 在判断中会考量并行启动和进程间通信带来的损耗。
在代价评估中，主要有两个参数起到约束和惩罚的作用：**Setup Cost** 与 **Communication Cost**。

1. **`parallel_setup_cost` (启动成本)**
   启动一个工作进程需要申请和分配内存、初始化进程等操作。优化器在此操作上设定了默认的固定开启阈值（默认 1000.0）。这意味着只有预估耗时较长的重型查询，才能够忽略这笔启动成本选择并行路径。
2. **`parallel_tuple_cost` (元组通讯成本)**
   涉及共享内存中队列的进出及主进程的数据提取等。这反映了进程间通信（IPC）引发的开销。若查询返回结果集庞大，这部分损耗将被累加至总体成本，系统借此模型计算选择更划算的执行模式。

在 `grouping_planner` 最终确立执行计划前，只有当携带了 `Gather` 及其所有下属成本计算节点总和（Total Cost）确实显著低于对应的串行执行路线时，系统才会为相应的 SQL 请求批准并行执行模式。
