## About `AActor::BeginPlay`

每一个 Actor 实例以及其派生对象, 都包括一个 BeginPlay 成员函数. 从名字容易看出它是在引擎初始化程序时执行的. 下面是部分代码.

```C++
void AActor::BeginPlay()
{
	TRACE_OBJECT_LIFETIME_BEGIN(this);
    
	SetLifeSpan( InitialLifeSpan );
	RegisterAllActorTickFunctions(true, false); // Components are done below.

	TInlineComponentArray<UActorComponent*> Components;
	GetComponents(Components);

	for (UActorComponent* Component : Components)
	{
		if (Component->IsRegistered() && !Component->HasBegunPlay())
		{
			Component->RegisterAllComponentTickFunctions(true);
			Component->BeginPlay();
			ensureMsgf(Component->HasBegunPlay(), TEXT("Failed to route BeginPlay (%s)"), *Component->GetFullName());
		}
	}

	if (GetAutoDestroyWhenFinished())
	{
		if (UWorld* MyWorld = GetWorld())
		{
			if (UAutoDestroySubsystem* AutoDestroySys = MyWorld->GetSubsystem<UAutoDestroySubsystem>())
			{
				AutoDestroySys->RegisterActor(this);
			}			
		}
	}
	ReceiveBeginPlay();
	ActorHasBegunPlay = EActorBeginPlayState::HasBegunPlay;
}
```

从源码我们可以看到一些信息 :

- `Tick` 函数是一个**十分关键的函数**. 它是每一帧运行一次的函数.
- 一个 Actor 执行 BeginPlay 的逻辑中, 会将其**所有挂载的组件也 BeginPlay**. (包括将它们的 Tick 函数注册)
-  `GetAutoDestroyWhenFinished()` 的部分与引擎的**自动销毁子系统**有关系.
  - 当属性被设置为自动销毁时, 该 Actor 会将自己注册到 World 的自动销毁子系统中, 由系统统一回收.
- `ReceiveBeginPlay();` 是 C++ 与蓝图连接的桥梁. 它会**触发蓝图系统**中的**Event BeginPlay**.
- `ActorHasBegunPlay = EActorBeginPlayState::HasBegunPlay;` 将 Actor 的状态机属性设置为完成 BeginPlay.

当继承 `AActor` 类时, 要求重写 `BeginPlay()`. 而重写的函数中第一行要求必须是 `Super::BeginPlay()`. (Super 是 UE 定义的类型别名, 等价于父类)

##  About `AActor::Tick()`

`Tick` 函数中的内容是 **每帧更新** 的逻辑.

蓝图中, 也有一个 `Event Tick` 节点. 