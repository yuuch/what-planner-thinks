# 核心数据结构 RelOptInfo

在 PostgreSQL 优化器中，一切皆 `RelOptInfo`。

**定义**: `src/include/nodes/pathnodes.h`

## 概念

`RelOptInfo` (Relation Optimizer Info) 用来表示一个“关系”。这个关系可以是：
1.  **Base Relation**: 一个真实的表（或子查询结果）。
2.  **Join Relation**: 两个或多个表连接后的结果。
3.  **Upper Relation**: 进行了聚合、排序后的结果。

在优化过程中，优化器会为每个参与的表创建一个 RelOptInfo，然后通过两两组合创建新的 Join RelOptInfo。

## 关键字段

```c
typedef struct RelOptInfo
{
    NodeTag     type;
    RelOptKind  reloptkind;     /* RELOPT_BASEREL, RELOPT_JOINREL, etc. */
    Relids      relids;         /* 包含的基表 ID 集合 (Bitmap) */
    
    Rows        rows;           /* 估算的行数 (Cardinality) */
    int         width;          /* 估算的平均行宽 */
    
    List       *reltarget;      /* 关系的输出列 (TargetList) */
    List       *pathlist;       /* 这是一个 Path 列表！存放所有可能的访问路径 */
    List       *ppilist;        /* ParamPathInfo list */
    Path       *cheapest_startup_path; /* 启动代价最低的路径 */
    Path       *cheapest_total_path;   /* 总代价最低的路径 */
    
    /* ... 统计信息, 索引信息, 等价类信息 ... */
} RelOptInfo;
```

## Path 与 RelOptInfo 的关系

*   一个 `RelOptInfo` 代表逻辑上的一个关系（比如 `A Join B`）。
*   `pathlist` 存储了实现这个关系的多种物理方式（比如 `NestLoop(A, B)`, `HashJoin(A, B)`）。
*   优化器的目标就是为最终的 `RelOptInfo` 找到 `cheapest_total_path`。

## Path 数据结构

```c
typedef struct Path
{
    NodeTag     type;
    NodeTag     pathtype;       /* T_SeqScan, T_IndexScan, T_NestLoop ... */
    RelOptInfo *parent;         /* 所属的 RelOptInfo */
    
    Cost        startup_cost;   /* 启动代价 (返回第一行前的代价) */
    Cost        total_cost;     /* 总代价 (返回所有行) */
    
    List       *pathkeys;       /* 结果的排序属性 */
} Path;
```

每种具体的路径（如 `IndexPath`, `NestPath`）都继承自 `Path` 结构体，并增加特定的字段。
