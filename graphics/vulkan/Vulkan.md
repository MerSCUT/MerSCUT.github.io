# Draw a Triangle

画出一个 Triangle 十分困难.  Vulkan 为用户提供了极强的灵活性和显式控制要求.

## Vulkan 对象梳理

### Vulkan 对象创建模版

在讲具体的对象之前, 跟一遍 Vulkan Tutorial 后, 能发现 Vulkan 对象的几种获取模版 :

1. 基于**描述符**显式创建 : 创建绝大多数**逻辑设备层**的Vulkan对象.
   1. 声明并填充一个 以 `CreateInfo` 结尾的结构体, 并详细配置其中的各种参数.
   2. 调用 `vkCreate` 前缀的函数, 传入上述结构体, 以及用于接收结果的句柄指针.
   3. 该类对象通常需要开发者手动调用 `vkDestroy` 来析构.
2. 双重调用枚举/查询 : 获取物理设备中的信息, 例如查询显卡的驱动支持功能, 扩展属性. 这一类对象事实上并不是由我们创建出来的, 而是去查询获得的.
   1. 声明 `uint32_t count` , 调用`vkEnumerate` 或 `vkGet`, 并将存放数据的指针设为 `nullptr`, 保存(物理对象/信息)的数量
   2. 然后在获取数量后, 为其分配内存, 通常使用 `std::vector<...> objects(count);`, 随后再次调用 `vkEnumerate`/`vkGet`, 将存放数据的指针设为 `objects.data()` 来接收最终的数据.
   3. 通常不需要手动析构 `objects`. 它们的生命周期要么与父对象绑定, 要么本就是物理底层存在的对象.
3. Object Pool 分配 : 最典型的是 `CommandBuffer` 和 `CommandPool` . 一般说来, 渲染循环中会被高频创建、重置或数量庞大的轻量级对象都会要求先创建 Pool, 并通过 Pool 分配具体对象.
4. 基于**继承, 缓存**创建 : 复用已有 Pipeline 中的部分子对象, 避免切换管线或上下文所带来的巨大开销.
5. 辅助结构 : 这一类对象通常不是完整的 Vulkan 对象, 而是创建完整对象的辅助结构. 
   1. 只需要创建 `vk...Description` 结构, 并填入对应信息. 它们会在最后传入完整的 Vulkan 对象的信息中. 
      1. 一个例子是 `Attachment` 和 `Subpass` 会以这种方式创建, 并传入 `RenderPass` 的 `CreateInfo` 中.

### Instance

这是基于 Vulkan 的应用程序的**基底**. 

- 创建 : 基于描述符显示创建

Instance 是一个 Vulkan 应用程序实例对象. 后面的所有对象创建的前提都是 Instance.

在该对象中, 你需要在 `CreateInfo` 中告诉 Vulkan, 关于应用程序的信息 (名称, 使用的 Vulkan API 版本), 需要激活的 Extension 和 Layer.

### Surface

Surface 创建在 Instance 对象之上. 

- 创建 : 基于描述符的显式创建. 
- **GLFW 提供了更便捷的创建方式**. `glfwCreateWindowSurface(instance, window, nullptr, &surface)`

它是 Vulkan 对各种 OS 的窗口的抽象.  一个 Vulkan 程序不一定需要将结果显示在窗口中, 所以关于 Surface 的功能都来源于 Khronos Group 扩展, 其 相关 API 都以 `KHR` 结尾.

需要注意, Surface 本身是不包含任何用来存储像素数据的内存的。在屏幕上显示图像的工作还需要 Surface 与后续对象一起进行. 在 `SwapChain` 对象中会有更多讨论.

### Physical Device

Physical Device 描述系统里实际存在的 **GPU 硬件**. 虽然它是通过双重枚举获取的, 但是查询的 API 仍然依赖于 `instance` 对象.

- 创建 : 双重枚举`vkEnumeratePhysicalDevices` 获取.

在获取了所有可用的 GPU 后, 我们会选择其中一个能满足应用程序使用要求的. 这部分逻辑通过 `isDeviceSuitable`来进行. 





物理设备中, 最重要的概念是 QueueFamily. 

