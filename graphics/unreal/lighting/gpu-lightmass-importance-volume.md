# GPU Lightmass 中 Lightmass Importance Volume 的实现逻辑

## 结论

GPU Lightmass 并没有把 Lightmass Importance Volume 当成“内部完整烘焙、外部降低反弹次数”的通用边界。

它的主要作用是控制 **Volumetric Lightmap（VLM）的覆盖范围和空间采样密度**：

- 表面 Lightmap 在 Volume 内外使用相同的采样设置。
- VLM 在 Importance Volume 内、靠近几何表面的地方分配高密度 brick。
- `CombinedImportanceVolume` 内但实际 Importance Volume 外的区域，由低分辨率 brick 覆盖。
- 最终 VLM 数据范围之外不生成 VLM brick。
- Volume 外的几何体仍然能够参与光线追踪，并影响 Volume 内的照明和遮挡。

## 1. 表面 Lightmap 不区分 Volume 内外

在 Full Bake 模式下，GPU Lightmass 会收集全部 `LightmapRenderStates`，按照空间 Morton 顺序排序，然后遍历每张 Lightmap 的全部 mip-0 tile。

相关实现：

- 构建全部 Lightmap 的排序列表：`Source/GPULightmass/Private/LightmapRenderer.cpp:3249`
- Full Bake 遍历全部 Lightmap 及其 X/Y tile：`Source/GPULightmass/Private/LightmapRenderer.cpp:3663`
- 统一的 GI pass 数来自 `GISamples`：`Source/GPULightmass/Private/LightmapRenderer.cpp:221`
- 所有 tile 使用同一个采样上限判断是否收敛：`Source/GPULightmass/Private/LightmapRenderer.cpp:2282`

表面 Lightmap 的实际行为如下：

| 区域或模式 | 行为 |
|---|---|
| Importance Volume 内 | 正常生成全部 Lightmap tile |
| Importance Volume 外 | 同样生成全部 Lightmap tile，不降低采样质量 |
| Bake What You See | 根据屏幕可见或已记录的 tile 调度，与 Importance Volume 无关 |

修改 Importance Volume 时，GPULightmass 只执行：

```cpp
Scene.bNeedsVoxelization = true;
```

见 `Source/GPULightmass/Private/GPULightmass.cpp:299`。这会要求重新构建 VLM 体素化数据，但不会使表面 Lightmap 的 revision 失效。

因此，传统 CPU Lightmass 中“Importance Volume 外降低间接光质量”的理解，不能直接套用到 GPU Lightmass 的表面 Lightmap 路径上。

## 2. Importance Volume 的收集方式

`FScene::GatherImportanceVolumes()` 遍历当前 World 中的 `ALightmassImportanceVolume`，将每个 Volume 转换为组件包围盒：

```cpp
CombinedImportanceVolume += LMIVolume->GetComponentsBoundingBox(true);
ImportanceVolumes.Add(LMIVolume->GetComponentsBoundingBox(true));
```

见 `Source/GPULightmass/Private/Scene/Scene.cpp:149`。

代码保存了两种范围：

- `ImportanceVolumes`：各个 Volume 独立的 AABB，用于决定哪些区域可以被细化。
- `CombinedImportanceVolume`：包围全部 Volume 的总 AABB，用于决定整个 VLM 数据域。

如果存在多个彼此分离的 Volume，它们之间的空白区域仍可能处于 `CombinedImportanceVolume` 内，但不属于任何单独的 `ImportanceVolumes`。

## 3. 没有 Importance Volume 时的处理

如果没有有效的 Importance Volume，代码会生成一个自动范围，见 `Source/GPULightmass/Private/Scene/Scene.cpp:164`：

1. 收集所有 `bCastShadow` 几何体的 world bounds。
2. 如果场景范围不大，则使用场景 bounds，并通过 `AutomaticImportanceVolumeExpandBy` 扩张。
3. 如果场景过大，则发出性能警告。
4. 对过大的自动范围进行限制，避免生成极其庞大的 VLM。

所以没有手动放置 Importance Volume 并不等于不生成 VLM，而是使用自动合成的范围。

## 4. VLM 总体数据范围

