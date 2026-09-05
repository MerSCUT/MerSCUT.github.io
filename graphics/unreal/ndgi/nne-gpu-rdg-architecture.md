# Unreal NNE GPU/RDG 架构笔记

## 1. 核心概念与分层

Unreal NNE（Neural Network Engine）不是单一的推理实现，而是一套公共接口、模型资产格式和可替换 Runtime 后端组成的框架。

最重要的分层是：

```text
模型格式层      ONNX
                  ↓
Unreal 资产层   UNNEModelData
                  ↓
NNE 接口层      CPU / GPU / RDG / NPU 等接口
                  ↓
Runtime 实现层  RDGHlsl / ORT / BasicCPU / IREE 等
                  ↓
执行技术层      CPU Kernel / DirectML / HLSL Compute Shader 等
                  ↓
硬件层          CPU / GPU / NPU
```

其中：

- ONNX 回答“神经网络如何被描述”。
- `UNNEModelData` 是 Unreal 中保存模型来源数据和 Runtime 专用数据的资产容器。
- NNE CPU、GPU、RDG 等接口回答“业务代码如何提交输入、输出和推理任务”。
- ORT、IREE、RDGHlsl 等 Runtime 回答“由谁解释和执行模型”。
- CPU Kernel、DirectML、HLSL Compute Shader 回答“最终使用什么机制进行计算”。

`NNERuntimeRDG` 和 `NNERuntimeORT` 通常是两条并列、可替换的执行路线，不是一个模型先经过 ORT、再经过 RDG。

## 2. 主要源码分布

### 2.1 NNE 核心接口

目录：

```text
Engine/Source/Runtime/NNE
```

主要文件：

- `Public/NNE.h`：NNE 模块入口和 Runtime 查询。
- `Public/NNEModelData.h`：`UNNEModelData` 模型资产。
- `Public/NNETypes.h`：Tensor 数据类型、形状和描述。
- `Public/NNERuntimeCPU.h`：CPU 推理接口。
- `Public/NNERuntimeGPU.h`：底层 GPU/RHI 推理接口。
- `Public/NNERuntimeRDG.h`：Render Dependency Graph 推理接口。

核心模块主要定义公共协议，并不自己实现 `Gemm`、`Conv`、`Relu` 等神经网络算子。

### 2.2 ONNX 导入和编辑器支持

目录：

```text
Engine/Source/Editor/NNEEditor
```

主要文件：

- `Private/NNEEditorModelDataFactory.cpp`：创建 `UNNEModelData` 资产。
- `Private/NNEEditorOnnxFileLoaderHelper.cpp`：读取 ONNX 文件。
- `Private/NNEEditorOnnxModelInspector.cpp`：检查模型、输入输出和相关元数据。

这部分位于 `Source/Editor`，因为 Content Browser 导入、模型检查和资产制作通常不应进入最终游戏包。

### 2.3 RDG/HLSL Runtime

目录：

```text
Engine/Plugins/Experimental/NNERuntimeRDG
```

内部模块职责：

```text
NNERuntimeRDGUtils
    ONNX/模型转换、验证、优化和 Runtime 格式构建

NNERuntimeRDG
    Runtime、Model、ModelInstance、Tensor 和算子调度

NNEHlslShaders
    Unreal Global Shader 声明、参数绑定和 Shader permutation

Shaders
    实际执行计算的 USF/HLSL Shader
```

典型调用关系：

```text
ONNX Gemm 节点
    ↓
NNERuntimeRDGGemm 算子实现
    ↓
NNEHlslShadersGemmCS Shader 封装
    ↓
对应 USF/HLSL
    ↓
GPU Compute Dispatch
```

### 2.4 ONNX Runtime 后端

目录：

```text
Engine/Plugins/NNE/NNERuntimeORT
```

ORT 是 Microsoft ONNX Runtime 的缩写。它是通用神经网络推理引擎，不等于某一种硬件执行方式。

```text
Unreal NNE 接口
    ↓
NNERuntimeORT 适配层
    ↓
Microsoft ONNX Runtime
    ↓
Execution Provider
├── CPU
└── DirectML 等 GPU 后端
```

原版 ONNX Runtime 还可能支持 CUDA、TensorRT 等 Execution Provider，但 Unreal 当前构建是否包含它们取决于平台和插件配置。

ORT GPU 路线与 RDG GPU 路线的区别：

