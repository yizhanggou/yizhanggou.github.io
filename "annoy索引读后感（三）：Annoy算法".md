---
title: "annoy索引读后感（三）：Annoy算法"
author: 一张狗
lastmod: 2019-07-06 08:39:43
date: 2018-07-31 20:03:20
tags: []
---


https://blog.csdn.net/hero_fantao/article/details/70245387
https://blog.csdn.net/LegenDavid/article/details/78490957

Annoy（Approximate Nearest Neighbors Oh Yeah）算法是应用于稠密矩阵的最近邻查找算法，Annoy的目标是建立一个数据结构，使得查询一个最近邻的时间是线性的。Annoy通过对输入矩阵建树来获取O(log n)的检索速度，具体实现方式如下：

annoy支持三种距离度量方式，cos距离，欧式距离和曼哈顿距离，在项目中我们采用了cos距离。

这里有几个变量需要注意：

n_descendants保存的是该节点到叶子节点所包含的所有节点的个数;

_s是node占有的空间大小, _get(item) 是 _get(i) = (Node*)((uint8_t *)_nodes + (_s * i))。

_nodes保存的是这块内存的起点。

_roots 存的所有树的根节点；

_n_items表示有多少个向量需要建索引

_n_nodes记录了一共有多少个节点，首先初始化为n_items，然后建树的时候每次找到一个法平面都要加1，最后再加上roots.size()，即所有根节点，每次加的时候都会开辟内存存放相应的数据（除了初始化的时候，初始化的节点会在make_tree的最后分配内存空间）。



检索过程：

1.对输入数据查找两个质心(two_means)

```
template<typename T, typename Random, typename Distance, typename Node>
inline void two_means(const std::vector<Node*>& nodes, 
        int f, Random& random, bool cosine, T* iv, T* jv) {
    //随机选择两个点，以这两个节点为初始中心节点，执行聚类数为2的kmeans过程，最终产生收敛后两个聚类中心点。
    //随机选择两个点赋值为iv和jv
    // 之后在两百次迭代中,每次随机选择一个点,靠近哪边(离iv近还是离jv近)就与那个点靠拢
    // 最后得到nodes中的两个质心iv jv
    /*
    This algorithm is a huge heuristic. Empirically it works really well, but I
    can't motivate it well. The basic idea is to keep two centroids and assign
    points to either one of them. We weight each centroid by the number of points
    assigned to it, so to balance it. 
    */
    static int iteration_steps = 200;
    size_t count = nodes.size();

    size_t i = random.index(count);
    size_t j = random.index(count-1);
    j += (j >= i); // ensure that i != j
    std::copy(&nodes[i]->v[0], &nodes[i]->v[f], &iv[0]);
    std::copy(&nodes[j]->v[0], &nodes[j]->v[f], &jv[0]);
    if (cosine) { 
        normalize(&iv[0], f); 
        normalize(&jv[0], f); 
    }

    int ic = 1, jc = 1;
    for (int l = 0; l < iteration_steps; l++) {
        size_t k = random.index(count);
        T di = ic * Distance::distance(&iv[0], nodes[k]->v, f),
        dj = jc * Distance::distance(&jv[0], nodes[k]->v, f);
        T norm = cosine ? get_norm(nodes[k]->v, f) : 1.0;
        if (di < dj) {
        //计算选出的点与iv、jv的距离，靠近哪边就以那边的点和本身做一次means
        //举例iv[z]，在某一次乘以ic再加上选出的点之后，除以ic+1,然后ic++
        //然后在下一次又选到iv的时候，乘以ic就还原出了上一次选中iv的时候的分子
        //这样在最后iv就是选中iv这边的点的均值
        //jv同理
            for (int z = 0; z < f; z++)
                iv[z] = (iv[z] * ic + nodes[k]->v[z] / norm) / (ic + 1);
            ic++;
        } else if (dj < di) {
            for (int z = 0; z < f; z++)
                jv[z] = (jv[z] * jc + nodes[k]->v[z] / norm) / (jc + 1);
            jc++;
        }
    }
}
```

2.利用两个质心找到法平面的法向量（法向量就是两个质心相减）

