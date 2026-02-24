# 多表连接顺序算法与等价类推导 (Join Ordering & Equivalence Classes)

当查询的复杂度从单表查询跃升为多表连接（Multiple Joins）时，优化器的最核心也是最耗时的终极任务就浮出了水面：**决定表的连接顺序（Join Order）以及连接方式（Join Methods）**。

在关系型数据库理论中，N 个表的连接方式理论上有 $N!$ （阶乘）种之多，甚至当考虑所有树结构（包括 Bushy Tree 灌木丛树组合）时，组合数会呈卡特兰数级别的恐怖爆炸。对于动辄数表、十数表的大型分析或者复杂业务查询，如果优化器（Planner）选错了连接顺序，由于连接数量膨胀导致的数据量级乘方，执行代价会以几何级数飙增，瞬间压垮系统内存或跑满 CPU。

在 PostgreSQL 中，只要参与连接的基表数量低于预设的 `geqo_threshold` 阈值（默认 12），**规划器就会坚定地走入最硬核、最精准的自底向上动态规划（Bottom-up Dynamic Programming）深水区**。而只有超过阈值时，才会回退到启发式的遗传算法（GEQO，后文详述）。

本章节将结合 PG 18.1 在 `src/backend/optimizer/path/joinpath.c`、`joinrels.c` 与 `equivclass.c` 等模块的源码，深度剥析动态规划建树过程。

---

## 1. 动态规划的基石：System R 风格的自底向上

PostgreSQL 骨子里继承了早年 IBM System R 数据库奠定的经典动态规划（DP）搜索骨架。

### 1.1 从单表到万物互联
动态规划的核心思想是利用最优子结构：为了求 N 个表的最优连接，我们先求 2 个表的、再求 3 个表的……直至求得全貌。在规划器中，负责推动这股齿轮的司令官是 **`make_rel_from_joinlist`** 函数（由 `make_one_rel` 呼叫入场）。

如果没有使用 GEQO，它将调用 **`standard_join_search`** 进入标准的 DP 检索流程：

*   **初始状态 (Level 1)**：
    其实就是前面 `set_base_rel_pathlists` 章节干出来的事。每一个单表（Base Relation）的最优访问路径早已躺在 `RelOptInfo->pathlist` 中整装待发。系统把所有基表放入一个层级数组 `join_rel_level[1]` 中。
*   **状态转移循环 (Level 2 到 Level N)**：
    外层一个巨大的循环从 `level = 2` 开始，一直运行到参与连接的总表数。在计算第 `N` 层（即 N 个表的连接组）时，算法会遍历切分方案。
    例如计算 $Level_K$，可以是由 $Level_1$ 加上 $Level_{K-1}$ 组合而成，也可能由 $Level_2$ 加上 $Level_{K-2}$ 组合而成。
    这个核心递推函数被称为 **`join_search_one_level`**。

### 1.2 Left-deep 还是 Bushy Tree？
如果你学过数据库理论一定知道连接树的方向：
*   **左深树 (Left-deep Tree)**：永远是一个 Join 的结果去同一个基表（Base Table）做 Join。（如 `( (A JOIN B) JOIN C ) JOIN D`）。这种树占用的内存上下文最少，特别符合 NestLoop 带 Index Scan 这种流式驱动的行为。
*   **灌木树/粗壮树 (Bushy Tree)**：允许两个都是 Join 的临时结果集互相去 Join。（如 `(A JOIN B) JOIN (C JOIN D)`）。当 C和D 的连接率极速塌缩（比如返回0行或1行）时，Bushy Tree 会表现出惊人的前瞻性优势。

在早期的 System R 中，为了保证时间可控只考虑左深树。但现代 PostgreSQL 的 `join_search_one_level` 会通过配置参数与组合检查，**同时探索左深树和 Bushy 树空间**。它不仅尝试用 $Level_1$（单表）连接 $Level_{k-1}$ 来构造出 $Level_k$（左深树路线），只要启用了灵活连接选项，它还会尝试 $Level_2$ 和 $Level_{K-2}$ 碰撞！

---

## 2. 推理大师的底牌：等价类 (Equivalence Classes, EC)

在进入暴力 DP 枚举之前，必须提到 PG 优化器里的一张隐秘且极其强大的王牌：**`src/backend/optimizer/path/equivclass.c`** 里的等价类推理机制。

### 2.1 隐式约束下推
假设有这样的 SQL：
```sql
SELECT * FROM a JOIN b ON a.id = b.id JOIN c ON b.id = c.id WHERE a.id = 5
```
如果你只是个呆板的机器，你会先拿着 `a.id = 5` 去过滤表 A。结果你在试图单独连接 B 和 C 时，发现不仅 A、B、C 成了交叉联合，B 和 C 还依然带着庞大的数百万行去拼表，最后才被 A 捞走一条孤零零的数据。

**PostgreSQL 是怎么解决这种信息壁垒的呢？靠建立等价类（EC）！**

在预处理时，遇到 `a.id = b.id` 并且 `b.id = c.id`，PG 会拉起一个 `EquivalenceClass`，在这个桶里装入全体“平权成员”：`{ a.id, b.id, c.id }`。
更美妙的是，由于发现了常量约束 `a.id = 5` 也在里面，它会把 `5` 作为 `em_is_const` 的特权会员投入桶内。

