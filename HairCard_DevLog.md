# HairCard ShadingModel 开发日志

> 引擎版本：Unreal Engine 5.7 (source build)
> ShadingModel ID：MSM_HairCard = 13 / SHADINGMODELID_HAIRCARD = 13
> 光照模型：Kajiya-Kay Dual Specular (R + TRT)
> 渲染路径：Legacy Deferred GBuffer（非 Substrate）

---

## 1. 修改文件清单

| # | 文件（相对 Engine/） | 修改摘要 |
|---|----------------------|----------|
| 1 | Source/Runtime/Engine/Classes/Engine/EngineTypes.h | EMaterialShadingModel 添加 MSM_HairCard；ESubstrateShadingModel 添加 SSM_HairCard |
| 2 | Source/Runtime/Engine/Public/Materials/MaterialExpressionShadingModel.h | ValidEnumValues meta 追加 MSM_HairCard |
| 3 | Source/Runtime/Engine/Private/Materials/HLSLMaterialTranslator.cpp | 添加 MATERIAL_SHADINGMODEL_HAIRCARD 预处理宏 define 块 |
| 4 | Source/Runtime/Engine/Private/Materials/Material.cpp | CustomData0/1 激活条件添加 MSM_HairCard；SSM 映射添加 SSM_HairCard |
| 5 | Source/Runtime/Engine/Private/Materials/MaterialShader.cpp | switch 添加 case MSM_HairCard 调试名称 |
| 6 | Source/Runtime/Engine/Private/Materials/MaterialAttributeDefinitionMap.cpp | Metallic 显示为 Spec Weight、CustomData0 显示为 Secondary Roughness、CustomData1 显示为 Lobe Separation |
| 7 | Source/Runtime/Engine/Private/ShaderCompiler/ShaderGenerationUtil.cpp | 添加 MATERIAL_SHADINGMODEL_HAIRCARD GBuffer slot 分配块 |
| 8 | Source/Runtime/RenderCore/Public/ShaderMaterial.h | FShaderMaterialPropertyDefines 添加 MATERIAL_SHADINGMODEL_HAIRCARD : 1 位域 |
| 9 | Source/Runtime/RenderCore/Private/ShaderMaterialDerivedHelpers.cpp | WRITES_CUSTOMDATA_TO_GBUFFER 条件追加 MATERIAL_SHADINGMODEL_HAIRCARD |
| 10 | Shaders/Private/ShadingCommon.ush | #define SHADINGMODELID_HAIRCARD 13；SHADINGMODELID_NUM 改为 14；调试色 float3(1,0.4,0.7) |
| 11 | Shaders/Private/ShadingModelsMaterial.ush | #if MATERIAL_SHADINGMODEL_HAIRCARD GBuffer 编码块 |
| 12 | Shaders/Private/DeferredShadingCommon.ush | IsSubsurfaceModel/HasCustomGBufferData 添加 HAIRCARD；Mobile Encode/Decode 添加分支 |
| 13 | Shaders/Private/ShadingModels.ush | 实现 HairCardBxDF() 及 4 个辅助函数（122 行）；IntegrateBxDF() switch 添加 case |
| 14 | Shaders/Private/ShadingModelsSampling.ush | 实现 SampleHairCardBxDF()；SampleBxDF() 和 SupportsSampleBxDF() 添加 case |
| 15 | Shaders/Private/MegaLights/MegaLightsMaterial.ush | 4 处 SHADINGMODELID_HAIR 判断扩展为 包含 SHADINGMODELID_HAIRCARD |

---

## 2. ShadingModel 注册流程

```
EMaterialShadingModel (EngineTypes.h) MSM_HairCard = 13
  -> MaterialExpressionShadingModel.h ValidEnumValues 允许材质编辑器选择
  -> HLSLMaterialTranslator.cpp #define MATERIAL_SHADINGMODEL_HAIRCARD 1
  -> ShaderMaterial.h FShaderMaterialPropertyDefines 位域标志
  -> ShaderMaterialDerivedHelpers.cpp WRITES_CUSTOMDATA_TO_GBUFFER = true
  -> ShaderGenerationUtil.cpp GBuffer slot 分配 (GBS_CustomData)
  -> ShadingCommon.ush #define SHADINGMODELID_HAIRCARD 13
  -> ShadingModelsMaterial.ush BasePass GBuffer 编码
  -> DeferredShadingCommon.ush GBuffer 解码 + Mobile 路径
  -> ShadingModels.ush HairCardBxDF() 光照计算
  -> ShadingModelsSampling.ush IBL 采样
  -> MegaLightsMaterial.ush MegaLights 集成
```

---

## 3. 渲染数据流

**BasePass (ShadingModelsMaterial.ush)：**

