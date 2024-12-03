This RFC introduces support for Compute shaders
to both *.MaterialPipelines and *.MaterialTypes

This feature is important for alternative rendering methods
like Texture/Object Space Shading that need to leverage
the O3DE Material pipeline.

# These are the basic steps on how to use this feature
## 1. Add 1 or more Shader Templates to a *.materialpipeline asset.  
In this example I'm adding a new shader template called `MyComputeSkin.shader.template`:
```json
{
    "shaderTemplates": [
        ...
        {
            "shader": "../Shaders/MyComputeSkin.shader.template",
            "azsli": "../Shaders/MyComputeSkin.azsli",
            "tag": "MyShaderTag"
        },
        ...
    ]
}
```
So far it doesn't look any different than a typical Raster (VS+PS) shader template.
  
## 2. Here is the content of `MyComputeSkin.shader.template`:  
```json
{
    "Source" : "TEMPLATE_AZSL_PATH",
    // The shader code historically assumed the shaders are pixel shader,
    // but this is a compute shader. With the SAMPLE_LEVEL_LOD macro, the shader code
    // will call Texture2D.SampleLevel() instead of Texture2D.Sample()
    "Definitions": [
        "SAMPLE_LEVEL_LOD=0"
    ],

    "ProgramSettings":
    {
        "EntryPoints":
        [
            {
              "name": "MainCS",
              "type": "Compute"
            }
        ]
    },

    "DrawList" : "MyCustomDrawListTag"
}
```
Two things are unique when compared to all other *.shader.template files that ship with O3DE:
- The `SAMPLE_LEVEL_LOD=0` macro definition, which, as explained in the comment, will make sure
that all the shader code will call Texture2D.SampleLevel(coord, SAMPLE_LEVEL_LOD) instead of
 Texture2D.Sample(coord).
- There's only one entry point of type `Compute`, in this case called `MainCS`.
  
## 3. Here is the content of `MyComputeSkin.azsli`:  
```cpp
#pragma once

// For legacy reason this macro also applies to Compute Shader.
#define MATERIALPIPELINE_SHADER_HAS_PIXEL_STAGE 1

#define OUTPUT_DEPTH 0
#define FORCE_OPAQUE 1
#define OUTPUT_SUBSURFACE 0
#define ENABLE_TRANSMISSION 0
#define ENABLE_SHADER_DEBUGGING 1
#define ENABLE_CLEAR_COAT 0

// Lighting
#define FORCE_IBL_IN_FORWARD_PASS              1
#define ENABLE_SHADOWS                         0
#define ENABLE_LIGHT_CULLING                   0
#define ENABLE_AREA_LIGHT_VALIDATION           0
#define ENABLE_SPECULARAA                      0 // SpecularAA requires ddx_fine() which is not available for compute shaders. 
#define USING_COMPUTE_SHADER_DIRECTIONAL_LIGHT 1 // Setting this to 1, otherwise the code assumes ddx_fine() is available.

// Material
#define ENABLE_DECALS                   0


//////////////////////////////////////////////////////////////////////////////////////////////////

#include MATERIAL_TYPE_AZSLI_FILE_PATH

//////////////////////////////////////////////////////////////////////////////////////////////////

#include "MySkinPassSrg.azsli"

#include <Atom/Feature/Common/Assets/Shaders/Materials/BasePBR/BasePBR_LightingData.azsli>
#include <Atom/Feature/Common/Assets/Shaders/Materials/Skin/Skin_LightingBrdf.azsli>
#include <Atom/Feature/Common/Assets/Shaders/Materials/Skin/Skin_LightingEval.azsli>

#include "MyComputeSkin_MainCS.azsli"
```
For the most part looks very similar to any other main `azsli` among the O3DE
PBR shaders, but a few macros are required to avoid compilation
issues related with hlsl functions like ddx_fine() which assumes the code
runs as a fragment shader:  
- `MATERIALPIPELINE_SHADER_HAS_PIXEL_STAGE 1`: For legacy reason this macro also applies to Compute Shaders.
This macro enables the inclusion of the `Surface` struct, etc.
- `ENABLE_SPECULARAA 0`: Disables SpecularAA because the implementation requires ddx_fine().
- `USING_COMPUTE_SHADER_DIRECTIONAL_LIGHT 1`: Same reason, because ddx_fine() doesn't exist in Compute shaders.
  