```
static inline void create_split(const std::vector<Node<S, T>*>& nodes, int f, Random& random, Node<S, T>* n) {
    //创建超平面,入参为当前空间的所有点nodes，维度f，随机函数random，分割节点n
    //首先求出nodes这个超平面的两个质心,然后算出这两个质心连线对应的向量,即分隔面的法向量,并归一化
    std::vector<T> best_iv(f, 0), best_jv(f, 0); // TODO: avoid allocation
    two_means<T, Random, Angular, Node<S, T> >(nodes, f, random, true, &best_iv[0], &best_jv[0]);
    for (int z = 0; z < f; z++)
        n->v[z] = best_iv[z] - best_jv[z];
    normalize(n->v, f);
}
```

3.对平面上每个点，通过与法向量相乘（向量点积，即计算它们的cos值），求出点与法向量夹角的正负，以正负来分出其属于左子树还是右子树。

```
D::create_split(children, _f, _random, m);//m保存了法向量!!!!!!
//把输入点放到children_indices(左右子树)中
for (size_t i = 0; i < indices.size(); i++) {
    //求出所有children属于哪个面(通过与m比较),然后push到children_indices向量中
    //children_indices这个向量就是要建立的子树
    S j = indices[i];
    Node* n = _get(j);
    if (n) {
        bool side = D::side(m, n->v, _f, _random);
        children_indices[side].push_back(j);
    }
}
```

这里还涉及到了side函数和margin函数：

```
static inline T margin(const Node<S, T>* n, const T* y, int f) {
    //计算两个向量的夹角
    T dot = n->a;
    for (int z = 0; z < f; z++)
        dot += n->v[z] * y[z];
    return dot;
}
static inline bool side(const Node<S, T>* n, const T* y, int f, Random& random) {
    //选择哪边，左子树保存了与法向量反向的，右子树保存了与法向量同向的
    T dot = margin(n, y, f);
    if (dot != 0)
        return (dot > 0);
    else
        return random.flip();
}
```

4.之后递归创建子树，直到空间中点少于k，把k放入叶子节点。二叉树底层是叶子节点记录原始数据节点，其他中间节点记录的是分割超平面的信息。

```
S _make_tree(const std::vector<S >& indices) {
    //二叉树底层是叶子节点记录原始数据节点，其他中间节点记录的是分割超平面的信息
    if (indices.size() == 1)
        return indices[0];

    if (indices.size() <= (size_t)_K) {//小于k个节点,那么放到 
        _allocate_size(_n_nodes + 1); 
        S item = _n_nodes++; 
        Node* m = _get(item);//_get()这个函数把item放到树中 
        m->n_descendants = (S)indices.size();
        // Using std::copy instead of a loop seems to resolve issues #3 and #13,
        // probably because gcc 4.8 goes overboard with optimizations.
        std::copy(indices.begin(), indices.end(), m->children);
        return item;
    }

    std::vector<Node*> children;
    for (size_t i = 0; i < indices.size(); i++) {
        S j = indices[i];
        Node* n = _get(j);
        if (n)
            children.push_back(n);
    }

    std::vector<S> children_indices[2];
    Node* m = (Node*)malloc(_s); // TODO: avoid
    //m是输入所有点的法平面的法向量
    D::create_split(children, _f, _random, m);//m保存了法向量!!!!!!

    //把输入点放到children_indices(左右子树)中
    for (size_t i = 0; i < indices.size(); i++) { 
    //求出所有children属于哪个面(通过与m比较),然后push到children_indices向量中 
    //children_indices这个向量就是要建立的子树 
        S j = indices[i]; 
        Node* n = _get(j); 
        if (n) { 
            bool side = D::side(m, n->v, _f, _random);
            children_indices[side].push_back(j);
        }
    }

    // If we didn't find a hyperplane, just randomize sides as a last option
    while (children_indices[0].size() == 0 || children_indices[1].size() == 0) {
        //如果左子树或者右子树数量为0,就是意味着没有找到超平面,那么就随机分配side
        if (_verbose && indices.size() > 100000)
            showUpdate("Failed splitting %lu items\n", indices.size());

        children_indices[0].clear();
        children_indices[1].clear();

        // Set the vector to 0.0
        for (int z = 0; z < _f; z++) m->v[z] = 0.0;

        for (size_t i = 0; i < indices.size(); i++) { S j = indices[i]; // Just randomize... children_indices[_random.flip()].push_back(j); } } int flip = (children_indices[0].size() > children_indices[1].size());

    m->n_descendants = (S)indices.size();
    //n_descendants保存的是该节点到叶子节点所包含的所有节点的个数
    for (int side = 0; side < 2; side++) // run _make_tree for the smallest child first (for cache locality) m->children[side^flip] = _make_tree(children_indices[side^flip]);//递归创建子树

    _allocate_size(_n_nodes + 1);
    S item = _n_nodes++;
    memcpy(_get(item), m, _s);//这里是把m复制到树中
    //_s是node占有的空间大小, _get(item) 是 _get(i) = (Node*)((uint8_t *)_nodes + (_s * i))
    //二叉树底层是叶子节点记录原始数据节点，其他中间节点记录的是分割超平面的信息
    free(m);

    return item;
}
```

