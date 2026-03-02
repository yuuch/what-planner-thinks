# 查询树的预处理 (Query Tree Preprocessing)

当 PostgreSQL 优化器接收到解析和重写完毕的查询树（Query Tree）后，首先会通过 `subquery_planner()` 函数进行预处理。
预处理的主要目的是简化查询树，消除冗余结构并尽可能展平嵌套。系统会将复杂的表达式简化并规范化，从而为后续负责处理连接逻辑的 `grouping_planner()` 与 `query_planner()` 提供更清晰、扁平的基础结构。

以下内容结合 PostgreSQL 18.1 源码（核心逻辑位于 `src/backend/optimizer/plan/planner.c` 中的 `subquery_planner`），介绍查询树预处理的几个核心步骤。

---

## 1. CTE 与子查询的处理

在处理具体表达式之前，优化器需要先将查询块和子句展开。

* **处理 WITH 语句 (CTE)**: 系统首先调用 `SS_process_ctes`。如果 CTE 满足内联（Inline）条件，优化器会将其展开并转化为普通的 `RTE_SUBQUERY`；如果不满足内联条件，则会为其单独生成 InitPlan 级别的 SubPlan，确保它在主查询执行前计算完毕。
* **补全 FROM 子句**: 通过 `replace_empty_jointree` 函数处理没有 `FROM` 子句的查询（例如 `SELECT 1`）。优化器会为其挂载一个基础的 `RTE_RESULT` 节点，以便后续统一使用标准的连表 (JoinTree) 算法。
* **SubLink (子引用) 提升 (ANY/EXISTS)**: 调用 `pull_up_sublinks`，将位于 `WHERE` 或 `JOIN/ON` 条件中的 `ANY` / `EXISTS` 等子查询提取出来，并尝试转化为半连接（Semi-Join）或反连接（Anti-Join）。
* **子查询扁平化 (Subquery Pull-up)**: 调用 `pull_up_subqueries`，尝试将 `FROM` 子句中基础的子查询与当前查询层级融合，减少查询嵌套层级。
* **UNION ALL 展开**: 遇到简单的 `UNION ALL` 时，优化器调用 `flatten_simple_union_all` 将其解构为平铺的追加关系（`appendrel`），从而避免复杂的集合操作开销。

---

## 2. 关系表范围 (RTE) 处理

RangeTable 记录了查询涉及的所有表、视图及函数。

* **函数 RTE 内联与常量化**: `preprocess_function_rtes` 负责处理 `FROM` 子句中的函数。如果函数不包含变量且为常量计算，系统会直接计算其结果；如果符合内联展开条件，则将其函数体展开并并入主查询。
* **虚拟生成列的展开**: `expand_virtual_generated_columns` 会对表进行排查。当发现查询中引用了生成列（Generated Columns 的 `Var` 引用）时，优化器会用其背后的计算表达式进行替换，以方便后续的表达式化简。

---

## 3. 表达式预处理 (preprocess_expression)

预处理阶段的核心步骤之一，是对 `TargetList`、`WHERE`、`HAVING`、`LIMIT` 和 `ON CONFLICT` 等各个部分的表达式调用 `preprocess_expression()` 进行规范化处理。其核心操作包括：

### 3.1 连接别名还原 (Join Alias Variables Flattening)
通过 `flatten_join_alias_vars`，优化器会将因为使用了 `JOIN ... USING` 或自然连接所产生的连接别名变量，替换回它们在底层基础表中的原始列名引用（底层的 `Var` 引用）。这保证了后续的优化逻辑只需处理物理表字段。

### 3.2 常量折叠 (Constant Folding)
调用核心化简函数 `eval_const_expressions`。
该函数除了计算简单的常量算术外，还会对内置稳定函数（Immutable Function）的常量输入进行提早计算。此外，它能将深层嵌套的条件进行扁平化处理，如将 `AND (A, AND(B, C))` 展平为 `AND(A, B, C)`。

### 3.3 条件范式化 (Canonicalize Qual)
经由 `canonicalize_qual` 函数，系统会对布尔条件（例如 `WHERE` 限定项）进行逻辑层面的范式转换，展平表达式并排查其中可能存在的互斥或冗余条件。

### 3.4 杂项 SubLink 转换与变量绑定
* 针对之前 `pull_up_sublinks` 阶段未能处理的子查询（例如出现在 `SELECT` 目标列集合中的标量子查询），会在此处转换为 `SubPlan` 操作符。
* `SS_replace_correlation_vars` 会查找子查询中对上层父查询变量的引用，并使用 `Param`（参数）将其绑定，建立父子查询之间的数据依赖关系。
* 最终，对于 `WHERE` 和 `HAVING` 顶层的条件判断，系统会调用 `make_ands_implicit` 剥除显式的 `BoolExpr` 包装，将其转换为隐式的 `Implicit-AND` 链表结构。

---

## 4. 后期合并与结构修正

在表达式处理完毕后，`subquery_planner` 还会进行一些清理操作，然后再将查询树交给 `grouping_planner`。

* **HAVING 转化为 WHERE**: 系统会检查 `HAVING` 子句中的条件。如果发现其中不包含聚合函数，也不含易失函数（Volatile Function），系统会将其移动或复制到 `WHERE` 子句中去（例如单纯的 `HAVING col = 1`）。这样可以在进入代价较高的 `Aggregate` 分组算子之前提早过滤数据。
* **外连接消除 (Outer Join Elimination)**: 通过调用 `reduce_outer_joins`。由于之前的步骤化简了 `WHERE` 表达式，系统会检查是否有 `WHERE` 条件要求由于 LEFT/RIGHT JOIN 产生的 NULL 值为非空（例如 `LEFT JOIN B ON ... WHERE B.id > 10`）。如果存在这样的严格限制条件，这个外连接就可以被安全降级为执行效率更高的 `INNER JOIN`。
* **移除无用 Result RTE**: 最后调用 `remove_useless_result_rtes`，清理前面步骤中产生的，但目前已经失去作用的冗余 `RTE_RESULT` 节点。

经过预处理阶段的优化后，查询树被转换为更扁平、逻辑结构更清晰的形式，随后传递给 `grouping_planner` 继续后续优化。

---

## 源码参考

*   **核心逻辑位置**：`src/backend/optimizer/plan/planner.c` 中的 `standard_planner` -> `subquery_planner`。
