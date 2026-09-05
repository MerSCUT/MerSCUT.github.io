# UE Lightmaps 数据流向分析

## Raw Data

经过 GPU Lightmass 计算后, 在函数 `ApplyFinishedLightmapsToWorld` ( 下称为 `AFLTW` ) 中会解压缩, 并按照<u>逐物体</u> ( 目前仅有 StaticMesh, 记为 $O$ )拼接为完整的贴图 :

- `IncidentLighting` : RGB 是三色光强, A 是有效位标记 (`A >= 0.0f` 有效) , 下称 `IL`, 数学符号简记 $I_O$
- `LuminanceSH` : RGB 是 1 阶球谐函数的系数, A 是 0 阶球谐函数系数 除以 一个常数. 下称  `LSH`, 数学符号记为 $L_O$

上述量都依赖于物体 $O$. 都是二维 4-channels 图.

## Light Sample Data

随后在 `ConvertToLightSample` 函数中被转移到 `LightSampleData`.

```C++
FLightSampleData ConvertToLightSample(FLinearColor IncidentLighting, FLinearColor LuminanceSH)
```



首先确定返回值 `FLightSampleData` 的数据结构, 它包含以下 4 个关键成员.

```C++
/**
* Coefficients[0] stores the normalized average color,
* Coefficients[1] stores the maximum color component in each lightmap basis direction,
* and Coefficients[2] stores the simple lightmap which is colored incident lighting along the vertex normal.
*/
float Coefficients[NUM_STORED_LIGHTMAP_COEF][3];

float SkyOcclusion[3];

float AOMaterialMask;

/** True if this sample maps to a valid point on a triangle.  This is only meaningful for texture lightmaps. */
bool bIsMapped;
```

`Coefficients` 和 `bIsMapped` 是需要着重关注的. 全局常量 `NUM_STORED_LIGHTMAP_COEF = 4`. 

下面看 ConvertToLightSample 如何填充这些字段 :

1. `bIsMapped` : 

   ```C++
   // 返回目标
   FLightSampleData Sample;	
   
   Sample.bIsMapped = IncidentLighting.A >= 0.0f;
   ```

   可见该字段做了一个二值转换 : `mask = true` 等价 `A >= 0.0f`.

2. 过渡容器 `DirLuma` : LSH 的 A 通道包括了一些亮度信息, RGB 通道近似表示主光源方向.

   ```C++
   float DirLuma[4];
   // Revert diffuse conv done in LightmapEncoding.ush for preview to get actual luma
   DirLuma[0] = LuminanceSH.A / 0.282095f; 
   
   DirLuma[1] = LuminanceSH.R;	// Keep the rest as we need to diffuse conv them anyway
   DirLuma[2] = LuminanceSH.G;	// Keep the rest as we need to diffuse conv them anyway
   DirLuma[3] = LuminanceSH.B;	// Keep the rest as we need to diffuse conv them anyway
   ```

   LSH 中的值有特殊含义, A 通道与环境亮度相关, 也是做归一化等缩放操作的重要依据. 因此它派生出了两个量 :

   ```C++
   float DirScale = 1.0f / FMath::Max(0.0001f, DirLuma[0]);
   float ColorScale = DirLuma[0];
   ```

3. `Coefficients` ( 下记 $C$ ): 开始填写核心数据.

   ```C++
   Sample.Coefficients[0][0] = ColorScale * IncidentLighting.R * IncidentLighting.R;
   Sample.Coefficients[0][1] = ColorScale * IncidentLighting.G * IncidentLighting.G;
   Sample.Coefficients[0][2] = ColorScale * IncidentLighting.B * IncidentLighting.B;
   
   Sample.Coefficients[1][0] = DirLuma[1] * DirScale;
   Sample.Coefficients[1][1] = DirLuma[2] * DirScale;
   Sample.Coefficients[1][2] = DirLuma[3] * DirScale;
   ```

   $C[0]$ 和 $C[1]$ 都是一个 Vec3f 向量, 前者存储亮度信息, 后者存储主方向光源信息. 代码清晰展示了它们的计算方法.

   而后续的 $C[2]$ 和 $C[3]$ 是他们的简单复制. 不同的是, 索引 0 和 1 存储的是 高质量 (HQ) 系数, 后者是 低质量 (LQ) 系数. 它们的区别会在后续量化, 压缩的过程中体现, 这里的 LightSample 将它们全部存储.