> 一个具体的 Physical Device（显卡）内部封装了多种不同功能的计算流水线，也就是队列簇（Queue Family），比如专门负责图形渲染的 Graphics Queue Family，或者专门负责内存传输的 Transfer Queue Family。

因此，`findQueueFamilies` 函数的作用是针对**某一个具体的** Physical Device，查询它内部是否包含我们需要的队列簇（例如同时支持图形渲染和呈现操作），并记录下这些队列簇的索引号。我们会对电脑上的所有物理设备执行这个查询，从而筛选出真正满足我们需求的显卡硬件。



除了最常用的查询 QueueFamily 支持功能以外, 拓展的支持也需要通过双重枚举的方式获取`vkEnumerateDeviceExtensionProperties`, 并逐一检查.

### (Logical) Device

Vulkan 抽象出 Logical Device（逻辑设备）对象 , 来保证应用程序**只能通过 LogicalDevice 提供的接口和上下文, 访问其申请的QueueFamily**.

- 创建 : 基于描述符的显式创建. 

创建 Logical Device 描述符中所依赖的对象是包括 `physicalDevice, QueueFamilyIndices` (后者是自定义结构体, 用以对 QueueFamily 需求的封装. ) 

`CreateInfo` 中还包括 **Device 所支持的 Extension 和 Layer**.



接上讨论 : 我们的应用程序大概率**只需要使用其中一部分QueueFamily的能力**。

通过抽象出 LogicalDevice 的方式, 多个应用程序 (或者一个应用程序的多个操作) 可以**异步使用同一个 PhysicalDevice 的不同 流水线簇 QueueFamily.** 尽可能榨干 GPU 的并发算力.





### Swap Chain

这是关于屏幕显示中最重要的对象之一. 

- 创建 : 基于显式描述符的创建. 依赖于 `Surface` 对象和 `PhysicalDevice` 对象.

总的而言, SwapChain 管理一块 VRAM 显存, 并综合 Surface 和 PhysicalDevice 的能力设置如何管理这块显存. 这块显存存储实际的图像数据, 最后会送到显示器上. 

简单来说,

- `Surface`  **决定画在哪**. (窗口显示在屏幕上的哪)
- `PhysicalDevice` **决定有哪些画画的能力** (支持的画画格式, 画画方式)
- `SwapChain`  **决定怎么画**. (怎么画上窗口)



`SwapChain` **对象真正地在 GPU 显存上分配了空间,** 用于存储屏幕输出的颜色.



首先来看看依赖关系. 交换链依赖于 `Surface` 和 `PhysicalDevice`.

-  `Surface` 封装了不同操作系统的窗口, 其背后的变量是**操作系统**, 体现的是**平台差异**

- `PhysicalDevice` 是物理设备, 其背后封装的变量是 **GPU 硬件**, 体现**硬件差异**.

在构建 `SwapChain` 的 `CreateInfo` 时, 我们需要知道当前系统的 `(Surface, PhysicalDevice)` 组合能支持的**操作**有哪些. 这些都是通过 `vkGetPhysicalDeviceSurfaceCapabilitiesKHR` 等 API 双重查询来获得的.

具体的核心操作有三种 :

- Capabilities : 画框的物理限制. 
  - 目前窗口的宽高分辨率范围 (Extent)
  - SwapChain 中最少/ 最多能拥有的图像数
  - 是否支持画面预旋转（比如手机横竖屏切换 `supportedTransforms`）
  - 是否支持透明窗口（`compositeAlpha`）
- Format : 像素在内存中的排列方式. (`VK_FORMAT_B8G8R8A8_SRGB`)
  - 按照什么格式**排列内存**. 色彩空间 是什么.
- PresentMode : 如何放置画面到窗口上. 以下是几种取值
  - IMMEDIATE : 立即模式, 通常会造成严重撕裂
  - FIFO：传统的 VSync 垂直同步（无撕裂，但如果渲染太慢会导致卡顿）。
  - MAILBOX：三重缓冲（后台有邮箱，画完了就更新邮箱，屏幕每次只拿最新的一封信，既无撕裂又低延迟）。



交换链对象中还包含了许多重要的信息, 部分信息会与上面查询到的三个核心支持操作有关 :

