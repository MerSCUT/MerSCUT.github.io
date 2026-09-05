# 前言

本文件会成为后续其它文件的导航文件 Archive，只记录大而泛的内容模块。

- 下面的第一级列表列点中是计划整理的知识板块
  - 在模块下的列表分点指出的是笔记中应当涉及的板块
  - 或者互联网上已有的文档和博客
  - 或者一个索引到其它 .md 自整理笔记的链接



# UE 源码剖析

- GPULM - GPUlightmass 的烘焙结果构成, 后处理流程, 拦截点, 导出.

- GPULM - EncodeTextures() 调用中, 将 Atlas 打包成 Atlas 的方法

- GPULM - UE 中 Lightmap 数据量化的数学原理

- Lightmap - UE 类体系剖析 - 如何组织 Lightmap 资源的存储和管理.
  - Registry
  - RenderResource
  - Shader

- RHI - UE 抽象的跨平台图形管线剖析
  - RHI - `FRHIUniformBuffer` 剖析

- LCI - Light Cached Interface 概述 : Mesh 与 Lightmap 的桥梁.

- Lightmap - 渲染管线如何使用烘焙好的 Lightmap 数据

- Lightmap - Shader 资源绑定 : 
  - PrecomputedLightingBufferParameter : 将 Mesh Lightmap UV 转换成真实的 UV
  - IndirectLightingCacheParameter : 服务于 Movable 物体, 未构建光照时的预览.
  - LgithmapResourceCluster : 真正绑定到纹理数据, 是预期的拦截点.

- Lightmap Policy - 一个 Mesh 应该如何使用 Lightmap
  - 常见的 Lightmap Policy  : `LMP_NO_LIGHTMAP` / `LMP_HQ_LIGHTMAP`
  - Policy 需要描述的内容 :
    - ShouldCompilePermutation
    - ModifyCompilationEnvironment
    - GetVertexShaderBindings
    - GetPixelShaderBindings

- Render Pass :  UE 如何组织 RenderPass 完成场景的渲染 ?
  - 什么是 Base Pass ? 还有什么 Pass ? (MeshPassProcessor.h)

- .usf / .ush : 文件后缀含义, 以及与 .cpp 文件的联系
  - `IMPLEMENT_MATERIAL_SHADER_TYPE` 宏的作用

- Lightmap UV01 : Engine 如何将 Atlas 的 Coefficients[0, 1] 打包进同一张纹理的上下半图 ?
  - 应在 LightMap.cpp 的 EncodeCoefficientTexture 完成.

