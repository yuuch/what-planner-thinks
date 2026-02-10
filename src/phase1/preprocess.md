# 查询树的预处理

在 PostgreSQL 优化器接收到查询树（Query Tree）后，第一步并非直接开始路径搜索，而是进行一系列的预处理。这些预处理步骤旨在简化查询结构，消除不必要的复杂性，并为后续的逻辑优化和物理优化打下基础。

## 主要流程

预处理主要发生在 `subquery_planner` 函数的早期阶段。

1.  **常量折叠 (Constant Folding)**:
    *   计算查询中可以预先确定的表达式。例如 `WHERE x = 1 + 2` 会被重写为 `WHERE x = 3`。
    *   利用 `eval_const_expressions` 函数完成。

2.  **表达式规范化**:
    *   将表达式转换为规范形式（Canonical Form），例如将 `x + 1 = 5` 转换为 `x = 4`（如果支持），或者将布尔表达式进行标准化（CNF/DNF）。
    *   处理 `ANY` / `ALL` 子查询转换等。

3.  **视图展开**:
    *   如果查询中引用了视图，重写器（Rewriter）阶段通常已经处理了规则（Rules），但在优化器阶段，可能通过 `pull_up_sublinks` 等进一步处理。

4.  **Qual 预处理**:
    *   将 `WHERE` 子句和 `JOIN/ON` 子句中的条件转换为隐式的 AND 连接列表（List of Expr）。
    *   处理 `NOT` 表达式下推。

## 源码入口

*   `src/backend/optimizer/plan/planner.c`: `standard_planner` -> `subquery_planner`
*   `src/backend/optimizer/plan/initsplan.c`
