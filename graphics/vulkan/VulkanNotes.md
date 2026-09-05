### 名词说明

为了构建清晰的对象生命周期, 依赖关系, 本文将通过几个维度来描述对象 :

- 一个 Vulkan 程序被定义为若干个阶段 :
  - PhaseA (init) : 初始化阶段. 创建关键对象, 资源分配, 清理临时辅助对象
  - PhaseB (render) : 渲染阶段. 执行渲染主循环. 每一帧都会执行的任务, 需要处理 Resize 导致的部分对象的重新构建和销毁.
  - PhaseC (destroy) : 资源根据依赖关系被依此销毁
- 对象类型.
  - 资源型 : 包含实际的硬件资源, 以及对应的句柄; 负责资源请求, 分配.
    - GPU 资源
    - CPU 资源
  - 管理型 : 资源消费. 解读资源, 利用资源进行计算.
  - 工具型 : 负责资源的上传/转移

### 资源生命周期

#### 全程存活

整个程序运行期间存活的对象包括 :

```C++
Window
VkInstance
VkSurfaceKHR
VkPhysicalDevice handle
VkDevice
VkQueue handles
```

这些内容在 init 阶段生成. 除非程序关闭重启, 否则一般不会 reconstruct.

除了 `Window` 窗口对象依赖于具体的系统和平台以外, 其他对象我们统一封装到 `VulkanContext` 中, 作为项目的全局属性.

`VulkanContext` 负责 `Instance, Surface, Device` 的创建, 以及 `PhysicalDevice, Queue` 句柄的获取.

- 需注意, `Surface` 依赖于 `Instance` 和 `Window`.



#### Renderer 周期

以下对象在 Phase2 中虽然可能帧之间变化, 但不会因为 Resize 而重建.

```C++
Mesh Vertex/Index Buffer
Texture Image/View/Sampler
DescriptorSetLayout
PipelineLayout
FrameContext
```

它们统一由 `Renderer` 来负责管理, 该类主要负责每一帧的图像绘制.