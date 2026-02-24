# 查询树的预处理

在 PostgreSQL 优化器（Planner）接收到解析和重写后的查询树（Query Tree）以后，第一步并非直接开始路径搜索，而是通过 `subquery_planner()` 函数进行一系列复杂的预处理。预处理的核心目标是消除查询中的冗余结构、展开嵌套，并对表达式进行规范化和简化，从而为后续的 `grouping_planner()` 和 `query_planner()` 减轻负担，提供尽可能扁平化和易于优化的结构。

以下内容基于 PostgreSQL 18.1 源码（`src/backend/optimizer/plan/planner.c` 中的 `subquery_planner`）详细解析查询树预处理的各个核心阶段。

## 1. CTE 与底层结构展开

在真正进入表达式级别的预处理之前，优化器必须先把从属的查询块或语法结构给“铺平”。

* **处理 WITH 语句 (CTE)**: 调用 `SS_process_ctes`。如果 CTE 可以被内联（Inline），则将其展开为 `RTE_SUBQUERY`；否则构建出 InitPlan 级别的 SubPlan，以便在主查询执行前计算完毕。
* **补全空 FROM 子句**: 通过 `replace_empty_jointree`。如果查询没有 `FROM`（如 `SELECT 1`），优化器会人为挂载一个 Dummy 的 `RTE_RESULT` 节点，统一后续的 JoinTree 处理逻辑。
* **SubLink 的显式提升 (ANY/EXISTS)**: 调用 `pull_up_sublinks`。将 `WHERE` 或者 `JOIN/ON` 里面涉及 `ANY` / `EXISTS` 的相关子查询（SubLinks）尽早转化为半连接（Semi-Join）或反连接（Anti-Join）。
* **子查询扁平化 (Subquery Pull-up)**: 调用 `pull_up_subqueries`。如果 `FROM` 子句中直接写了普通的简单子查询，系统会尝试将其合并到当前查询层级，消除不必要的嵌套。
* **UNION ALL 的扁平化**: 遇到简单的 `UNION ALL`，调用 `flatten_simple_union_all` 将其重写为并列的追加关系 `appendrel`，避免复杂的集合操作开销。

## 2. RTE (RangeTableEntry) 预处理

RangeTable 维护了查询涉及的所有表、视图、函数。

* **函数 RTE 内联与常量化**: `preprocess_function_rtes` 针对 `FROM` 子句中的函数进行计算。如果是简单的常数化函数，直接得出结果；如果有机会，则像展开视图一样将函数展开（Inline）。
* **虚拟生成列展开**: `expand_virtual_generated_columns` 会扫描所有的表，将查询内涉及到 Generated Columns 的 `Var` 引用直接替换为其背后的计算表达式，方便一并在后续步骤被折叠。

## 3. 表达式预处理阶段 (preprocess_expression)

这是预处理代码块中占篇幅最大、也最为细腻的一环。系统不仅要处理 `WHERE` 里的条件，还要将 `TargetList`、`HAVING`、`LIMIT` 及 `ON CONFLICT` 等各处的表达式利用 `preprocess_expression()` 统统洗一遍。核心动作包括：

### 3.1 连接别名展开 (Join Alias Variables Flattening)
通过 `flatten_join_alias_vars`，优化器将由于 `JOIN ... USING` 或自然连接引发的连接别名变量，替换为来自底层基表的原始列（`Var` 引用）。这确保后续的优化只需要面对最底层的真实列名。

### 3.2 常量折叠 (Constant Folding)
调用核心函数 `eval_const_expressions`。
不仅会将 `1 + 1` 化简为 `2`，还能对各种内部函数的常量输入进行预计算（前提是函数为 Immutable）。重要的是，它会压平嵌套的 `AND/OR`（将 `AND (A, AND(B, C))` 变成 `AND(A, B, C)`）以保持逻辑树平坦。

### 3.3 条件标准化 (Canonicalize Qual)
利用 `canonicalize_qual` 函数，如果表达式是一个布尔条件（如 `WHERE` 里的项），它会尝试做一定的范式转化（比如尽力将其拉平或者排查明显的互斥矛盾）。

### 3.4 剩余 SubLink 转换与外层变量绑定
* 经过 `pull_up_sublinks` 还没解决的杂项子链接（如出现在 `SELECT` 目标列里的标量子查询），在此被包装成 `SubPlan`。
* `SS_replace_correlation_vars` 会找出对上层查询列的引用，并使用 `Param`（参数）进行绑定，使得父子查询产生实质的数据关联参数。
* 最后对于 `WHERE` / `HAVING` 的顶层条件，调用 `make_ands_implicit` 退化为 Implicit-AND 的 `List` 结构存放，不再采用显式的 `BoolExpr`。

## 4. 后期合并与结构修正

经过表达式洗刷后，`subquery_planner` 在交付给 `grouping_planner` 之前会做最后的清理。

* **HAVING 下推为 WHERE**: 这是经典的优化手段。系统会扫描 `HAVING` 子句，寻找那些不包含聚合函数、不包含易变函数（Volatile Function）并且不需要等到空分组集产生结果的过滤条件（例如 `HAVING col = 1`）。这些被认为是退化的聚合后过滤，系统会将其克隆或移动到 `WHERE` 列表中，从而在进入 `Aggregate` 算子之前尽早淘汰无关数据。
* **外连接消除 (Outer Join Elimination)**: 调用 `reduce_outer_joins`。因为上一步各种 `WHERE` 表达式已经被处理，这一步系统会检查如果存在 `WHERE` 强约束要求某空侧列非空（比如 `LEFT JOIN B ON ... WHERE B.id > 10`），此时原本的 `LEFT JOIN` 已经失去保留 null 侧的意义，该外连接会被安全地降级成 `INNER JOIN`。
* **剔除无用的 Result RTE**: 调用 `remove_useless_result_rtes`。前面补充的或中间产生的不再起实质作用的 Dummy 关联在此被扫地出门。

至此，一个极其干净、扁平化且逻辑清晰的等价 Query Tree，才会正式交给 `grouping_planner`。

## 源码入口

*   `src/backend/optimizer/plan/planner.c`: `standard_planner` -> `subquery_planner`
*   该逻辑作为第一阶段的总览，其余各子项优化详情，请查看侧边栏对应小节。
