# Smart Pointer

2026-3-24 Mergic

本文将讨论 `unique_ptr`, `shared_ptr`, `weak_ptr` 的实现逻辑和内存布局.

# `unique_ptr`

`unique_ptr` 有两个特性

- 所指向的对象是被指针独占的. 其他 Smart Pointer 不能指向这个对象. 
- RAII : 当被指向的对象退出作用域时, 会被自动回收.

**独占性**是由**编译器**在编译时期保证的. **RAII** 是由编译器自动分析对象作用域, 在适当位置插入`delete` 来保证的.

在底层的内存布局上,  `unique_ptr` 对象仅包含了一个裸指针. 这是一种**零成本抽象**. 

默认情况下的内存布局十分简单.

-----

但是, 这种简单的机制并不能满足所有应用场景.

 若我们需要引入**自定义的 Deleter** 以满足特定的高性能系统的需求时, `unique_ptr` 就需要一些更灵活的内存布局来存储信息了.

> 例如, 我们为对象写了一个内存池来统一管理内存资源, 那么当对象退出作用域时, 不能让编译器自动执行`delete`. 而是应该传入自定义的析构函数, 来将对象内存归还到内存池中.

自定义 Deleter 的实现方法不同, `unique_ptr` 的内存开销也会有不同

1. **使用函数指针** : 在 `unique_ptr` 的内存中额外保存一个指向 Deleter 的函数指针.
   1. 这种方案会将 其内存大小 (`sizeof`) 从 $8$ 膨胀到 $16$.
2. **使用无状态仿函数(Stateless Functor)** : 这是目前 `unique_ptr` 在接受 外部 Deleter 时的实现.
   1. 在使用**空基类优化**后, 这种方案不会扩大 `unique_ptr` 的大小.

什么是  Functor ? 它是指一个普通的**类或结构体**, **重载了 `()` 运算符**, 使得实例可以像函数一样被调用.

什么是 Stateless ? 是指类或结构体中**没有任何成员变量和虚函数**.

```C++
// 这是一个无状态仿函数，常用于自定义删除器
struct MyDeleter {
    void operator()(int* p) const {
        delete p;
    }
};
```

`MyDeleter` 是可以实例化的. C++ 要求每个对象的大小都不能是 $0$, 否则无法通过指针找到这个对象. 因此执行`sizeof(MyDeleter)` 时会输出 $1$.

倘若我们这样组织 `unique_ptr<int>` :

```C++
class unique_ptr_for_int{
	int* ptr;
    MyDeleter deleter;
}	// sizeof() == 16
```

由于**内存对齐**机制, 使用 `sizeof()` 运算符会得到大小为 $16$. 这与直接传入函数指针的效果差不多.

所以此时要引入**空基类优化 Empty Base Class Optimization (EBO)**了.

> C++ 标准规定 : 当一个空类作为基类 (Base Class) 被继承时，**只要它(空基类)不与派生类的第一个非静态数据成员同类型**，编译器就可以优化掉这 1 字节的空间，让它不占用任何内存。
>
> 人话 : 派生类中第一个non-static**新成员类型不是基类**.

> 上述机制实现是通过一个叫 `compressed_pair` 的模版类. 

利用这个机制, 我们这样组织智能指针 :

```C++
class unique_ptr_for_int_with_EBO : public MyDeleter{
	int* ptr;
}
```

此时, 基类是 `MyDeleter`, 派生类的第一个non-static成员是`int*`, 不同类型.

而这里因为有一个对象`ptr`, 空基类并不会触发 `sizeof == 0` 自动补充 1 字节的机制. 

所以我们完美地让 `unique_ptr` 仅占用一个裸指针的大小, 却也可以**自定义Deleter**的功能.



# `shared_ptr`

C++ 允许多个  `shared_ptr` 同时指向一个对象. 并在最后一个使用者离开时精准释放资源.

`shared_ptr` 对象在内存中的布局也不复杂, 其对象包含**两个裸指针**, 因此`sizeof(shared_ptr)`的大小固定为 16 B :

1. 指向**被管理对象内存的首地址**
2. 指向**被管理对象的控制块CB的首地址**.

==**控制块Control Block, CB**==是智能指针管理对象时使用到的数据结构. (有点像进程控制块 PCB 那样的一个数据结构). 每个被 `shared_ptr` 管理的对象都包含其专属的 CB, 其中维护着以下重要信息