1. 图像规格 : 

   - `minImageFormat` : 交换链中最少的图像数量.

   - `imageFormat` & `imageColorSpace`：像素的颜色格式（如 RGBA）和色彩空间（如 sRGB）。

   - `imageExtent`：图像的分辨率大小（宽和高，通常与窗口分辨率一致）。

2. 图像用途 :

   - `imageUsage` : 用这个图像做什么. 例如 `VK_IMAGE_USAGE_COLOR_ATTACHMENT_BIT` 表示该图像将作为最终的彩色渲染目标

3. Queue 并发模式

   - `imageSharingMode` : 如果程序需要的两个 QueueFamily 落在同一个流水线簇中, 那么可以设置为`EXCLUSIVE` 获取更好的性能. 否则, 需要 `CONCURRENT` (并发) 模式

4. 图像预处理

   - `preTransform` : 图像送往屏幕前, 是否需要进行变换. (手机屏幕的横竖屏旋转适配)
   - `presentMode` : 画面防撕裂策略 (垂直同步, 三缓冲, 或者不处理策略)



### Image / Image View

SwapChain 对象在 VRAM 上分配了显存来存储图像数据. 这块内存可以视为**图像数组**, Vulkan 使用 `VkImage` 去描述其中存储的每一章图像.

- 创建 :  `SwapChain` 被显式创建后, `VkImage` 自动生成. 通过双重查询获得.

纯粹的 `VkImage` 图像并不能被我们所用, **它仅仅代表 VRAM 中的一块 存储** (归 SwapChain 管理的一部分显存).

我们需要一些额外信息去解读这一块图像内存. 

**Vulkan 使用 `VkImageView` 来完成这个任务.** 对于每一个 `VkImage` 对象, 都在此基础上创建 `VkImageView` 对象, 并配置解读方式和接口.

- 创建 : 基于描述符的显式创建. 依赖于 `VkImage` 对象.

创建时配置的信息与 SwapChain 对象的信息有部分交集.



### Attachment

接下来需要创建 `GraphicsPipeline` 对象, 它是**实际的画师** (先这样理解), 是渲染过程中画画的自动流水线机器. 其中包括了许多 着色器编译后代码和固定函数的配置参数. 

除此之外, 另一个更重要的创建信息是 `RenderPass` 对象, 其中又包括多个 `Attachment` 对象和 `Subpass` 对象. 本节先解释 `Attachment` (附件) 的概念.

#### Why Attachment ? 

前面定义了 `SwapChain`, 它在 GPU 上分配了一段存储几张 `VkImage` 的空间 (在双缓冲, 三缓冲下通常是 2 或 3).  并且定义了 `VkImageView` 去描述这段空间, 包括颜色, 格式.  

实际渲染时, 管线需要以某种方式, 从 SwapChain 申请一张空闲的 `VkImage` 来作为渲染结果颜色图的输出. 

在渲染过程中, **管线完全不知道 SwapChain 给了哪一张具体的 VkImage** (它不会知道 得到的 `VkImage` 首地址), 取而代之, 它直接在某个抽象的数据结构上作画. 这个结构就是 `Attachment`, 它的作用就是**"接受得到的`VkImage`".**

这是 Vulkan 的解耦式设计. `Attachment` 像是一个插槽一样, 可以将任何符合的`VkImage`插入其中. Pipeline 只管对着 `Attachment` 画就好了. 底层的逻辑会自动处理**实际写到哪个地址中去**.

Attachment 的定义, 可以使用 `VkAttachmentDescription` 结构体描述. 它包括以下成员

- `format` : 规定了将要进入的`VkImage`的格式要求. 
- `samples` : 采样方式. (后面我们会在其他对象中再次见到这个成员)
- `loadOp`/ `storeOp` : 规定了 `VkImage` **进入附件后的第一个操作/离开附件前的最后一个操作**
  - 例如, 每一帧开始时, 申请一个新的 `VkImage`, 在插入 Attachment 后立刻清屏为背景色, 再开始后续步骤. 
- `stencilLoadOp`/ `stencilStoreOp` : 与上面类似, 不过是作用在 stencil 上的.
- `initalLayout` : 要求在**进入 Attachment 前**, `VkImage` 已经是指定的布局 (仅仅是一个要求, 实现这个要求是由其他对象完成的) .
- `finalLayout` : 要求在**离开 Attachment 前**, `VkImage` 已经是指定的布局. (附注同上一条)

