# 左值引用, 右值引用, 移动语义

在 C++ 11 以前, 只有一种引用, 也是初学者比较熟知的**左值引用**

```c++
int x = 10;
// lvalue-reference
int &a = x;
```

而 C++ 11 引入了 **右值引用**, 以及延伸出来的**移动**语义. 

其结果是通过**减少不必要的深拷贝 Deep Copy, 极大地提升了程序的性能**.

## 左值与右值

**左值** 是具有"名字"的值, 在内存中具有**固定的内存地址** (所以可以对其使用 取地址运算`&`)

- 例如一般的变量.

**右值** 是程序运行过程中**临时创建的值**, 没有固定地址. 当语句执行完以后立即销毁.

- 例如 调用函数的(非引用)返回值, 整数字面值.



## 左值引用

顾名思义, **左值引用**是**绑定==到==左值上**的引用.

```cpp
int x = 10;
int &a = x;			// x 是左值, 故可以定义左值引用 a

// Error:
int &b = 10;		// 10 是右值, 不能定义其左值引用
```

> 特例 : 若要强行绑定`10` 这种右值的 左值引用, 可以使用**常量左值引用**.
>
> ```cpp
> const int& b = 10;		// 正确
> ```
>
> 但是这样的应用价值不大.

左值引用 的 常见应用在**函数的参数上**.

```cpp
void swap(int &x, int &y)
{
	int temp = x;
	x = y;
	y = temp;
}
```

这是实现**在函数内修改传入参数**的一种方案. 指针可以做到类似的事情, 但是出于指针极高的灵活性, 这种方案并不安全. 且左值引用的方法在代码上更简洁.

另一个常见应用是**类的复制构造函数**.

- 即使没有在类内显示定义复制构造函数, 编译器也会自动生成一个**默认复制构造函数**, 实现**浅拷贝**构造.

```cpp
class myClass{
    myClass(){}; //普通构造函数
    myClass(const myClass& other) {} // 复制构造函数
}
```



## 右值引用

右值引用是**绑定到右值上的引用**.

通常, 右值在执行完产生右值的语句后, 就会被销毁. 例如调用 `x = func(y);`, `func(y)`  会创建一个临时对象 `obj` (右值), 保存返回结果. 然后执行 `x = obj;` . 随后销毁 `obj`.

右值引用的语法如下 :

```cpp
int&& a = 10;
```

经此, 右引 `a` 指向了一个执行完就销毁的右值 `10`. 既然 `10` 在语句执行完毕就会销毁, 暂时保留它也不会影响到后续程序运行. 而保留它的价值在于**移动语义**的实现.

## 移动语义 Move Semantics

在移动语义出来以前, 临时对象与其他变量之间的拷贝都是**深拷贝**, 即在内存中创建一个与变量一模一样的临时副本. 对于一些较大的对象, (例如一张100万像素的图片对象)  复制的开销十分重, **但是临时对象在执行完一条语句后就销毁了**, 这些时空代价使用得非常低效.

**移动语义** (C++ 11) 出现以后, 上述问题被极好地解决了. 观察下面类的"移动构造函数" :

```cpp
class HugeMem {
public:
    int* data;
    int size;

    // 构造函数
    HugeMem(int s) : size(s) {
        data = new int[size]; // 分配一大块内存
        std::cout << "构造: 分配内存" << std::endl;
    }

    // --- 1. 拷贝构造函数 (Copy Constructor) ---
    // 输入是 const 左值引用，保证不修改原对象
    HugeMem(const HugeMem& other) : size(other.size) {
        data = new int[size]; // 【慢】必须重新分配内存
        std::copy(other.data, other.data + size, data); 
        // 【慢】必须逐个复制数据
        std::cout << "拷贝: 深拷贝完成" << std::endl;
    }

    // --- 2. 移动构造函数 (Move Constructor) ---
    // 输入是右值引用 (&&)，表示 other 是个临时对象，马上要挂了
    HugeMem(HugeMem&& other) noexcept {
        // 【快】直接“偷”走它的指针
        this->data = other.data;
        this->size = other.size;

        // 【关键】把原对象的指针置空，防止它析构时删除这块内存
        other.data = nullptr; 
        other.size = 0;
        
        std::cout << "移动: 指针所有权转移" << std::endl;
    }

    // 析构函数
    ~HugeMem() {
        delete[] data; 
    }
};
```

`Hugemem` 类中, 拷贝构造函数 Copy Constructor 与移动构造函数 Move Constructor 的区别在于传入的参数类型. 后者传递的是一个右值引用 `HugeMem&&`, 从其定义解释, 传入的是一个**即将要被销毁的临时对象**.

Move Constuctor 中, `other` 的实际数据没有被移动, 而是其指针 `data` 被"过继"给了新的构造对象. 这是**移动**与**浅拷贝**的区别 : 移动保证了没有 double-free.

在需要两个独立的对象时, 浅拷贝是一个潜在问题, 即 `HugeMem b(a);` 时, `b` 的指针与 `a` 的指针一致, 指向同一块数据, 一物二主, 容易导致 Double-Free 或其他潜在问题. 若 `a` 与 `b` 表示两张独立的图片的话, 这是不合适的, 此时需要**深拷贝**.

