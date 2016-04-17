---
layout: post
title: DFS性能优化--剪枝策略
author: hawker
permalink: /2016/03/DFS-Optimization.html
date: 2016-03-19 23:00:00
category:
    - 算法
tags:
    - DFS
    - 剪枝优化
---

前段时间研究了一下过必经点最短路径问题。本篇文章主要讨论一下DFS算法的性能优化及相关剪枝策略。

### 图的数据结构
为了同时具有邻接矩阵和邻接表的优势，使用下面的双向邻接结构。

{% highlight C++ %}
struct node{
    	short node_l;   //同一行左边的邻居
    	short node_r;   //同一行右边的邻居
    	short node_u;     //同列上一个邻居
    	short node_d;     //同列下一个邻居
    	short weight;     //到该点的边权重
    	short index;      //边的编号
}graph[MAXSIZE][MAXSIZE];
{% endhighlight %}
`node_l`表示与目前顶点同一起点，的上一个终点。
`node_r`表示与目前顶点同一起点，的下一个终点。
`node_u`表示与目前顶点同一终点，的上一个起点。
`node_d`表示与目前顶点同一终点，的下一个起点。
矩阵的第一行表示`[0, i]`：指向以i为终点的第一个起点。
矩阵的第一列表示`[j, 0]`：指向以j为起点的第一个终点。

图的初始化，直接上代码：
	
{% highlight C++ linenos%}
void add_edge_into_matrix(short from, short to, short weight, short index){
    short k = -1;
    graph[from][to].weight = weight;
    graph[from][to].index = index;
    if (to == dest) {
        k = graph[from][0].node_r;
        graph[from][0].node_r = to;
        graph[from][k].node_l = to;
        graph[from][to].node_r = k;
        if(!graph[from][to].node_l)  graph[from][to].node_l = to;
    } else {
        if (graph[from][0].node_r == 0)
            graph[from][0].node_r = to;
        else {
            k = graph[from][0].node_l;
            graph[from][k].node_r = to;
            graph[from][to].node_l = k;
        }
    }
    if (graph[0][to].node_d == 0) {
        graph[0][to].node_d = from;
    } else {
        k = graph[0][to].node_u;
        graph[k][to].node_d = from;
        graph[from][to].node_u = k;
    }
    if (to != dest)
        graph[from][0].node_l = to;
    graph[0][to].node_u = from;
}
{% endhighlight %}

### 高效的Dijkstra实现
因为剪枝策略时，需要求两点之间最短路径的权重。我们需要实现一个基于上面图数据结构的Dijkstra。使用C++中的优先队列，构造堆，优化Dijkstra算法的性能。

{% highlight C++ linenos%}
void Dijkstra(short s, short* weight) {
    for (short i = 0; i < MAXSIZE; i++)
        weight[i] = INF;
    weight[s] = 0;
    using ii = pair<short, short>;
    using vii = vector<ii>;
    priority_queue<ii, vii, greater<ii>> heap;
    heap.push(make_pair(0, s));
    while (!heap.empty()) {
        ii top = heap.top(); heap.pop();
        short v = top.second, d = top.first;
        if (d > weight[v]) continue;
        for (short from = graph[0][v].node_d; from; from = graph[from][v].node_d) {
            int tmp = weight[v] + graph[from][v].weight;
            if (tmp < weight[from]) {
                weight[from] = tmp;
                heap.push(make_pair(tmp, from));
            }
        }
    }
}
{% endhighlight %}

### DFS剪枝策略
下面来看主函数中的图的初始化和相关预处理，DFS函数中将展示如何进行剪枝优化。

{% highlight C++ linenos%}
short origin, dest; //存储起点与终点
short total_weights = INF;  //不连同边的权重
bool cycle_check[MAXSIZE] = {false};  //遍历路径中，已经经过点的集合。
short to_dest_weight[MAXSIZE];        //剪枝预处理，到终点的每个点的最短路径权重
short to_designated_weight[CHECK_SIZE][MAXSIZE]; //到每个必经点，最短路径权重
short designated_point[CHECK_SIZE]; unsigned short designated_len = 0; //必经点集合
bool cutflag = false;  //拓扑稍大时，启用的剪枝标志
using namespace std;
vector<short> route;   //遍历路径集合
vector<short> short_path;//最短路径

