# 一些模版代码

## 快速幂

### 无模

```cpp
long long qpow(long long a, long long b){
	long long ans = 1;
    
    while(b){
        if (b & 1){
            // 奇数 只需要操作一次.
            ans *= a;
        }
        
        // 分解
        a *= a;
        b >>= 1;
    }
}
```

### 有模

```cpp
const long long MOD = 1e9+7;

long long qpow(long long a, long long b){
    long long ans = 1;
    a %= MOD;
    
    while(b){
        if (b & 1){
            ans *= a;
            ans %= MOD;
        }
        
        a = a * a % MOD;
        b >> = 1;
    }
    
    return ans;
}
```

### 无模 - 递归

好理解, 但是有递归开销, 建议少用.

```cpp
long long qpow(long long a, long long b){
    if (b == 0){
        return 1;
    }
    
    half = qpow(a, b/2);
    
    if(b & 1){
        return a * half * half;
    }
    return half * half;
}
```



## 双堆

双堆的典型应用是动态维护一个持续添加的序列的中位数.

- `addNum(int x)`
- `int findMedian()`
- 

```cpp
class MedianFinder {
public:
    priority_queue<int> lower;
    priority_queue<int, vector<int>, greater<int>> upper;
    
    void addNum(int x){
        // 1.决定放哪边
        
        if (lower.empty() || x <= lower.top()){
            lower.push(x);
        } else {
            upper.push(x);		// 这种写法会将 lower 和 upper 中间的值偏向于 upper入堆
        }
        
        // 2. 维护 Size 平衡
        // 需要保证 lower.size() == upper.size() [+ 1]
        if (lower.size() > upper.size() + 1){
            upper.push(lower.top());
            lower.pop();
        }
        
        if (upper.size() > lower.size()){
            lower.push(upper.top());
            upper.pop();
        }
    }
    
    double findMedian() {
        if (lower.size() > upper.size()){
            return lower.top();
        }
        return ((long long) lower.top() + upper.top()) / 2.0;
    }
};

```

## 单调队列

> 高级题 : Shortest Subarray with Sum at Least K. 
>
> 转化后 : 给定一个数 $k$ 和一个前缀和序列 $pre$, 请优化 :
> $$
> \min(i-j)\\
> s.t.\quad  pre_i - pre_j \geq k
> $$

思路 : 优化 `for i : for j` 双循环, 先考虑固定一个 $i$, 看看 $j$ 的最优候选在暴力优化以外, 有没有优化空间.

注意到一个核心的性质 : 如果 $j_1 < j_2 < i$ 满足 $pre[j_1] \geq pre[j_2]$, 那么 $j_1$ 已经不可能是最优解了.

也就是说, 如果我们所维护的候选集合中, 扫描到新的 $j$ 时, 集合里所有 $\geq j$ 的元素都应该被删除.

实现这个操作需要用到 **单调队列**.对于一个新的 $j$, 按照以下次序维护队列 :

- 若队尾 $pre[back] \geq pre[j]$, 则 `pop_back()`, 直到前面的条件不成立为止
- `push_back(j)`;

这样维护下, 队列将是单调的. 队头会成为队列 (候选集合) 中最小的元素. 此时可以直接通过 `front()` 来判断当前 $i$ 是否存在可行解. 如果可行, 则可以 `pop_front()` [这里是最难想的一步], 因为这个队头已经不再可能是"未来的最优解"了.

所以, 遍历到 $i$ 时还要先执行以下操作 :

- 若 $pre[i]- pre[front] >= k$ 则一直 `pop_front()` 直到条件不成立. 需要记录最后一个成立的 front, 这是当前固定 $i$ 下的最优解.

```cpp
deque<int> q;
q.push_back(0);
int maxAns = 0;
for(int i = 1; i < n; i++){
    // 判断可行性
    int cache = -1; // 记录最后一个可行解 (是当前 i 下的最优解)
    while (!q.empty() && pre[i] - pre[q.front()] >= k){
        // 可行
        cache = q.front();
        q.pop_front();		// 不再是最优解候选, 已过期.
    }
    if (cache != -1){
        // 新的更优解
        maxAns = max(maxAns, i-cache);
    }
    
    // 新元素作为 j 维护加入单调队列中
    while(!q.empty() && pre[q.back()] >= pre[i]){
        q.pop_back();
    }
    q.push_back(i);
}
return maxAns
```



