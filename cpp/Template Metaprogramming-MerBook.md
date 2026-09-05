# 模版元编程 TMP

2026-3-24 mergic

模版元编程将**程序运行时的性能开销**转移到**编译期**.

本文默认已经了解 <u>函数模版, 类模版编程及其主要特性.</u>

# 1. SFINAE

标题的含义是 **Substitution Failure is not an Error.** **替换失败不是错误.**

当编译器尝试为一个模版参数寻找匹配的类型时, 某种尝试导致了**非法的代码**,  编译器不会报错, 而是将这种尝试从可选名单上划掉.

以**函数模版**为例. 当你调用一个模板函数时，编译器会经历以下过程：

1. 查找同名的函数模版; 
2. 代入模版参数 `T`.
3. 有几种情况 :
   1. 代码合法, 则加入候选名单
   2. 若非法, 由 SFINAE 机制, 编译器不会报错而停止编译.
4. 当候选名单非空, 则按照**重载规则**选择最优. 若为空, 才会抛出**找不到匹配函数**的错误.



那我们是不是可以**故意**制造一些**替换失败**，来人为**引导编译器选择我们想要的模板**？

这就是一个经典应用工具 `std::enable_if` 的由来. 这**同样是一个模版**, 它接受两个参数

1. `bool` 条件, `true` or `false`.
2. 一个类型 参数`T`, 默认是 `void`.

如果第一个参数为`true`, 则在内部定义一个 `T`. 否则就直接抛出**替换失败**, 让编译器直接跳过目前模版的替换.

`enable_if` 的核心源码很简洁 :

```C++
// 1. 泛型通用模板：默认情况（条件 B 为 false 时匹配）。里面是空的。
template<bool B, class T = void>
struct enable_if {};

// 2. 偏特化版本：明确指出当条件 B 为 true 时匹配。里面定义了 type。
template<class T>
struct enable_if<true, T> { typedef T type; };
```

分析一下这段源码 :

- `enable_if` 是一个`struct`模版.  
- 当传入的模版参数 `B` 为 `true` 时, **编译器替换上述两个模版时都会成功**, 此时会选择**特化程度最高的那个模版**作为最终结果. 所以此时, `enable_if` 中执行了这一句类型别名定义 `typedef T type;` 意思是说, 现在`enable_if` 中,  `type` 这个类型名字就等同于 `T`.
- 若传入的模版参数`B` 为 `false`, 编译器仅能通过第一个模版的替换, 此时`enable_if` **没有 `type` 这个类型名字.** 现在再去访问 `enable_if<...>::type` 就会造成**类型缺失**, 导致编译器判断为**替换失败**.

----

看看下面一个应用实例 :

```C++
// 模板 1：只接收int (Integral)
template <typename T>
typename std::enable_if<std::is_integral<T>::value, void>::type
process_number(T num) {
    // 处理整数逻辑
}
// 模板 2：只接收float 
template <typename T>
typename std::enable_if<std::is_floating_point<T>::value, void>::type
process_number(T num) {
    // 处理浮点数逻辑
}
```

- 代码中引用了另一个模版类 : `std::is_integral<T>`. 
- 这个类的行为是 : 当 `T` 为 `int` (或者其他可以视为/隐转为 `int` 的类型) 时, 结构体中的`value` 为 `true`. 否则为 `false`. 底层的源码实现**同样是利用模版特化**.
- 初见该例时, 你可能会注意到 `std::enable_if` 的前面还有一个比较突兀的 `typename` 关键字. 使用它的原因是区分**类型名**和**静态成员**.
  - C++ 中, 当你想引用一个类的静态成员时, 格式是 `ClassName::staticMember`. 
  - 上面的代码中, 这种写法完全可以将`type`解释为 `enable_if` 中的**静态成员变量**, 这也是编译器的默认行为. 关键字 `typename` 的作用就是告诉编译器 `type` 不是静态变量, **而是一个类型别名**.
  - 在此之后, 才符合一个函数的定义语法 : `返回类型 process_number(T num)`.

当调用 `process_number(1);` 时, 可以通过 `enable_if` 中的检查, 成功匹配处理整数的逻辑.

---

再看一个复杂写的例子.

我们引入一个新的用来判断分支的模版类 `is_trivially_copyable<T>::value`. 当类型 `T` 的对象是在**内存中连续存储**的类型时 (也称为**可平凡拷贝类型**),  上述`value`值为 `true`, 否则为`false`.

对于那些连续存储的类型, 我们想要复制它们的值可以使用`std::memcpy` 这个效率十分高的方法. 而对于 `T` 中有许多虚函数, 指针跳转的复杂结构, 就只能老实地循环赋值了.

应用模版元编程完成上述逻辑 :

```C++
// Trivially copyable
template <typename T>
typename std::enble_if< std::is_trivially_copyable<T>::value, void>::type
fast_copy(T* dest, const T* src, size_t count){
	std::memcpy(dest, src, count);
}
// non-trivially copyable (注意 enable_if 后有非运算 !...::value)
template<typename T>
typename std::enable_if<!std::is_trivially_copyable<T>::value, void>::type	
fast_copy(T* dest, const T* src, size_t count)
{/* 循环复制的逻辑 */}
```

