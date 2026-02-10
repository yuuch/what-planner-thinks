# 路径键 (PathKeys)

PathKeys 是 PostgreSQL 优化器用来表示数据流**排序属性**的数据结构。

## 为什么需要 PathKeys？

在查询执行过程中，许多操作依赖于数据的顺序：
1.  **Merge Join**: 需要输入数据按连接键排序。
2.  **Group By (Sort group)**: 需要按分组键排序。
3.  **Order By**: 最终输出需要满足用户指定的排序。
4.  **Distinct**: 有时通过排序去重。

如果一个访问路径（Path）——比如 B-Tree 索引扫描——产生的输出已经是有序的，优化器就需要用一种方式标记这种“有序性”，以避免后续不必要的 Sort 操作。PathKeys 就是这种标记。

## PathKeys 与等价类

PathKey 不仅仅记录“按某列排序”，而是记录“按某个等价类排序”。

*   **Canonical PathKeys**: PathKey 指向一个 Equivalence Class (EC)。
*   **逻辑**: 如果 `A.x = B.y` (同一个 EC)，那么“按 `A.x` 排序”和“按 `B.y` 排序”在物理顺序上是一致的（假设操作符语义一致）。
*   因此，PathKey 列表实际上是一组 EC 的列表。例如 `ORDER BY A.x, C.z` 对应 `[EC(A.x), EC(C.z)]`。

## 数据结构

```c
typedef struct PathKey
{
    NodeTag     type;
    EquivalenceClass *pk_eclass; /* value to be ordered */
    Oid         pk_opfamily;    /* btree opfamily defining the ordering */
    int         pk_strategy;    /* sort direction (ASC/DESC) */
    bool        pk_nulls_first; /* do NULLs come before normal values? */
} PathKey;
```

*   `pk_eclass`: 关联的等价类。
*   `pk_opfamily`: 排序使用的操作符族（B-tree opfamily）。

## 比较 PathKeys

优化器经常需要比较两条路径的 PathKeys 是否“有用”：
*   `pathkeys_contained_in(keys1, keys2)`: 判断 keys1 是否是 keys2 的子集（前缀）。
*   如果查询要求 `ORDER BY x, y`，而路径提供了 `x, y, z` 的顺序，则是满足需求的。

通过 PathKeys，优化器可以智能地识别出哪些索引扫描或连接结果可以直接满足排序需求，从而省略昂贵的 Sort 节点。