1. 强引用计数 : 记录 `shared_ptr` 同时指向该对象的个数.
2. 弱引用计数 : 记录 `weak_ptr` 同时指向该对象的个数.
3. 自定义 Deleter 
4. 空间分配器 Allocator

**对象与CB内存都是在堆上分配的**.



`shared_ptr` 有以下两种初始化方式

- `shared_ptr<Obj> ptr(new Obj());`
- `shared_ptr<Obj> ptr = make_shared<Obj>();`

乍一看二者都能做到创建一个新的 `Obj` 对象并让 `ptr` 指向它. 

为什么 C++ 标准库要专门搞了一个 `make_shared<>()` 方法来完成 `new` 的效果? 

因为**第一种写法会让系统****分配两次内存**. 分别是 :

1. 执行 `new Obj()` 时第一次分配内存给 `Obj()` 对象.
2. 执行 `shared_ptr<Obj> ptr(...)` 时第二次分配内存给 **控制块 CB**.

系统分配内存是一次**系统调用**, 会大幅**增加时间开销**, 同时上述两步分配都是在堆中进行, 若分配的空间比较碎片化, 则对**CPU缓存不友好**.

第二种初始化方法就是用来解决这个问题的. 它提前计算出了**对象 + CB 所需占用的的大小**, 然后一次申请一块**连续的内存**, 将二者对象与 CB 的内容放在一起.

这种做法不仅减少一次系统调用的昂贵开销, 还提升了缓存局部性.

> 使用第二种初始化方法时, 由于**只进行了一次内存分配**, 所以对强弱引用计数管理规则如下 :
>
> - **当强引用计数归零时, 析构 (Destruction) 对象, 但是不释放 (Deallocate)**.
> - **当强引用计数与弱引用计数都归零时, 释放(Deallocate) 对象和CB所在的连续内存块**.
>
> 而 `new` 方法则不一样, 它分配了两次内存, 它们释放的过程时也是独立的.

| **行为**           | **`new `方法**                  | **`make_shared` 方法**             |
| ------------------ | ------------------------------- | ---------------------------------- |
| **内存布局**       | 离散（两块内存）                | 连续（一块大内存）                 |
| **强引用归零时**   | **析构**对象 + **释放**对象内存 | **仅析构**对象                     |
| **弱引用 > 0 时**  | 对象内存已回收到系统            | **对象内存仍被占用**（即使已析构） |
| **全部计数归零时** | 释放控制块内存                  | 释放整块（对象+控制块）内存        |

可见在以下情况使用 `make_shared` 时要慎重 :

1. **需要自定义Deleter时**.
   1.  `make_shared<>()` 方法没有传入 自定义 Deleter 的接口.
2. 用 `make_shared<Obj>()` 分配的内存中包括对象+CB. 而这块大内存仅当**强, 弱引用计数**都为 $0$ 时才会释放. **虽然对象可以正常析构(Destruction), 但是只要弱引用计数还未归零, 包括对象在内的这块内存就无法归还(Deallocate) 给系统**. 
   1. 上述情况在**对象内存空间占用大 + `weak_ptr` 生命周期长时会有显著影响**.

这些应用情况下, 第一种**对象与CB分别allocate**是更灵活的选择. 不过上述情景比较菜少见, 95% 选择 `make_shared` 会更好.

## 循环引用

这是 `shared_ptr` 面临的一个主要问题. 下面讲的 `weak_ptr` 是一种解决方案.

看以下例子 :

```C++
struct A {
    std::shared_ptr<B> ptrB;				// A 中持有 指向 B 的共享指针
    ~A() { std::cout << "A destroyed\n"; }
};

struct B {
    std::shared_ptr<A> ptrA;				// B 中持有指向 A 的共享指针
    ~B() { std::cout << "B destroyed\n"; }
};

int main() {
    {
        auto a = std::make_shared<A>();
        auto b = std::make_shared<B>();
        
        a->ptrB = b;
        b->ptrA = a; // 此时形成循环引用
    } 
    // 离开作用域, a 和 b 被销毁. 
    // 但是 A 和 B 的析构函数均不会被调用, 因为强引用计数不会归零
    return 0;
}
```

此时只需要将其中一个类的指针对象修改一下, 例如把 `ptrB` 改为 `weak_ptr` 即可.

## 别名构造

这是 `shared_ptr` 拥有的特殊**构造函数**. 来看看其应用场景.

在一个巨大的 `struct Scene` 中 包含许多其他类的子对象 :