- RenderDoc : 如何与 UE 引擎结合使用 ?
  - [官方文档-使用RenderDoc分析虚幻引擎画面](https://dev.epicgames.com/documentation/unreal-engine/using-renderdoc-with-unreal-engine?lang=zh-CN)
  - [How Unreal Renders a Frame - UE4](https://interplayoflight.wordpress.com/2017/10/25/how-unreal-renders-a-frame/)
  - Event Browser 屏蔽指令过滤 : `$action() -Copy -Clear -ResolveQueryData -ExecuteCommandLists`
  - Event Browser 中的部分 Marker 标记可以在源码中追踪 `RDG_EVENT_SCOPE` 宏后面的标记名称.

- Render Resource Viewer
  - [官方文档-渲染资源查看器](https://dev.epicgames.com/documentation/unreal-engine/render-resource-viewer-in-unreal-engine)

- Render Dependencies Graph (RDG) : UE 中的 Render Graph
  - Render Graph 是对 GPU Pass 和资源依赖关系的声明式描述系统，它根据“谁读谁、谁写谁”自动完成排序、资源生命周期管理、显存复用和同步屏障插入。

- Compute Shader GPU 编程模型 : 类似 CUDA 的 线程组概念.

- Compute Shader : 如何在 UE 中添加一个自己的 Compute Shader ? (实战经验篇)
  - [官方文档-在虚幻引擎中添加全局着色器](https://dev.epicgames.com/documentation/unreal-engine/adding-global-shaders-to-unreal-engine?utm_source=chatgpt.com)
  - [官方文档-]() : 需要注意, RDG 的内容 

- NNE : Neural Network Engine (Beta) 概述
  - [官方文档-NNE](https://dev.epicgames.com/documentation/unreal-engine/neural-network-engine-in-unreal-engine)
  - [GPT源码总结笔记](../ndgi/nne-gpu-rdg-architecture.html)

- Render Pass in UE : UE 如何组织 各种 Render Pass ?
  - [Unreal’s Rendering Passes](https://unrealartoptimization.github.io/book/profiling/passes/#basepass)

- GPU Crash : 如何排查并解决显存爆炸导致的 Crash 的情况 ? 

- Render Thread : UE 的典型线程结构

  - 1Game Thread + 1Main Render Thread + Many Render SubThreads
  - RHI Thread ?
  - `IsInRenderingThread()` 和 `IsInParallelRenderingThread()` 的区别

- HLSL : UE 的 Compute Shader 与 CUDA 核函数的对应关系 ?

  ```
  CUDA：Grid → Block → Thread
  HLSL：Dispatch → Group → Thread
  ```

  - 为什么要设计 XYZ 方向 的线程数 ? X、Y、Z 维度是为了让 GPU 线程布局直接对应数据的维度。

    它们不是 GPU 上真实存在的三维空间，而是一种方便的线程编号方式。

- UE Shader : 如何让一个自定义 Shader 被 UE 识别, 编译 并运行 ?

  - `IMPLEMENT_GLOBAL_SHADER` 宏 : 向 UE 注册 "Shader 类型", 使其能被编译并查找.

- UE 命令行系统 : CVar 和 Console Command

- Texture Streaming : Mip Level 与 驻留 (Resident)

- Reflection Capture : 原理和代码实现

- Unreal Insight : 性能分析案例 (EnableNeural 切换卡顿) 

  - [(待链接) NDGI 性能分析与优化]()

- NNERuntimeTRT : TensorRT For RTX By NVIDIA.

  - [NNERuntimeTRT NVIDIA 博客](https://developer.nvidia.com/blog/speed-up-unreal-engine-nne-inference-with-nvidia-tensorrt-for-rtx-runtime/)

- UE Asset : 如何创建一个自己的资产, 参与 Cook 和 Packaging, 

  - `UFactory`, `FReimportHandler`

- Packaging : 在 UE 中完成 Build, Cook, Package 操作, 将项目转为 .exe 可执行文件

- 关于移动端真机测试 :

  - 

- 如何在 Android Devices 上执行 Time Insight :

  - [Insight On Android Devices 官方文档](https://dev.epicgames.com/documentation/unreal-engine/how-to-use-unreal-insights-to-profile-android-games-for-unreal-engine?application_version=5.5) : 启用 AFS 插件 ; ProjectSettings Enable AFS ; 配置 UECommandLine.txt; UAFT 连接 devices 并推送 CommandLine.txt
  - [Andriod Performance Analyzer (APA) - Download and QuickStart](https://developer.android.com/android-performance-analyzer/quickstart) : 手机端真机 GPU 性能分析工具. 在个人的 Honor Magic5 Pro 上, 可以获得 UE 无法详细获取 GPU 的内容. 操作方法与 Unreal Insight 类似, 可以在 APA 内部启动 Trace.

- Ambient Occlusion 烘焙 : GPULightmass 烘焙结果中的另一张纹理.

- UE Plugins 体系 : 



# UE Gameplay 开发

- UE 与 git 配合注意事项
- 

# 开发技巧

## JetBrains Rider

- 大型 C++ 项目开发中的常用功能 : 书签 / 结构 / 导航到派生类 / 代码跳转 / 常用设置

## Unreal Engine Editor 5.5

- 场景管理操作
- 截图方法
- UE Python Scripts

## 关于 AI 的使用策略

- Prompt / Harness Engineering 在 AI 高度优化的今天还能怎么应用
- 常用的提问技巧 : 阅读代码, 可控代码修改等等.
- 

## NDGI 实践经验

- 神经网络相关模型, 模型的能力 vs 开销 之间的平衡.
- AI + 应用的经验

## Vulkan API

- 
