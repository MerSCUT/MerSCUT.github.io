# Descriptor 系统

Descriptor（描述符）系统可以理解为：

> Shader 使用 GPU 资源时，Vulkan 用 Descriptor 告诉 Shader：“你声明的这个资源，实际对应哪块 Buffer 或哪张 Image。”

## Descriptor 解决的问题

Shader 通常会定义 UBO : 

```C++
layout(set = 0, binding = 0) uniform UniformBufferObject {
    mat4 model;
    mat4 view;
    mat4 proj;
} ubo;
```

Vulkan 中, CPU 侧准备好 model, view, proj 数据后, 传输到 GPU 的 `Buffer` 中. 

Shader 能看懂 `set = 0, binding = 0`,  对应的是哪个 `VkBuffer` (从中取数据), 依赖的就是 Descriptor 系统.  

不同的 `set = i, binding = j` , 指向哪块显存是如何定义的 ?

## 主要对象

Descriptor 系统中包括 4 个主要对象, 分成两条依赖关系 :

1. `DescriptorSetLayout -> PipelineLayout -> Pipeline`
2. `DescriptorPool -> DescriptorSet -> Buffer / ImageView / Sampler`



### DescriptorSet

一个 DescriptorSet 包含以下概念性结构 :

```
Descriptor Set 0
├── binding 0：Uniform Buffer，供 Vertex Shader 使用
└── binding 1：Combined Image Sampler，供 Fragment Shader 使用
```

这就是 Shader 中, `set = 0, binding = 0` 背后暗藏的内容. 

可以看见, DescriptorSet 的不同 binding 可以绑定到不同的内容 :

- 数据 : Uniform Buffer, 例如 UBO 中的矩阵数据
- Sampler : 纹理/图片采样器
- ImageView

在 Shader 中, 它们被这样接受 :

```
// Vertex Shader
layout(set = 0, binding = 0) uniform UniformBufferObject {
    mat4 model;
    mat4 view;
    mat4 proj;
} ubo;

// Fragment Shader
layout(set = 0, binding = 1) uniform sampler2D texSampler;
```

`set = i, binding = j` 是全局信息. 对应的变量通常都以 `uniform` 修饰.



### DescriptorSetLayout

该对象定义了 `DescriptorSet` 的结构. 在上一节中, 我们拥有以下两个信息需要提前定义

1. `DescriptorSet` 拥有 `2` 个 bindings ;
2. 索引 0 的 Bindings 绑定到某个 UniformBuffer 上, 索引为 1 的 Bindings 绑定到一个 Sampler.

 每一个 `bindings` 的内容, 通过 `DescriptorSetLayoutBinding` 对象定义 :

```C++
// binding = 0
VkDescriptorSetLayoutBinding uboBinding{};
uboBinding.binding = 0;
uboBinding.descriptorType = VK_DESCRIPTOR_TYPE_UNIFORM_BUFFER;
uboBinding.descriptorCount = 1;
uboBinding.stageFlags = VK_SHADER_STAGE_VERTEX_BIT;

// binding = 1
VkDescriptorSetLayoutBinding samplerBinding{};
samplerBinding.binding = 1;
samplerBinding.descriptorType =
    VK_DESCRIPTOR_TYPE_COMBINED_IMAGE_SAMPLER;
samplerBinding.descriptorCount = 1;
samplerBinding.stageFlags = VK_SHADER_STAGE_FRAGMENT_BIT;
```

每一个 binding 都有其定义信息, binding 的个数构成了 Descriptor Set 的总 binding 个数. 

注 : 这里并没有绑定具体的数据资源, 只是定义了接口.

简单理解为在 Shader 中定义了这样一个结构体 (而没有实例化 [绑定资源] )

```C++
struct MaterialResources {
    UniformBuffer* transform;
    Texture* texture;
};
```

所有的 `DescriptorSetLayoutBinding` 会作为创建信息, 传入 `DescriptorSetLayout` 中 :

```C++
std::array<VkDescriptorSetLayoutBinding, 2> bindings{
    uboBinding,
    samplerBinding
};
// CreateInfo
VkDescriptorSetLayoutCreateInfo layoutInfo{};
layoutInfo.sType =
    VK_STRUCTURE_TYPE_DESCRIPTOR_SET_LAYOUT_CREATE_INFO;
layoutInfo.bindingCount = static_cast<uint32_t>(bindings.size());
layoutInfo.pBindings = bindings.data();

VkDescriptorSetLayout descriptorSetLayout = VK_NULL_HANDLE;

if (vkCreateDescriptorSetLayout(
        device,
        &layoutInfo,
        nullptr,
        &descriptorSetLayout) != VK_SUCCESS) {
    throw std::runtime_error("Failed to create descriptor set layout");
}
```