| 材质引脚 | GBuffer 目标 | 说明 |
|----------|-------------|------|
| Roughness | GBuffer.Roughness | 主粗糙度 R lobe |
| CustomData0 | GBuffer.CustomData.x | 次粗糙度 TRT lobe |
| CustomData1 | GBuffer.CustomData.y | Lobe Separation |
| Opacity | GBuffer.CustomData.z | Backlit 透射强度 |
| Metallic | GBuffer.CustomData.w | Spec Weight 高光权重 |
| Specular | GBuffer.Specular | TRT 吸收混合因子 |
| WorldTangent | GBuffer.WorldTangent | 发丝切线方向 |
| ShadingModelID | 13 (HAIRCARD) | 着色模型标识 |

**Deferred Lighting (ShadingModels.ush)：**
IntegrateBxDF() -> case SHADINGMODELID_HAIRCARD -> HairCardBxDF()
解码 GBuffer -> 双高光 Kajiya-Kay + Wrap Diffuse + Transmission

**MegaLights (MegaLightsMaterial.ush)：**
bIsHair = true -> 启用 Hair 复杂光照路径
bIsComplex = true -> 启用各向异性采样



---

## 4. GBuffer CustomData 通道分配表

| 通道 | 含义 | 来源引脚 | 推荐值范围 | 说明 |
|------|------|----------|-----------|------|
| .x | Secondary Roughness (TRT 粗糙度) | CustomData0 | 0.3 ~ 0.8 | 通常比主粗糙度大 0.1~0.2 |
| .y | Lobe Separation (切线偏移量) | CustomData1 | 0.02 ~ 0.15 | 控制 R/TRT 高光分离距离 |
| .z | Backlit (背光透射强度) | Opacity | 0.0 ~ 1.0 | 0=无透射, 1=全透射 |
| .w | Spec Weight (高光总权重) | Metallic | 0.3 ~ 0.8 | 控制高光 vs 漫反射能量分配 |

额外通道：
- GBuffer.Specular -> TRT 吸收混合因子 (0~1)，控制 TRT lobe 颜色偏移程度
- GBuffer.Roughness -> 主粗糙度 (R lobe)，推荐 0.2~0.5

---

## 5. 关键设计决策及理由

1. **Kajiya-Kay 而非 Marschner**：Hair Card 是面片几何体，没有真实的圆柱截面信息。Marschner 模型依赖精确的入射角和方位角分解，在面片上无法正确计算。Kajiya-Kay 基于切线方向的简化模型更适合面片渲染，且性能开销更低。

2. **双高光 (R + TRT) 而非单高光**：单一 Kajiya-Kay 高光无法表现真实头发的视觉复杂度。R lobe（表面反射）提供锐利的白色高光，TRT lobe（内部透射-反射-透射）提供柔和的彩色次级高光，两者组合可逼近 Marschner 的视觉效果。

3. **借用 Metallic/Opacity/Specular 引脚**：UE 的材质系统只提供 CustomData0 和 CustomData1 两个自定义引脚。HairCard 需要 6 个参数（双粗糙度、分离度、背光、高光权重、TRT 混合），因此复用 Metallic（对头发无物理意义）、Opacity（Deferred 不透明路径中未使用）、Specular 引脚来传递额外参数。

4. **Legacy GBuffer 路径而非 Substrate**：UE 5.7 的 Substrate 系统仍在过渡期，Hair Card 作为新增 ShadingModel 优先在稳定的 Legacy 路径实现，避免 Substrate 的复杂性和潜在不稳定性。

5. **ShadingModel ID = 13**：UE5.7 使用 4-bit ID（最大 15），现有最大 ID 为 12（Strata）。13 是下一个可用值，留出 14、15 供未来扩展。

6. **MegaLights 复用 Hair 路径**：HairCard 的各向异性特性与 Hair Strand 类似，复用 bIsHair 和 bIsComplex 标志可确保 MegaLights 正确处理各向异性采样和复杂透射。

---

## 6. 与输入文档的对照验证

| 文档 | 核心设计目标 | 实现状态 |
|------|-------------|---------|
| HairCard_Plan.md | 10 步修改计划 + 5 个额外 C++ 文件 | 全部 15 个文件已修改 |
| HairCard_Plan_backup.md | 备份计划，内容与主计划一致 | 同上 |
| HairCard_DeepShadow_Feasibility.md | Deep Shadow Map 可行性分析 | 未实现（属于后续优化，非核心 ShadingModel） |
| HairCard_References_Analysis_R2.md | R/TT/TRT 独立粗糙度、Chiang BSDF 参考 | 双粗糙度已实现；完整 Chiang 模型留作后续优化 |
| HairCard_References_Analysis.md | Kajiya-Kay 双高光方案确认 | 已实现 R + TRT 双高光 |
| UE57_Hair_Analysis.md | UE5.7 Hair 渲染系统源码分析 | 参照 HairBxDF 模式实现，GBuffer 编码/解码对齐 |