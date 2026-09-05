## UE, Swarm 和 Lightmass 的关系

UE 和 Swarm 的关系是 **客户端（任务发起者）vs 分布式调度中间件（任务管家）**

UE -> **导出场景数据**交给本地 Swarm Agent 

-> Swarm Coordinator 分发任务 

-> 各个节点的 Swarm Agent 启动 UnrealLightmass.exe 计算 

-> 结果通过 Swarm 返回给 UE 

-> UE 将光照贴图应用到场景。