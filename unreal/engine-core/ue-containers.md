# TArray 源码

## AddUninitialized()

这一部分涉及数组容量管理 :

```cpp
FORCEINLINE SizeType AddUninitialized()
{
    CheckInvariants();

    const USizeType OldNum = (USizeType)ArrayNum;
    const USizeType NewNum = OldNum + (USizeType)1;
    // 先更新大小
    ArrayNum = (SizeType)NewNum;				
    
    // 检查是否超容量
    if (NewNum > (USizeType)ArrayMax)
    {
        ResizeGrow((SizeType)OldNum);
    }
    else
    {
        SlackTrackerNumChanged();
    }

    return OldNum;
}
```

`AddUnintialized()` 会添加 1 个有效元素 (`NewNum = OldNum + 1`) 到末尾, 并判断是否需要扩容.

## AllocatorInstance

该成员是`ElementAllocatorType` 类型. 定义上是一个 `isTrue ? A : B` 的结构 (编译期类型推断专用模版)

```cpp
using ElementAllocatorType = std::conditional_t<
    AllocatorType::NeedsElementType,							// 需要元素
    typename AllocatorType::template ForElementType<ElementType>,
    typename AllocatorType::ForAnyElementType
>;
```

`TArray` 中最核心的三个成员就是 `ArrayNum(size), ArrayMax(capacity)` 以及 `AllocatorInstance`. 大部分 TArray 的逻辑都是在管理 `ArrayNum` 和 `ArrayMax`, 具体的内存管理是由 `AllocatorInstance` 完成的.

可见, `AllocatorInstance` 都是 `AllocatorType::ForAnyElementType` 或者 `ForElementType<ElementType>`. 这个类囊括了非常多的内存操作接口. 它定义在 `ContainerAllocationPolicies.h` 中.



## ForAnyElementType

追溯源码, 可看到这是一个**嵌套类**, 定义在 `TSizedHeapAllocator` 类下. (也可以分析出, TArray 中的 AllocatorType 在使用时会被实例化为 Allocator 等分配器类型.) 在末尾可以看见它仅有一个数据成员 `FScriptContainerElement* Data`.

- 继续追溯 `FScriptContainerElement`, 可以发现它是一个空结构体. **后续可以分析为什么需要这样**

该类具有 5 类关键接口 :

- `GetAllocation()` : 返回 `Data` / 获取堆内存首地址
- `ResizeAllocation(...)` : 真正调用 `Alloc` 相关函数, 进行内存分配的接口.
- `CalculateSlackGrow/Shrink/Reserve(...)` : 计算实际容量. 
- `GetInitialCapacity()` : 获取初始容量
- `MoveToEmpty(...)` 







# TQueue 源码

源码位于`I:\UnrealEngine\Engine\Source\Runtime\Core\Public\Containers\Queue.h`

> 如果比较了解多线程编程的话，那你肯定知道多线程中最常用的一个容器就是消息队列，解决的就是生产者-消费者问题。
>
> from https://zhuanlan.zhihu.com/p/367807315

TQueue 在 UE 中主要用于 **多线程同步**数据.  一个 TQueue 有不同级别的**并发访问的安全性**. 其功能如注释所言. 

安全级别越高的同时, 性能也就相应越差.

```cpp
/**
 * Enumerates concurrent queue modes.
 */
enum class EQueueMode
{
    /** Multiple-producers, single-consumer queue. */
    Mpsc,

    /** Single-producer, single-consumer queue. */
    Spsc,

    /** Single-threaded - no guarantees of concurrent safety. */
    SingleThreaded,
};
```

接下来深入到 TQueue 中 :

```cpp
template<typename T, EQueueMode Mode = EQueueMode::Spsc>
class TQueue
```

类注释提到 :

> This template implements an **unbounded non-intrusive queue using <u>a lock-free linked list</u>** that stores copies of the queued items.

> TQueue选择了简单的单链表这样的结构，从链表的结构上就能很好的解决这个问题

其数据成员也能体现上面这一点 :

```cpp
private:
	/** Structure for the internal linked list. */
	struct TNode
	{
		/** Holds a pointer to the next node in the list. */
		TNode* volatile NextNode;
		/** Holds the node's item. */
		FElementType Item;
        
		// Other Member Function;
	};

	/** Holds a pointer to the head of the list. */
	MS_ALIGN(16) TNode* volatile Head GCC_ALIGN(16);

	/** Holds a pointer to the tail of the list. */
	TNode* Tail;
```

> MS_ALIGN 和 GCC_ALIGN 是两种 C++ 编译期不同的内存对齐宏. 
>
> 64 位机器中, 指针占用 8 B. 若不对齐

> `volatile` 关键字是让编译期**不要生成带优化的汇编代码**. 
>
> 保证每次访问都是从内存读取和写入, 这是为了保证**多线程并发安全**的.

TNode::NextNode 不需要对齐, 因为它唯一的使用点是 TQueue 的 构造时 使用的 `new`, 而这个 `new` 对应 UE 的内存池管理 `FMemory`, 已经自动完成了对齐.

