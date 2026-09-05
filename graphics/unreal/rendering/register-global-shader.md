借用 本次功能开发，分析创建一个 Shader，让 UE 识别它，让 UE 运行它的语法需要完成什么工作

## 0. 概述

创建自定义 Shader 并让 UE 能编译和运行，需要两个完成两个部分的工作：

1. 在 .cpp 侧声明, 注册 Shader 类型, 指明 Shader 的各项参数 / 配置 ;
2. 在 .usf 侧写好 Shader 的 HLSL 代码.

## 1. 注册 Shader 类型

一个 Shader 需要在 C++ 中创建一个对应的类型. 最通用的 C++ Shader 类继承自 `FGlobalShader`

```C++
class MyCS : public FGlobalShader
{};
```

这个类负责向 UE 描述：

- Shader 在哪些平台编译 ;
- Shader 参数有哪些 ;
- 对应哪个 `.usf` 文件 ;
- 对应哪个入口函数 ;
- 它是 Compute Shader 还是其他 Shader Stage.

首先要在类中用宏声明 Shader :

```C++
DECLARE_GLOBAL_SHADER(MyCS);
SHADER_USE_PARAMETER_STRUCT(MyCS, FGlobalShader);
```

- `DECLARE_GLOBAL_SHADER` 告诉 UE : `MyCS` 是一个 Global Shader 类型，需要参与 Shader 类型注册、编译和序列化。
- `SHADER_USE_PARAMETER_STRUCT` 则告诉 UE : Shader 中的变量类型参考后续的 `FParameters` 结构体. 第二个参数传入其父类.

`SHADER_USE_PARAMETER_STRUCT` 宏内部会查找对象 `MyCS::FParameters`

接下来, 需要定义 `FParameters` 结构体. 这个名称是 UE 固定的类型名, 不能随意更换为其它标识符.

```C++
BEGIN_SHADER_PARAMETER_STRUCT(FParameters, )
    SHADER_PARAMETER(FIntPoint, TextureExtent)
    SHADER_PARAMETER_RDG_TEXTURE(Texture2D<float4>, SourceLightmap)
    SHADER_PARAMETER_RDG_TEXTURE_UAV(RWTexture2D<float4>, DestinationLightmap)
END_SHADER_PARAMETER_STRUCT()
```

最核心的成员已经完成. 同时, 也可以在类内添加一些可选的函数, 告诉 UE 在哪些平台才需要编译这个 Shader. 

- `ShouldCompilePermutation` : 返回 `true` 则为这个 Platform + Permutation 编译 Shader。否则不编译, 不放入 Shader Map
- `ModifyCompilationEnvironment` : 负责在 __决定需要编译后__, 修改编译环境. 在这里, 它的作用是设置 HLSL 宏 (NDGI_LIGHTMAP_COPY_THREADGRUP_SIZE)

```C++
static bool ShouldCompilePermutation(const FGlobalShaderPermutationParameters& Parameters)
{
    return EnumHasAllFlags(Parameters.Flags, EShaderPermutationFlags::HasEditorOnlyData)
        && IsFeatureLevelSupported(Parameters.Platform, ERHIFeatureLevel::SM5);
}

static void ModifyCompilationEnvironment(
    const FGlobalShaderPermutationParameters& Parameters,
    FShaderCompilerEnvironment& OutEnvironment)
{
    // 调用父类
    FGlobalShader::ModifyCompilationEnvironment(Parameters, OutEnvironment);
	// 自定义 设置 HLSL 宏;
    // NDGI_LIGHTMAP_COPY_THREADGROUP_SIZE <- FComputeShaderUtils::kGolden2DGroupSize
    OutEnvironment.SetDefine(TEXT("NDGI_LIGHTMAP_COPY_THREADGROUP_SIZE"), FComputeShaderUtils::kGolden2DGroupSize);
}
```

该函数是静态方法, 不能声明为虚函数. 所以它实现选项扩展的原理是 __子类声明同名函数, 会隐藏父类实现__. 当访问 `MyCS::ShouldCOmpilePermutation` 时, 

- 若类中有额外定义同名函数, 则正常使用子类版本; 
- 若没有, 则从子类访问该函数会调用父类的版本;

至此, `MyCS` 的任务已经结束.

## 2. 建立 Shader 映射

接下来, 需要在 `MyCS` 类外使用以下宏, 建立 `MyCS -> .usf -> HLSL 入口 -> Stage` 的映射 :

```C++
IMPLEMENT_GLOBAL_SHADER(
    FNDGICopyNativeLightmapCS,
    "/Plugin/GPULightmass/Private/NDGI/NDGILightmapCopy.usf",
    "MainCS",
    SF_Compute);
```

 此宏仅仅建立映射, 让 UE 能够找到 正确的 Shader 文件并编译, 并非执行 Shader.



## 3. Copy Pass

Shader 的注册, 编译工作已经完成. 但是 Shader 代码还无法执行. 为了简单, 我们利用 RDG 来自动管理, 调度 Shader 的执行. 因此我们给出以下函数 : 

```C++
void AddNDGICopyNativeLightmapPass(
    FRDGBuilder& GraphBuilder,
    FRDGTextureRef SourceTexture,
    FRDGTextureRef DestinationTexture)
{
    check(SourceTexture);
    check(DestinationTexture);
    check(SourceTexture->Desc.Extent == DestinationTexture->Desc.Extent);
    check(SourceTexture->Desc.Extent.X > 0 && SourceTexture->Desc.Extent.Y > 0);
    check(EnumHasAnyFlags(DestinationTexture->Desc.Flags, TexCreate_UAV));

    // MergicC: Why: This pass copies the complete physical W x 2H texture at mip 0. RDG owns the SRV-to-UAV transitions, while the later resource
    // manager remains responsible for creating PF_R8G8B8A8, extracting it with SRVGraphics final access, and publishing it only after graph execution.
    const FIntPoint TextureExtent = SourceTexture->Desc.Extent;
    FNDGICopyNativeLightmapCS::FParameters* PassParameters =
       GraphBuilder.AllocParameters<FNDGICopyNativeLightmapCS::FParameters>();
    PassParameters->TextureExtent = TextureExtent;
    PassParameters->SourceLightmap = SourceTexture;
    PassParameters->DestinationLightmap = GraphBuilder.CreateUAV(DestinationTexture);

    TShaderMapRef<FNDGICopyNativeLightmapCS> ComputeShader(GetGlobalShaderMap(GMaxRHIFeatureLevel));
    FComputeShaderUtils::AddPass(
       GraphBuilder,
       RDG_EVENT_NAME("NDGI.CopyNativeLightmap(%dx%d)", TextureExtent.X, TextureExtent.Y),
       ComputeShader,
       PassParameters,
       FComputeShaderUtils::GetGroupCount(TextureExtent, FComputeShaderUtils::kGolden2DGroupSize));
}
```