**连锁反应 (generate_implied_equalities)**：
此后，在 DP `join_search_one_level` 搜索无论哪层路径、试图扫描哪怕是任意单表 `b` 或者 `c` 之前，优化器会去查它的等价桶，立刻推导出“哦！原来还有一个原 SQL 里没写的隐式条件：`b.id = 5` 以及 `c.id = 5`！”。
这就是 **等价类约束下推**（Implied Equalities）。在进入任何 Join 前，基表自身就已经通过被下推得到的隐式约束利用单表 Index Scan 把所有无关据消灭在了萌芽之中，给接下来的动态规划卸去了数百倍的负担。

---

## 3. 生成特定的连接途径：`populate_joinrel_with_paths`

当 DP 决定要撮合 RelA 和 RelB 组成一个新 Rel (Rel A+B) 层级后，它调用 `make_join_rel`。这里的关键就是 **到底以何种算法、何种物理操作符（Operator）将这两个子结果拼在一起**。

负责实施拼合的是大名鼎鼎的 `populate_joinrel_with_paths()` （位于 `joinpath.c`），它为这个新结合的关系打磨生成这三大门派的物理路径集合，并一股脑丢给 `add_path` 去进行适者生存淘汰：

### 3.1 预处理：无序匹配与巢状循环 (NestLoop / HashJoin)
通过调用 `match_unsorted_outer`，系统开始假定：如果不要求特殊的顺序，就原生态地把外表和内表塞进去。
这时候会衍生出两种主流路径：
*   **NestLoop Join**：朴实的双循环。但 PostgreSQL 在这里非常聪明！它会检查有没有我们上一章讲到的 **参数化路径 (Parameterized Paths)** 可供使用。如果有，它会让驱动表（外表）在流式遍历时，把它的行变量利用 `ParamPathInfo` 直接塞到被驱动表（内表）底层作为 Index Qual 发射！这种做法极大地盘活了 Nestloop。
*   **Hash Join**：如果探测到了连接条件有等号（`hashjoinable = true`），优化器也会塞入 Hash 路径。Hash Join 是无序流对接的最强利器，代价估算中它会考虑内表尺寸去建模内存里修建 Hash Bucket 的代价（`cost_hashjoin`）。

### 3.2 有序匹配：排序归并 (Merge Join)
随后调用 `sort_inner_and_outer`。
**Merge Join 极其依赖于数据输入的预先排好序**。优化器会检查两边输出集合如果自身带着 `pathkeys`（比如从 Index Scan 出来自带 B-Tree 排序，或上一层已经做了显式 Sort），是否正好命合了当前这个 Join Qual （连接条件）的排序诉求！
如果两边天然有序，那构成 Merge Join 的启动成本将暴雪式骤降。如果不具备或者只有单边具备，PG 也会为它强行添加在内存中做 Sort Node 的成本（`cost_sort`），然后构筑起 Merge Join 路径进行公平比稿。

### 3.3 缓存魔法：Materialize 路径辅助
在各种嵌套内部中，如果被频繁重复读取内部关系时（尤其是没有命中参数化索引降维的情形），会调用辅助流程加上隐式缓存节点操作，即 `create_material_path`。这个节点能在内存（或临时文件盘）中将查询出的子查询结果物理钉住，从而挽回大量重算的性能流失。

---

## 4. 防御爆炸：启发式空间约束与剪枝

如果你允许对所有左深树、灌木树，搭配全排列的各种 Nested/Hash/Merge 连接全部敞开探索，DP 即使再优，时间复杂度本身依然是呈极高的多项式的。特别是遇到笛卡尔积（Cross Join，没写 ON 条件盲拼）乱入，空间状态更是翻倍。

**为了保证查询不会为了节约两秒的执行时间，反而在 Planning Server 活活死等算了一分钟计划**，PG 利用参数做出了强烈的启发式空间剪枝：

*   `join_collapse_limit`：强行限制由 `FROM` 罗列起来隐式交叉拼凑而来的列表展平数量。超过限制就不打乱你的拼装顺序。
*   `from_collapse_limit`：针对显式 `JOIN` 写的查询，同理，一旦查询中明明白白写的表堆叠超过这个阈值，就不再允许跨多边界把里面的嵌套展开来大乱炖，直接在原始代码结构小圈子里进行局部 DP。
*   **推迟笛卡尔积（Defer Cross Join）**：在搜索某一 Level 时，任何缺乏 `JoinClause` 限制、必定导致乘方数据膨胀的关系匹配，如果不属于最终必须强制捏合的表尾，优化器都**极度拒绝对其进行早期评估联合**！它宁可在别的地方找有连接条件的表生成 Left-Deep 链，也不会轻易提早把没关联的结构连在一起从而拖死执行栈。

至此，在这个被精心裁剪过搜索树分支的 DP 主框架里，PostgreSQL 像编织藤网一般，最终凝聚出位于最高层级 `join_rel_level[levels_needed]` 里的一份终极 `RelOptInfo`。里面存下了一颗挂满算子、代价和排序策略最优解的物理查询计划树干。