## 图论

DFS / BFS : 邻接表模版代码

```cpp
int n,m;
vector<vector<int>> graph;
vector<bool> vis;

void dfs(int u){	// 遍历到 u 节点 - 递归算法
    vis[u] = true;
    
    // =======================
    // 处理节点 u, 按照合适的逻辑
    // =======================
    
    for(int v : graph[u]){
        if (!vis[v]){
            dfs(v);
        }
    }
}

void bfs(int start){
    // 队列法
    queue<int> q;		// 记录节点编号
    vis[start] = true;
    q.push(start);
    
    while(!q.empty()){
        int u = q.front();
        q.pop();
        // =======================
        // 处理节点 u, 按照合适的逻辑
        // =======================
        for (int v : graph[u]){
            if (!vis[v]){
                vis[v] = true;			// 注 : 入队时就要标记, 否则可能多次入队
                q.push(v);
            }
        }
    }
}

int main(){
	cin >> n >> m;			// 节点数 >> 边数
    
    graph.resize(n+1);		// 提前分配并构造空间
    vis.resize(n+1, false);
    
    for(int i = 0; i < m; i++){
        int u,v;
        cin >> u >> v;		// u -> v 边
        // 建图
        graph[u].push_back(v);
        graph[v].push_back(u);		// 无向图则双边都加
    }
    
    // 图可能不连通时
    for(int i = 0; i < n; i++){
        if (!vis[i]){
            dfs[i];		// ||  bfs[i];
        }
	}
}
```

## 拓扑排序

拓扑排序常用例子是 "课程依赖". 给定一个依赖关系 (有向无环图), 按什么顺序能不违反依赖关系的情况下修完所有课程 ? 换句话说, 如果有边 `u->v`, 那么拓扑排序是一个节点序列 : 满足 $u$ 在 $v$ 的前面.

我们只需要统计每个节点的入度, 就好了.

```cpp
vector<int> topoSort(int n, vector<vector<int>>& graph){
    vector<int> indegree(n+1, 0);
	// 这里的 graph 构建可以看上面
    for (int u = 1; u <= n; i++){
        for(int v : graph[u]){
            indegree[v]++;
        }
    }
    
    // 存储没有依赖关系的节点
    queue<int> q;
    
    for (int i = 1; i <= n; i++){
        if (indegree[i] == 0){
            q.push(i);
        }
    }
    // 答案
    vector<int> order;
    
    while(!q.empty()){
        int u = q.front();
        q.pop();
        
        order.push_back(u);
        
        for(int v : graph[u]){
            indegree[v] -- ;		// 遍历到 u, 则 u 指向的节点依赖已经结束
            
            if (indegree[v] == 0){
                q.push(v);
            }
        }
    }
    return order;
}
```

## 并查集

标准并查集的接口包括 :

- `init(int n)` : 根据并查集容量初始化
- `int find(int x)` : 查找 $x$ 的 parent. 具有 "路径压缩优化"
- `bool unite(int x, int y)` : 添加新的连通关系 $(x, y)$, 具有 "根据集合大小优化"
- `bool same(int x, int y)` : 判断是否属于同一集合
- `int size(int x)` : 获得 $x$ 的大小.

```cpp
class DisjointUnion{
public:
	vector<int> parent;
    vector<int> sz;			// size
    
    // Initialization
    DisjointUnion(int n){	// n -> capacity
        parent.resize(n);
        sz.assign(n, 1);	// 记录每个类的大小
        
        for(int i = 0; i < n; i++){
            parent[i] = i;
        }
    }
    
    int find(int x){
        // 路径压缩
        if (parent[x] != x){
            parent[x] = find(parent[x]);
        }
        return parent[x];
    }
    
    bool unite(int x, int y){
        int rx = find(x);
        int ry = find(y);
        
        if (rx == ry){
            return false;
        }
        
        // 小集合挂在大集合下, 减少find时间
        if (sz[rx] < sz[ry]){
            swap(rx, ry);
        }
        
        parent[ry] = rx;
        sz[rx] += sz[ry];
        
        return true;
    }
}
```



## Trie 前缀树

- `init()` : 构造根结点
- `insert(string & s)` / `insert(int x)`
- `search(string &s)` : 
- `prefix(string &s)` 

