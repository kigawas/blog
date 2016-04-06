---
title: 有向图强连通分量的算法
excerpt: 有向图强连通分量的算法简介和代码（CPP）
---


## 强连通分量的概念


在有向图G中，如果两个顶点间至少存在一条路径，称两个顶点**强连通(strongly connected)**。如果有向图G的每两个顶点都强连通，称G是一个**强连通图**。非强连通图有向图的极大强连通子图，称为**强连通分量(strongly connected components)**。

例如，下图中，子图{1, 2, 3, 4}为一个强连通分量，因为顶点1, 2, 3, 4两两可达。{5}, {6}也分别是两个强连通分量，因为一个顶点到自身必然可达。

![strong connected](https://www.byvoid.com/upload/wp/2009/04/image1.png)

## Kosaraju算法
Kosaraju算法的步骤：
1. 先找图G的拓扑序列。
2. 在图G的**逆图**上以拓扑序列的顺序，对每一个点运行DFS。添加一个辅助数组id[ ]，每一次的DFS对应同样的id。
3. 有同样id的点即为连通分量。

Kosaraju算法的原理：
拓扑序列实际上是DFS完成访问结点顺序的**逆序**，
例如上图，假设DFS访问的顺序为1->2->4->6->3->5，那么完成节点的顺序则是6，4，2，5，3，1，拓扑序列则是1，3，5，2，4，6。

根据拓扑序列的性质可以知道，图G中拓扑序列在前的点**一定**能访问到拓扑序列在后的点；
而逆图GR中则是拓扑序列在后的点一定能访问到拓扑序列在前的点，若我们以拓扑序列的顺序在逆图上运行DFS，则可以找到**两两可达**的那些点，这些点构成了强连通分量。

代码：

- 拓扑序列:

```cpp
void dfs( DiGraph &G, int v ) {
    marked[v] = true;
    for( vector<int>::iterator w = G.get_adj(v).begin();
                               w != G.get_adj(v).end();
                               ++w ) {
        if( !marked[*w] ) dfs( G, *w );
    }
    toplogical_order.push_back( v );
}
```

- Kosaraju:

```cpp
Kosaraju( DiGraph &G ){
    Digraph &G = G.reverse();
    //注意vector要反过来遍历
    for( vector<int>:reverse_iterator s = toplogical_order.rbegin();
                              s != toplogical_oder.rend();
                              ++s ) {
        if( !marked[*s] ) dfs( G, *s );//每一次运行dfs都需要检测该点是否已经访问过
        ++count;
    }
}
void dfs ( Digraph &G, int v ){
    marked[v] = true;
    id[v] = count;
    for( vector<int>::iterator w = G.get_adj(v).begin();
                               w != G.get_adj(v).end();
                               ++w ) {
        if( !marked[*w] ) dfs( G *w );
    }
}
```
