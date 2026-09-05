# Mer

本科在读，主要关注 **游戏引擎、实时渲染与图形技术**。

目前的学习与实践主要围绕 Unreal Engine、C++、Vulkan、Graphics Pipeline，以及神经网络在实时图形与游戏引擎中的应用展开。

---

## Featured Notes

这里整理了一些持续更新的技术笔记与学习总结。

- [Graphics](./graphics/)  
  实时渲染、Graphics Pipeline、光照、抗锯齿、可见性与 GPU 相关内容。

- [Unreal Engine](./unreal/)  
  UE 渲染架构、RDG、线程模型、资源管理与源码学习记录。

- [C++](./cpp/)  
  C++ 语言机制、并发、内存管理、STL 与工程实践。

- [Systems](./systems/)  
  操作系统、并发同步、底层机制与系统设计相关内容。

- [Algorithms](./algorithms/)  
  算法题、数据结构与面试训练记录。

---

## Projects & Practice

### Neural Lightmap Compression

在 Unreal Engine 中进行 TOD Lightmap 神经网络压缩与运行时重建相关实践，涉及：

- PyTorch 模型训练与压缩方案验证
- Unreal Engine 渲染源码与 Lightmap Pipeline
- NNE / RDG Runtime 集成
- GPU Resource 调度与性能优化
- Android 真机部署与性能分析

### CPU Path Tracing Renderer

使用 C++ 实现的离线 Path Tracing Renderer，主要包括：

- PBR 材质与路径追踪
- BVH 加速结构
- 光线求交与采样
- 渲染性能优化

### Vulkan Rendering Practice

基于 Vulkan 进行现代实时渲染 Pipeline 的学习与实践，关注：

- Vulkan 显式资源管理
- Descriptor / Command Buffer / Synchronization
- Render Pass 与 Pipeline
- 实时渲染架构设计

---

## Current Focus

近期主要关注：

- Unreal Engine 渲染源码
- Graphics Pipeline 与 GPU 执行模型
- Vulkan 与现代图形 API
- Shader 与实时渲染
- 游戏引擎中的多线程与资源管理
- Neural Rendering / Neural Compression

---

## About

我希望从底层机制和工程实践两个角度理解游戏引擎与实时渲染，并持续把学习过程整理成可复用的技术笔记。

GitHub: [MerSCUT](https://github.com/MerSCUT)