> Shader 中写的 `layout (location = 0) ...` , 其中的 Location = 0 就是附件编号. 由此也能看出, 管线只知道对 `location = 0` 操作, 而不知道具体数据的位置.



### Render Pass

`GraphicsPipeline` 中存储了许多逻辑和配置, 编译好的 Shader, 以及如何进行 Rasterization 的逻辑, 全都会在创建时配置好.

前一小节已经解释, 这些 Shader 和 固定函数 所操作的抽象对象是 Attachment. 那么自然, GraphicsPipeline 被要求定义它需要使用的所有 Attachment. 承载这个 `Attachment` 的数据结构, 就称为 `RenderPass`. (事实上, `RenderPass` 做的远比单纯的 Attachment 数组要更多. 不过我们后面再补充.)

`RenderPass` 是创建 `GraphicsPipeline` 时需要传入的信息之一. 

一个 `RenderPass` 中可以包含多个 Attachment, 是一份管线与硬件之间非常严格的**契约**. 管线会根据这个契约, 配置好底层的硬件资源.

如果传入的 `VkImage` 在**数量(Attachment 的个数)和格式要求(各个Attachment的要求)**上, 没有办法匹配上 `RenderPass` 中定义的所有 Attachment 插槽, 就会导致严重的渲染崩溃.



> 上面的描述显得 Render Pass 依赖于 Graphics Pipeline, 但实际情况完全相反. **一个 Graphics Pipeline 是绑定在一个 Render Pass 上的 (在后面的 Subpass 一章, 我们能看到一个管线其实会绑定在一个 Subpass 上)**. 实际情况中, 我们可以有两个管线同时满足一个 RenderPass 的要求, 而不可能有一个 GraphicsPipeline 同时使用两个 Render Pass.
>
> 实际中, 我们首先定义 RenderPass, 它就是后续所有渲染步骤的基础. 图形管线都在此之上进行操作, 后面讲述的帧缓冲也需要符合 RenderPass 的定义.

### Subpass

Subpass 是 Vulkan 引入的另一个机制. 从名称上看, 它与 Render Pass 关系匪浅. 

我们前面说, Render Pass 除了接受 Attachment 清单以外, 另一个需要接受的对象就是 Subpass 了.

Subpass 所带来的性能提升可以从一个最典型的例子看出 : **延迟渲染**. 这种渲染方式走了两遍管线 , 分别执行了不同的任务 :

1. 几何处理阶段 : 记录每个像素的几何信息 (法线, 颜色等).
2. 光照阶段 : 考虑所有光照, 材质, 纹理等信息 以及前一阶段记录的几何信息, 计算出每个像素的真正颜色.

每个阶段都会从 顶点处理开始, 依次走过 Vertex Shader, Rasterization, Fragment Shader. 

如果不引入 Subpass 机制, 我们需要使用两个 RenderPass, 去完成上面的两个不同的任务. 在 Vulkan 中, 一个 GraphicsPipeline 只能有一个 Renderpass, **而我们不仅需要切换管线, 还需要切换 RenderPass.** 

但这种方式是有优化空间的. 由于第二个阶段的数据是依赖于第一阶段数据的, **在第一个 Render Pass 结束后, GPU 会清空自己内部访问速度非常快的缓存**. 而延迟渲染需要保留第一个阶段的数据, 所以需要将它们转移到 GPU 的全局显存 VRAM 上. 在切换到另一个 Renderpass 后再从 VRAM 上读回, 这一次开销极大的 VRAM 读写是完全不必要的.

而 Subpass 正是为了解决该优化问题而引入的. 一个 Render pass 允许有多个 subpass. 每一个 pass 在**逻辑上是顺序执行的** , 且代表一轮渲染计算管线. 

在 RenderPass 中切换 Subpass 的开销, 可以用 CPU 下在进程中切换线程的开销来类比. Subpass 的切换不再需要将所有缓存数据导入到 VRAM 中, 只需要重新修改需要使用的数据, 并做好状态切换, 就可以无缝继续下一阶段的计算.

> 注 1 : 尽管逻辑上它们是顺序执行的, 但在物理硬件上, 多个 subpass 在同一时刻有可能同时进行, 而这就有可能导致竞态条件等等一系列并发情景下会出现的同步/互斥问题. 
>
> 这也是我们后面需要引入 Subpass Dependencies 对象的原因