现在, `LightSampleData` 就是一张未经压缩的原始光照贴图数据了. 当然它也与物体 $O$ 有关. 记为 $S_O$.



## Quantized Lightmap Data

接下来是对 LightSampleData 执行量化操作. 目标的数据结构为 `FQuantizedLightmapData` :

```C++
// 量化后的光照数据
struct FLightMapCoefficients
{
	uint8 Coverage;
	uint8 Coefficients[NUM_STORED_LIGHTMAP_COEF][4];
	uint8 SkyOcclusion[4];
	uint8 AOMaterialMask;
};
struct FQuantizedLightmapData
{
    /** Width or a 2D lightmap, or number of samples for a 1D lightmap */
    uint32 SizeX;

    /** Height of a 2D lightmap */
    uint32 SizeY;

    /** The quantized coefficients */
    TArray<FLightMapCoefficients> Data;

    /** The scale to apply to the quantized coefficients when expanding */
    float Scale[NUM_STORED_LIGHTMAP_COEF][4];

    /** Bias value to apply to the coefficients. */
    float Add[NUM_STORED_LIGHTMAP_COEF][4];

    /** The GUIDs of lights which this light-map stores. */
    TArray<FGuid> LightGuids;

    bool bHasSkyShadowing;
    
    FQuantizedLightmapData() :
    	bHasSkyShadowing(false)
    {}
    ENGINE_API bool HasNonZeroData() const;
    ENGINE_API void Serialize(FArchive& Ar);
};
```

需要注意 :

- `SizeX, SizeY` : 贴图的长和宽
- `TArray<FLightMapCoefficients> Data;` : 量化后的光照贴图数据
  - `FLightMapCoefficients` 中的 `Coverage` 和 `Coefficient` 分别对应 上一节 LightSample 中的 `bIsMapped` 和 `Coefficient`.
- `Scale` 和 `Add` :  存储了将原始 Light Sample Data 量化为 Quantized Lightmap Data 所使用的变换参数.

具体的量化从 `AFLTW` 中下面的入口处开始 :

```C++
QuantizeLightSamples(
    LightSampleData, 				// 原始光照贴图数据
    QuantizedLightmapData->Data, 	// 这里以及之后的参数都是引用
    QuantizedLightmapData->Scale, 
    QuantizedLightmapData->Add
);
```

函数头如下

```C++
void QuantizeLightSamples(
	TArray<FLightSampleData> InLightSamples,
	TArray<FLightMapCoefficients>& OutLightSamples,
	float OutMultiply[NUM_STORED_LIGHTMAP_COEF][4],
	float OutAdd[NUM_STORED_LIGHTMAP_COEF][4]
)
```

代码很长, 具体的行为概括以下 : 

- 定义一些经验值; 将 LightSample 中的内容按 4096 / Task 分块,  
- 第 1 次 并行执行 : 对每一个分块,
  - 分配并初始化 `MinCoefficient`, `MaxCoefficient` ( 简记 MinC 和 MaxC )系数向量.
  - 遍历 InLightSamples 中每一个 texel 的数据 ( 记为 `SourceSample` ) :
    - 若当前 texel 无效则跳过;
    - 若有效, 将 `SourceSample` 的 RGB 色强转换为 LUVW 格式存储 ( 接口为 `GetLUVW`, HQ/LQ 的区别在转换过程中可以体现 ). 将代表光强的 `L` 转为对数存储 `LogL`. [适应人眼对暗部细节敏感的特点]
      - 按照 HQ 和 LQ, 对 SourceSample[0] 和 [2] 执行不同的 `U, V, W` 预处理 ;
      - 对 SourceSample [1] 和 [3] 会执行相同的处理, 并且会
    - 更新 MinC, MaxC 系数向量的值. 它们存储当前 Tile 中所有 texel 的 `LogL, U, V, W` 的最小和最大值. (在 LQ 中`U,V,W` 的位置存储的是 `LogR, LogG, LogB`.)