## Constructor 和 Destructor

```cpp
/** Default constructor. */
	TQueue()
	{
		Head = Tail = new TNode();	// 有一个默认空节点. 
	}

	/** Destructor. */
	~TQueue()
	{
		while (Tail != nullptr)
		{
			TNode* Node = Tail;
			Tail = Tail->NextNode;

			delete Node;
		}
	}
```

构造函数中, 直接新建了一个默认的空节点. 后面很多地方都不需要直接加入空判断.

## EnQueue 和 DeQueue

Dequeue 相对简单

```cpp
bool Dequeue(FElementType& OutItem)
	{
		TNode* Popped = Tail->NextNode;

		if (Popped == nullptr)
		{
			return false;
		}

		if constexpr (Mode != EQueueMode::SingleThreaded)
		{
			TSAN_AFTER(&Tail->NextNode);
		}

		OutItem = MoveTemp(Popped->Item);

		TNode* OldTail = Tail;
		Tail = Popped;
		Tail->Item = FElementType();
		delete OldTail;

		return true;
	}
```

而 Enqueue 需要根据队列的并发安全级别来进行不同的操作 :

```cpp
bool Enqueue(const FElementType& Item)
	{
		TNode* NewNode = new TNode(Item);

		if (NewNode == nullptr)
		{
			return false;
		}

		TNode* OldHead;

		if constexpr (Mode == EQueueMode::Mpsc)
		{
            OldHead = (TNode*)FPlatformAtomics::InterlockedExchangePtr((void**)&Head, NewNode);
			TSAN_BEFORE(&OldHead->NextNode);		// 可忽视
			FPlatformAtomics::InterlockedExchangePtr((void**)&OldHead->NextNode, NewNode);
		}
		else
		{
			OldHead = Head;
			Head = NewNode;

			if constexpr (Mode == EQueueMode::Spsc)
			{
				TSAN_BEFORE(&OldHead->NextNode);
				FPlatformMisc::MemoryBarrier();
			}

            OldHead->NextNode = NewNode;
		}

		return true;
	}
```

### Mpsc 

`InterlockedExchangePtr` 的详细定义 : 

```cpp
static FORCEINLINE void* InterlockedExchangePtr( void*volatile* Dest, void* Exchange )
	{
		#if !(UE_BUILD_SHIPPING || UE_BUILD_TEST) 
			if (IsAligned(Dest, alignof(void*)) == false)
			{
				HandleAtomicsFailure(TEXT("InterlockedExchangePointer requires Dest pointer to be aligned to %d bytes"), (int)alignof(void*));
			}
		#endif

		return ::_InterlockedExchangePointer(Dest, Exchange);
	}
```

return 后面的函数是 Windows 中的接口, 声明为 :

```cpp
PVOID InterlockedExchangePointer(
	PVOID volatile *Target,
	PVOID 		   Value
)
```

作用是 : 将 `Value` 中的值赋给 `Target` 指向的对象, 并返回旧值. 

- `OldHead = (TNode*)FPlatformAtomics::InterlockedExchangePtr((void**)&Head, NewNode);` 对应下面的 `OldHead = Head; Head = NewNode`.

使用该函数要求 `Target` 对齐到 16 字节, 否则会导致未定义行为. 这是为什么要判断 `IsAligned(Dest, alignof(void*)) == false` 的原因.

该函数保证**原子性**, 且一个线程调用 InterlockedExchangePtr 时会拦截其他相同的调用. 适用于访问**被多线程 shared 的变量.** 

> **图片待补充**：`image-20260625120907557.png`

这部分相当于是两个原子操作. 这种设计的确是可以保证访问安全的.

- 线程 A 和 B 都分别调用第一个`InterlockedExchangePtr`,  Head 会进行两次移动. 而变量 `OldHead` 和 `NewNode` 都依赖于线程上下文, 完全不影响第二次原子赋值操作.

### Spsc

这是双线程 (1消费1生产的情形.) 两个线程一个操作 Head, 一个操作 Tail, 不需要像 Mspc 中的原子操作.

> **图片待补充**：`image-20260625132815808.png`

Spsc 中插入了一个 `FPlatformMisc::MemoryBarrier()`. 前面提到的专栏作者对此的解释限制在了单线程的指令读取顺序上, 有失偏颇.

内存屏障是用于 **在面对 CPU, 编译器优化导致的指令乱序执行的环境下, 保障<u>多线程</u>之间的数据同步**. 

`MemoryBarrier` 之后的`OldHead->NextNode = NewNode;` 是在**公布新节点**. 此时这个节点是对消费者可见的. 内存屏障保证了 : 

- 生产者线程中, 在节点公布给消费者之前, 最开始的 `	TNode* NewNode = new TNode(Item);` 已经将数据准备完毕. (注意在生产者单线程中, `new TNode` 和 `OldHead->NextNode` 的读写是没有数据依赖的, 因为 `new` 先分配内存地址, 再执行构造函数. )