```text
RDG 路线：NNE → NNERuntimeRDG → Unreal Global Shader/RDG → GPU

ORT 路线：NNE → NNERuntimeORT → ONNX Runtime/DirectML → GPU
```

ORT 路线通常不会再经过 `NNERuntimeRDG` 的算子实现、`NNEHlslShaders` 和对应 USF。

### 2.5 IREE 和其他 Runtime

IREE 是 Intermediate Representation Execution Environment。它强调把模型转换成中间表示，经过编译、优化和可能的算子融合后，再生成适合目标设备的执行形式。

```text
神经网络模型
    ↓ 中间表示 IR
编译与图优化
    ↓
目标平台执行形式
    ↓
IREE Runtime
```

Unreal 中的实验性适配器位于：

```text
Engine/Plugins/Experimental/NNERuntimeIREE
```

其他相关模块：

- `NNERuntimeBasicCpu`：另一种 CPU Runtime 实现。
- `NNEDenoiser`：NNE 的上层使用者和实际案例，不是通用 Runtime 后端。

## 3. NNE 接口与 Runtime 实现的关系

“CPU、GPU、RDG”与“ORT、IREE、RDGHlsl”属于不同层级。

### 3.1 接口类型

接口规定调用方式、资源类型和调度语义：

| 接口 | 输入输出存储 | 主要执行位置 | 调度模型 |
|---|---|---|---|
| `INNERuntimeCPU` | CPU 内存 | CPU | 通常为同步式工作流 |
| `INNERuntimeGPU` | RHI GPU Buffer | GPU | 较底层 GPU/RHI 工作流 |
| `INNERuntimeRDG` | RDG Buffer | GPU | 加入 `FRDGBuilder`，由 RDG 调度 |

`INNERuntimeGPU` 与 `INNERuntimeRDG` 都可能在 GPU 上执行，但抽象层不同：前者更接近底层 RHI 资源，后者直接融入 Unreal Render Graph。

当前引擎分支还可能包含 NPU、RunSync 等相关接口，因此 NNE 接口并不永远严格只有 CPU、GPU、RDG 三种。

### 3.2 Runtime 实现

具体插件可以实现一种或多种 NNE 接口。例如：

```text
NNERuntimeRDGHlsl
    → RDG 接口
    → Unreal HLSL Compute Shader

NNERuntimeORTCpu
    → CPU 类接口
    → ONNX Runtime CPU

NNERuntimeORTDml
    → GPU 类接口
    → ONNX Runtime + DirectML

NNERuntimeBasicCpu
    → CPU 类接口
    → 基础 CPU 算子
```

准确的 Runtime 名称和接口支持情况由当前引擎版本、平台、插件设置及编译配置决定。

接口本身只定义虚函数契约；真正执行模型和创建内部资源的是实现接口的具体对象。例如：

```text
IModelInstanceRDG              公共接口
          ▲
          │ implements
FModelInstanceRDGHlsl          具体实现
```

## 4. UNNEModelData 与 Runtime 专用数据

在 Editor 中选择并导入 `.onnx` 文件后，会形成 `UNNEModelData` 资产。但不同 Runtime 通常不是简单地从一份完整数据中挑选几个字段，而是使用模型来源数据生成自己的专用表示。

```text
model.onnx
    ↓ NNEEditor 导入
UNNEModelData
├── 模型来源格式，例如 ONNX
├── 原始模型数据，主要用于编辑器转换
└── Runtime 专用数据或缓存
    ├── RDG Runtime 数据
    ├── ORT Runtime 数据
    └── IREE Runtime 数据
```

NNE 接口规定如何创建模型和提交推理，但不规定每个 Runtime 必须如何存储计算图和权重。

同一份 ONNX 数据可以被转换为：

- RDG Runtime 的算子图、Tensor 索引、Shader 映射和重新布局的权重。
- ORT 可以创建 Session 的模型数据。
- IREE 的编译或运行时数据。

在打包后的程序中，原始 ONNX 数据不一定继续存在；通常只需保留目标平台和目标 Runtime 所需的 Cook 后数据。

## 5. 完整的 RDG GPU 推理过程

以下以 `28 → 64 → 64 → 3` MLP 为例。

### 5.1 导入并形成模型资产

```text
model.onnx
    ↓ NNEEditor
UNNEModelData (.uasset)
```