而在移动语义下, 我们的假设是, `a` 对象是个右值引用, 其图片数据不再需要使用, 那么我们没必要进行深拷贝, 而是直接浅拷贝指针即可.  

> 注意到 Move Constructor 的声明时添加了 `noexcept` 修饰.
>
> - 含义是 : **开发者向编译器保证该函数调用不会抛出任何异常**. 如果真的发生了异常, 程序直接 `std::terminate()` **终止**, 此时应当已经发生了十分严重的情况才会产生报错. 
> - 应用场景 :  `std::vector` 对象调用`push_back`, 发现容量不够了, 尝试在内存中开辟更大的一块区域, 并将原有数据复制过去. 在复制过程中, 若编译器没有看到移动构造函数后面的 `noexcept`,  那么它只敢降级用拷贝构造函数在 新的区域初始化元素, 因为这样做能在复制过程发生异常而中断时保留原有数据安全.
> - C++ 的黄金准则中, 有一条指出需要将Move构造函数和赋值函数都声明为`noexcept`, 不论代码本身是否需要这个修饰. 

在下面的例子中, Move Constructor 会被调用 :

```cpp
HugeMem createHuge(){
    return HugeMem(10000);
}

int main()
{
    HugeMem c(createHuge());	
    // 调用 createHuge() 生成了临时对象(暂记为 obj) 保存返回对象值
    // 编译器识别到 HugeMem c(obj) 传入了一个右值, 会优先调用移动构造函数, 直接将临时对象所拥有的数据原封不动地"转交"给了 c, 没有发生任何的拷贝.
}
```



## `std::move()` 方法

 `std::move()` 函数也是移动语义下诞生的. 它的一种应用场景如下

```cpp
int main()
{
	HugeMem a(1000);		// a 是左值
    //...
    // 倘若在完成下面函数的传参以后, 不再需要使用 a 了, 那么可以直接将 a 的资源转让出去
    HugeMem b(std::move(a));
}
```

`std::move()` 的作用是将 `a` 从**左值强制转换为右值**. 上面的代码中, `b` 的初始化会调用 Move Constructor, `a` 的资源会直接转让给 `b`. 

此后, 尽管 `a` 还是一个有效的变量, 但是它内部的数据已经被重置, 需要重新赋值才能使用了.

`std::move()` 的恰当使用能极大提升程序的性能. 但也有一些情景是必须使用 `std::move()` 方法的.

1. **Copy构造函数被禁用的类**

   例如 `std::unique_ptr`, 该智能指针**独占**其指向的内存区域. 

   ```cpp
   std::unique_ptr<int> p = std::make_unique<int>(10);
   ```

   考虑以下简单的指针解引用函数 :

   ```cpp
   void func(std::unique_ptr<int> a)
   {
   	std::cout << *a << std::endl;
   }
   ```

   若此时, 在主流程中直接传递 `func(p)` 会引发错误. 因为独占指针 `p` 会复制到形参变量`a` (也是个独占指针)中, 此时 `p`, `a` 同时指向了 `10` 所在的内存单元, 这违背了 `unique_ptr` 的定义. 

   所以, `func(std::move(p))` 是传递`unique_ptr`参数的唯一方法. 此时 `p` 被**转换为右值**, 执行了 `a = std::unique_ptr<int>(p)`, 原来 `p` 的资源 通过 Move Constructor 继承到了 `a`中.

   - `p` 在调用完函数后就已经被置为 `nullptr` 了, 不能直接解引用.
   - 现在 `10` 所在内存单元由 `a` 掌管, 当 `func` 函数执行结束时, `a` 被销毁前会释放`10`的单元, 不会出现内存写漏的情况.

2. **移动赋值函数** : 这是除了移动构造函数以外最常用的情况之一. 它是指在类内基于**移动**语义重载`=`赋值运算符. 应用的语句常见有 `a = std::move(b);`

   ```cpp
   // 重载赋值运算符 operator=
   HugeMem& operator=(HugeMem&& other) noexcept {
       std::cout << "移动赋值: 清理旧资源，接管新资源" << std::endl;
   
       // 0. 自我赋值检测（防止 a = std::move(a) 导致自杀）
       if (this == &other) return *this;
   
       // 1. 释放自己原本占用的内存（非常重要！否则内存泄漏）
       delete[] data;
   
       // 2. 偷走 other 的资源
       data = other.data;
       size = other.size;
   
       // 3. 把 other 置空
       other.data = nullptr;
       other.size = 0;
   
       return *this;
   }
   ```

   

----

## 杂例

标准库中的交换函数 `std::swap` 曾经是基于 复制 Copy 的. 当 C++ 11 引入移动机制后, 更新为基于 移动 Move 的 :

```cpp
void swap(T& a, T& b)
{
	T temp = std::move(a);		// 移动赋值
    a = std::move(b);
    b = std::move(temp);
}
```

