# standard_planner 函数入口

`standard_planner` 是 PostgreSQL 默认优化器的主要入口函数。

**位置**: `src/backend/optimizer/plan/planner.c`

## 整体流程

`standard_planner` 的执行流程大致如下：

```c
PlannedStmt *
standard_planner(Query *parse, const char *query_string, int cursorOptions,
                 ParamListInfo boundParams)
{
    /* 1. 初始化 */
    root = makeNode(PlannerInfo);

    /* 2. 预处理 (Subquery Pull-up, Constant Folding 等) */
    /* 实际上这部分主要在 subquery_planner 中完成 */
    root->parse = parse;

    /* 3. 调用 subquery_planner */
    /* 这是递归优化的核心，处理子查询和当前层级的优化 */
    plan = subquery_planner(root->glob, parse, NULL, false, tuple_fraction);

    /* 4. 最终收尾 */
    /* 将 Plan 树转换为 PlannedStmt，准备交给 Executor */
    result = makeNode(PlannedStmt);
    result->planTree = plan;
    
    return result;
}
```

## subquery_planner

`subquery_planner` 负责处理特定查询层级的逻辑。

1.  **pull_up_subqueries**: 提升子查询。
2.  **preprocess_expression**: 规范化表达式。
3.  **reduce_outer_joins**: 外连接消除。
4.  **grouping_planner**:
    *   这是真正进行路径搜索的地方。
    *   它首先调用 `query_planner` 来处理 Scan/Join 优化。
    *   然后自己处理 Group By, Agg, Sort, Limit 等非 Join 操作。

## query_planner

`query_planner` 是核心中的核心 (The Core)：

1.  **setup_simple_rel_arrays**: 初始化 RelOptInfo 数组。
2.  **add_base_rels_to_query**: 识别所有基表。
3.  **make_one_rel**:
    *   `set_base_rel_pathlists`: 生成单表访问路径。
    *   `make_rel_from_join_list`: 生成多表连接路径 (DP / GEQO)。
    *   返回一个代表最终结果关系的 `RelOptInfo`。

## 总结

调用栈：
`standard_planner` -> `subquery_planner` -> `grouping_planner` -> `query_planner` -> `make_one_rel`