> 注 2 : 在桌面级独立显卡中, 即使使用 Subpass, 部分显卡驱动仍然会启动多次 VRAM 读写. 这部分的性能优化在 Tile-Based Rendering 架构上比较明显. 但 Vulkan 是一个跨平台的 API, 我们应当总是在类似的情境下使用 Subpass.



在引入 Subpass 机制后, 我们可以补充细化 Graphics Pipeline 的绑定信息 :

GraphicsPipeline 在创建时, 会**绑定在某个 Renderpass 的某个 Subpass 上.** 

我们仍然以经典的延迟渲染管线为例, 实现一个延迟渲染管线需要提前定义以下对象 :

- 一个 RenderPass : 包含了延迟渲染管线两个计算阶段的所有 Attachment 定义, 以及两个阶段的 Subpass.
- 两个 Subpass : 分别用于几何处理, 光照计算.
- **两个 Graphics Pipeline** : 一个管线用于几何处理, 它绑定到 RenderPass 下的 几何处理 Subpass. 另一个管线用于光照计算, 同前.

在实际的进行延迟渲染时, 我们发出的命令可以用以下伪代码叙述 :

```
启动 Render Pass{
(自动)准备好 Subpass 0 的 Attachment 及其布局切换

绑定到 Graphics Pipeline 0 .
调用 Draw 命令, 执行几何处理管线

调用 NextPass 指令, 指出现在将进入到下一个 Subpass 1, 同时调整 Attachment 及内存布局

绑定到 Graphics Pipeline 1.
调用 Draw 命令, 执行光照计算管线

(自动)根据 finalLayout 的信息调整附件布局
} 结束 Render Pass
```



### Subpass Dependency [TODO]

待补充



### Graphics Pipeline

我们描述了管线中最重要的蓝图结构 : `Render Pass` 以及其下的两个辅助结构 `Subpass` 和 `Attachment`.

前文并没有涉及这些对象在 Vulkan 中的创建范式, 这里补充 :

- Render Pass 创建 : 显式描述符创建, 创建信息的主要成员包括 Subpass 数组和 Attachment 数组.
- Subpass / Attachment 创建 : 它们并不是一个完整的 Vulkan 对象, 可以视为创建 Render Pass 的辅助信息. 
  - Attachment : 通过 `vkAttachmentDescription` 来定义描述结构.
  - Subpass : 通过 `vkSubpassDescription` 来定义描述其结构. 在其中, 还需要使用 `AttachmentRef` 对象来引用当前 Subpass 会使用全局 Attachment 中的哪些附件, 以及这些被使用附件的内存布局是什么.



现在我们可以真正的看看 Graphics Pipeline 的创建信息了 :

1. Shader : 着色器编译后的产生的字节码 (.spv文件), 传入 `ShaderModule` 对象中. 
2. Dynamic-State : 有什么属性或成员在管线执行中是动态可变的 (Dynamic). 
3. Fix-Function State : 配置管线的固定流程的行为. 
   1. 例如 Rasterizer 是否执行背面剔除或深度测试, 是否使用重采样来抗锯齿.
4. Pipeline Layout : 设置管线布局信息, 主要用来向 Shader 中传入 Uniform 变量或其他外部信息.
5. Render Pass : 前面已经详尽叙述.
6. 其他信息, 不过多展开.

搞定 Render Pass 以后, 其他配置都比较简明易懂.



### FrameBuffer

首先需要指明, **这里的 `FrameBuffer` Vulkan 对象并不特指 "颜色缓冲区".** 这可能与部分图形学初学者使用的帧缓冲概念冲突. 

在介绍 Attachment 时, 我们说它是一个插槽, 规定了送进来的 `VkImage` 应当具有什么格式要求.

在实际的渲染循环中, 还需要解决一个问题, **谁来决定如何插入 `VkImage` 到 Attachment 中 ?**