5.这里还存在一个问题，一个点的邻居不一定在这个区域内，解决方法是建多颗树。

6.检索过程_get_all_nns是使用一个优先队列，首先把所有树的root入栈，然后如果找到叶子节点，则将该树节点对应的所有子节点加入到待输出的nns变量中，如果是非叶子结点，则将两棵子树都加入到优先队列中，权值是目标向量与法向量的夹角，左子树则取反，因为左子树保存的是与法平面相反方向的，需要取反才能是正数，才可以与右子树作比较。之后每次都从优先队列中pop出最大值再重复过程。

```
void _get_all_nns(const T* v, size_t n, size_t search_k, std::vector<S>* result, std::vector* distances) {
    //检索过程,输入是 &vec_list[0], expand_num, -1, &result, &dis
    //vec_list[0]是目标向量,expand_num是扩展多少个,result是结果,dis是结果和目标间的距离
    std::priority_queue<std::pair<T, S> > q;
    //S是uint, T是float

    if (search_k == (size_t)-1)//如果search_k传-1,则初始化为expand_num乘以树的数目
        search_k = n * _roots.size(); // slightly arbitrary default value

    for (size_t i = 0; i < _roots.size(); i++) {
        //先保存存所有的root,root的距离是无穷大
        q.push(std::make_pair(std::numeric_limits::infinity(), _roots[i]));
    }

    std::vector<S> nns;
    while (nns.size() < search_k && !q.empty()) {
        const std::pair<T, S>& top = q.top();//初始的时候q保存了所有树的root
        T d = top.first;//T是float,d是当前节点到目标节点的距离,
        S i = top.second;//S是uint,第几个节点
        Node* nd = _get(i);
        q.pop();
        if (nd->n_descendants == 1 && i < _n_items) { 
        //叶子节点,All nodes with n_descendants == 1 are leaf nodes 
            nns.push_back(i); 
        } else if (nd->n_descendants <= _K) { 
        //有大于1小于k的子节点,将该树节点对应的所有子节点加入到nns中 
            const S* dst = nd->children;
            nns.insert(nns.end(), dst, &dst[nd->n_descendants]);
        } else {//非叶子节点,则将两棵子树都加入到优先队列中
            T margin = D::margin(nd, v, _f);//margin计算两个向量的cos值
            //在建树的时候,夹角为正的放到右子树(children 1),夹角为负的放到左子树(children 0)
            //因为是cos,而优先队列是大的值优先,所以这里使用优先队列是合理的,向量越靠近cos值越大
            q.push(std::make_pair(std::min(d, +margin), nd->children[1]));
            q.push(std::make_pair(std::min(d, -margin), nd->children[0]));
        }
    }

    // Get distances for all items
    // To avoid calculating distance multiple times for any items, sort by id
    sort(nns.begin(), nns.end());
    std::vector<std::pair<T, S> > nns_dist;
    S last = -1;
    for (size_t i = 0; i < nns.size(); i++) { S j = nns[i]; if (j == last) continue; last = j; nns_dist.push_back(std::make_pair(D::distance(v, _get(j)->v, _f), j));
    }
    //最后对nns中的id去重后计算距离返回。
    size_t m = nns_dist.size();
    size_t p = n < m ? n : m; // Return this many items
    std::partial_sort(&nns_dist[0], &nns_dist[p], &nns_dist[m]);
    for (size_t i = 0; i < p; i++) { if (distances) distances->push_back(D::normalized_distance(nns_dist[i].first));
        result->push_back(nns_dist[i].second);
    }
}
```

检索后输出的时候，distance计算输出的向量与目标向量的距离（(a/|a| – b/|b|)^2）。

![20170803173722568](http://yizhanggou.top/imgs/2019/07/20170803173722568.jpeg)
![20170803173652347](http://yizhanggou.top/imgs/2019/07/20170803173652347.jpeg)

