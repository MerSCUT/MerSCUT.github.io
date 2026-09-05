# RHI 源码剖析

`FRHIResource` 是最底层抽象的资源. 

> **图片待补充**：`image-20260730111735211.png`

类中有这样一个枚举类型说明资源具体所属的种类

> **图片待补充**：`image-20260730112010977.png`

可以是一个 Buffer, Texture, Sampler, Shader 等等. 由此可以看出 RHI 的设计中, 如何规划 GPU 资源的类别.

> **图片待补充**：`image-20260730112217946.png`

在 JetBrains Rider 中可以查看其所有派生类

> **图片待补充**：`image-20260730113356932.png`

在这里可以看到, 所有的派生类可以被划分为几个部分. 以 `FRHI*` 开头的类是跨平台的通用类. 它们直接继承自 `FRHIResource`. 再下面的就与具体图形 API 关联了, 它们实现 `FRHI` 中所定义的通用接口.

以 `FRHIUniformBuffer` 为例. 它实际上只定义了一些通用的接口, 而没有真正调用 图形 API

> **图片待补充**：`image-20260730113933650.png`

再次查看派生类

> **图片待补充**：`image-20260730114019037.png`

这就是 API 专用的 UniformBuffer 实现了. 跳转到 `FVulkanUniformBuffer`, 就可以看见熟悉的 Vulkan API 了

> **图片待补充**：`image-20260730114214435.png`

UE 仍然对 Vulkan 的语法做了一些封装. 例如 `FVulkanDevice` 中的 (部分) 成员实际上包括 :

> **图片待补充**：`image-20260730114353547.png`

除了最基础的 `VkDevice` 以外, UE 还在 VulkanRHI 中定义了许多 `*Manager`, 用来管理 Device 派生的各种资源.

---

回到 `FvulkanUniformBuffer` :

> **图片待补充**：`image-20260730114812422.png`

- `Device` 指向了一个 UE 封装的 VulkanDevice 类.
- `Allocation` 是属于 VulkanRHI 中的类,用于管理 GPU 的内存. 对应到 Vulkan 中也即 `VkDeviceMemory`. 
- `Usage` 如类型所述, 描述该 UniformBuffer 的用途.