在绘制一个三角形的简单任务中, 我们只需要将 SwapChain 在每一帧分配给我们的 `VkImage` 所对应的视图 `VkImageView` 传入到 Render Pass 定义中唯一的那个 Attachment 中就好.  更复杂一些的情况呢? 比如我们的 Render Pass 定义了两个 Attachment, 一个用来存放颜色图, 另一个用来存放深度图. 我们需要将两个 `ImageView`放入同一个数组中, 送入 Attachment 数组. 

****Vulkan 定义了 `FrameBuffer` 对象, 用来完成将 `ImageView` 插入 `Attachment` 的任务.****

Frame Buffer 也是通过描述符显示创建的. 它的 `CreateInfo` 中, 两个重要的成员就是  **RenderPass 和 attachments 数组.** 一个不正式的定义如下 :

```cpp
VkRenderPass renderPass;
VkImageView ImageView1, ImageView2;
// Create renderPass and ImageView;

VkImageView attachments[] = {
    ImageView1, 
    ImageView2
};

VkFramebufferCreateInfo createInfo{};
createInfo.renderPass = renderPass;
createInfo.attachmentCount = 2;
createInfo.pAttachments = attachments;
// ... 其他信息
// Create FrameBuffer
```

而 `VkImageView` 数组 被命名为 attachments, 这是因为 FrameBuffer **将严格按照其排列顺序** 插入到 RenderPass 中的 attachment. 在这个意义下, `VkImageView` 与 `VkAttachmentDescription` 是一一对应的. 

传入的 `RenderPass` 对象, 会作为 FrameBuffer 的 "出厂设置". 其中的信息可以帮助驱动程序提前准备好 GPU 中的硬件状态 和配置, 使得运行时的效率最大化. 在运行时, 虽然 Vulkan 不会检查, 但是我们必须**确保  FrameBuffer 与 Renderpass 是兼容的(Compatible).** 

Vulkan Tutorial 中创建 Framebuffer 的代码写得较为简单, 从 SwapChain 的每个 ImageView  都创建一个 FrameBuffer 就好了. 但这样的代码可能使读者对它更灵活的用法没有概念.

假设现在我们的渲染过程需要包含两个 Attachment 的 RenderPass, 一个是颜色图, 另一个是辅助计算用的深度图. 

- 颜色图最终需要显示到屏幕上, 所以 FrameBuffer 的第一个附件元素会从 SwapChain 手中申请一张 `VkImage`, 传入其 `VkImageView`. 
- 而辅助计算的深度图可不归 SwapChain 管, 它只负责要被显示的结果图. 因此我们需要手动从 GPU 申请一段空间, 然后在渲染一帧的指令中, 存放 `VkImage` 及其 `VkImageView`, 作为 Framebuffer 的第二个附件元素.

阐述这个例子的原因是, 我觉得这样能更好理解 FrameBuffer 与 SwapChain 各自的任务是什么, 以及它们并没有必然的对应关系. 

如果仅仅是为了画一个三角形, 只需要向 Framebuffer 传入一个最终会被放到 SwapChain 中的颜色图, 那么创建 FrameBuffer 的代码看起来很像是 FrameBuffer 依赖于 SwapChain . 但事实并非如此. 在一些 离屏渲染 off-screen 中, 没有 SwapChain 时, FrameBuffer 仍然有它重要的任务要做.

### 关于图片信息的解耦

我们已经接触过几个与`VkImage` 相关的对象了. 它们在创建时, 对图片的**格式**, 图片的**尺寸**信息有不同的要求.

- `VkImageView`  : 创建需要 `format`, 同时需要传入的 `VkImage` 结构已经包含宽高 (`width`, `height`)信息 (因此不需要重复写).
- `Attachment` : 只需要 `format`, 完全不需要宽高信息.
- `FrameBuffer` : 完整的创建结构体还包含宽高信息, 但没有 `format`.

Vulkan 将图片的尺寸与图片的格式**解耦**了. 核心原因是, **玩家会拖拽改变窗口大小.** 所以, Vulkan 需要将尺寸剥离出图片的具体存储方式, 也剥离出 RenderPass, GraphicsPipeline 等重构开销巨大的对象.

也因此, 如果回看 RenderPass 和 Graphics Pipeline 的创建过程, 本身并没有描述窗口尺寸的代码. 