void dfs_visit(short node, short total) {
//如果已经到达终点，处理如下
    if(node == dest) {
        //查询路径候选路径是否经过所有必经点
        if (route.size() < designated_len) return;
        for (short i = 0; i < designated_len; i++)
                if (!cycle_check[designated_point[i]]) return;
        //如果该路径的权重小于目前最短路径的权重，该路径成为当前的最短路径
        if(total_weights > total) {
            total_weights = total;
            short_path = route;
        }
        return;
    }
//未达到最终点时的处理：
    for (short from = graph[node][0].node_r; from; from = graph[node][from].node_r)
    {
        //已经经过的点，不处理，避免出现环
        if (cycle_check[from]) continue;
        //剪枝策略1:计算到目前node节点的路径权重＋该点到终点的最短路径权重(w+tmp)，用于上限剪枝
        short w = total + graph[node][from].weight;
        //下一点到终点的权重。
        int tmp = to_dest_weight[from];
        if (cutflag) {
            //剪枝策略2: 如果图的规模较大，计算剩余必经点到终点的权重。
            cycle_check[from] = true;
            for (int i = 0; i < designated_len && tmp <= INF; i++) {
                short v = designated_point[i];
                //选择tmp可能最大值的最大值
                if (!cycle_check[v]) tmp = max(tmp, to_designated_weight[i][from]+to_dest_weight[v]);
            }
            cycle_check[from] = false;
        }
        //上限剪枝策略，预测该条路径到终点的权重最小值，必需小于已有的最短路径的最小值；如大于不向下遍历。
        if (w+tmp < total_weights ){
            cycle_check[from] = true;
            route.push_back(graph[node][from].index);
            dfs_visit(from, w);
            route.pop_back();
            cycle_check[from] = false;
        }
    }
}
	
//1. topo是原始点边数据输入
//2. edge_num边的个数
//3. demand必经点。
void search_route(char *topo[5000], int edge_num, char *demand)
{
    //初始化起点、终点及必经点集合
    stringstream ss;
    string str = ""; char trash;
    ss.str(demand);
    ss >> origin >> trash >> dest >> trash >> str;
    origin++;
    dest++;
    ss.clear(); ss.str(str);
    while (getline(ss, str, '|'))
        designated_point[designated_len++] = stoi(str) + 1;
	//初始化双向邻接图结构
    short from, to, e_index, e_weight;
    for (int i = 0; i < edge_num; i++) {
        ss.clear();
        ss.str(topo[i]);
        ss >> e_index >> trash >> from >> trash >> to >> trash >> e_weight;
        add_edge_into_matrix(from+1, to+1, e_weight, e_index);
    }
    //计算图中所有点到终点的最短路径权重
    Dijkstra(dest, to_dest_weight); 
    //当图的规模大于一定规模时，启用的剪枝策略，预处理图中点到每个必经点的最短路径
    if (edge_num > THRESHOLD) {
        cutflag = true;
        for (short i = 0; i < designated_len; i++)
             Dijkstra(designated_point[i], to_designated_weight[i]);
     }
	//深度优先遍历查询最短路径
    dfs_visit(origin, 0);
    //结果输出
    for (auto &i : short_path)
        record_result(i);
}
{% endhighlight %}

本方案的主要剪枝优化策略是，上限剪枝。基本的思路如下：DFS算法搜索到某个顶点`s`，当遍历该顶点邻接节点`e`时，计算每一个节点到终点的最短路径权重`tmp`，并将这个权重加上已遍历路径的权重`w`。如果(`tmp`+`w`)大于已经存在的最短路径权重，那么没有必要深度遍历`e`节点，剪枝可以大大减少不必要的计算。此外，当图的规模较大，必经点很多时，可以增加一个剪枝策略。我可以在没有权重预测时，计算剩余必经点到终点的权重，并于前面提到`tmp`取最大值，这样可以优化剪枝策略，提升性能。算法中的Dijkstra主要是用来预处理，到终点和各个必经点最短路径的权重。

上面的剪枝策略，只是针对单点到终点的权重计算。如果能根据图的特性，计算经过所有剩余必经点的最短路径权重，可以达到更好的优化效果。此外，可以舍弃一定的精确度，模糊预测某个点到终点，而且必须经过剩余所有必经点的适当权重；如果这个路径权重比较大并且能保证计算的误差较小，对于整体性能的提升效果也很显著。

当然，经过分析会发现，制约当前版本的DFS性能的主要因素是初始最短权重的值；如果可以通过预处理找到一个较小的最初的路径权重可以减少很多不要的遍历。最后，提出一种进一步优化算法的思想，可以采用双队列BFS方法去优化目前的最短路径算法。BFS维护两个队列`Q1`与`Q2`: `Q1`是从起点开始广度遍历的待处理顶点队列；`Q2`是从终点开始广度遍历的待处理顶点队列。如果处理两个队列时，邻接顶点出现在对方队列里，则找到一条通路；验证这条通路是否满足需要。这种方式可以避免搜索遍历太深，也是一种剪枝的思想。
	
