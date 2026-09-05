# Arena Allocator

2026-3-22 Mer

# 1. Introduce allocator 

allocator, 又称为__内存分配器__, 其功能是什么, 为什么需要它?



我们之前常使用 `new` 和 `delete` 操作符来管理 程序的堆内存 (heap). 细看下来, `new` 这个操作包括两个部分

1. __内存空间分配__ : 在堆内存中申请一块空间, 此时内存中都是垃圾值.
2. __构造对象__ : 在申请的空间中执行__构造函数__, 构造出有实际意义的对象.

而`delete`操作类似, 它包括

1. __析构对象__
2. __释放内存空间__.

在一些特殊的__高性能__需求场景下, **空间分配**与**对象管理**一起进行会导致许多不必要的开销, 下文会有介绍.

本文的主题 Allocator 内存分配器, 就是用来完成上面两部操作的**解耦**, 将__堆内存上的空间分配(allocate)与释放(deallocate), 对象的构造(construct) 与析构(destroy)__ 分别视为 4 个独立的动作. 

一个 Allocator 应该提供以下接口 :

1. 分配 : `allocate(n);`
   1. $n$ 表示对象的个数. 
2. 构造 : `construct(ptr, arg);`
   1.  `ptr` 表示进行构造的位置(首地址)
   2. `arg` 表示构造函数的参数.
3. 析构 : `destroy(ptr);`
   1. `ptr` 表示进行析构的对象的首地址.
4. 释放 : `deallocate(ptr, n);`
   1. `ptr` 表示释放内存的首地址
   2. `n` 表示释放的对象个数 (用以计算释放的内存大小).



# 2. Why Allocator?

要说明引入 Allocator 的必要性, 就得看看不使用 allocator 的方案都存在什么问题. 为此先复习一下 `std::vector` 中的方法. 

----

## 2.1 `Size` / `Capacity` of std::vector

回忆一下 `std::vector<T>` 中的重要属性和方法, 下面的高性能场景通常也以此为基础. 

一个 `vector` 对象有两个基本属性 :

1. `size` : vector 中实际存在的, 已经构造的元素数量.
2. `capacity` : vector 在不重新分配内存的情况下, 最大可以容纳的元素数量.

在日常使用时, `capacity` 涉及到内存管理, 可能更难以被关注到. 上述两个概念对应于 `vector` 的两个重要方法.

假设一开始,  `std::vector<T> vec;` 对象仅进行了默认初始化. 现在已知需要往 `vec` 中添加 $n$ 个 `T` 类型的元素, 我们有两种提前处理的选择 : 

1. `resize(n)` : 在 vector 上分配 $n$ 个元素的内存大小, __并在其上执行 $n$ 次默认构造函数, 构造了 $n$ 个元素__ 
   1. 执行后 `size` = $n$, `capacity` = $n$.
2. `reserve(n)` : **仅**在 vector 上分配 $n$ 个元素的内存大小.
   1. `size` = $0$ , `capacity` = $n$. 

从中可以看出 `size` 和 `capacity` 的区别所在.

> 补充 : 更一般地, 如果某时刻 `vec` 中已经有了 $k$ 个元素,  `size` = $k$, `capacity` = $m \geq k$, 那么上面两个函数的行为如下 :
>
> - `resize(n)` : 若 $ k > n$, 那么 第 $n$ 个元素后的其他元素__会被全部析构__. 若 $k < n$, 那么不足的部分会通过构造函数补充元素. (若 `capacity` 不够也会将内存扩容). 
>   - 核心 : 会将向量的元素个数__强行改为 `n`__. 少了就补默认构造, 多了就析构.
> - `reserve(n)` : 若 $m \geq n$, 则 `vec` 已经至少包含 $n$ 个对象的内存空间. 此时**该函数什么都不做**. 反之若 $m < n$, 则会分配内存空间到 $n$ 个元素的容量.
>   - 核心 : 保证对象的内存有至少 $n$ 个元素的容量. 少了就分配到 $n$, **多了的部分不会释放**.

可见, `reserve` 是内存管理这个话题下更加常用且重要的方法. 也是避免触发扩容机制的一个方案.

## 2.2 `push_back` / `emplace_back` of std::vector

两个函数都是在 `vector` 对象的末尾添加一个元素, 但是过程不一样. 下面我们限定这样一个情景

有一个类 `myClass`, 有一个 双参数构造函数 `myClass(int n, float f);` 现在有一个 `vector<myClass> vec` 对象. 向其中添加元素时, 下面两种方法进行了这样的行为 :

- `vec.push_back(myClass(n, f))` : 
  1. construct : 调用 `myClass(n,f)` 构造了临时对象 `obj`;
  2. move : 将 `obj` 通过 __赋值/移动__ copy 到 `vec` 的末尾 ;
  3. destroy : 调用析构函数 `~myClass()` 销毁临时对象 `obj`.
- `vec.emplace_back(n, f)` : 
  1. 直接在 `vec` 的末尾处, 调用 `myClass(n,f)` 构造对象.

可见 `push_back` 在这种情形下多了一次移动/赋值与析构的开销. 在数据量较大时会带来显著的影响.

注意上述 `emplace_back` 是直接传入构造函数的参数. 如果写成 `vec.emplace_back(myClass(n,f))`, 其行为等价于 `push_back`. 只是将第二步的 move 赋值/移动 改为调用 __拷贝构造函数__. 开销上是相似的.