- 在 CPU 主线程侧, 合并所有分块的 MinC 和 MaxC, 得出全局的 最大最小值. 下面将基于此计算量化的参数
- 定义 `CoefficientMultiply` 和 `CoefficientAdd`, (它们对应量化结构体中的 `Scale` 和 `Add` 字段.) 计算所有 Coefficient, Colorindex 下的 Scale 值和 Add 值 (4 * 4) , 并针对 SH [0,1,2,3] 的防除 0 和 SH [1, 3] 常数项作特殊处理 
- 第 2 次并行执行 : 对每一个分块
  - HQ : 具体应用前面算出的 Scale 和 Add 对 SourceSample 中的数据作线性变换, 变换到范围 $[0,1]$ 中, 取根号 (U, V , W; LogL 不开根) 后 * 255, 并截断尾部精度为 `uint8`.
  - LQ : 重复 HQ 的大部分操作, 在一些细节上有区别.



## Lightmap Allocation

量化完后, 函数进行了安全性检查和验证, 最终于 `FLightMap2D::AllocateLightMap` 中创建光照贴图.

```C++
MeshBuildData.LightMap = FLightMap2D::AllocateLightMap(
    Registry,
    QuantizedLightmapData,
    bUseVirtualTextures ? ShadowMaps : EmptyShadowMapData,
    StaticMeshComponent->Bounds,
    PaddingType,
    LMF_Streamed
);
```

注意到这一步还可能与 `ShadowMaps` 有关系. 通过 `bUseVirtualTextures` 来控制.

结果将以 `TRefCountPtr<FLightMap2D>` 类型返回到 MeshBuildData.LightMap 中.

函数头为 :

```C++
TRefCountPtr<FLightMap2D> FLightMap2D::AllocateLightMap(UObject* LightMapOuter,
	FQuantizedLightmapData*& SourceQuantizedData,
	const TMap<ULightComponent*, FShadowMapData2D*>& SourceShadowMapData,
	const FBoxSphereBounds& Bounds, ELightMapPaddingType InPaddingType, ELightMapFlags InLightmapFlags)
```

声明中第一个参数是 `LightMapOuter`, 而外部传入的是 `UMapBuildDataRegistry* Registry`, 表示当前场景 Scene 的静态构建数据仓库, 用来存储被 UE 直接使用/采样的 静态 Atlas. 它在后续的逻辑中会作为<u>资源 Outer</u>.



### AllocateLightMap 宏观行为

该函数先创建返回目标.

```C++
TRefCountPtr<FLightMap2D> LightMap = TRefCountPtr<FLightMap2D>(new FLightMap2D());
```



再创建一个 Allocation 对象 `FLightMapAllcation`

```C++
TUniquePtr<FLightMapAllocation> Allocation = MakeUnique<FLightMapAllocation>();
```



随后, 所有的信息都被转移到对象中 (后面的小节会具体说明其中重要的字段) :

> **图片待补充**：`image-20260626195530440.png`

> **图片待补充**：`image-20260626195747136.png`

后续的代码几乎都在填充 Allocation 中的字段. 最后, 将信息完备的 Allocation 加入到 AllocationGroup 中 :

```C++
AllocationGroup.Allocations.Add(MoveTemp(Allocation));
```

准备好 AllocationGroup, 将其加入待处理队列 `PendingLightMaps` 中 :

```C++
PendingLightMaps.Add(MoveTemp(AllocationGroup));
```

其中的数据会在后面的 `EncodeTextures` 中被打包, 压缩. 



函数将在一开始创建的 `LightMap`返回给调用者. 

而在准备数据的过程中,  同一个 LightMap 被挂载到了 Allocation 上 :`Allocation->LightMap = LightMap`, 一起送入 AllocationGroup 和 PendingLightMaps. 

它最后会用来承载贴图烘焙结果.

### FLightMap2D

`FLightMap2D` 继承自 `FLightMap`, 父类包括如下关键成员 :