## 4. The actual compute Entry Point
Content of `MyComputeSkin_MainCS.azsli`:  
```cpp
#pragma once

[numthreads(8,8,1)]
void MainCS(uint3 dispatch_id: SV_DispatchThreadID)
{
    VsOutput IN = CalculateVsOutputFromStreamBuffers(dispatch_id.xy);

    // ------- Geometry -> Surface -> Lighting -------

    PixelGeometryData geoData = EvaluatePixelGeometry(IN, true /*isFrontFace*/);

    float3 viewPos = PassSrg::GetViewPosition();


    // Early Out
    float3 dirToCamera = normalize(viewPos - IN.worldPosition);
    float NdotV = dot(geoData.vertexNormal, dirToCamera);
    if(NdotV < -0.05)
    {
        return;
    }

    Surface surface = EvaluateSurface(IN, geoData);

    LightingData lightingData = EvaluateLighting(surface, IN.position, viewPos);

    // ------- Output to RW Texture (Uses BindlessSRG) -------

    uint diffuseTextureIndex = ObjectSrg::m_diffuseTextureIndexRW;

    const uint2 outputIndex = IN.position.xy;
    RWTexture2D<float4> diffuseTexture = Bindless::GetRWTexture2D(diffuseTextureIndex);
    diffuseTexture[outputIndex] = float4(lightingData.diffuseLighting.rgb, 1.0f);

    uint specularTextureIndex = ObjectSrg::m_specularTextureIndexRW;
    RWTexture2D<float4> specularTexture = Bindless::GetRWTexture2D(specularTextureIndex);
    specularTexture[outputIndex] = float4(lightingData.specularLighting.rgb, 1.0f);
}
```
The example shows that it is possible to leverage all the existing PBR functions
from the material system into a Compute shader as if it were a Pixel shader.  
The example shows that we are using the Bindless SRG to locate the output textures
for both Diffuse and Specular colors.

## 5. The C++ Side Of Things...
A few small, yet important, changes are required in MeshDrawPacket::DoUpdate().
In particular the PipelineState attached to each DrawItem is now defined according
to the PipelineStateType of the ShaderAsset, which can be of type `Compute` or `Draw`
where in the past was only assumed to be of `Draw` type.

Also because there's no IA (Input Assembly) stage for Compute shaders, there's
no automatic binding of the Vertex Streams. Instead, APIs will be provided to manually
bind vertex streams as ByteAddressBuffer shader resources.

The most important takeaway is that there are no changes to DrawPacket nor changes to the DrawItem
structs. So this addition will not impact the memory footprint of the DrawItems.

## 6. It Is Expected The Creation Of Custom Render Pass Subclasses.
Typically the developer was used to add RPI::RasterPass(es) to a pipeline,
and effortlessly render DrawPackets by defining the same DrawListTag as
those defined in the `*.shader` or `*.shader.template` files.  
  
Unlike Raster Shaders, Compute Shaders are a different beast in the sense that the amount of Compute Threads
are customizable and arbitrary. This RFC brings new, optional, APIs that will help with binding
vertex streams, and the creation of the RHI::DispatchItems, where the developer will have the opportunity to define
the total number of Threads at runtime.

The expectation is that a developer will have at least one custom/proprietary FeatureProcessor that talks
to the MeshFeatureProcessor to get MeshHandles and use the new APIs to request the creation of the DispatchItems.  
The developer will also have custom `RPI::RenderPass` subclass(es) that will submit those DispatchItems.
  

## 7. New APIs
  