> DescriptorSet 可以理解为 Descriptor 的 "集合". 所以 `Descriptor` 这个概念出现时, 可以把它理解为 `bindings`.

### PipelineLayout

该对象是 GraphicsPipeline 与 DescriptorSetLayout 之间的桥梁. 

DescriptorSetLayout 定义的是一个 `set` 的接口. 而管线允许拥有多个 `DescriptorSet`. `set = i`.

创建 PipelineLayout 的样板代码 如下 :

```C++
VkPipelineLayoutCreateInfo pipelineLayoutInfo{};
pipelineLayoutInfo.sType =
    VK_STRUCTURE_TYPE_PIPELINE_LAYOUT_CREATE_INFO;
pipelineLayoutInfo.setLayoutCount	 = 1;
pipelineLayoutInfo.pSetLayouts = &descriptorSetLayout;

// Create
vkCreatePipelineLayout(
    device,
    &pipelineLayoutInfo,
    nullptr,
    &pipelineLayout);
```

除此以外, PipelineLayout 还接受 Push Constant. 其设计目标是 :

> 高效地向 Shader 传递少量、频繁变化、通常每次 Draw 都可能不同的数据，同时避免创建和管理 Buffer、Descriptor Set。

如果不使用 PushConstant, 将 CPU 侧的参数运输到 GPU 上都要进行一次 `Buffer -> AllocateMemory -> Binding -> copy` 等繁琐操作.

这个问题当然也可以通过将需要传递的小参数统一打包为大参数一次性传递来解决. 

使用 Push Constant 只需要 :

1. 在 Pipelinelayout 定义 Push Constant Range
2. 渲染循环中, 每帧录制命令使用 `vkCmdPushConstants`

总结 :

- Camera 数据每帧变化一次：适合 UBO
- 材质纹理偶尔变化：适合 Descriptor Set
- Model Matrix 每个物体变化：可以使用 Push Constant
- 顶点数据：适合 Vertex Buffer

### DescriptorPool / DescriptorSet

前面所述的对象都是 "接口规范" 的定义, 与实际资源无关.

本节开始叙述的依赖链条与实际资源有关系.

`DescriptorPool` 类似于 `CommandPool`, DescriptorSet 从其中分配与回收.

创建 Pool 时需要说明：

- 最多分配多少个 Descriptor Set
- Pool 中包含多少个 Uniform Buffer Descriptor
- 包含多少个 Sampler Descriptor
- 包含多少个 Storage Buffer Descriptor 等

Descriptor Set 通过 `vkAllocateDescriptorSets` 从 Pool 中分配:

- 核心依赖信息 : `allocateInfo.pSetLayouts = &descriptorSetLayout;`

Pool 根据 Layout 所定义的结构进行分配. 此后, 可以如下理解 

```C++
descriptorSet
├── binding 0 → 尚未写入
└── binding 1 → 尚未写入
```

于是接下来就是 **写入空的 DescriptorSet**. 所有写入的信息准备到若干个 `BufferInfo`/`SamplerInfo` 等等, 传入 `VkWriteDescriptorSet`. 最终通过录制命令 `vkUpdateDescriptorSets` 来实现写入.

```C++
std::array<VkWriteDescriptorSet, 2> writes{};

writes[0].sType = VK_STRUCTURE_TYPE_WRITE_DESCRIPTOR_SET;
writes[0].dstSet = descriptorSet;
writes[0].dstBinding = 0;
writes[0].dstArrayElement = 0;
writes[0].descriptorType = VK_DESCRIPTOR_TYPE_UNIFORM_BUFFER;
writes[0].descriptorCount = 1;
writes[0].pBufferInfo = &bufferInfo;

writes[1].sType = VK_STRUCTURE_TYPE_WRITE_DESCRIPTOR_SET;
writes[1].dstSet = descriptorSet;
writes[1].dstBinding = 1;
writes[1].dstArrayElement = 0;
writes[1].descriptorType =
    VK_DESCRIPTOR_TYPE_COMBINED_IMAGE_SAMPLER;
writes[1].descriptorCount = 1;
writes[1].pImageInfo = &imageInfo;

vkUpdateDescriptorSets(
    device,
    static_cast<uint32_t>(writes.size()),
    writes.data(),
    0,
    nullptr);
```

