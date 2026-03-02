# 单表扫描路径生成 (Access Paths)

在 PostgreSQL 的查询优化流程中，当查询树完成了逻辑重写、子查询提升、表达式扁平化及外连接消除等预处理步骤后，规划器（Planner）将进入基于代价的优化阶段（Cost-Based Optimization, CBO）。

代价优化的首要任务，是为查询中涉及的每一个基表（Base Relation）计算可用的**单表访问路径（Single Relation Access Paths）**。在这个环节，无论是顺序扫描、索引扫描还是并行扫描，都会被枚举和评估，为后续的代价比较和表连接（Join）提供基础。

本章结合 PostgreSQL 18.1 源码（主要位于 `src/backend/optimizer/path/allpaths.c` 和 `indxpath.c`），介绍单表扫描路径的生成机制及其数据结构。

---

## 1. 扫描路径生成入口：`make_one_rel`

生成基础路径的工作主要由 `make_one_rel()` 函数（位于 `allpaths.c`）统筹。它的最终目标是确定多表连接的最佳顺序，但在开始表连接之前，它必须先为每个基表生成访问路径。

### 1.1 两阶段生成机制
在 `make_one_rel` 中，单个基表的路径生成分为两个主要阶段：
1. **估算规模 (`set_base_rel_sizes`)**：首先估算该表的输出行数（Rows）和行宽（Width）等统计信息。
2. **生成路径 (`set_base_rel_pathlists`)**：结合上一步估计的规模，生成读取该表的物理路径（Path）。

**分阶段的原因：**
这是因为并行查询和参数化路径的判定非常依赖对表规模的精确估计。例如，如果优化器在估算时发现一个表的数据量非常小，就会将 `consider_parallel` 标记为 false，从而跳过并行扫描的路径生成。分离规模估算和路径生成，可以为全局规划提供更准确的比对基准。

### 1.2 `RelOptInfo`：关系优化信息
在规划阶段，每张基表（包括中间结果和子查询）都会被封装为一个重要的结构体——**`RelOptInfo`**（Relation Optimization Info，定义于 `nodes/pathnodes.h`）。

在路径生成阶段，`RelOptInfo` 包含以下几个核心字段：
*   **`rows` / `reltarget`**: 预估的输出行数和列宽，直接影响后续的扫描代价计算。
*   **`baserestrictinfo`**: 与当前表相关的 `WHERE` 过滤条件，已被解析为 `RestrictInfo` 构成的列表。
*   **`pathlist`**: 一个链表（List），存放由 `set_base_rel_pathlists` 生成的候选扫描路径（Path）。
*   **`cheapest_startup_path` / `cheapest_total_path`**: 记录了执行代价最低的两条评估结果——**启动代价最小的路径**和**总代价最低的路径**，它们会被缓存以便后续快速选用。

---

## 2. 顺序扫描 (Sequential Scan) 与并行扫描

顺序全表扫描（SeqScan）是最基础的访问方式。当缺乏可用索引或索引代价过高时，数据库会通过遍历表的数据块来读取记录。

在 `allpaths.c` 的 `set_plain_rel_pathlist` 函数中：

### 2.1 基础顺序扫描：`create_seqscan_path`
优化器默认会添加一条 `SeqScan` 路径：通过调用 `add_path(rel, create_seqscan_path(root, rel, required_outer, 0))` 实现。
顺序扫描按照堆表（Heap Table）的物理存储顺序逐块读取。
*   **代价组成**：主要由表占用的页数（`rel->pages`）决定连续 I/O 的开销，其次由行数（`rel->rows`）决定 CPU 进行条件过滤的开销。

### 2.2 并行化扫描 (`create_plain_partial_paths`)
在支持并行查询后，系统提供了**部分顺序扫描（Partial Sequential Scan）**。
在添加了基础的顺序扫描之后，优化器会检查是否符合并行条件：
```c
if (rel->consider_parallel && required_outer == NULL)
    create_plain_partial_paths(root, rel);
```
如果条件允许，内部核心逻辑调用 `compute_parallel_worker()` 根据数据规模和相关阈值参数计算出需要启动的**后台工作进程 (Background Workers)** 数量。
当决定并行执行后，系统会通过 `add_partial_path` 添加一条指定了 `parallel_workers` 的分支战线。多个 Worker 共享数据块分配游标，并发读取数据。

---

## 3. 索引扫描 (Index Paths) 机制解析

对于需要高选择性过滤（High Selectivity）的查询，B-Tree 等索引能显著减少数据读取量，从而提升查询效率。
在 `set_plain_rel_pathlist` 之后，系统调用 `indxpath.c` 中的 `create_index_paths()`，生成多种基于索引的候选路径。

### 3.1 `Index Scan`：标准的索引扫描
当查询的 `WHERE` 条件与某个建立的索引 (`IndexOptInfo`) 匹配时，`build_index_paths` 会生成一条 `Index Scan Path`。

