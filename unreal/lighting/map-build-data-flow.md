# 前言

在 GPULM 后处理函数 ApplyFinishedLightmapsToWorld 的 `EncodeTextures` 完成 Atlas 打包, 编码等工作, 最终的结果被存放到一个叫 MapBuildDataRegistry 的仓库. 该仓库与 Level 对应.

# Lightmap 引用声明和绑定

```cpp
UMapBuildDataRegistry* Registry = StorageLevel->GetOrCreateMapBuildData();

Registry->SetupLightmapResourceClusters();
```

在 Setup Lightmap Resource Clusters 函数中, GPULM 完成了一系列资源的声明和引用绑定, 并将数据上传到 GPU Uniform Buffer 中.

注 : 在前序流程中, Registry 已经存储了实际的 Lightmap 数据.

原始 Lightmap 会经过如下流程 :

- 进入 `GetClusterInput()`, 获取每个 Map 对应的 Input 信息,
- 初始化 Clusters 数组, 将对应元素绑定到 Mesh 数据中的 Cluster 指针
- 对每个 Cluster 进行资源初始化 `BeginInitResource(&Cluster);`

`UMapBuildDataRegistry` : 持有 Level 需要使用的所有光照烘焙**数据引用**. 内部的关键成员包括 : 

- `TMap<Guid,FMeshBuildData>` : 存放 Level 中所有 Meshes (用 Guid 映射对应) 的 光照贴图数据. 其中会保存所属的 Atlas 族以及坐标变换等数据. 同时, 会用一个 `FLightmapResourceCluster` 引用下面数组的某一个元素.
- `TArray<FLightmapResourceCluster>` : 是 Registry 拥有的 Lightmap Shader 绑定集合, 用于让多个 Mesh 共享同一组 Lightmap 纹理 和 Uniform Buffer. 关键成员包括 :
  - `FLightmapClusterResourceInput` : Input 是一整套资源的引用, 这些信息会被用于生成其它成员
  - `FUniformBufferRHIRef` : 



# 实际 RHI Texture 绑定

关注以下函数 

```cpp
GetLightmapClusterResourceParameters()
```

这里是将前一步准备的数据提取, 绑定到 RHI 的核心逻辑, 也是 NDGI 的核心侵入点之一.