至此, 我们创建了一个符合 Layout 接口, 且绑定到实际资源上的 DescriptorSet.

### 绘制时多帧执行

实际中, 假设同时使用  2 帧交替绘制, 创建时期会定义两个相同 Layout 的 `std::vector<DescriptorSet> descriptorSets (2)`,  在绘制第 $k$ 帧时, 通过录制命令 :

```C++
vkCmdBindDescriptorSets(
    commandBuffer,
    VK_PIPELINE_BIND_POINT_GRAPHICS,
    pipelineLayout,
    0,
    1,
    &descriptorSets[k],
    0,
    nullptr);
```

将第 $k$ 帧对应的 DescriptorSet 绑定到管线中, Shader 访问时, 就会访问到具体的资源了.





# 同步对象 SYNC Object

在标准的渲染循环流程中, 有两类 Vulkan 对象实现了同步机制 : 

- 两组 `VkSemaphore`：GPU/WSI 与队列 Queue 之间的同步。
- 一组 `VkFence`：GPU 与 CPU 之间的同步。

它们分别解决三个完全不同的问题：

| 同步对象                    | 数量与归属                | 谁 signal                     | 谁 wait        | 解决的问题                        |
| --------------------------- | ------------------------- | ----------------------------- | -------------- | --------------------------------- |
| `imageAvailableSemaphore_`  | 每个 frame slot 一个      | `vkAcquireNextImageKHR` / WSI | Graphics Queue | Swapchain image 何时可以开始写    |
| `renderFinishedSemaphores_` | 每个 swapchain image 一个 | Graphics Queue                | Present Queue  | 渲染何时完成，可以显示            |
| `inFlightFence_`            | 每个 frame slot 一个      | Graphics Queue                | CPU            | 当前 frame slot 何时能被 CPU 重用 |

仅从对象名上看,

-  `Semaphore` 实现的是 GPU-GPU 或 GPU-渲染系统 的同步.
- `Fence` 是 CPU - GPU 同步. (CPU 等待 GPU 触发)

在目前 Vulkan 项目的重构架构中, 

- `imageAvailableSemaphore` 和 `inFlightFence` 定义在 `FrameContext` 中,  是属于一帧的同步对象.
- `renderFinishedSemaphores` 则定义在 `Renderer` 中, 属于一张 Swap Chain `VkImage` .

一次完整的 渲染同步链 :

```
CPU
 │
 │ wait inFlightFence[currentFrame]
 │
 │ 更新 Frame UBO
 │ 重置并录制 CommandBuffer
 │
 ▼
vkAcquireNextImageKHR
 │
 │ signal imageAvailable[currentFrame]
 ▼
Graphics Queue Submit
 │
 │ wait imageAvailable[currentFrame]
 │ 执行渲染命令
 │ signal renderFinished[imageIndex]
 │ signal inFlightFence[currentFrame]
 ▼
Present Queue
 │
 │ wait renderFinished[imageIndex]
 ▼
显示 swapchain image
```



## ImageAvailableSemaphore

这是个 Semaphore, 实现的是 GPU graphics pipeline 与 SwapChain 关联的显示系统的同步.

> 常见误解 : 当 CPU 线程 跑到 `vkAcquireNextImageKHR` 后, 
>
> 并不是
>
> - 阻塞 CPU 直到 SwapChain/显示系统解封, 给出下一张空闲 VkImage. 
>
> 而是 :
>
> - SwapChain 直接向 CPU 提供下一张空闲的图片 Index, 而**不管这张空闲图片是否仍然在被 Present Queue, 显示系统扫描**.
> - CPU 在获取 index 以后立马进入下一步 :  Update Uniform Buffers 与 Record Command Buffer

所以, 这个信号量的同步机制与 CPU 没有关系, 仅仅是由 CPU 创建了这个信号量, 传递给了触发者 vkAcquireNextImageKHR. 而 等待者 是执行 Command 的 GPU Graphics queue, 这个信号量会在 submitInfo 中提供, 告知 Graphics queue 等待它被触发后再执行写入 VkImage 的命令.

再次强调 : `vkAcquireNextImageKHR` 返回 Index 成功不等于 GPU 可以无条件立即写入。真正用于 GPU 队列同步的是它 signal 的 semaphore。