VLM 的最小坐标来自：

```cpp
VolumeMin = Scene->CombinedImportanceVolume.Min;
```

然后使用 `VolumetricLightmapDetailCellSize` 计算细节网格，并将尺寸向上补齐到顶层 brick 的整数倍，见 `Source/GPULightmass/Private/VolumetricLightmap.cpp:151`。

关键常量为：

```cpp
const int32 BrickSize = 4;
const int32 MaxRefinementLevels = 3;
```

见 `Source/GPULightmass/Private/VolumetricLightmap.cpp:94`。

因此一个最高层级 brick 在每个轴向上覆盖 64 个最细 detail cell：

```text
4 ^ 3 = 64
```

实际 `FinalVolume` 的最大边界可能略微超过 `CombinedImportanceVolume.Max`，这是网格尺寸向上对齐造成的。

## 5. Importance Volume 内的自适应细化

### 5.1 标记 Importance Volume

每个 Importance Volume 会先写入最细层的体素纹理。写入前，边界会向外扩张两个 `TargetDetailCellSize`：

```cpp
Parameters->ImportanceVolumeMin = ImportanceVolume.Min - 2 * TargetDetailCellSize;
Parameters->ImportanceVolumeMax = ImportanceVolume.Max + 2 * TargetDetailCellSize;
```

见 `Source/GPULightmass/Private/VolumetricLightmap.cpp:246`。

Shader 将范围内的体素标记为：

```cpp
BRICK_IN_IMPORTANCE_VOLUME
```

见 `Shaders/Private/BrickAllocationManagement.usf:29`。

这意味着 Volume 边界并不是完全硬切换的，细化候选区在边界外还有两个 detail cell 的缓冲带。

### 5.2 只在几何表面附近分配最细 brick

随后，GPU Lightmass 只提交与任意 Importance Volume 相交的以下几何体进行 VLM 体素化：

- Static Mesh
- Instance Group
- Landscape

见 `Source/GPULightmass/Private/VolumetricLightmap.cpp:294`。

像素着色器在几何表面附近检查 `BRICK_IN_IMPORTANCE_VOLUME` 标记，并将其改成 `BRICK_ALLOCATED`。同时向周围扩张一个 cell，以减少不同 mip 拼接时丢失细节的问题：

```cpp
if (VoxelizeVolume[...] == BRICK_IN_IMPORTANCE_VOLUME)
{
	VoxelizeVolume[...] = BRICK_ALLOCATED;
}
```

见 `Shaders/Private/VolumetricLightmapVoxelization.usf:183`。

没有靠近几何表面的 `BRICK_IN_IMPORTANCE_VOLUME` 体素，在 brick 计数阶段会重新变为未分配状态：

```cpp
if (VoxelizeVolume[VoxelPos] == BRICK_IN_IMPORTANCE_VOLUME)
{
	VoxelizeVolume[VoxelPos] = BRICK_NOT_ALLOCATED;
}
```

见 `Shaders/Private/BrickAllocationManagement.usf:63`。

因此，即使位于 Importance Volume 内，也不是所有空间都使用最高采样密度；最细 brick 主要集中在几何表面附近。

## 6. Volume 外的粗层覆盖

最细体素会继续生成两个较粗的 mip，见 `Shaders/Private/BrickAllocationManagement.usf:42`：

- 中间 mip：只要子层中存在细化 brick，就分配父 brick。
- 最高 mip：通过 `bIsHighestMip == 1` 无条件覆盖整个 VLM 网格。

brick 分配完成后，代码按照最高 mip 到最细 mip 的顺序写入 indirection texture；更细的 brick 会覆盖先前写入的粗层映射，见 `Source/GPULightmass/Private/VolumetricLightmap.cpp:596`。

最终空间分配可以概括为：

```text
Importance Volume 内、靠近几何表面
    -> 最细 brick

Importance Volume 内、远离几何表面
    -> 通常使用粗 brick

多个 Importance Volume 之间、但位于 Combined Bounds 内
    -> 使用最高 mip 的粗 brick

最终 VLM Bounds 外
    -> 不生成 VLM brick
```

