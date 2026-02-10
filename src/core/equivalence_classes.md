# 等价类 (Equivalence Classes)

等价类（Equivalence Class, EC）是 PostgreSQL 优化器中用于推导隐含约束条件的核心数据结构。它主要基于 SQL 中的 `WHERE` 或 `JOIN/ON` 子句中的等值条件（Equality Clauses）。

## 概念

如果查询包含 `A.x = B.y` 和 `B.y = C.z`，根据传递性（Transitivity），我们可以推导出 `A.x = C.z`。优化器通过将 `{A.x, B.y, C.z}` 放入同一个等价类来管理这种关系。

## 作用

1.  **减少冗余 Quals**: 不需要显式地存储所有两两组合的等值条件，只需维护 EC。
2.  **生成连接条件**: 在考虑表 A 和表 C 连接时，即使原始 SQL 没写 `A.x = C.z`，优化器可以从 EC 中提取出该条件，允许生成 A 和 C 的直接连接路径。
3.  **排序顺序推导**: 如果 `ORDER BY A.x`，而 `A.x` 与 `B.y` 等价，那么按 `B.y` 排序的结果也满足 `ORDER BY A.x`。这对利用索引扫描（Index Scan）或 Merge Join 非常重要。

## 数据结构

定义在 `src/include/nodes/pathnodes.h`:

```c
typedef struct EquivalenceClass
{
    NodeTag     type;
    List       *ec_members;     /* list of EquivalenceMembers */
    List       *ec_sources;     /* list of generating restrictinfos */
    List       *ec_derives;     /* list of derived restrictinfos */
    Relids      ec_relids;      /* all relids appearing in ec_members */
    bool        ec_has_const;   /* does EC contain a constant? */
    bool        ec_below_outer_join; /* equivalence applies below OJ? */
    bool        ec_broken;      /* failed to generate canonical pathkeys? */
    Oid         ec_collation;   /* collation, if datatypes are collatable */
    /* ... operations info ... */
} EquivalenceClass;
```

*   `ec_members`: 包含等价的表达式（如 `A.x`, `B.y`, `42`）。
*   `ec_has_const`: 如果 EC 中包含常量（如 `A.x = 42`），则意味着 EC 中所有成员都必须等于该常量。这极大地约束了扫描范围。

## 构建过程

在 `query_planner` -> `make_one_rel` 的早期，通过 `generate_base_implied_equalities` 处理。优化器会扫描所有的 RestrictInfos，寻找 Merge-joinable 的等值子句来构建 EC。