另一个关键点 : GraphicsQueue 事实上不需要在 管线开头就阻塞等待, 可以先执行本次提交 Command 中, 不会写入 VkImage 的命令. 这个设定是怎么告知 GPU 的 ?

- 也是通过 submitInfo :

  ```C++
  waitStage = VK_PIPELINE_STAGE_COLOR_ATTACHMENT_OUTPUT_BIT;
  
  submitInfo.waitSemaphoreCount = 1;
  submitInfo.pWaitSemaphores = &imageAvailable;
  submitInfo.pWaitDstStageMask = &waitStage;
  ```

  waitStage 取值 : `COLOR_ATTACHMENT_OUTPUT`, 就指代 Queue 将渲染结果写入 VkImage 的命令.

  关于 "Stage" 的概念非常复杂, 后续再提.

还有一个问题 : 为什么是 每帧一个  (而不是每 SwapChain Image 一个) ?

- 

## inFlightFence

### Why ?

CPU 什么时候可以重用 frame context 中的信息 (并推送到 GPU 上) ? 

Frame Context 中都是用于 Shader 计算的信息 (Graphics Queue). 项目中缓存了 ` MAX_FRAMES_IN_FLIGHT = 2` 帧的信息.

假设渲染器某个渲染循环中, 即将开始 `currentFrame = 0` 的渲染, 需要注意 : 我们没办法确保 前一次跑到 第 0 帧时所准备的数据已经被 GPU 用完了.

假设现在确实不巧, GPU 仍然在使用前一次准备的数据, 而如果我们不作任何同步, CPU 再次一路高歌猛进, 覆写了 frameContexts[0] 并推送至 GPU 覆盖前一次的数据,  那 GPU 所消费的数据就全乱套了, 会产生未定义结果.

在一帧的渲染流程中, CPU 会与 GPU 异步执行, 完成以下操作

```C++
vkResetCommandBuffer(...)		// 重置 CommandBuffer
UpdateUniformBuffer(...)		// 更新 UBO 数据 到 GPU
RecordCommandBuffer(...)		// 录制新的 Command Buffer
```

不论是 CommandBuffer 还是 UniformBuffer, GPU 正在使用它时都不能被 CPU 覆写这些内容.

CPU 需要用 InFlightFence[0] 来确认, 上一次使用这个 frame slot 的 graphics submission 已完成。 

当 GPU 触发 Fence 时, 当 fence 被 signal 时，表示：

> 这次 `vkQueueSubmit` 提交的所有 Queue 工作已经执行完成。

可以保证 :

- command buffer 不再被 GPU 执行；
- Frame UBO 不再被 GPU 读取；
- graphics submission 已经消费 `imageAvailable`；
- 当前 frame slot 可以安全重用。

### How ?

Fence 归属 FrameContext.

1. 如何告诉 GPU 触发 Fence ?
   - 在 `vkQueueSubmit` 中传入 Fence
2. 如何告诉 CPU 等待 Fence ?
   - 在 `vkWaitForFences` 中传入Fence

注 : Fence 在 signaled 后, 需要手动 Reset. 项目的代码的方案是**尽可能晚 reset**, 直到 `vkQueueSubmit` 前才重置. 

- 不这么做, 在 waitFence 被释放后立刻 reset 会怎样 ?  如果在 QueueSubmit 前的录制指令/数据准备阶段抛出了异常, 在处理异常回来后, 就没有任何线程能够再将 Fence 从 unsignaled -> signaled, CPU 将永远阻塞在 WaitFence.

## renderFinishedSemaphore

### Why ?

该信号量完成的是 Graphics Queue  -> Present Queue 的同步.

原理很简单 : CPU 在提交 vkQueueSubmit 和 vkQueuePresentKHR 时是不会在乎是否同步的. 在展示图片之前, GPU 当然需要把图片先渲染完. 

### How ?

该信号量由 Renderer 持有, 每 SwapChain Image 一个.

1. 触发者是 Graphics Queue :
   1. 在 `vkQueueSubmit` 的 submitInfo 中传入
2. 等待者是 present Queue :
   1. 在 `vkQueuePresentKHR` 的 presentInfo 中传入



## 关于同步机制的推理

假设现在处理第 currentFrame = 0 帧, 流程中几个关键的同步节点是 :