```cpp
class Trie{
public:
    struct Node{
      	int next[26];		// 假设字符集大小是 26;
        bool isEnd;
        
        Node(){
            memset(next, -1, sizeof(next));
            isEnd = false;
        }
    };
    
    vector<Node> trie;			// 顺序存储的树
    
    Trie(){
        trie.emplace_back();	// 构造根结点.
    }
    
    void insert(const string & s){
        int p = 0;		// root
        for(auto& c : s){
            int x = c - 'a';		// 获取字符索引
           	if (trie[p].next[x] == -1){
                // 该节点未创建
                trie[p].next[x] = trie.size();
                trie.emplace_back();
            }
            // 遍历
            p = trie[p].next[x];
        }
        
        trie[p].isEnd = true;
    }
    
    bool search(const string & s){
        int p = 0;
        for(auto & c : s){
            int x = c - 'a';
            if (trie[p].next[x] == -1){
                return false;
            }
            p = trie[p].next[x];
        }
        return trie[p].isEnd;		// 必须完全匹配
    }
    
}
```

## 树状数组

也称为 Fenwick Tree / Binary Indexed Tree (BIT)

它的最经典应用是 "单点修改 + 区间查询".

- 如果怕与 "区间修改 + 单点查询" 混淆的话, 想象一下这种情况其实用差分数组就能完成. 而上面这个更难.

树状数组也需要几个接口 / 模版

- `init(int n)` : 
- `lowbit(int x) { return x & (-x) ; }` 
- `add(int i, long long delta)`
- `sum(int i)` : 快速求前缀和 
- `query(int l, int r)` : 调用 两次 `sum`

```cpp
class BIT{
public:
    int n;		// size
    vector<long long> tree;
    
    BIT(int nn) : n(nn){
        tree.resize(n+1);
    }
    
    // 返回 x 最右侧的 1
    inline lowbit(int x){
        return x & (-x);
    }
    
    void add(int i, long long delta){
       	// add 过程中, 需要不断 x + lowbit(x) 
        // 该操作相当于在末尾 1 上再加一个 1
        while(i <= n){
            tree[i] += delta;
            i += lowbit(i);
        }
        // O(log n)
    }
    
    int queryLegacy(int l, int r){
       	// 区间查询
        // 最大的理解难点在于, 如何分划区间 ?
        // 例如 [3,6] 很大概率是要分划成 [3,4] + [5,6] 的, 
        // 结果应该是 tree[4] - tree[2] + tree[6]
        // 我们首先需要发现, tree[4] 覆盖了 1~4, 而 [3,4] 没有. 我们需要一种方法, 
        // 判断 4 所覆盖 left, 与 l 的大小关系. 如果是等于, 皆大欢喜, 直接返回 tree[4]. 
        	// 1. 4 覆盖的范围更大, 那这时候就需要减法.
        	// 2. 4 覆盖的范围小, 那就砍掉 4 已经覆盖的部分, 继续query.
        // 我们需要找到 r 的左侧, 第一个比它高的索引 com.
        int com = r - lowbit(r);
        if (com + 1 == l){  
            return tree[r];
        }
        if (l < com + 1){
            // 应该截断继续 query
            return tree[r] + query(l, com);
        }
        if (l > com + 1){
            // 意味着 tree[r] 多了.
            return tree[r] - query(com + 1, l-1);
        }
    }
    
    // 事实上我们可以先查询前缀和
    long long sum(int i){
        int res = 0;
        while(i != 0){
            res += tree[i];
            i -= lowbit(i);
        }
		return res;
    }
    
    // 再查询任意区间和
    long long query(int l, int r){
        return sum(r) - sum(l-1);
    }
};
```



## 线段树*

针对几种情形, 有不同的模版:

- 单点修改 + 区间查询 : 最基础, 无需 lazy. 作用与 BIT 类似.
- 区间修改 + 区间查询 : BIT 做不到, 在线段树中也需要额外 lazy propagation 避免复杂度爆炸.
- 动态开点线段树 : 节点很稀疏时用.



单点修改 + 区间查询 :

