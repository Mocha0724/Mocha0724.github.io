---
title: 【基础算法】Dijkstra
date: 2025-04-28 21:49:45
tags:
  - 算法笔记
  - 最短路径
  - GIS
mathjax: true
---


> Dijkstra算法的流程
> 以及到A*的优化和正确性问题的数学推导

## 最短路径：从Dijkstra到A-Star

### Dijkstra 算法简介

Dijkstra 算法用于在一个图（Graph）中求“从起点到所有点的最短距离”，前提是边权重为**非负**。

![Dijkstra示意](/images/Dijkstra/广度优先到启发式.png)
<p class="img-caption">从左到右为BFS、Dijkstra、A*，在距离成本均匀分布时Dijkstra和BFS是一致的</p>


核心思想是：每次从当前已知最短距离集合外选出距离起点最近的那个节点，把它的最短距离固定下来，然后更新它的邻接点。

- 维护一个 `dist[u]`：表示从起点到节点 `u` 的当前最短距离
- 维护一个优先队列（min-heap）：按 `dist` 取出当前距离最小的节点
- 更新近邻：对边 `u -> v`，
  - 若 `dist[u] + w(u, v) < dist[v]`，则更新 `dist[v]` 并记录前驱（用于回溯路径）

![Dijkstra均匀](/images/Dijkstra/Dijkstra.gif)
<p class="img-caption">在距离成本均匀分布时Dijkstra</p>

![Dijkstra成本](/images/Dijkstra/Dijkstra_成本.gif)
<p class="img-caption">在距离成本不均时Dijkstra</p>

在栅格/栅格图（例如把栅格像元当作节点）里，通常会把：

- 每个像元当作一个节点
- 相邻像元（4 邻域或 8 邻域）当作图的边
- 边权重由“从当前像元到邻居像元的移动成本”给出

### Dijkstra 伪代码（用于把概念写成流程）

```text
Input: 图 G(V, E), 边权 w(u, v) >= 0, 起点 s, （可选终点 t）
dist[v] = +infinity, prev[v] = None for all v in V
dist[s] = 0
Q = priority_queue()  // 按 dist 最小弹出
Q.push((0, s))

while Q not empty:
  (d, u) = Q.pop_min()
  if u == t: break  // 可选：只求 s->t

  if d > dist[u]: continue  // 跳过过期条目（优先队列常见写法）

  for each neighbor v of u:
    alt = dist[u] + w(u, v)
    if alt < dist[v]:
      dist[v] = alt
      prev[v] = u
      Q.push((dist[v], v))  // 或 decrease-key

Output: dist, prev（prev 可回溯得到路径）
```

### A* 算法简介（在 Dijkstra 上加启发式）

A*（A-star）仍然使用累计代价来保证最优性，但会引入启发式函数 `h(n)` 来“估计从节点 `n` 到终点还需要多少代价”，从而更快地把搜索聚焦到终点附近。

在 A* 中，优先队列一般按下面的评估函数排序：

- `f(n) = g(n) + h(n)`

其中：

- `g(n)`：从起点到节点 `n` 的已累计成本（类似 Dijkstra 的 `dist`）
- `h(n)`：从 `n` 到终点的启发式估计成本

![Astar](/images/Dijkstra/Astar.gif)
<p class="img-caption">Astar算法</p>

在栅格最短路场景中，常见的 `h(n)` 选择包括：
- 基于网格距离的估计（如曼哈顿距离用于 4 邻域，欧氏距离用于 8 邻域）
- 或者乘以一个最低可能移动成本的系数（让启发式更贴合成本栅格）

### 如何保证A*正确性（以及数学推导）
**正确性要求：`h(n)` 不要高估真实最短剩余成本**

设终点为 `t`，从节点 `n` 到终点的真实最短剩余代价为：

$$
h^{\ast}(n) = C^{\ast}(n \rightarrow t)
$$

要求启发式满足（对所有节点 `n`）：

$$
0 \le h(n) \le h^*(n)
$$

接着给出常见的最优性推导思路（说明当启发式可采纳时，A* 弹出 `t` 时得到最短路径）：

1) 设存在一条最优路径 \(P^{\ast}\)：

$$
P^{\ast}=(n_{0}=s, n_{1}, \dots, n_{k}=t), \quad C^{\ast} = \sum_{i=0}^{k-1} w(n_{i},n_{i+1})
$$

2) 对路径上任意节点 \(n_i\)，记从起点到 \(n_i\) 沿 \(P^{\ast}\) 的代价为 \(g_i\)，从 \(n_i\) 到终点沿 \(P^{\ast}\) 的代价为 \(r_i\)，显然：

$$
g_i + r_i = C^*
$$

3) 由于 \(P^{\ast}\) 从 \(n_i\) 到 \(t\) 的这段路径只是一种“候选”，真实最短剩余代价满足：

$$
h^*(n_i) \le r_i
$$

因此可采纳启发式给出：

$$
h(n_i) \le h^*(n_i) \le r_i
$$

4) A* 在搜索过程中维护的累计代价 \(g(n_i)\) 是“当前已知的起点到 \(n_i\) 最优近似”，一定满足：

$$
g(n_i) \le g_i
$$

于是可得该路径节点在 A* 的评估值：

$$
f(n_i)=g(n_i)+h(n_i) \le g_i + r_i = C^*
$$

也就是说：**最优路径上的节点在 OPEN/候选集中始终存在某些 `f <= C*` 的节点**，因此 A* 不可能“过早”弹出一个总代价大于 $C^*$ 的解节点。

5) 当终点 `t` 被弹出时（通常满足 $h(t)=0$），有：

$$
f(t)=g(t)+h(t)=g(t)
$$

由于上面的推理保证 `f(t) <= C*`，并且 $C^*$ 本身是全局最短代价，故：

$$
g(t) \ge C^* \quad 且 \quad g(t) \le C^* \Rightarrow g(t)=C^*
$$

因此 A* 返回的路径代价是最优的。