### 7.1. New APIs Added To MeshFeatureProcessorInterface
```cpp
namespace AZ::Render
{
    class MeshFeatureProcessorInterface : public RPI::FeatureProcessor
    {
    public:
         ...
        //! A helper function, typically called by another FeatureProcessor, when Compute or RayTracing shaders need to bind
        //! Mesh Input Streams like "POSITION", "NORMAL", "UV1" etc as regular AZ::RHI::BufferViews.
        //! This function instantiates a concrete Builder-like object that helps creating the RHI::BufferViews. 
        virtual AZStd::unique_ptr<StreamBufferViewsBuilderInterface> CreateStreamBufferViewsBuilder(const MeshHandle& meshHandle) const = 0;

        //! MaterialTypes and MaterialPipelines support Compute Shaders (With DrawListTag) in their ShaderItem collections.
        //! Given that this is an uncommon use case, the DispatchItems are not created automatically by the MeshDrawPacket.
        //! Additionally DispatchItems require knowledge of the Total number of threads X,Y,Z, which should be customizable.
        //! The following function helps the creation of the DispatchItems and the user must supply a callback that
        //! allows full control on the number of Total Threads X,Y,Z.
        //! REMARK 1: It is recommended to call this function whenever ModelDataInstanceInterface::MeshDrawPacketUpdatedEvent is signaled.
        //! REMARK 2: This function is typically called by a custom FeatureProcessor that leverages the MeshFeatureProcessor.
        //!           The custom FeatureProcessor will own the returned list and submit the DispatchItems in a custom Pass.
        using DispatchArgumentsSetupCB = AZStd::function<void(uint32_t /*lodIndex*/, uint32_t /*meshIndex*/, uint32_t /*drawItemIdx*/, const RHI::DrawItem*, RHI::DispatchDirect&)>;
        //! DisptachItems will be created for the DrawItems that match both the @drawListTagsFilter and @materialPipelineFilter.
        //! Also, only DrawItems whose PipelineState is of Compute type will be considered.
        virtual DispatchDrawItemList BuildDispatchDrawItemList(
            const MeshHandle& meshHandle,
            const uint32_t lodIndex,
            const uint32_t meshIndex,
            const RHI::DrawListMask drawListTagsFilter,
            const RHI::DrawFilterMask materialPipelineFilter,
            DispatchArgumentsSetupCB dispatchArgumentsSetupCB) const = 0;
    };
} // namespace AZ::Render
```

### 7.2 AZ::Render::StreamBufferViewsBuilderInterface
```cpp
// This helper interface is typically used to manually define a set of Stream Buffers, identifiable
// by their Shader Semantics, like POSITION, NORMAL, UV0, UV1, etc... and create Buffer Views that can be bound
// to a Shader (typically Compute or RayTracing, because Raster shaders get the streams automatically bound to the InputAssemblystage).
// The most common use case is a non-Raster shader that uses the BindlessSrg and needs to know the indices
// of each StreamBuffer within the BindlessSrg::m_ByteAddressBuffer.
// To create one of these builders please use ModelDataInstanceInterface::CreateStreamBufferViewsBuilder().
class StreamBufferViewsBuilderInterface
{
public:
    AZ_RTTI(AZ::Render::StreamBufferViewsBuilderInterface, "{B0004EA8-C829-427D-8F3B-0FBB060CB385}");
    virtual ~StreamBufferViewsBuilderInterface() = default;

    //! All streams that need to be queried must be added before calling BuildShaderStreamBufferViews();
    virtual bool AddStream(const char* semanticName, AZ::RHI::Format streamFormat, bool isOptional) = 0;

    //! Returns the number of streams that were successfully added via @AddStream(...)
    virtual AZ::u8 GetStreamCount() const = 0;

    //! If @AddStream() is never called, the returned  @ShaderStreamBufferViewsInterface may only be useful
    //! for GetIndexBufferView().
    //! The returned ShaderStreamBufferViewsInterface can only get the Streams on the particular
    //! @meshIndex within the @lodIndex.
    virtual AZStd::unique_ptr<ShaderStreamBufferViewsInterface> BuildShaderStreamBufferViews(
        uint32_t lodIndex, uint32_t meshIndex) = 0;

    //! For informational purposes. Gets a reference to the MeshHandle used when this builder was created.
    virtual const MeshFeatureProcessorInterface::MeshHandle& GetMeshHandle() const = 0;
};
```  
  