此时只是模型资产，还没有创建 GPU Buffer，也没有执行推理。

### 5.2 查找具体 RDG Runtime

业务模块通过 NNE 核心注册表获取实现 `INNERuntimeRDG` 的具体 Runtime，概念代码如下：

```cpp
TWeakInterfacePtr<INNERuntimeRDG> Runtime =
	UE::NNE::GetRuntime<INNERuntimeRDG>(TEXT("NNERuntimeRDGHlsl"));
```

调用关系：

```text
业务模块请求 INNERuntimeRDG
    ↓
NNE 核心 Runtime 注册表
    ↓
返回 NNERuntimeRDG 插件注册的具体实现
```

### 5.3 创建 Model

概念代码：

```cpp
TSharedPtr<IModelRDG> Model = Runtime->CreateModelRDG(ModelData);
```

具体 Runtime 从 `UNNEModelData` 取得属于自己的模型数据，反序列化或构造不可变 Model。Model 通常包含：

- 输入输出 Tensor 描述。
- 算子执行顺序。
- Tensor 连接关系。
- 常量和权重。
- Runtime 专用算子对象。

MLP 可能表示为：

```text
Input [N, 28]
    ↓
Gemm 或 MatMul + Add
    ↓
Relu
    ↓
Gemm 或 MatMul + Add
    ↓
Relu
    ↓
Gemm 或 MatMul + Add
    ↓
Output [N, 3]
```

### 5.4 创建 ModelInstance

```cpp
TSharedPtr<IModelInstanceRDG> Instance =
	Model->CreateModelInstanceRDG();
```

Model 与 ModelInstance 分开是为了区分：

- Model：可共享的图、权重、算子定义。
- ModelInstance：某次使用的输入形状、输出形状和执行状态。

同一个 Model 可以拥有多个不同输入形状的 Instance。

### 5.5 设置输入形状

对 Lightmap Atlas 中的 `PixelCount` 个像素：

```text
Input Shape  = [PixelCount, 28]
Output Shape = [PixelCount, 3]
```

设置输入形状后，Runtime 可以：

- 验证输入维度。
- 推导中间 Tensor 和输出 Tensor 形状。
- 计算 Buffer 字节数。
- 确定算子配置和 Shader 参数。

这里传递的是 `FTensorShape` 等元数据，还不是实际特征数据。

### 5.6 创建并绑定输入输出 RDG Buffer

业务模块通过同一个 `FRDGBuilder` 创建或注册输入输出 Buffer：

```cpp
FRDGBufferRef InputBuffer =
	GraphBuilder.CreateBuffer(InputDesc, TEXT("NNE.Input"));

FRDGBufferRef OutputBuffer =
	GraphBuilder.CreateBuffer(OutputDesc, TEXT("NNE.Output"));

FTensorBindingRDG InputBinding;
InputBinding.Buffer = InputBuffer;

FTensorBindingRDG OutputBinding;
OutputBinding.Buffer = OutputBuffer;
```

关系如下：

```text
GraphBuilder.CreateBuffer
    ↓ 创建当前图中的逻辑资源
FRDGBufferRef
    ↓ 放入绑定结构
FTensorBindingRDG
    ↓ 传给 ModelInstance
EnqueueRDG
```

模型的 Tensor 描述定义类型、形状和次序，`FTensorBindingRDG` 提供实际存储数据的 RDG Buffer。

### 5.7 输入数据写入 Buffer

输入可以从 CPU 上传：

```text
CPU 特征
    ↓ RDG Upload
InputBuffer
```

也可以由前置 Compute Pass 直接生成：

```text
Feature Texture/Buffer
    ↓ Feature Sampling Compute Pass
InputBuffer
```

对于大规模 Lightmap 像素，第二种方式能避免 GPU 到 CPU 再返回 GPU的数据往返。

### 5.8 EnqueueRDG 添加推理 Pass

概念代码：

```cpp
Instance->EnqueueRDG(
	GraphBuilder,
	InputBindings,
	OutputBindings);
```

`EnqueueRDG()` 通常不是立即完成 GPU 推理，而是向同一个 `FRDGBuilder` 添加算子 Pass：

```text
InputBuffer
    ↓ Gemm Pass 0
IntermediateBuffer0
    ↓ Relu Pass
IntermediateBuffer1
    ↓ Gemm Pass 1
IntermediateBuffer2
    ↓ Relu Pass
IntermediateBuffer3
    ↓ Gemm Pass 2
OutputBuffer
```