- CPU 等待 inFlightFence[0]
  - 核心 : 如果 CPU 能被放行, 说明前一次处理 第 0 帧 时, 前一次提交的 Graphics Queue 已经执行完毕, 而这又说明, <u>imageAvailble[0] 已经被触发并消费,</u> 所以下一步, CPU 仍然能将该信号量传递给 vkAcquire*...
- CPU 获取 SwapChain Image Index 并<u>提供 imageAvailable[0] 信号量</u>, 设立 显示系统为触发者
- CPU 录制指令, 更新数据. 重置 inFLightFence[0].
- CPU 提交 Graphics Queue 任务, 提交 PresentKHR Queue 任务, 并设立同步信号量 renderFinished[0].
- CPU 进入下一个渲染流程.

同时, GPU 侧异步执行 CPU 的调用.

- GraphicsQueue, PresentQueue, 显示系统在疯狂的异步执行, 由上述两个信号量同步.



### 常见错误 : 为什么 renderFinished 不能设为每帧一个 ?

如果是每帧一个, 我们会因为异步执行得到一些偶发的 **信号量 double signaled 非法行为**. 下面来推演一下

我们假设仍然在两次渲染第 currentFrame = 0 时引发了冲突, 这两次渲染先后分别记为 PhaseA 和 PhaseB. 同时, 由于 SwapChain 可能会在 PhaseA 和 B 分别给出两个不同的 VkImage 索引, 我们记为 $x$ 和 $y$. 在极端情况下, 我们需要 $x \neq y$.

1. 在 PhaseA, CPU 提交了 Graphics and Present 任务, 设置好了两个 Queue 中的同步 renderFinished[0], 以及 PhaseA-GracphisQueue 与 显示系统释放 ($x$) 的同步 imageAvailable[0], PhaseA-GraphicsQueue -> CPU 的 inFlightFence[0].
2. 可能存在这样的情况 : PhaseA-GraphicsQueue 执行完毕 ( !!! 注意, 这蕴含了 imageAvailable[0] 已经被显示系统触发并消费 ), 触发了 renderFinished[0]->signaled, 和 inFlightFence[0] -> signaled. 但是 PhaseA-presentQueue 还没有消费 renderFinished[0], 保持为 signaled. 
3. 在 PhaseB 中, CPU 发现 inFlightFence[0] 已经触发, 于是开始着手准备 PhaseB 的帧数据, 再一次提交 Graphics and Present Queue 指令, 绑定了 PhaseB-GraphicsQueue 与 显示系统释放 ($y$) 的信号量 imageAvailable[0].
4. **不幸的是 PresentQueue 仍然没有消费 renderFinished[0].** 
5. 随后, 显示系统释放了 $y$, 触发了 imageAvailable[0], 使得 PhaseB - Graphics queue 毫无阻碍地完成了工作, 又触发了一个已经被触发的 renderFinished[0], 导致了非法行为.

除了上述情况以外, 其它情况都不会使得 renderFinished[frame] 方案出现问题. 

相信即使推演完这种比较极端的情况, 仍然不好理解为什么 renderFinished[frame] 不安全以及为什么下面的方案可以解决这个问题. 因此现在立刻切换到 renderFinished[ImageIndex] 方案, 看看上面的推演过程导致的问题在什么位置被预防住了.

在更换安全的方案后, renderFinished[] 不应再用 帧编号 0 来索引, 而是使用 $x$ 和 $y$ 索引.

这就是问题的关键, 

我们看看 $x \neq y$ 的情况 : 关注上面推演的第 4 和 第 5 步, 此时它们应该变成

> 4. **不幸的是 PresentQueue 仍然没有消费 renderFinished[ $x$ ].** 
> 5. 显示系统释放了 $y$, 触发了 imageAvailable[0], 使得 PhaseB - Graphics queue 毫无阻碍地完成了工作, 触发了 renderFinished[ $y$ ].

注意到, 此时没有两个 renderFinished[0] 的冲突, 该方案是安全的.

如果 $x = y$, 事实上, 这种情况不可能推演到第 5 步. 因为 PresentQueue 在没有消费 renderFinished[$x$] 的情况下, 第 5 步是不可能出现 "显示系统释放了 $y = x$" 的情况的, 即 PhaseB - GraphicsQueue 无法执行完毕后触发非法行为. 所以在 $x = y$ 成立时, 即使是 renderFinished[frame] 的方案也是安全的, 它不会出现上一步的极端情况. 但是我们没办法保证 SwapChain 总是提供 $x = y$.