```cpp
class SegmentTree{
public:
    int n;					// 覆盖范围是 n 个离散元素 (而不是节点数)
    vector<int> tree;		// 数组化表示的线段二叉树
    vector<int> lazy;		// Lazy Propagagtion 模版记录区间的修改
    
    // Initialize
    SegmentTree(vector<int>& nums) : n(nums.size()) {
        tree.resize(4 * n);
        build(nums, 1, 0, n-1);
    }
    
    void pushUp(int p){
        //==========================================
        // 这里按区间和维护, 实际需要根据维护的信息进行修改
        //==========================================
        tree[p] = tree[2 * p] + tree[2 * p + 1];
    }
    
    // Lazy 模版
    void pushDown(int p, int l, int r){
        if (lazy[p] == 0){
            return;
        }
        
        int mid = l + (r-l)/2;
        
        tree[2*p] += lazy[p] +  (mid - l + 1);
        lazy[2*p] += lazy[p];
        tree[2*p + 1] += lazy[p] + (r - mid);
        lazy[2*p + 1] += lazy[p];
        
        lazy[p] = 0;
    }
    
    // build
    // p : parent, 当前节点的编号. 
    // l,r : parent 所负责管理的区间范围. (而非孩子节点编号, 因为这可以用 2*p[+1] 算出来)
    void build(vector<int>& nums, int p, int l, int r){
        
        if (l == r){
            tree[p] = nums[l];
            return;
        }
        
        int m = l + ( r - l ) / 2; 
        build(nums, 2*p, l, mid);
        build(nums, 2*p+1, mid+1, r);
        
        pushUp(p);
    }
    
    // udpate interval
    void update(int p, int l, int r, int ul, int ur, int val){
        if (ul <= l && r <= ur){
            tree[p] += val * (r - l + 1) ;
            lazy[p] += val;
            return;
        }
        
        // 需要遍历子节点, 先pushDown
        pushDown(p, l, r);
        
        int mid = l + (r-l)/2;
        
        if (mid >= ul){
            update(2*p, l, mid, ul, ur, val);
        }
        if (mid + 1 <= ur){
            update(2*p+1, mid+1, r, ul, ur, val);
        }
        pushUp(p);
    }
    
    // update single
    void update(int p, int l, int r, int pos, int val){
       	if (l == r && r == pos){
            tree[p] = val;
        }
        
        int m = l + (r-l)/2;
        // 左右选一个分支.
        if (mid >= pos){
            update(2 * p, l, mid, pos, val);
        }
        else {
            update(2 * p + 1, mid+1, r, pos, val);
        }
        
        pushUp(p);
    }
    // 封装
    void update(int pos, int val){
        update(1, 0, n-1, pos, val);
    }
    
    // query [ql, qr]
    int query(int p, int l, int r, int ql, int qr){
        if (ql <= l && r <= qr){
            return tree[p];
            // 不再继续搜索
        }
        
        // 需要继续遍历, 先 PushDown (Lazy)
        pushDown(p, l, r);
        
		//==========================================
        // 这里按区间和维护, 实际需要根据维护的信息进行修改
        int ans = 0;
        //==========================================
        
        
        int mid = l + (r-l)/2;
        
        if (ql <= mid){
            int lson = query(2 * p, l, mid, ql, qr);
            ans += lson;
        }
        if (qr >= mid+1){
            int rson = query(2 * p, mid+1, r, ql, qr)
            ans += rson;
        }
        return ans;
    }
    // 封装
    int query(int ql, int qr){
        query(1, 0, n-1, ql, qr);
    }
};
```

## Floyd 算法

无向图之间求两两最短路径.

这个算法原理比实现难.

首先, 无向图的最短路径满足三角不等式

$$

d(u,v) \leq d(u,w) + d(v,w)

$$

但是我们可以观察到, 如果枚举所有 $w$, 一定有一条是在最短路径上, 使得 $d(u,w) + d(w,v)$ 最小.

这就是 Floyd 的核心思路. 每枚举一个点 $w$, 都将它视为 "其它两组点的中间点", 更新 $d(u,v)$.

它的算法实现核心非常简单 : 

```cpp
vector<vector<int>> dist(n + 1, vector<int>(n + 1, INF));		// INF -> 1e9
// Init : 自己到自己距离为 0
for (int i = 1; i <= n; i++) {
    dist[i][i] = 0;
}
// Init : 读边
for (int i = 0; i < m; i++) {
    int u, v;
    cin >> u >> v;

    dist[u][v] = 1;
    dist[v][u] = 1;
}
// 核心三重循环
for (int k = 1; k <= n; k++) {
    for (int i = 1; i <= n; i++) {
        for (int j = 1; j <= n; j++) {
            dist[i][j] = min(dist[i][j],  dist[i][k] + dist[k][j]);
        }
    }
}
```