二者都会让 `vec.size()` 增加, 倘若 `size > capacity`, 则会**触发扩容**. 在堆上重新申请一块更大的内存空间, 将原来的 `vec` 的元素全部复制过去, 然后释放原有的空间. 这是一步开销十分大的操作, 应该尽可能提前 `reserve` 来避免. 另外, 下面的实例 2.3 也是基于此机制出现的.

## 2.3 Instance : Sequential storage of Tree

假设一棵二叉树的结点定义为

```C++
struct Node{
	int val;
	Node* left;
	Node* right;
};
```

在构建一个二叉树时, 为了提高 CPU 的缓存命中率, 我们希望这棵树的各个结点存放的内存位置接近.  有几种方案 :

1. 使用 `vector<Node>` 来保存结点. 此时每个元素都包含左右子节点指针.
   - 这种方案的最大缺陷在于, 如果我们不能预先用 `reserve(n)` 出足够的空间 (或者我们错误地估计了结点树 $n$), 那么在往  `vector<Node>` 中添加结点时, **触发扩容**, 会**导致所有的元素被迁移到另一处地址**. 但是`vector<Node>` 中已经保存的指针 `left` 和 `right` 却**仍然指向原来的那片地址区域**, 程序也无法再正常运行下去了.
   - 这是`vector` 这个类中的已经写好的内存分配机制, 它在一些应用情景上可能会带来麻烦. 

2. 一个缓解该问题的方案是, 我们不存储 指针 `Node*`, 而是存储左右子结点在 `vector<Node> vec` 中的下标.
   - 这是一个很容易想到的方案. 它确实能有效解决前一个方案中的问题. 
   - 相比于直接使用指针, 存储下标的方案在访问子节点时引入了数组索引计算的开销. 虽然十分微小, 但是下面第三种方案可以兼顾到二者.

3. 使用自定义的 allocator. 可以完全控制内存的操作方法和布局. 例如我们可以提前根据元素分配更大的空间; 可以在内存空间不足时引入链式存储来避免元素迁移带来的野指针问题.



## 2.4 Instance : Arena Allocator

继续看 2.3 中的实例. 我们仍用 `vector<Node> vec` 和 allocator 方案在内存中连续存储树的结点. 

计算机图形学, 特别是游戏 / 影视领域中, 常常需要高速地计算一个__动态__的模型, 模型的位置在变化, 也意味着模型对应的树结构 (用于渲染计算加速) 也在高速地变化. 

也就是说, 现在有这样一个需求 : **高频地重构`vec`中的树结构.**

看看两种方案会如何解决这个问题 :

1. `vector<Node> vec` : 设`vec` 中已经存储了一个树结构 (一堆`Node`). 现在需要往其中存入另一个树结构 (另一堆`Node`).
   1. 首先, 用`vec.clear()` 析构 `vec` 中的所有结点.
   2. 其次, 再向其中用`push_back`/`emplace_back` 存入新的树结点
2. allocator 方案 : 我们重定义一个 allocator 类. 它包含一个 __"偏移量指针offset"__, 代表当前内存块已经使用的数量. 初始时, allocator 中已经存了一堆`Node`. 
   1. 当需要切换树结构时, 直接__重置offset__ : `offset = 0`, 仅一行整数赋值代码, 我们就完成了原有树结构的重置. 
      - 此过程**没有执行任何析构函数**. 但是因为我们认为 `offset = 0`, 前一棵树的结点没有被真正删除, 但已经**被我们视为无效值.**
   2. 当存入新的结点时, 我们重新**在原来的对象位置开始构造construct,** 直接**覆盖**掉上一棵树的结点.

这就是 allocator 的强大之处. 在高性能需求领域, 自定义 allocator 是十分常见的.

上面提到的这个方案, 可以归为 __Arena Allocator__. 它对于__不调用析构函数没有任何副作用的数据类型是十分好用的__.

> 不调用析构函数可能会有副作用. 例如__某个对象持有了一个外部资源(文件, 打印机等).__ 它们释放资源的途径是析构函数. 
>
> 倘若不执行析构函数, 就会导致__所有资源永久泄露__. 

对于这个情景, `Node` 并不持有任何外部资源 (这种自定义类型我们称为==**Trivial（平凡）类型**==). 所以可以放心使用 Arena Allocator, 来避免析构上一棵树中的百万结点带来的性能开销.



# 3. A Brief Implementation of Arena Allocator

下面是一个极简的 Arena Allocator 内存分配器类实现. 

```C++
// By Gemini Pro
#include <cstddef>
#include <new>
#include <vector>

class ArenaAllocator {
private:
    char* buffer;        // 预分配的大块内存 🪵
    size_t capacity;     // 总大小
    size_t offset;       // 当前分配到了哪个位置 📍

public:
    // 构造时直接申请一大块堆内存
    ArenaAllocator(size_t size) : capacity(size), offset(0) {
        buffer = new char[size];
    }

    ~ArenaAllocator() {
        delete[] buffer;
    }

    // 核心分配逻辑：只移动指针
    void* allocate(size_t size) {
        if (offset + size > capacity) return nullptr; // 空间不足
        
        void* ptr = buffer + offset;
        offset += size; // 指针“单向奔跑”
        return ptr;
    }

    // 一键清空：$O(1)$ 时间复杂度
    void reset() {
        offset = 0;
    }
};
```