----

# 2. 编译器分支

前一个例子中, 仅仅是为了实现一次分支就写了一大串非常难读的类型描述符.

自==C++ 17==后, 编译器分支语句`if constexpr`  的出现改善了这个问题. 可以精简如下 : 

```C++
template <typename T>
void fast_copy(T* dest, const T* src, size_t count) {
    if constexpr(std::is_trivially_copyable<T>::value){
        std::memcpy(dest,src,count);
    }
    else{
        // 循环复制逻辑...
    }
}
```

这段代码的效果与前一个例子一致. 当 `if constexpr` 中判断成功时, 编译器会直接丢弃掉 `else` 中的代码, 连**编译过程都不执行**.

再回顾最开始那个 `process_num` 的例子, 它需要限制模版参数为特定的类型 (如 `int` 和 `float`).

 `if constexpr` 可以充当编译器的 `if`, 但无法充当`switch` 的角色. 这使得我们仍然需要用很多个 `enable_if` 来充当过滤器. 

这就是下一节的**约束**将要解决的问题.

# 3. 约束

 ==C++ 20== 引入了 **Concepts (概念 / 约束, 为避免歧义, 下文都用 Concepts 来避免与中文混淆)** 机制以及 `requires` 关键字.

直接看看如何使用 `requires` 来改写前面**处理整数的逻辑**. 这个例子中使用的是 Required Clause (子句)

```C++
// 旧版写法
template <typename T>
typename std::enable_if<std::is_integral<T>::value, void>::type
process_number(T num) {
    // 处理整数逻辑
}

// C++ 20 后支持的写法
template <typename T>
requires(std::is_integral<T>::value)
void process_number(T num){
    // 处理整数逻辑
}
```

事实上还能用 Concepts 进一步简化代码 :

```C++
#include <concepts>		// for std::integral

// 甚至连 requires 关键字都可以省略，直接把概念当作类型名使用！
template <std::integral T>
void process_number(T num) {
    // 处理整数逻辑
}
```

而对于一些复杂的场景, 需要精细地自定义 Concepts 时, 也可以结合 `concept` 和 `requires` 关键字 (此时`requires`的用法是 requires expression) :

```C++
#include <concepts>
#include <string>

// 定义一个概念，要求类型 T 必须有一个名为 name() 的方法，且返回类型与 std::string 相同
template <typename T>
concept HasName = requires(T a) {
    { a.name() } -> std::same_as<std::string>;
};
```

我们先详细解释一下 `concept` 和 `requires` 关键字的含义和用法.

- `concept` 关键字定义 `HasName` 是一个**约束名(词)**.

  - 给定一个类型后, 约束可以视为**在编译器就计算出值的`bool`常量**. 如果一个类型 `T` 符合 `HasName` 定义的条件, 就称符合这个 concept.

  - 定义了 `HasName` 后, 可以直接在函数中使用它.

    ```C++
    void printName(HasName auto const & obj);
    ```

    编译器会在**编译期**检查传入的值是否符合 concept.

- `requires` 关键字有两种用法, 一种叫 requires expression (表达式), 另一种是 requires clause (子句).

- Requires Expression : 定义约束的**动词/谓词逻辑**. 即描述类型检查的逻辑. 通常用作 `concept` 的定义体代码.

  - ==`concept` 和 `requires` 表达式的关系就像是, `int a = 1;` 中 `int` 和 `1` 的关系.==

  - `requires` expression 可以包含以下 4 种类型的检查逻辑 :

    ```C++
    template <typename T>
    concept SmartBuffer = requires(T t) {
        // 1. 简单要求 (Simple Requirement)
        t.clear();               
        // 含义 : 类型 T 包含一个 clear() 方法, 且可以不带参数调用
        
        // 2. 类型要求 (Type Requirement)
        typename T::value_type;  
        // 含义 : 类型 T 中必须定义 `value_type` 类型
        
        // 3. 复合要求 (Compound Requirement)
        { t.size() } noexcept -> std::convertible_to<std::size_t>; 
        // 含义 : 包含了多个检查.
        	// a. 类型 T 必须包含一个 size 方法, 且可以不带参数调用
        	// b. a 中的方法必须声明为 noexcept (不抛出异常)
        	// c. 返回值必须能隐式转换为 std::size_t
        
        // 4. 嵌套要求 (Nested Requirement)
        requires sizeof(T) <= 64; 
        // 含义 : 要求类型 T 下执行的代码能跑通, 且要求表达式计算出的结果为 true
        
        // 注意这里的 requires 是 嵌套用法. 后面跟随的是 常量表达式 constexpr
        // 嵌套用法中, requires (constexpr) 要求常量表达式必须是 true 才算通过编译.
    };
    ```

    

- Requires Clause : 第一个例子中的就是 requires 子句. 它的作用是模版的准入开关. 格式为

  ```C++
  template<typename T>
  requires(std::is_integral<T>::value)
  // 函数模版定义
  ```

  通常紧跟在 `template<typename T>` 后面, 若 `T` 不满足 `requires` 子句的要求, 则编译器直接跳过对该模版的匹配.