由 `BrickSize = 4` 和三层 refinement 可以推得各层的大致 cell 间距：

| VLM 层级 | cell 采样间距 |
|---|---:|
| mip 0，细层 | `DetailCellSize` |
| mip 1，中层 | `4 × DetailCellSize` |
| mip 2，粗层 | `16 × DetailCellSize` |

## 7. 内外差异来自空间密度，而不是单个 brick 的采样质量

VLM 的统一 pass 数为：

```cpp
NumTotalPassesToRender =
	GISamples * VolumetricLightmapQualityMultiplier;
```

如果启用 Irradiance Cache，还会加上 `IrradianceCacheQuality`。见 `Source/GPULightmass/Private/VolumetricLightmap.cpp:97`。

每个已分配 brick 都包含 `4 × 4 × 4 = 64` 个 cell，并以相同的 pass 数执行路径追踪，见 `Source/GPULightmass/Private/VolumetricLightmap.cpp:843`。

因此：

- 细 brick 和粗 brick 的单 brick 采样设置相同。
- 质量差异来自一个 brick 所覆盖的实际世界空间大小不同。
- Volume 内靠近表面的区域，单位空间拥有更多 brick 和更多 VLM cell。
- Volume 外的粗层区域，单位空间的 sample 数明显更少。

## 8. Volume 外几何仍参与光线追踪

Importance Volume 只控制 VLM brick 的分配位置，不会将 Volume 外几何从光线追踪场景中剔除。

`FCachedRayTracingSceneData::SetupFromSceneRenderState()` 会遍历全部场景 Static Mesh 和 Instance Group，Landscape 也会被加入 ray tracing scene。这里没有进行 Importance Volume 相交测试：

- `Source/GPULightmass/Private/LightmapRenderer.cpp:648`
- `Source/GPULightmass/Private/LightmapRenderer.cpp:975`

VLM 路径追踪使用这个完整的场景 TLAS：

```cpp
PassParameters->TLAS = Scene->RayTracingSceneSRV;
```

见 `Source/GPULightmass/Private/VolumetricLightmap.cpp:778`。

因此，Volume 外的几何体仍然可以：

- 遮挡射向 Volume 内样本的光线；
- 参与间接反射；
- 影响 Volume 内的 VLM 结果。

区别只在于这些几何体附近不会因为它们自身位于 Volume 外而获得细 VLM brick。

## 9. Stationary Directional Light 的附带影响

Stationary Directional Light 的静态阴影深度图会使用 `CombinedImportanceVolume` 变换到光源空间后的 bounds：

```cpp
const FBox LightSpaceImportanceBounds =
	Scene.CombinedImportanceVolume.TransformBy(WorldToLight);
```

阴影图分辨率和追踪范围都由该 bounds 决定，见 `Source/GPULightmass/Private/Scene/Lights.cpp:395`。

因此 Importance Volume 也会限制 Stationary Directional Light 的 static shadow depth map 覆盖范围。

Spot、Point 和 Rect Light 的对应实现主要使用各自的锥体范围或 `AttenuationRadius`，不能统一理解为都由 Importance Volume 限定。

## 总结

GPU Lightmass 对 Lightmass Importance Volume 内外区域的处理可以归纳为：

1. **表面 Lightmap**：Volume 内外没有特殊质量分级，全部有效 Lightmap tile 使用统一采样设置。
2. **Volumetric Lightmap 总范围**：由所有 Volume 的合并 AABB 决定，并向网格边界对齐。
3. **Volume 内靠近几何表面**：分配最细 VLM brick，空间采样密度最高。
4. **Volume 内远离表面，以及多个 Volume 之间的空白区域**：由较粗 VLM brick 覆盖。
5. **最终 VLM Bounds 外**：不分配 VLM brick。
6. **Volume 外几何**：仍然保留在完整的光线追踪场景中，能够影响 Volume 内的光照结果。
7. **Stationary Directional Light**：其静态阴影深度图范围会受到 `CombinedImportanceVolume` 限制。

最准确的一句话描述是：

> GPU Lightmass 使用 Importance Volume 控制 VLM 的空间覆盖与自适应采样密度，而不是控制表面 Lightmap 的内外烘焙质量。
