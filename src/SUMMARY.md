
# Summary

[简介](README.md)
[优化器概览](overview.md)

# 第一阶段：逻辑重写与扁平化
- [查询树的预处理](phase1/preprocess.md)
- [子查询提升 (Pull-up)](phase1/subquery_pullup.md)
- [外连接消除](phase1/outer_join_elimination.md)

# 第二阶段：基础设施 (The Core)
- [等价类 (Equivalence Classes)](core/equivalence_classes.md)
- [路径键 (PathKeys)](core/pathkeys.md)
- [选择率与代价模型](core/selectivity_cost.md)

# 第三阶段：路径搜索 (Scan & Join)
- [单表扫描路径生成](phase2/scan_paths.md)
- [多表连接顺序 (Dynamic Programming)](phase2/join_order.md)
- [遗传算法 (GEQO)](phase2/geqo.md)

# 第四阶段：后置处理
- [聚合与排序优化](phase3/post_optimization.md)
- [并行执行计划生成](phase3/parallel_planner.md)

# 源码深度拆解
- [standard_planner 函数入口](source/standard_planner.md)
- [核心数据结构 RelOptInfo](source/reloptinfo.md)