```C++
public:
	/** The GUIDs of lights which this light-map stores. */
	TArray<FGuid> LightGuids;
protected:
	/** Indicates whether the lightmap is being used for directional or simple lighting. */
	bool bAllowHighQualityLightMaps;

private:
	int32 NumRefs;		// 引用计数
```

该类自身囊括许多数据结构.

> **图片待补充**：`image-20260626192147346.png`

值得关注的是 : 

- `Textures[2]` : 指向 `ULightMapTexture2D`  的指针 ; 还有许多类型的纹理指针.
- `ScaleVectors`

---

### FLightMapAllocation / FLightMapAllocationGroup

细看 `FLihgtMapAllocation`, 其中包括许多重要的信息. 

- 它决定了当前的量化贴图将被映射到 Textures Atlas 中的位置. 

- 它会与下文的 AllocationGroup 结合使用.

> **图片待补充**：`image-20260626193847788.png`

注意 :

- `OffsetX, OffsetY` 定义了在 Atlas 中的左上角 texel 坐标, `TotalSizeX, Y` 则是长和宽 (texel)
- `MappedRect` 是 2D 整数值矩阵. 注意它是用来<u>框定**当前贴图 (不是 Atlas!)** 的一个矩形区域</u>, 这个矩行区域会被映射到 Atlas 中. 但具体映射到哪里是通过 前面的 `Offset` 和 `TotalSize` 来决定的. 没有特殊需求的话, 矩形应该框定整张纹理, 即 `MappedRect.Min.X = MappedRect.Min.Y = 0`, `MappedRect.Max.X = SizeX`, `Mappedrect.Max.Y = SizeY`
  - 成员包括 `Min` 和 `Max`, 表示矩阵左上角和左下角的整数点 (点的索引用 X 和 Y).
- `RawData` 成员用来承载贴图的 Coefficient 数据; `Scale` 和 `Add` 成员用来承载贴图的 Scale 和 Add. 与前面的结果是对应的.

## Encode Textures

前面的所有流程, 都会在 StaticMesh, InstanceGroup 和 Landscape 类物体中进行, 不同的类别在一些处理方式上有细微不同. 

直到所有物体都处理完, 放入了 `PendingLightMaps`后, 整个 AFLTW 终于进入到了最核心的编码操作 :

```C++
FLightMap2D::EncodeTextures(&LightingContext, true, true);
```

函数头如下

```C++
/**
 * Executes all pending light-map encoding requests.
 * @param	bLightingSuccessful	Whether the lighting build was successful or not.
 * @param	bMultithreadedEncode encode textures on different threads ;)
 */
void FLightMap2D::EncodeTextures(const FStaticLightingBuildContext* LightingContext, bool bLightingSuccessful, bool bMultithreadedEncode)

```



### 算法流程

1. 配置信息, 调整后续的打包和压缩策略.
2. 若可裁剪 (`GAllowLightmapCropping`), 则执行 `unmapped texel` 即无效纹素的裁剪. (`CropUnmappedTexels`)
3. **保守估计**所有 AllocationGroup 的总 Texel 数. ( 长宽按4倍向上对齐. )
4. 按照估计的总 Texel 数降序排列 `PendingLightMaps` ( 优先放大纹理 )
5. 遍历 `PendingLightMaps` 中的所有贴图 Group, 让每个 Allocation 的所有权转移到某个大 Textures 中 :
   - 遍历目前已有的纹理 `PendingTextures` , 如果找到了一个纹理 `ExistingTexture`能够放下 目前的 Group, 则直接将其放进去. 跳出循环. 
   - 若没有可以放下的 Texture, 则根据 1. 中的配置信息, 估算目前的 Group 需要多大的矩形区域, 添加一个新纹理. 尺寸是 2 : 1 或者 1 : 1.



### CropUnmappedTexels

修改 `Allocation.MappedRect`, 使其边界不包含无效像素.

### FLightMapPendingTexture::AddElement

该函数的任务是真正将 `PendingTexture` 分配到 Atlas 中的某个特定区域. 底层的算法数据结构是 **二叉分割树**.

内部会分别调用 `FTextureLayout::AddElement` , `FTextureLayout::AddSurfaceInner`.