**回表 (Heap Fetch)**：
标准的索引扫描在找到符合条件的索引条目和目标 `TID`（Tuple ID）后，仍需要回表读取堆表（Heap）中的实际记录块，以提取 SELECT 所需的其他列数据及进行多版本可见性（MVCC）的判断。

### 3.2 `Index Only Scan`：覆盖索引与可见性映射
如果查询的目标列完全被索引覆盖（如 `SELECT id FROM table WHERE id = 5`），则可能不需要回表读取。

**MVCC 挑战与 VM (Visibility Map) 设计**：
由于 PostgreSQL 在 MVCC 机制下，通过表记录的标头 (`xmin`, `xmax`) 来判断事务可见性，索引条目本身通常不包含可见性信息。
为了减少回表开销，PostgreSQL 使用了 **VM (Visibility Map，可见性映射文件)**。在考虑 `Index Only Scan` 时，只有当 VM 标记对应数据块为 `all-visible` （即该块中的所有数据对所有活跃事务清晰可见）时，查询此时可以直接使用索引数据而跳过回表，进一步降低代价。

### 3.3 `Bitmap Index/Heap Scan` (位图扫描)
当查询包含多个涉及不同索引的条件，或者使用 `OR` 和 `IN` 等查询模式时，独立的 `IndexScan` 在频繁回表时可能带来大量的随机 I/O (Random I/O) 操作。为此，系统采用了**位图扫描（Bitmap Scan）**方案。在 `indxpath.c` 中，主要由 `generate_bitmap_or_paths` 处理。

*   **位图索引扫描 `Bitmap Index Scan`**：各个索引分支独立检索，且不立刻回拉真实数据，而是记录下符合条件的表块位点，在内存中形成一个按位标识（Bit）的图谱。
*   **位图操作 `BitmapAnd` / `BitmapOr`**：通过对不同的索引结果图谱进行按位与、按位或操作，快速合并出最终的目标图谱。
*   **位图堆表扫描 `Bitmap Heap Scan`**：根据合并后的位图结果，严格按照物理数据块地址的前后顺序去批量读取真表。这种方式将大量分散的随机读取转化为**类似顺序读取的连续 I/O 查询**，大幅减少了磁盘寻址时间。

---

## 4. 参数化路径 (Parameterized Paths) 的运用

在评估基表的访问路径时，PostgreSQL 能够前瞻多表连接（主要是嵌套循环连接 NestLoop Join）在实际执行时的需求。

当大表 A 和小表 B 进行连接（如 `A JOIN B ON A.val = B.id`）时，若 B 作为被驱动的内表，可能会根据 A的每一行重复扫描全表。此时如果将关联条件（如 `A.val`）作为一个**在途传输参数（Parameter）**下移到 B 的扫描阶段，B 可以利用此参数在读取前提前应用索引过滤。

*   在 PostgreSQL 术语中，这称为**参数化借力路径 (Parameterized Path)**。
*   系统通过构造 `ParamPathInfo`（参数化信息），并在 `create_index_paths` 时生成不仅依赖自身 `WHERE` 条件、还依赖于外部关系过滤项的索引路径。这种设计虽然自身执行的可行性取决于外部条件，但在被 `NestLoop Join` 采纳后，可以大幅降低连接运算的全局代价。

---

## 5. 路径剪枝 (Path Pruning) 与 `add_path` 函数

由于一条 SQL 可能会为每个基表生成多达几十条不同的扫描路径（如不同索引的单查、组合位图扫、并行等），如果全部保留它们将导致多表连接组合时发生指数级路径爆炸（Path Explosion）。为此，优化器采取了**路径剪枝（Path Pruning）**策略。

所有的候选扫描路径都会通过统一的函数入口进行过滤校验：**`add_path(RelOptInfo *parent_rel, Path *new_path)`**。

`add_path` 会比较新路径和 `pathlist` 中现有的旧路径，评估是否保留新传入的路径：
1.  **按代价修剪（Cost Fudge 裁决）**：该函数会对新路径和已有路径的起步代价（`startup_cost`）与总消耗代价（`total_cost`）进行比较。如果一条新路径在启动代价和总代价上都显著高于现有的某条路径，则该新路径被认为劣势明显，将被直接淘汰。系统在比较时引入了一个极小比例（如 `fuzz` 系数`1e-10`级别）的缓冲乘数来忽略微小的舍入误差。
2.  **受排序属性影响的特例 (Pathkeys 的保护)**：当一条新路径因为执行效率稍慢，原本要被淘汰，但它附带的输出特征恰好满足了全局必要的排序序列（即具备 `pathkeys` 排序特质，如依靠 B-Tree 生成的顺序）时，它有可能不会被直接剔除。预先排好序的路径在后续的 `ORDER BY` 或 `Merge Join` 中可以免除巨大的重新排序开销，因此 `add_path` 将会根据排序价值对其通融和保留。

经过 `add_path` 严格的层层淘汰后，`RelOptInfo->pathlist` 列表中最终只保存了在各个性能维度下最具竞争力的少数几条扫描路径，这大大提升了优化器在后续关联查询排布阶段的处理效率。
