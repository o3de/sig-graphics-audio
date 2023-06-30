Summary
=======

The Atom renderer intends to be modular and extensible, allowing game teams to easily customize the render pipeline to suit their needs. The material system currently hinders this customization because it is tightly coupled to the default render pipeline that ships with Atom. In order to work around these issues, bespoke material types would have to be created for each pipeline which would not be a viable workflow for a large scale project. To address this issue we need to improve the material system to be agnostic of the render pipeline, allowing the same materials to be used on different pipelines.

A similar RFC was published last year (https://github.com/o3de/sig-graphics-audio/blob/main/rfcs/rfc-prs-20210913-1.md), and here I attempt to improve upon that proposal. Some aspects of the prior RFC have been replicated here (thanks @jeremyong-az). If approved, this RFC will supersede the prior one.

What is the relevance of this feature?
======================================

The Atom renderer that ships with O3DE currently provides a _shader-centric_ workflow for authoring materials, where the material type definition is composed of complete shader programs. A tool called Material Canvas is in development that will add support for _node-editing_ workflows for authoring materials, but the current shader-centric workflow is being maintained (in addition to graph-based authoring of full shaders). The current problem can be simply stated by observing that the implementation of material algorithms is strongly coupled to the shaders used to author them. These shaders are hard-wired with assumptions about the render pipeline including the render target layout, vertex input streams, and resource binding strategy. The rendering pipeline for the current StandardPBR, EnhancedPBR, and similar material types are implemented in a Forward+ rendering pipeline. Without architectural changes, the material implementations cannot be easily ported to alternative rendering pipelines (e.g. deferred, visibility buffer, etc.). This is especially important for workloads that need to render high-density meshes (the Achilles’ heel of a Forward+ pipeline, due to quad-dispatch inefficiencies), but also restricts research and development of alternative rendering pathways. It is similarly difficult to make any changes to the default pipeline because it would impact the shader code for every material type. Over time, materials authored using the current abstraction are liable to ossify architectural decisions made early.

O3DE should provide an abstraction that allows engineers and artists to define how a material is shaded and lit _once_, and leverage this abstraction to drive _any_ particular rendering pipeline (be it forward, deferred, v-buffer, ray tracing, or any other future technique).

Feature Design Description
==========================

Implementing a render pipeline abstraction is a complex task with multiple aspects that would be difficult to solve all at once. Each render pipeline can vary widely with respect to passes, render target layouts, texture binding strategies, geometric inputs, BRDFs, light types, and more. In order to get the ball rolling and start providing value to users, we can narrow the scope of the problem to focus on solving a couple specific use cases, while keeping more use cases in mind (but not necessarily targeting them in the design at this stage). So let's focus on the following goals:

1.  Allow materials to simultaneously target the current MainPipeline, the current LowEndPipeline, and a hypothetical deferred pipeline.
2.  Keep the design as simple and focused as possible, leveraging existing patterns, minimizing the need to build new systems.
3.  Implement the pipeline abstraction at the material type level (rather than material graph) to remain decoupled from the Material Canvas development effort.
4.  Maintain general compatibility with the Material Canvas template design.
5.  Establish a conceptual framework that can hopefully be expanded in the future to address other use cases such as texture binding, geometry input, and lighting abstractions.

This proposal is an alternative to the design presented in https://github.com/o3de/sig-graphics-audio/blob/main/rfcs/rfc-prs-20210913-1.md. Key differences from this RFC include:

1.  The proposal does not separate the lighting code from the pipeline code. This abstraction is left for future enhancements.
2.  The proposal includes the ability to perform custom vertex position adjustments.
3.  No new AZSL features are proposed. We'll take a similar overall approach but use simple macros and stitching rather than introduction of virtual shader stages.
4.  All shaders are defined within the material pipeline rather than leaving some in the material type.
5.  A scripting system configures the material pipeline, rather than use singular pipeline tags to identify predefined shader combinations.

Current Material System
-----------------------

The material type file lists all of the specific .shader files that could be used. It has lua scripts that use material properties to control which shaders to enable, what passes to use, and change blend states. These shaders and the scripts that configure them are hard-coded to only work with the MainRenderPipeline.

![image](https://user-images.githubusercontent.com/55155825/183226335-9cc807e9-1848-458c-a032-5a5b2074879a.png)

Proposed Material System
------------------------

The material type file indicates a lighting model that it will use (i.e. Standard, Enhanced, Skin, Eye, etc.). This is a direct user choice. In a material graph, this defines the topology of the final output node. At a lower level, this selects the AZSL definition of the Surface struct. The material type also provides material code, which is partial shader code specific to the materials. This code only describes how to fill the Surface struct, or perform other simple operations like adjusting vertex position, depth, or discarding pixels.

A new _material pipeline_ feature will define how materials are processed by a render pipeline. This includes shader templates for each pass, and for each lighting model, with gaps that can be filled in by the material type's code. The association between the material type and the material pipeline is indirect, and the material type can work with any pipeline. The material build system is responsible for pairing each material type with the available material pipelines for compilation. The material-specific code for configuring the per-pixel Surface data is defined once, and is automatically compiled into separate shaders for each available render pipeline.

There are scripts in each material pipeline that configure the shader compilation based on the material type that is being compiled. For example, this allows the forward pass to be used for some material types and the transparent pass to be used for others.

![image](https://user-images.githubusercontent.com/55155825/183226342-90e4d288-2e5f-4500-b89b-200347d718a2.png)

Technical Design Description
============================

Material Code
-------------

The shader code in the material pipeline has gaps that must be filled in by the material type. Several functions can be provided:

*   MaterialVertexFunction() - optional function that can manipulate vertex data.
*   MaterialShadingFunction() - this is the main function that prepares the Surface data for the lighting model to consume.
*   MaterialDepthAndOrClippingFunction() - optional function that can be run in pixel shaders for depth-only passes to perform clipping and-or depth adjustments.

Along with these functions, the material type will provide the definition for the MaterialSrg and any shader options that the material needs. This data is only used by the material code and so it can be defined together with the functions listed above.

> **_Note_**: **_Material SRG Abstraction_** - In the future we expect there will be new abstractions to hide the details of how material resources are accessed. Instead of providing a MaterialSrg, the material type would just provide forward declarations for accessor functions like GetBaseColor() or SampleBaseColorTexture(). The material pipeline (described in a later section) would be responsible for implementing these functions using some form of code-generation. The material pipeline might generate a MaterialSrg and the functions would simply read from there, or the pipeline might generate functions that integrate with a bindless resource system, for example. In any case, it should be fine for the material type to provide the MaterialSrg directly for now, and any future abstraction would be implemented in a backward-compatible way, using metadata from the material type. 

There may need to be additional metadata or splitting of the functions into separate files to help the build system stitch the material code together with the shader template code. Those details can be determined at implementation time.

Note that this code should be able to include the Material Canvas insertion comments used by the code generator, making this compatible with the material graph template design (see "Appendix C - Lighting Model Output Node and Templates Notes" in https://github.com/o3de/sig-graphics-audio/issues/51).

**MaterialCode.azsli**
```hlsl
ShaderResourceGroup MaterialSrg : SRG_PerMaterial
{
    // GENERATED_MATERIAL_SRG_BEGIN
    float3 m_baseColor;
    Texture2D m_baseColorTexture;
    ...
    // GENERATED_MATERIAL_SRG_END
 }
 
void MaterialVertexFunction(inout float3 localPosition)
{
    // GENERATED_VERTEX_INSTRUCTIONS_BEGIN
    outPosition.z += sin(SceneSrg::m_time) * 0.1;
    // GENERATED_VERTEX_INSTRUCTIONS_END
}
 
void MaterialShadingFunction(VSOutput IN, out Surface outSurface, inout float depth)
{
    // GENERATED_SHADING_INSTRUCTIONS_BEGIN
    float4 inBaseColor = float4(1.0, 1.0, 1.0, 1.0);
    float4 inEmissive = float4(0.0, 0.0, 0.0, 0.0);
    float inMetallic = 1.0;
    float inRoughness = 0.5;
    float inSpecularF0Factor = 0.5;
    // GENERATED_SHADING_INSTRUCTIONS_END
   
    // GENERATED_DEPTH_OR_CLIPPING_INSTRUCTIONS_BEGIN
    // It could adjust depth here
    // It could check alpha clipping here
    // GENERATED_DEPTH_OR_CLIPPING_INSTRUCTIONS_END
    // (... or we could just call MaterialDepthAndOrClippingFunction() below, but this kind of nuance can be clarified as we build the system).
 
    surface.position = IN.m_worldPosition.xyz;
    surface.normal = normalize(IN.m_normal);
    surface.vertexNormal = normalize(IN.m_normal);
    surface.roughnessLinear = inRoughness;
    surface.CalculateRoughnessA();
    surface.SetAlbedoAndSpecularF0(inBaseColor.rgb, inSpecularF0Factor, inMetallic);
    surface.clearCoat.InitializeToZero();
}
 
void MaterialDepthAndOrClippingFunction(VSOutput IN, inout float depth)
{     
    // GENERATED_DEPTH_OR_CLIPPING_INSTRUCTIONS_BEGIN
    // It could adjust depth here
    // It could check alpha clipping here
    // GENERATED_DEPTH_OR_CLIPPING_INSTRUCTIONS_END
}
```
  

Shader Template
---------------

For each pass, the material pipeline will include one or more shader templates, with gaps that will be filled by each material type. For depth passes and similar, there will only be one or two shader templates. But for lighting passes there will be at least one shader template for each of the available lighting models. (In the future, we could also abstract out the lighting model to reduce the number of shader templates, and the build system can combine them automatically like it does for the material types).

The main gap that must be filled in is the material code. For now I propose the template .azsli file will simply forward-declare the functions that should be provided by the material. The material build pipeline will generate a .azsl file that \#includes the material code and the template .azsli file to stitch the two together. (In particular we should try to avoid stitching the files together programatically as part of a code-gen process as this would complicate the asset builder's source dependencies, discussed in the "Material Shader Build System" section).

The other way that shader templates are configured is through preprocessor flags (which are driven by material pipeline settings, more on this later).

**StandardLighting\_Forward.azsli**
Here is an example of what the forward pass shader template might look like. Similar files will exist for each pipeline, for each lighting model (i.e. StandardLighting.azsli or EnhancedLighting.azsli, etc.). Note that the specific design of the "VERTEX\_" flags might vary from what's presented here, what's important is the general idea of using preprocessor flags to configure the vertex structs.
```hlsl
// Common #includes here
 
// Preprocessor flags select the members of the vertex structs, the build system will configure this based on the material type requirements
 
#ifndef VERTEX_POSITION
#define VERTEX_POSITION 1
#endif
 
#ifndef VERTEX_NORMAL
#define VERTEX_NORMAL 0
#endif
 
#ifndef VERTEX_TANGENT
#define VERTEX_TANGENT 0
#endif
 
#ifndef VERTEX_BITANGENT
#define VERTEX_BITANGENT 0
#endif
 
#ifndef VERTEX_UV_COUNT
#define VERTEX_UV_COUNT 0
#endif
 
struct VSInput
{
#if VERTEX_POSITION
    float3 m_position : POSITION;
#endif
 
#if VERTEX_NORMAL
    float3 m_normal : NORMAL;
#endif
     
#if VERTEX_TANGENT
    float4 m_tangent : TANGENT;
#endif
     
#if VERTEX_BITANGENT
    float3 m_bitangent : BITANGENT;
#endif
  
#if VERTEX_UV_COUNT >= 1
    float2 m_uv0 : UV0;
#endif
     
#if VERTEX_UV_COUNT >= 2
    float2 m_uv1 : UV1;
#endif
};
 
struct VSOutput
{
#if VERTEX_POSITION
    precise linear centroid float4 m_position : SV_Position;
    float3 m_worldPosition : UV0;
#endif
     
#if VERTEX_NORMAL
    float3 m_normal: NORMAL;
#endif
     
#if VERTEX_TANGENT
    float3 m_tangent : TANGENT;
#endif
     
#if VERTEX_BITANGENT
    float3 m_bitangent : BITANGENT;
#endif
     
#if VERTEX_UV_COUNT >= 1
    float2 m_uv0 : UV1;
#endif
     
#if VERTEX_UV_COUNT >= 2
    float2 m_uv1 : UV2;
#endif
};
 
// Forward-declare functions that must be provided by the material type or generated
// by the material build pipeline.
void MaterialVertexFunction(inout float3 localPosition);
void MaterialShadingFunction(VSOutput IN, out Surface outSurface, inout float depth);
void MaterialDepthAndOrClippingFunction(VSOutput IN, inout float depth);
 
VSOutput MainPassVS(VSInput IN)
{
    VSOutput OUT;
     
    MaterialVertexFunction(IN.m_position);
  
    float4x4 objectToWorld = ObjectSrg::GetWorldMatrix();   
    float3 worldPosition = mul(objectToWorld, float4(adjustedPosition, 1.0)).xyz;
  
    OUT.m_worldPosition = worldPosition;
    OUT.m_position = mul(ViewSrg::m_viewProjectionMatrix, float4(OUT.m_worldPosition, 1.0));
     
    float3x3 objectToWorldIT = ObjectSrg::GetWorldMatrixInverseTranspose();
 
    // Note it doesn't support material manipulation of TBN yet
#if VERTEX_NORMAL
    OUT.m_normal = normalize(mul(objectToWorldIT, IN.m_normal));
#endif
     
#if VERTEX_TANGENT
    OUT.m_tangent = normalize(mul(objectToWorld, float4(IN.m_tangent.xyz, 0)).xyz);
#endif
     
#if VERTEX_BITANGENT
    OUT.m_bitangent = normalize(mul(objectToWorld, float4(IN.m_bitangent, 0)).xyz);
#endif
     
    return OUT;
}
 
#if ENABLE_CUSTOM_DEPTH
ForwardPassOutput MainPassPS(VSOutput IN)
#else
ForwardPassOutputWithDepth MainPassPS(VSOutput IN)
#endif
{
    ForwardPassOutput OUT;
     
    #if ENABLE_CUSTOM_DEPTH
    OUT.m_depth = IN.m_position.z;
    #endif
     
    // ------- Surface -------
 
    Surface surface;
     
    float unusedDepth;
     
    MaterialShadingFunction(IN, surface,
    #if ENABLE_CUSTOM_DEPTH
        OUT.m_depth
    #else
        unusedDepth
    #endif
    );
     
    // ------- LightingData -------
 
    LightingData lightingData;
    lightingData.tileIterator.Init(IN.m_position, PassSrg::m_lightListRemapped, PassSrg::m_tileLightData);
    lightingData.Init(surface.position, surface.normal, surface.roughnessLinear);
    lightingData.specularResponse = FresnelSchlickWithRoughness(lightingData.NdotV, surface.specularF0, surface.roughnessLinear);
    lightingData.diffuseResponse = 1.0f - lightingData.specularResponse;
    lightingData.emissiveLighting = inEmissive;
 
    // ------- Lighting Calculation -------
 
    // Apply Decals
    ApplyDecals(lightingData.tileIterator, surface);
 
    // Apply Direct Lighting
    ApplyDirectLighting(surface, lightingData, IN.m_position);
 
    // Apply Image Based Lighting (IBL)
    ApplyIBL(surface, lightingData);
 
    // Finalize Lighting
    lightingData.FinalizeLighting();
 
    PbrLightingOutput lightingOutput = GetPbrLightingOutput(surface, lightingData, inBaseColor.a);
 
    // ------- Output -------
 
    OUT.m_diffuseColor = lightingOutput.m_diffuseColor;
    OUT.m_diffuseColor.w = -1; // Subsurface scattering is disabled
    OUT.m_specularColor = lightingOutput.m_specularColor;
    OUT.m_specularF0 = lightingOutput.m_specularF0;
    OUT.m_albedo = lightingOutput.m_albedo;
    OUT.m_normal = lightingOutput.m_normal;
 
    return OUT;
}
```


**DepthPass.azsli**
This depth pass example shows how MaterialDepthAndOrClippingFunction() is used. (Similar files could exist for each material pipeline, but could also be shared as the depth pass will be pretty standard).
```hlsl
// #include the same preprocessor-configurable VSInput and VSOutput structs shown above
 
// Other #include headers...
 
// Forward-declare functions that must be provided by the material type or generated
// by the material build pipeline.
void MaterialVertexFunction(inout float3 localPosition);
void MaterialDepthAndOrClippingFunction(VSOutput IN, inout float depth);
 
VSOutput DepthPassVS(VSInput IN)
{
    // This is exactly the same as the forward pass above, can be factored out
}
 
struct PSDepthOutput
{
    precise float m_depth : SV_Depth;
};
 
// Actually, we probably don't need to use preprocessor flags here as this will just be ignored if this entry
// point is not referenced, but for the sake of example, I'll keep them here.
#if ENABLE_CUSTOM_DEPTH || ENABLE_PIXEL_CLIPPING
PSDepthOutput DepthPassPS(VSOutput IN)
{
    PSDepthOutput OUT;
    OUT.m_depth = IN.m_position.z;
     
    MaterialDepthAndOrClippingFunction(IN, OUT.m_dept);
     
    return OUT;
}
#endif
```
  
**StandardLighting\_Transparent.azsli**
Finally, an example of the transparent pass highlights how the same MaterialShadingFunction() will be reused in passes with completely different render targets. There's also an interesting example of how we could report errors about material types that use an unsupported configuration.
```hlsl
// #include the same preprocessor-configurable VSInput and VSOutput structs shown above
 
// Other #include headers, especially the lighting model...
 
// Forward-declare functions that must be provided by the material type or generated
// by the material build pipeline.
void MaterialVertexFunction(inout float3 localPosition);
void MaterialShadingFunction(VSOutput IN, out Surface outSurface, inout float depth);
 
VSOutput TransparentPassVS(VSInput IN)
{
    // This is exactly the same as the forward pass, can be factored out
}
 
#if ENABLE_CUSTOM_DEPTH
#error Custom depth not supported for transparent materials
#endif
 
struct TransparentPassOutput
{
    float4 m_color;
    #if TRANSPARENCY_MODE==TRANSPARENCY_MODE_TintedTransparent
    float4 m_color1; // for dual source blending
    #endif
}
 
TransparentPassOutput TransparentPassPS(VSOutput IN)
{   
    // ------- Surface -------
 
    Surface surface;
     
    MaterialShadingFunction(IN, surface);
     
    // ------- LightingData -------
 
    // This is basically the same as the forward pass code
 
    // ------- Output -------
 
    TransparentPassOutput OUT;
    OUT.m_color.rgb = lightingData.diffuseLighting * lightingData.alpha;
    OUT.m_color.rgb += lerp(lightingData.specularLighting, lightingData.specularLighting * alpha, surfaceSettings.opacityAffectsSpecularFactor);
    OUT.m_color.a = lightingData.alpha;
     
    #if TRANSPARENCY_MODE==TRANSPARENCY_MODE_TintedTransparent
        OUT.m_color1.rgb = surface.baseColor * (1.0 - lightingData.alpha);
        OUT.m_color1.a = 1.0;
    #endif
     
    return OUT;
}
```
  

Lighting Model
--------------

The first new file format proposed is a .lightingmodel file. When the user creates a material graph in Material Canvas, a lighting model must be selected (this could happen manually or automatically behind the scenes, depending on the desired user workflow). Similarly, when hand-authoring a material type, the user must indicate which lighting model it will use. For now this only provides limited information:

1.  A name. This will be used by the material pipeline system to decide which shaders to compile.
2.  A description to show in Material Canvas or other tools.

**Standard.lightingmodel**
```json
{
    "name": "Standard",
    "description": "Atom's standard lighting model."
}
```

Obviously this data is rather sparse. All the shader code for the lighting model will be #included into the shader templates that exist in the material pipeline. Thus each material pipeline will be hardwired to each available lighting model. In the future, we could move the lighting shader code to live alongside the lighting model definition, and use the build system to combine this code together with pipeline code and material code, allowing easier addition of new light types and lighting models. But for now the proposal does not include any significant abstraction to separate the lighting code from the render pipeline, as that isn't necessary for the time being. 

Material Canvas will need a way to associate the lighting model file with data that describes the final output node. That data shouldn't be included inside the lighting model file directly because the lighting model concept exist at a lower level. The details of how to make this association are yet to be determined, but one approach could be to have a _material graph template_ data file. The user would have to select a material graph template when creating a new material graph, and this would include references to the output node(s) metadata, the material template files, and the lighting model file.

Material Pipeline
-----------------

A Material Pipeline forms the back-end for the material system, allowing materials to run in a particular render pipeline. Each material pipeline has metadata and shader templates that match the material-related passes in the render pipeline. It also has a script that configures the pipeline to build the appropriate shaders for a specific material type. Here is an example collection of files that would form a material pipeline:

*   MainPipeline.materialpipeline
*   MaterialPipeline.lua
*   Shader Templates for Standard Lighting
    *   StandardLighting\_Forward.shader.template
    *   StandardLighting\_BlendedTransparent.shader.template
    *   StandardLighting\_TintedTransparent.shader.template
*   Shader Templates for Enhanced Lighting
    *   EnhancedLighting\_Forward.shader.template
    *   EnhancedLighting\_BlendedTransparent.shader.template
    *   EnhancedLighting\_TintedTransparent.shader.template
*   Other Shader Templates
    *   SkinLighting\_Forward.shader.template
    *   EyeLighting\_Forward.shader.template
    *   DepthPass.shader.template
    *   DepthPass\_WithPS.shader.template
    *   ShadowPass.shader.template
    *   ShadowPass\_WithPS.shader.template
    *   DepthPassTransparentMin.shader.template
    *   DepthPassTransparentMax.shader.template
    *   MeshMotionVector.shader.template

We can use a .template extension to prevent the Asset Processor from processing these files since they are not complete source assets.

The set of shaders will include the combination of every lighting model and every render pass. When it comes time to build a material type's shaders, we need a way to tell the build system which shaders to compile, how to configure the preprocessor, or maybe perform other configuration we haven't thought of yet. So each material pipeline will expose a collection of settings to the material type, and run a script to configure the build. Examples of these settings might include _UV stream count_, _enable custom depth_, _enable pixel clipping_, _is transparent_, _transparency mode_, and/or _needs tangents_. These properties can be set in the .materialtype file and exposed in the Material Canvas UI. Of course they'll each have metadata for descriptions, data types, and visibility. In the context of Material Canvas, some settings might be exposed to the user to configure the graph, some might be hidden and configured automatically (like setting _needs tangents_ based on whether normal mapping is used), and we _might_ even allow some to be connected to material properties and exposed on a per-material basis (this would be required to make things like normal maps optional at the material level; it needs more thought).

We will use Lua for these scripts because that's what material functors use. While Lua isn't necessary (python could be an option since these scripts aren't used at runtime), it should provide nice continuity because material functors use Lua. Also, the material pipeline might even include a script that does execute at runtime, so we should be prepared for that possibility (for example, the material pipeline might know how to enable/disable certain shaders at runtime).

Each pipeline will define its own settings but there will likely be common ones where the same property name could map to multiple pipelines. For now we'll just merge the pipeline setting names from all pipelines, and assume that duplicate names will have the same usage in all pipelines. This should give the most convenient experience to the user, as there's no need to set the same value for each pipeline. If needed in the future, we could provide a way to disambiguate in case different pipelines use the same name for different things.

Each setting could be connected directly to a preprocessor flag, or processed by the pipeline's script to select which shaders to compile, or show/hide other properties. (We might want to consider just always exposing every setting as a preprocessor flag rather than having to specify one explicitly in the properties list. If a setting is to be controlled automatically rather that explicitly set by the user, then there's no real reason to provide metadata for it, except to do error checking for unhandled settings. It could use more thought).

**MainPipeline.materialpipeline**
Here is an example .materialpipeline file. (Note that it also lists all the shader templates, so the asset builder can report source file dependencies).
```json
{
    "propertyLayout": {
        "properties": [
            {
                "name": "ENABLE_CUSTOM_DEPTH",
                "displayName": "Enable Custom Depth Values",
                "description": "The pixel shader can modify the depth value.",
                "defaultValue": false,
                "visibility":  "Hidden"
            },
            {
                "name": "ENABLE_PIXEL_CLIPPING",
                "displayName": "Enable Per-Pixel Clipping",
                "description": "The material can be clipped on a per-pixel basis (i.e. alpha clipping). This forces depth passes to include a pixel stage.",
                "defaultValue": false,
                "visibility":  "Hidden"
            },
            {
                "name": "IS_TRANSPARENT",
                "displayName": "Is Transparent",
                "description": "Indicates that the material should be rendered in a transparent pass with blending enabled.",
                "defaultValue": false
            },
            {
                "name": "TRANSPARENCY_MODE",
                "displayName": "Transparency Mode",
                "description": "If transparency is enabled, this selects the kind of blending to perform.",
                "type": "Enum",
                "enumValues": [ "Blended", "TintedTransparent" ],
                "defaultValue": "Blended"
            },
            ...
        ]
    },
    "shaderTemplates": [
        "StandardLighting_Forward.shader.template",
        "StandardLighting_BlendedTransparent.shader.template",
        "StandardLighting_TintedTransparent.shader.template",
        "EnhancedLighting_Forward.shader.template",
        "EnhancedLighting_BlendedTransparent.shader.template",
        "EnhancedLighting_TintedTransparent.shader.template",
        "SkinLighting_Forward.shader.template",
        "EyeLighting_Forward.shader.template",
        "DepthPass.shader.template",
        "DepthPass_WithPS.shader.template",
        "ShadowPass.shader.template",
        "ShadowPass_WithPS.shader.template",
        "DepthPassTransparentMin.shader.template",
        "DepthPassTransparentMax.shader.template",
        "MeshMotionVector.shader.template"
    ],
    "pipelineScript": "MaterialPipeline.lua"
}
```

**MaterialPipeline.lua**
Here is an example of the script file for the above pipeline. Note there is an implicit _lightingModel_ variable that has the name of the material type's selected lighting model. (This is using pseudo-code, not actual lua, and the actual APIs might look a bit different, for example calling a GetPropertyValue\_bool("") function instead of just referencing a global variable).
```
EnableShaderCompilation("MeshMotionVector.shader.template")
    
SetPropertyVisible("TRANSPARENCY_MODE", IS_TRANSPARENT)
    
if lightingModel=="Standard"
    if(IS_TRANSPARENT)
        EnableShaderCompilation("DepthPassTransparentMin.shader.template")
        EnableShaderCompilation("DepthPassTransparentMax.shader.template")
        if(TransparencyMode=Blended)
            EnableShaderCompilation("StandardLighting_BlendedTransparent.shader.template")
        else if(TransparencyMode=TintedTransparent)
            EnableShaderCompilation("StandardLighting_TintedTransparent.shader.template")
    else if(ENABLE_CUSTOM_DEPTH || ENABLE_PIXEL_CLIPPING)
        EnableShaderCompilation("DepthPass_WithPs.shader.template")
        EnableShaderCompilation("ShadowPass_WithPs.shader.template")
        EnableShaderCompilation("StandardLighting_Forward.shader.template")
    else
        EnableShaderCompilation("DepthPass.shader.template")
        EnableShaderCompilation("ShadowPass.shader.template")
        EnableShaderCompilation("StandardLighting_Forward.shader.template")
if lightingModel=="Enahanced"
    if(IS_TRANSPARENT)
        EnableShaderCompilation("DepthPassTransparentMin.shader.template")
        EnableShaderCompilation("DepthPassTransparentMax.shader.template")
        if(TransparencyMode=Blended)
            EnableShaderCompilation("EnhancedLighting_BlendedTransparent.shader.template")
        else if(TransparencyMode=TintedTransparent)
            EnableShaderCompilation("StandardLighting_TintedTransparent.shader.template")
    else if(ENABLE_CUSTOM_DEPTH || ENABLE_PIXEL_CLIPPING)
        EnableShaderCompilation("DepthPass_WithPs.shader.template")
        EnableShaderCompilation("ShadowPass_WithPs.shader.template")
        EnableShaderCompilation("EnhancedLighting_Forward.shader.template")
    else
        EnableShaderCompilation("DepthPass.shader.template")
        EnableShaderCompilation("ShadowPass.shader.template")
        EnableShaderCompilation("EnhancedLighting_Forward.shader.template")
else if lightingModel=="Skin"
    EnableShaderCompilation("DepthPass.shader.template")
    EnableShaderCompilation("ShadowPass.shader.template")
    EnableShaderCompilation("SkinLighting_Forward.shader.template")
else if lightingModel=="Eye"
    EnableShaderCompilation("DepthPass.shader.template")
    EnableShaderCompilation("ShadowPass.shader.template")
    EnableShaderCompilation("EyeLighting_Forward.shader.template")
```
  

> **_Note_**: You may notice that the features of the the material pipeline's property and script feature feel very similar to the material type concept. Actually an early suggestion from @VickyAtAZ was to leverage the material type concept directly to form this abstraction. The two systems are different enough that we can't use material type directly, but there will almost certainly be opportunity to refactor and share some of the existing material type code for properties and lua scripts.

  

Material Type Changes
---------------------

In order to utilize the material pipeline build system, a .materialtype file will omit the "shaders" section, add a "lightingModel" setting to reference the .lightingmodel file, and add a "materialShaderCode" setting to reference the .azsli file that contains the material shader functions. It will provide values for material pipeline settings as appropriate in a new "materialPipelineSettings" section. The material type's functors must avoid using any APIs that enable, disable, or edit shaders (including setting render states). If possible, we'll disable these functions and/or report errors if they are called at runtime.

**Example.materialtype**
```json
{
    "propertyLayout": {
        "propertyGroups": [
            ...
        ]
    },
    "lightingModel": "Standard.lightingmodel",
    "materialShaderCode": "ExampleMaterialType.azsli",
    "materialPipelineSettings":
    {
        "IS_TRANSPARENT":  true,
        "TRANSPARENCY_MODE": "TintedTransparent"
    },
    "functors": [
        ...
    ]
}
```
  

Material Shader Build System
----------------------------

In order to build the material types, we will need a new stage of asset processing, utilizing the new intermediate assets feature: Material Type Prep Builder. This new stage will first do any code-gen or stitching that's required to produce the full set of necessary shaders that are ready for compilation. There will be a MaterialPipelineSystem class that knows the list of all available pipelines. (How we configure this system is left for another discussion, for now it can just be a registry setting with a list of .materialpipeline files for each platform). For each material pipeline, pass the lighting model name and the pipeline settings, and run the configuration script to get a list of shader templates. For each shader template, it will find the .azsli and stitch it together with material code using \#include statements, and also prepend the list of preprocessor definitions such as "ENABLE\_CUSTOM\_DEPTH". These .shader and .azsl files will be output as intermediate assets for the ShaderAssetBuilder to consume.

The final MaterialTypeAsset is going to need to reference the final ShaderAssets, which aren't ready at this stage. Somehow the AP will need to determine the final AssetIds for each shader before it can produce the final MaterialTypeAsset that references them. There needs to be a Material Type Builder job that depends on the Shader Asset Builder jobs. However, since ProcessJob is not able to create other jobs with dependencies (all it can do is output products), we can make it output another "alternate" .materialtype file to the intermediate asset folder. This alternate version will have the _lightingModel_ field removed and instead will list all the intermediate shaders that are being compiled. Then the Material Type Builder can use the list of intermediate shader files to create the necessary job dependencies, and assign the final shader AssetIds in the final MaterialTypeAsset.

![image](https://user-images.githubusercontent.com/55155825/183226371-02975e3b-0245-414f-b2c0-d1fa6c5c5e38.png)
  
Each MaterialAsset will continue referencing a single MaterialTypeAsset to define its property layout and behavior, but can be rendered in any pipeline. Therefore, the final result from a .materialtype source file will be a _single_ MaterialTypeAsset that can run in any render pipeline, rather than a separate MaterialTypeAsset _per_ pipeline. All we need to do is add some new information to each ShaderItem in the ShaderCollection that indicates which pipeline(s) should use that shader. This will make the "alternate" .materialtype format naturally backward compatible, as the absence of this exta info means the shader can be used in any pipeline. (This also allows for the possibility of the same shader asset being used in multiple pipelines, like a common depth pre-pass).

At runtime, the system will select the appropriate shaders for any given pipeline using a DrawFilterMask. There are several ways we could approach this. The MeshDrawPacket could create separate draw packets for every pipeline that the material type can target, and call DrawPacketBuilder::SetDrawFilterMask() using the RenderPipelineId. Or we could change the DrawPacketBuilder and DrawPacket classes to have a DrawFilterMask for each DrawItem. This latter approach is what I prototyped and it worked quite well.

![image](https://user-images.githubusercontent.com/55155825/191451918-7bbde1d7-36f4-415f-98ec-b3462e29ef97.png)

We should also consider the possibility that a single material pipeline could target multiple render pipelines. For example, consider a VR scenario where there are two similar render pipelines, one for each eye, but only one eye renders the shadowmaps. Since the materials's shaders only use passes that are common between the two pipelines, we shouldn't have two separate material pipelines as that would double the number of shaders that have to be compiled, and those shaders would be duplicate. So the material pipeline should be able to list multiple render pipelines that it works with, or we need a tagging system separate from the render pipeline name. This could be determined at implementation time, but I would lean toward having an independent "materialPipelineTag" name.

One more thing to keep in mind is that the Material Type Prep Builder will need to have appropriate dependencies so that if anything in a material pipeline changes, it will trigger reprocessing of all material types. Material Type Prep Builder's CreateJobs should be able to do this by asking the MaterialPipelineSystem for a list of all .materialpipeline files. These will list the pipeline script file and all of the shader template files (as described in the Material Pipeline section), so the builder can report these as source dependencies. Since the generated intermediate .azsl files will \#include the template .azsli file (rather than having the builder inject the material code directly into the .azsl file) there is no need to report source dependencies here. The Shader Asset Builder will automatically report the included files as dependencies just like any other azsl include file.

Material Canvas Changes
-----------------------

This new material pipeline abstraction should work well with Material Canvas's general approach of injecting data into a material graph template. However, Material Canvas's technical design will require some changes.

1.  The template concept is still used, just with a different set of template files. Instead of having separate template shaders for the different passes, there will only be a template for the material code AZSLI file, with functions like MaterialVertexFunction() and MaterialShadingFunction().
2.  Material Canvas will not be able to insert instructions into the render pipeline code which is being moved to the material pipeline. Instead, it will send flags to the MaterialPipelineSystem to configure the preprocessor. For example, instead of inserting specific fields into the VSInput struct, it will set flags like "VERTEX\_NORMAL" to include normal vectors in the input and output structs. Material Canvas should set flags like this automatically based on whether normal vectors are referenced in the node graph. The flags are set using the new materialPipelineSettings section of the .materialtype file.
3.  Material Canvas will need to expose material pipeline settings to the user (the ones marked Visible), and set them in the materialPipelineSettings section of the .materialtype file. For example, the user could set an "Is Transparent" value. Since each pipeline could have different settings available, Material Canvas will iterate through all available pipelines. It should probably merge any duplicate settings into a single display where possible (so the user doesn't have to set "Is Transparent" separately for every pipeline), but will also need to handle settings with the same name but different types or metadata. The specific design for how to present these settings is outside the scope of this RFC.
4.  When the user creates a new material graph, they will need to somehow select a lighting model. This could be done by selecting a material graph template that references a lighting model and other graph-specific data. Or in the future it could provide a more automatic workflow without impacting the overall design presented here. For example, Enhanced lighting is similar to Standard lighting, it just has a few extra pins for features like anisotropy and subsurface scattering. These pins could be always present, the selection of Standard vs Enhanced would happen automatically based on whether the additional enhanced pins are connected. But for now we should probably take the simplest approach and iterate from there.

What are the advantages of the feature?
=======================================

1.  Adding a new render pipeline or lighting code can be done without impacting existing materials and shaders.
2.  The same library of materials could be used for rendering on a high end or low end PC, game console, mobile device, and VR headset.
3.  Since the material pipeline system exists at a lower level than the node graph concept...
    1.  These abstractions can be implemented quickly, unhindered by dependencies on the Material Canvas development schedule.
    2.  Rather than being forced to use Material Canvas, users can continue hand-authoring material types if they prefer, while still having the benefits of pipeline abstraction.
4.  Authoring new material types is simpler because common boilerplate code is clearly separated from the code that makes each material type unique.
5.  New abstraction features can be added without significant impact to the overall design, because of the script layer that exists between the material and material pipeline.

What are the disadvantages of the feature?
==========================================

*   This RFC decouples materials from the render pipeline, but does not yet decouple material resource binding. This will need to be addressed in a future RFC before materials can take advantage of bindless resources or virtual texture sampling. I think this abstraction can be added in the future while maintaining support for material types that are implemented under the proposed design.
*   Lighting models and light types are implemented as part of the material pipeline. Adding a new light type would require updating shader code that is hard-wired to each material pipeline. A future RFC could separate the lighting code from the pipeline code and use the MaterialPipelineSystem to combine them at build-time.
*   The build system for material types will be more complex.
*   There may be some new restrictions on the ways materials can be configured, because material types will only have indirect control over the shaders.

How will this be implemented or integrated into the O3DE environment?
=====================================================================

Initially the new features will be implemented alongside the existing material type build system and will not impact existing material types. Existing material types will continue to work, but they will not take advantage of the new pipeline abstractions and will remain bound to the default render pipeline.

Any additions to the .materialtype file will be backward compatible. We will add new sections to the .materialtype file format for specifying the lighting model, material pipeline settings etc.  The existing material asset builder will ignore any .materialtype files that include these new fields, and we'll introduce a new material type asset builder that only processes those .materialtype files. This will give us time to flesh out the new system, experiment, and discover unforeseen limitations. We'll greadually build up equivalents of the core material types (StandardPBR, EnhancedPBR, Skin, etc) under the new system. Once those reach maturity, we will replace the old material types with these new ones. 

I am optimistic that the new matrial types can be fully compatible with existing materials. Originally I expected that the material pipeline system will introduce some new content limitations, so the conversion would not be 1:1. For example, StandardPBR currently supports both opaque and transparent surfaces, but in the new system these might need to be separate material types. But I put a lot of time brainstorming the different ways we might introduce new content limitations, and I the only change I foresee is documented here https://github.com/o3de/sig-graphics-audio/issues/74 and that change should not have any significant impact on existing material data.

IF we somehow discover some incompatibility, we will follow a deprecation process to phase out the old material types. The extent of any new limitations is difficult to predict, but we will do our best to provide clear migrations paths.

Material Canvas will continue its current design pattern of using material graphs to automatically generate corresponding material types. We will update it to target the new material type system that supports pipeline abstraction.

Are there any alternatives to this feature?
===========================================

The original RFC (https://github.com/o3de/sig-graphics-audio/blob/main/rfcs/rfc-prs-20210913-1.md) is one alternative. The main difference is that the new design includes a scripting layer in the material pipeline. Other differences are described in the Feature Design Description section.

One idea we considered was to implement the pipeline abstraction at the material graph level instead of the material type level. The main advantage would be consolidation of tech. Instead of material graphs and material types both having complex builders with shader code generation, the Material Canvas builder would handle all the complexity of code genreation and shader stitching, while the material type would remain closely coupled to the generated shaders. Then we could focus all our efforts on making a great node graph authoring experience. Also, node graph authoring would be a natural fit for abstraction as details are all hidden behind the final output node anyway. But there are multiple disadvantages to this approach. Users would not have the option of hand-authoring material types, they would be forced to use Material Canvas. The development schedule for pipeline abstraction would be dependent on Material Canvas and likely take longer to convert existing material types to support multiple pipelines. Material Canvas would not be able to continue using its template approach for code generation.

How will users learn this feature?
==================================

For Material Canvas users: The new system will be almost entirely hidden from them, providing a convenient intuitive workflow. There will be tutorial content based around Material Canvas, but the pipeline abstraction system will be an afterthought for these users, which is exactly what we want.

For material type authors: We will update the material system documentation to reflect the new .materialtype file spec, and describe the new .lightingmodel and .materialpipeline files and relevant APIs. We will provide new versions of published material type authoring tutorials (which will become much simpler with the new system).

For material pipeline authors: We will provide new material pipeline documentation similar to what's described in this RFC. We could also write a tutorial that shows how to implement a new render pipeline and add it to O3DE, which will cover both the pass system and the material pipeline system.

Are there any open questions?
=============================

**How will this impact existing material types?**

Existing material types will continue to function a they do now, but will not take advantage of the new pipeline abstraction features. We will port Atom's core material types to the new system, and we expect users will want to do the same with any custom material types they have created. I expect that the material pipeline system will be able to support porting most material types, but the extent of the impact to special edge-case material types is difficult to predict, and we will do our best to provide migration guidance.

**Should we allow per-material properties to configure the material pipeline, or only expose material pipeline settings to material-types?**

I was originally thinking that the material pipeline would not allow materials to configure shaders in any way, including enabling or disabling shaders. After giving this a lot of thought, I am giving up on that idea. So there is no reason to consider material properties that configure the material pipeline _build system_. However we will probably need a way for material properties to enable or disable a particular shader, like they do now, to make a material transparent or opaque. So the material pipeline will probably need some support for runtime properties/scripting in addition to its build-time properties/scripting.

**How can we reduce redundant compilation of duplicate shader code?**

We could reduce compilation of duplicate shaders by detecting whether the material impacts the vertex stage. In most cases it will not. So for any materials that don't use the pixel stage, and don't edit vertex data, we could pre-compile the depth-only shaders once per pipeline, and use these instead of recompiling once per material type.

Maybe we can reduce redundant shader compilation with a new AP feature that can see if two jobs have the same source data checksums, and copy the other job's product instead of running a redundant job.

**Should we allow material types to set any preprocess flag regardless of whether the material pipeline explicitly exposes it in the property list?**

Initially we should allow this for invisible settings that aren't exposed in Material Canvas. This will make it easier for us to iterate and experiment with new preprocessor options without having to fiddle with property metadata that will only be used for validation. We can always add this restriction later.

**How will the MaterialPipelineSystem define the list of material pipeline configurations for each project?**

For now we can just have registry settings that list the material pipelines for each pipeline. In the future we might want a more automatic way to discover pipelines, so that the user can simply enable a gem and any new material pipelines will be added automatically.

**How will parallax and PDO work with this system? How will multi-layer material types work?**

These cases tend to add more complexity to material shaders and need a closer to look to really know how they might impact the proposed design.

**How will geometry and texture code abstraction work under this system?**

The proposed design does not go into detail of these areas, but the idea is that the material pipeline simultaneously abstracts all of these concerns from the material type. The script interface layer should hopefully extend to be able to cover the configuration of these abstractions as well. As described in the Material Code section, we will eventually need to abstract out the MaterialSrg definition and make the material pipeline generate accessor functions instead. This will be a future design effort.

**How will light types and light models be abstracted in the future?**

I have some design ideas and notes that I was working on while considering the proposed design. I hope to refine and publish those ideas later as time allows.

**How can we expand this design to cover per-vertex manipulation of normals and tangents?**

We have plans to move some of the TBN processing from the vertex shader to the pixel shader. I'd like to get that change in place and then take a fresh look at this question. The primary need is for vertex position manipulation (to do minor wind bending for example), and the design covers this.

**How will we manage shader variant trees for the many per-pipeline shaders?**

I'm hoping that we can provide some default process for enumerating system level shader options and just ignoring the material-level options. If users need more performance they could author new material types that are tailored to the necessary materials, so no material-level shader options will be needed. At least for now. Otherwise the user is going to need to manage .shadervariantlists for many generated shader assets. Maybe we should make a way for one .shadervariantlist to apply to multiple shader assets, because a collection of pipeline-specific shader assets for the same material type will have the same options.