尺寸只有在 `FrameBuffer` 将 `VkImageView` 递交给 RenderPass 时, 才确定下来. 玩家改变窗口大小时, 只需要重新构建 SwapChain, VkImageView 和 Framebuffer 对象就好了. 这些相比于前面两个, 开销小非常多. 当然, 由于 Graphics Pipeline 中包含 Viewport 的属性, 必须将其设为**Dynamic State**, 否则当玩家拖拽改变 Viewport 时, 管线依然需要销毁后重新构建.



### Command Buffer / Command Pool [TODO]

Vulkan 不允许像 OpenGL 一样, 直接通过 CPU 调用 API 向 GPU 发送命令. 曾经我们使用的命令 `vkCreatexxx` / `vkAllocate...` 都是纯 CPU 行为. 向 GPU 发送命令的函数通常都以 `vkCmd` 开头. (除了某些扩展中的函数)

Vulkan 强制要求使用 `CommandBuffer` 对象来向 GPU  发出命令. 具体流程是 :

1. CPU 创建 `CommandBuffer` 对象. 在此之前我们需要先创建 `CommandPool`, 从中分配 CommandBuffer.
2. 开启 `CommandBuffer` 录制. 随后,  `vkCmd` 命令都会记录在缓冲区中.
3. 录制完成后, 使用 `vkQueueSubmit` 来一次性像某个 QueueFamily 提交所有录制的指令. 



### Buffer / Device Memory

待补充



### Descriptor Set / Descriptor Set Pool

#### PipelineLayout

待补充

#### DescriptorSetLayoutDescription

待补充

#### DescriptorSetLayout  / DescriptorSetLayoutBinding



> 未整理.

首先, Pipeline 在创建时会指定 PipelineLayout, 其中指定了管线(Shader的视角)中会包含哪些全局layout.



然后, 接下来是如何将CPU内存中的数据对接上 PipelineLayout 的问题. 完成对接过程的对象是 DescriptorSet, 它需要从 Pool 中分配.

而描述一个 DescriptorSet 能"承载什么内存格式的数据", 这个工作是由 VkDescriptorSetLayout 来完成的. 它规定了这个描述符集中包含几个 Binding (且这个 Binding 似乎是与 VertexInput 中的Binding 是独立的?), 每个 Binding 中的内容是什么 (有点套娃的感觉, Binding 中的内容是由另一个辅助数据结构 VkDescriptorSetLayoutBinding 来叙述的).

总结一下, DescriptorSet 对象的格式包括多个 Binding. 每个 Binding 都有具体的 Descriptor 描述符信息. (我在看代码时, 发现描述符    uboLayoutBinding.descriptorCount = 1; 的这个字段, 我不太理解. 一个 Binding 居然可以有多个 Descriptor ? 我突然有点想不明白 Descriptor 和 DescriptorSet 的关系了.)



那么至此, 我们完整的定义了 管线一端的 PipelineLayout, 以及输入一端的装配对象 DescriptorSet. 现在要做的, 就是让 DescriptorSet 去绑定到一块具体的内存 (VkBuffer).



绑定的过程, 我们也需要指定清楚, DescriptorSet 中每一个 Binding 需要绑定到哪一块 Buffer.

Buffer 的具体信息会被汇集到 VkWriteDescriptorSet 这个结构体中, 这个结构体会被送入 VkWriteDescriptorSet 结构体, 再传入调用vkUpdateDescriptorSets 真正地将 Descriptor 中的哪些个 Binding 绑定到 具体的 Buffer 上.

> 现在，我们来集中解决让你感到困惑的那个问题：**为什么一个 Binding 可以有多个 Descriptor？Descriptor 和 DescriptorSet 到底是什么关系？**
>
> 我们可以用一个“工具箱”的类比来完美对应这三者的关系：
>
> 1. 🧰 **Descriptor Set (描述符集):** 它是一个总的**工具箱**。
> 2. 🕳️ **Binding (绑定点/插槽):** 它是工具箱里的一个个**专属隔层**（比如 0 号隔层、1 号隔层）。
> 3. 🔧 **Descriptor (描述符):** 它是放在隔层里的**具体工具**（指向一块特定 Buffer 或 Image 的指针）。
>
> 

> **Descriptor 的 Binding 和 Vertex 的 Binding 是完全独立的两个平行宇宙**。它们互不相干，只是恰好借用了同一个英文单词（相当于“插槽”）。