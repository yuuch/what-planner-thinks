# 单表扫描路径生成 (Access Paths)

在这一步，优化器为查询中的每一个基表（Base Relation）生成可能的访问路径（Access Paths）。

## 常见的扫描方式

### 1. 顺序扫描 (Sequential Scan)

*   **机制**: 从头到尾读取表的所有页面。
*   **适用场景**: 表很小，或者需要读取大部分行（Selectivity 高）。
*   **代价**: 主要取决于表的大小（Pages）。

### 2. 索引扫描 (Index Scan)

*   **机制**: 遍历 B-Tree（或其他索引），找到匹配的 Tuple ID (TID)，然后回表（Heap Fetch）读取数据。
*   **适用场景**: 选择率低（High Selectivity），只需要极少数行。
*   **Index Only Scan**: 如果查询的所有列都在索引中（Covering Index），且可见性映射（VM）显示页面全可见，则无需回表，性能极佳。

### 3. 位图扫描 (Bitmap Heap Scan / Bitmap Index Scan)

*   **机制**:
    1.  **Bitmap Index Scan**: 扫描索引，将满足条件的 TID 记录到位图（Bitmap）中。如果使用了多个索引，可以对位图进行 AND/OR 操作。
    2.  **Bitmap Heap Scan**: 根据位图中的 TID，按物理顺序（Physical Order）读取堆表页面。
*   **优势**: 解决了 Index Scan 的随机 I/O 问题，将其转化为更顺序的 I/O。
*   **劣势**: 无法利用索引的排序特性（输出是无序的）。

### 4. TID Scan

*   **机制**: 直接根据用户提供的 `ctid` 访问行。
*   **场景**: `WHERE ctid = '...'`。

## 源码流程

入口：`set_rel_pathlist` -> `set_plain_rel_pathlist`

对于每个表，优化器会：
1.  创建一个 `SeqScan` 路径（作为保底）。
2.  查找所有可用的索引。
3.  对于每个索引，评估是否可以使用（检查 WHERE 条件是否匹配索引列）。
4.  如果可用，创建 `IndexScan` 或 `BitmapScan` 路径。
5.  将所有生成的路径添加到 `RelOptInfo->pathlist` 中。

## Pruning (剪枝)

在生成路径的过程中，优化器会利用 `add_path` 函数。该函数会比较新路径与现有路径。如果新路径“明显更差”（代价更高且没有更好的排序特性），则会被直接丢弃，不加入路径列表。