### 7.3. AZ::Render::ShaderStreamBufferViewsInterface
```cpp
// An Interface that contains all Stream BufferViews (AZ::RHI::BufferView)
// requested to @StreamBufferViewsBuilderInterface.
// This is useful to manually bind Mesh Stream Buffers in a Compute or RayTracing Shader.
class ShaderStreamBufferViewsInterface
{
public:
    AZ_RTTI(AZ::Render::ShaderStreamBufferViewsInterface, "{3A80C85C-DD3A-4A1D-B564-291EB463CD0B}");
    virtual ~ShaderStreamBufferViewsInterface() = default;

    //! Returns the Shader-bindable RHI::BufferView for the Vertex Indices from a particular mesh. 
    virtual const AZ::RHI::Ptr<AZ::RHI::BufferView>& GetIndexBufferView() const = 0;
    
    //! Returns the Shader-bindable RHI::BufferView for a Vertex Stream, identifiable by @shaderSemantic, from a particular mesh.
    virtual const AZ::RHI::Ptr<AZ::RHI::BufferView>& GetStreamBufferView(const AZ::RHI::ShaderSemantic& shaderSemantic) const = 0;
    
    //! Same as above, but provides the convenience of finding the vertex stream by name, like "POSITION" or "UV1", etc.
    virtual const AZ::RHI::Ptr<AZ::RHI::BufferView>& GetStreamBufferView(const char* semanticName) const = 0;
    
    //! For informational purposes. Returns the LOD Index of the Mesh.
    virtual uint32_t GetLodIndex() const = 0;
    // For informational purposes. Returns the Mesh Index (Within the current LOD index) of the Mesh.
    virtual uint32_t GetMeshIndex() const = 0;
};
```
  
## 8. Practical Use Case
The overarching use case is that a developer will have custom MaterialPipelines.  
Also the developer will have at least one custom FeatureProcessor and one custom subclass of RPI::RenderPass.
CustomFeatureProcessor will get a reference of the MeshHandles they care about (by working with the MeshFeatureProcessor).
CustomFeatureProcessor will create the DispatchItems with the new API:  
`MeshFeatureProcessorInterface::BuildDispatchDrawItemList(const MeshHandle& meshHandle, ...)`.  
  
Just like `RPI::RasterPass`, the `CustomRenderPass` will get the DrawItems that are visible as in `RasterPass::UpdateDrawList()`.

Every frame, the list of DispatchItems will get filtered with the list of visible DrawItems, and only the surviving DispatchItems
will be submitted by `CustomRenderPass` during `BuildCommandListInternal()`.
  

# What are the advantages of the feature?
Developers will have the option of adding Compute shaders to MaterialTypes and MaterialPipelines.
This is very important for alternative rendering methods like Texture Space Shading and having fine control
on how much work should be done any given frame on behalf of a particular DrawItem.  

Another major advantange is that this proposed solution has zero impact in DrawPacket memory footprint and zero performance
impact to the existing PBR pipeline.

This feature leverages the Culling system, so the Compute shaders will run on only the visible DrawItems (as DispatchItems).

# What are the disadvantages of the feature?
Minor disadvantages only. For example, the addition of Macros like `SAMPLE_LEVEL_LOD` throughout some of the O3DE PBR shaders
to prevent compilation errors related to Compute shaders attemping to use the existing calls to Texture2D.Sample().

Also this implementation is not 100% data driven. It requires custom subclases of `RPI::FeatureProcessor` and `RPI::RenderPass` to be able to create and use DispatchItems from DrawItems. But also this a strength of the solution because it has negligible impact in performance and C++ footprint to the Atom CommonFeature Gem. 

# How will this be implemented or integrated into the O3DE environment?
New Public Interfaces will be added to the `Atom_Feature_Common.Public`.
Concrete implementations will be added to `Atom_Feature_Common.Private.Object`.
Some macro definition and `#ifdef SAMPLE_LEVEL_LOD` code changes throughout the O3DE PBR `*.azsli` files will be added
to prevent compilation errors. 

# Are there any alternatives to this feature?
More than alternatives, there may be enhancements in the future where the RPI can add support of a new subclass of `RPI::RenderPass`
called sommething like `RPI::ComputeItemPass` and provide data driven mechanisms to define Total X,Y,Z Thread counts and also automate the creation of DispatchItems. But those new APIs will be relevant only if We start to notice more O3DE developers taking advantage of Compute Shader to render DrawItems.
  
# How will users learn this feature?
Since it will not require developers intervention, a simple announcement of the feature will suffice. Documentation will need to be updated to show how users can add Compute shaders to Material-related files.  
  
Also an example will be added to the AtomSampleViewer (or maybe AutomatedTesting).

# Are there any open questions?

