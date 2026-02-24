# 多表连接次序与等价类推导 (Join Ordering & Equivalence Classes)

当查询从单表跃升为多表连接（Multiple Joins）时，优化器迎来了它最耗算力的终极考验：**决定表的连接顺序（Join Order）以及连接方式（Join Methods）**。

在理论中，N个表的联接顺序组合数呈阶乘爆炸。为了寻找最优解，只要参与联接的表数量低于预设的 `geqo_threshold` 阈值（默认 12），**规划器就会坚定地踏入自底向上的动态规划（Bottom-up Dynamic Programming）深水区**。

本章节紧扣 PG 18.1 在 `src/backend/optimizer/path/joinpath.c`、`joinrels.c` 乃至 `equivclass.c` 的源码，深度剖析动态规划建树过程。

---

## 1. 动态规划底盘：致敬 System R 

PostgreSQL 骨子里流淌着 IBM System R 奠定的经典动态规划（DP）搜索思维。

### 1.1 从单打独斗到万物互联
动态规划秉持最优子结构思想：求 N 张表合并的最优解，就从底部的任意 2 个表搭伙算起，再算 3 张的……以此类推。负责发号施令的主函数为 **`make_rel_from_joinlist`**。

算法调用 **`standard_join_search`** 进入标准的 DP 轮回：
* **第一层 (Level 1)**：单表路径（来自 `set_base_rel_pathlists` 的成果）业已就绪，被投入到 `join_rel_level[1]`。
* **状态跃迁 (Level 2~N)**：外围庞大的循环从 `Level = 2` 开始爬坡。每一层的新组合都由之前计算出的最优子集拼合而来，执行核心递归计算枢纽是 **`join_search_one_level`**。

### 1.2 Left-deep 与 Bushy Tree
在连接树的探索过程中，有两种典型的生长结构：
* **左深树 (Left-deep Tree)**：老连接结果集直接和单个新基表去拼凑。这种操作占用的内存上下文极少，非常契合 NestLoop 带 Index Scan 这种流式循环。
* **灌木丛树 (Bushy Tree)**：允许两个中途拼出的临时大合集互相合并。当复杂查询两侧同时具备高过滤性时，它能打出断代级别的性能优势。

相比古典引擎死守左深树，PG 在 `join_search_one_level` 推进下不仅包容左深树路线，更敢于让老手互拼去探索狂放的 Bushy 魔树空间！

---

## 2. 推理大师的逆袭：等价类 (Equivalence Classes, EC)

在放飞动态规划之前，绝对绕不开一张最强底牌：位于 `src/backend/optimizer/path/equivclass.c` 的等价推理机制。

### 2.1 隐式约束的高维度拦截
如果有句 SQL 写着 `WHERE A.id = B.id AND B.id = C.id AND A.id = 5`。传统引擎往往忘了通知还没见面的 B 和 C！
但 PG 凭借等价类结界 `EquivalenceClass` 实体桶，直接将 `{A.id, B.id, C.id, 5}` 当成同一根绳上的蚂蚱死死绑定。

随后祭出 **`generate_implied_equalities`** 神技：不管优化器当下在哪个角落想要扫描底层的 `B` 或者 `C`，它都会下意识查老底。一看 EC 里存在常量 `5`，立马如获至宝地凭空变出 `B.id = 5` 和 `C.id = 5` 的前置隐式拦截指令。
这就赶在多表庞大合并引发雪崩之前，通过直接调用单表的 Index Scan，硬生生把上亿行的烂摊子秒杀在前线阵地上。

---

## 3. 点兵列阵的终极派活：`populate_joinrel_with_paths`

当 DP 决定拿 RelA 和 RelB 拼作一个大联盟（Rel A+B）后，它会将打磨三大物理连接方式的脏活苦活悉数推给 `populate_joinrel_with_paths()` 熔炉函数。

### 3.1 狂野盲连与死磕：NestLoop / HashJoin
该函数一上来大喊 `match_unsorted_outer`，测试原生态无序对抗：
* **NestLoop Join**：如果此时手头带有参数化借力武器（Parameterized Paths），外表便可将循环的每行值作为狙击参数下发给内表，从而使得内表直接唤醒沉睡中的索引发挥神效，绝境逢生翻盘！
* **Hash Join**：要是碰到了等值交换，并且算盘先生评估内表塞进内存哈希桶（Hash Bucket）的开销尚能忍受，这就成了大批流灌数据相互撕咬最高效的无序神兵。

### 3.2 讲究排场的序列：Merge Join
随后的 `sort_inner_and_outer` 则要高雅得多。
Merge Join 极度依赖双方自带的序列。它如饥似渴地扒着双边结果集的 `pathkeys`（输出特征），看看它们原生的排序是否对上眼了联接条件。如果恰好都有命合，这招的发起成本便是一路雪崩跌到谷底；如果不巧没有，哪怕硬着头皮加上高昂内存 `Sort` 花费，PG 也会把它送上决策席比对。

### 3.3 Materialize 的缓存魔法
如果在多层 Nestloop 内部嵌套里发现某个内部小圈子被反复频繁读取而累得喘不上气，系统会自动呼唤 `create_material_path` 在途中加塞一个钉子般的缓存节点，保留救命结果集。

最终，各显神通的所有招式会被集体塞入无情的 `add_path` 刽子手门外。它像极其功利的督工一样排查评估谁跑得快谁带的好处多，把没用的统统扔进深渊，剩下的真金战神统统稳坐 `RelOptInfo->pathlist` 决胜席位。