Runtime 内部维护从模型 Tensor 到实际 RDG Buffer 的映射，并为权重和常量准备 GPU 资源。

### 5.9 Shader 选择与执行

`NNERuntimeRDG` 通常不会针对任意 ONNX 在运行时新生成一份 HLSL，而是：

1. 检查 ONNX/NNE 算子是否受支持。
2. 将算子映射到已有的 Runtime 算子实现。
3. 选择预先实现的 Unreal Global Shader 和 Shader permutation。
4. 绑定 SRV、UAV、维度、步长和算子属性。
5. 向 RDG 添加 Compute Pass。

如果模型包含没有对应实现的算子，模型验证或创建会失败；Runtime 不会自动发明新的 Shader。

### 5.10 后续 Lightmap 编码 Pass

NNE 输出通常仍是 Tensor Buffer：

```text
OutputBuffer = [PixelCount, 3] Neural RGB
```

它不会自动成为原生 Lightmap Texture。业务模块需要添加后续编码 Pass：

```text
NNE OutputBuffer
    ↓ Native Lightmap Encoding Compute Pass
    ├── 使用 Scale[0]/Add[0]
    ├── 保留 native lower-half RGB
    └── 写入匹配的 LogL residual
    ↓
PF_R8G8B8A8_UNORM RDG Texture UAV
```

由于前置特征采样、NNE 推理和编码都加入同一个 RDG 图，RDG 可以根据资源读写关系安排顺序并插入必要的状态转换。

### 5.11 执行 RDG 图

当图被执行时，GPU 才真正运行此前声明的工作：

```text
Feature Sampling CS
→ NNE Gemm/Relu CS
→ Lightmap Encoding CS
→ Lightmap Texture
```

如果没有请求 GPU Readback，模型输入、推理结果和 Lightmap 编码可以始终停留在 GPU。

## 6. Buffer 的创建和生命周期

“NNE Runtime 创建中间 Buffer”是一种简写。准确含义是：具体 Runtime 实现通过调用者传入的 `FRDGBuilder` 声明内部逻辑资源。

```text
IModelInstanceRDG             公共接口
          ▲
          │ implements
具体 RDG ModelInstance        插件实现
          ↓
GraphBuilder.CreateBuffer
          ↓
RDG 负责生命周期和物理资源分配
```

职责通常如下：

| 资源 | 谁发起创建或提供 | 谁管理生命周期 |
|---|---|---|
| 模型输入 Buffer | 业务模块 | RDG |
| 模型最终输出 Buffer | 业务模块 | RDG |
| 中间 Tensor Buffer | 具体 NNE Runtime 实现 | RDG |
| 临时算子 Buffer | 具体 NNE Runtime 实现 | RDG |
| 模型权重资源 | Runtime 实现负责准备 | Runtime、RDG 或 RHI |
| 实际物理显存 | RDG/RHI 决定 | RDG/RHI |

Runtime 调用 `GraphBuilder.CreateBuffer()` 首先创建的是逻辑 RDG 资源描述；RDG 在编译和执行计算图时决定资源生命周期、别名复用以及实际物理资源分配。

接口本身不执行创建操作。业务代码调用接口虚函数后，具体实现才使用 `FRDGBuilder` 创建资源和添加 Pass。

## 7. 简要结论

1. NNE 核心是接口和资产框架，不是某个具体推理引擎。
2. ORT、IREE、RDGHlsl 是可替换的 Runtime 技术路线。
3. RDG Runtime 不会任意生成新的 Shader，而是把受支持的模型算子映射到已有 Shader，并将其组织成 RDG Pass。
4. 输入和最终输出 Buffer 通常由业务代码提供；中间 Buffer 由具体 Runtime 实现通过同一个 `FRDGBuilder` 声明。
5. RDG Pass 边界用于表达资源依赖，不应仅为减少 Pass 数量而把多个相关 Dispatch 隐藏在一个 Pass 内。
6. 对 NNE Compute 推理而言，真正有价值的性能优化通常是减少 Dispatch、中间 Tensor 和显存往返。
7. NDGI 路线中，NNE 应只负责 MLP Decoder；特征采样、native Lightmap 编码和纹理绑定继续由自定义 RDG/渲染代码负责。