```C++
struct Scene{
 	Camera camera;
    Object object;
    //......
}scene;

std::shared_ptr<Scene> scene_ptr(&scene);
```

此时我们想将 `scene` 中的子对象 `camera` 拿出来, 传递给另一个函数使用.

那么我们必须保证 `camera` 在这个过程中, 用`scene_ptr` 管理的 `scene` 不会因为**强引用计数归零** 而被**提前析构**. 

直接用 `shared_ptr<Camera> camera_ptr` 去管理这个子对象`camera` 显然不能阻止 `scene` 的强引用计数归零. 此时就是**别名构造函数**大显身手的时候. 它的函数签名为 :

```C++
template<class T>
shared_ptr(const shared_ptr& r, element_type* ptr);	
```

- `shared_ptr` 是另一个智能指针, 在我们的例子中应该是 `scene_ptr`.
- `ptr` 是**指向子对象首地址**的指针, 这里应该是 `&scene_ptr->camera`.

我们以这种方式去创建一个子对象的智能指针 : 

```C++
std::shared_ptr<Camera> camera_ptr(scene_ptr, &scene_ptr->camera);
```

它的效果是 : **`camera_ptr` 指向子对象期间,  `scene` 的强引用计数加1, 而 `scene.camera` 的强计数不会增加.** 注意此时, 也不会创建属于 `scene.camera` 的 CB.

细究到内存布局上, `camera_ptr` 智能指针中有两个指针 : 一个指向对象首地址, 另一个指向CB首地址. 

- 对象首地址就是 `&scene.camera`
- CB 首地址直接复用 `scene`  的 CB 首地址. 在别名构造的逻辑中自动完成.



# `weak_ptr`

这类智能指针与`shared_ptr` 进行协作, 它作为 `shared_ptr` 对象的观察者, 会影响对象的**弱引用计数**. 

同时, `weak_ptr` 还能被升级成强引用 `shared_ptr`.

为了保持与`shared_ptr` 无缝的协作关系, `weak_ptr` 对象同样也仅**包含两个裸指针**, 且含义与前者完全一致.

1. 指向对象首地址
2. 指向 CB 首地址 (管理其中的弱引用计数).



`weak_ptr` 的初始化方法通常是**绑定到一个 `shared_ptr` 上, 与其协作管理 CB**. 这种方法会使得 `obj_shared_ptr` 管理的对象**弱引用计数 + 1.** 

```C++
shared_ptr<Obj> obj_shared_ptr = make_shared<Obj>();
weak_ptr<Obj> obj_weak_ptr(obj_shared_ptr);				// 常用


weak_ptr<Obj< obj_another_weak_ptr(obj_weak_ptr);		// 拷贝构造
weak_ptr<Obj< obj_another_weak_ptr(move(obj_weak_ptr)); // 移动构造
```

另一种是拷贝构造 / 移动构造, 简单的复制.



`weak_ptr wptr;` 仅作为一个观察者. 它有几种方法去访问对象的状态 :

1. `wptr.expired()` 探活方法 : 判断对象是否还活着. 逻辑就是检查 CB 中 strong count == 0 是否成立.

2. `weak_ptr` 并没有重载 `operator->` 和  `operator*`, 这说明`weak_ptr` 对象**没办法直接访问观察对象的值或调用其方法.** 

   要访问对象, ==**唯一合法**==的方法是调用 `lock()` 方法将 `weak_ptr` **升级**为 `shared_ptr`. 该方法会尝试返回一个被观察对象的 `shared_ptr`. 若对象已析构, 则返回的是 空的 `shared_ptr`. (调用`.get()`方法获取对象指针时返回`nullptr`)

> 为什么升级的方法名叫 `lock()`, 即 "加锁" ? 容易联想到并发编程中的锁. 事实上确实与此有关.
>
> `lock()` 被设计成一个**原子操作**, 完成**强引用计数+1, 构造一个`shared_ptr`**. 
>
> 在并发系统中, 如果我们允许 `weak_ptr` 在探活以后直接用 `*wptr` 访问对象, 就会遇到以下问题 :
>
> 1. 一个线程 A 的`wptr`进行 `expired()` 探活成功后, 被线程调度打断, 切换到另一线程 B
> 2. 另一个线程 B 中`shared_ptr`恰好退出作用域, 销毁了本来是活的对象. 
> 3. 切换回 线程 A , 其`wptr`已经通过了探活, 认为对象仍然存活, 使用 `*wptr`访问对象, 引发内存错误.



