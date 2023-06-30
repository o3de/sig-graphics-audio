# Apple Shader converter integration


As shown in the diagram below our current solution goes through three steps for metal shader pipeline.

* Step 1: Azsl → HLSL → SPIR-V
* Step2: SPIR-V → MSL
* Step3: MSL → Metal bytecode library

![image](https://github.com/o3de/sig-graphics-audio/blob/main/rfcs/AppleShaderConverterIntegration/ShaderPipeline.png.png)

**Problems** 

1. Issue 1- Shader pipeline throughput. While a couple extra steps to compile an individual shader doesn’t *seem* like much on the surface, AAA titles can compile up to (or beyond) a *million* such shaders to package a title for a single RHI. This number shrinks for engines that don’t permit fully dynamic material assignment, but is actually realistic for engines that do (which includes Unreal Engine for example). Such titles can take easily hours to package on a single machine. Those extra steps above is a strict multiplicative factor on top of an already slow and expensive process. Compiling a material shader on Windows to go from HLSL to DXIL takes on the order of 1 to 2 seconds (empirical measurement taken across several engines). Compiling the same shader for Metal can take over 5 seconds depending.



1. Issue 2 - Intermediate step of passing through SPIR-V is limiting in that source context is lost to do accurate debugging. Bytecode-to-source translation doesn’t preserve the semantics of how the code was originally authored, so if there are crashes in Metal shaders, debugging these crashes is a lot trickier than debugging the same issues on, say, Windows.



1. Yet another issue is that the conversion from SPIR-V to MSL is potentially lossy or buggy, owing to the fact that there may be latency from the time features are added to MSL vs SPIRV-cross (this latency can be measured in years, meaning that features available in Metal are not “really available”). We also run into codegen-related problems related to Argument buffer data layout mismatches due to the dxc and spirv-cross stripping SRG layout entries not used by the shaders. This can lead to issues with 

    * Atomic instructions operating on `RWBuffer` memory
    * Hacky shader pipeline code to re-attach stripped resources in order to ensure Argument buffer data layout correctness.
    * Lack of support in SPIRV-cross for features otherwise supported in HLSL (e.g. mesh & tile shaders)


**Solution**
Integrate Apple metal shader converter into Metal RHI backend - https://developer.apple.com/metal/shader-converter/. With the converter the new shader pipeline would look like this

* Step 1: Azsl → HLSL → Dxil
* Step2: Dxil → Metal bytecode library

By going directly from DirectX IR to Apple IR we are able to address a lot of the problems described above. There are two ways we could integrate the shader converter to O3de


**Solution 1** - Automatic layout strategy

With this layout strategy the shader converter will build a linear layout of resources into the top-level Argument Buffer. So if you have a shader with 4 SRGs it will expand them all out in a linear layout in increasing order of register and space id. This type of solution has a performance advantage in that the command list can directly access the resource with no indirection but it brings with it a severe drawback. This type of solution means that each shader will have a unique resource binding layout and will need to be updated individually which goes against the Atom’s SRG architecture. 

Currently on Atom we update all the SRG gpu descriptors during SRG compilation time. This will essentially need to be moved to execution time as each SRG is directly embedded into a draw packet’s PSO. So if a PSO is referencing 3 SRGs then at execution time we will need to extractive SRG values and then re-update the descriptors embedded into the PSO. This solution will not work due to excessive cpu overhead and we can safely ignore it. 

**Solution2** - Explicit layout strategy (Viable strategy)

This solution will be possible via use of root signature that allows for indirection. We will introduce a new Argument buffer that will represent root signature for a PSO (pipeline state object). This new root signature argument buffer will reference SRG’s argument buffer as described below. Each SRG will represent an argument buffer which is equivalent to a descriptor table from DX12. The root signature AB will hold reference to Argument buffers for each SRG and hence when SRGs are updated at SRG execution time everything should still work out. 

Let's imagine if you have a shader with 3 SRGs → ViewSRg, PassSrg, MaterialSrg

```
ViewSRG
ConstantBuffer<ViewSRGConstants> ViewSRG_Constants : register(b0, space0);
StructuredBuffer<LightData> ViewSRG_LightData : register(b1, space0);
StructuredBuffer<DecalData> ViewSRG_DecalData : register(b2, space0);

PassSRG
ConstantBuffer<PassSRGConstants> PassSRG_Constants : register(b0, space1);
Texture2D<float4> PassSrg_m_brdfMap : register(t0, space1);
Texture2DArray<float> PassSrg_m_LightShadowmap : register(t1, space1);
SamplerState PassSrg_samp : register(s0, space1);

MaterialSRG
ConstantBuffer<MaterialSRGConstants> MaterialSRG_Constants : register(b0, space2);
Texture2D<float4> MaterialSrg_m_baseColorMap : register(t0, space2);
SamplerState MaterialSRG_samp : register(s0, space2);
```

Root Signature layout fed by shader pipeline

|TopLevelArgumentBufferLayout (Explicit layout)	|	|	|
|---	|---	|---	|
|Root Signature AB	|SRG ABs	|SRG entries	|
|uint64 (gpu address for ViewSrgArgumentBufer)	|uint64 (gpu address for ViewSRG_Constants) ->	|MTLBuffer for ViewSRG_Constants	|
|	|uint64 (gpu address for ViewSRG_LightData) ->	|MTLBuffer for ViewSRG_LightData	|
|	|uint64 (gpu address for ViewSRG_DecalData) ->	|MTLBuffer for ViewSRG_DecalData	|
|uint64 (gpu address for PassSrgArgumentBufer)	|uint64 (gpu address for PassSRGConstants) ->	|MTLBuffer for PassSRGConstants	|
|	|uint64 (gpu address for PassSrg_m_brdfMap) ->	|MTLTexture for PassSrg_m_brdfMap	|
|	|uint64 (gpu address for PassSrg_m_LightShadowmap) ->	|MTLTexture for PassSrg_m_LightShadowmap	|
|	|uint64 (gpu address for PassSrg_samp) ->	|MTLSamplerState for PassSrg_samp	|
|uint64 (gpu address for PassSrgArgumentBufer)	|uint64 (gpu address for MaterialSRG_Constants) ->	|MTLBuffer for MaterialSRG_Constants	|
|	|uint64 (gpu address for MaterialSrg_m_baseColorMap) ->	|MTLTexture for MaterialSrg_m_baseColorMap	|
|	|uint64 (gpu address for MaterialSRG_samp) ->	|MTLSamplerState for MaterialSRG_samp	|

In theory the above layout should work but in order for it work we will need to do the following

* In the shader pipeline we will use the Metal shader converters companion header’s helper functions to feed explicit layout described above. Example code will look like this. The variable **params** (which is IRRootParameter1) holds information related to the layout above.

```
    IRVersionedRootSignatureDescriptor rootSigDesc;
    memset(&rootSigDesc, 0x0, sizeof(IRVersionedRootSignatureDescriptor));
    rootSigDesc.version                    = IRRootSignatureVersion_1_1;
    rootSigDesc.desc_1_1.NumParameters     = sizeof(params) / sizeof(IRRootParameter1);
    rootSigDesc.desc_1_1.pParameters       = params;
    rootSigDesc.desc_1_1.pStaticSamplers   = samps;
    rootSigDesc.desc_1_1.NumStaticSamplers = sizeof(samps) / sizeof(IRStaticSamplerDescriptor);

    IRError* pRootSigError    = nullptr;
    IRRootSignature* pRootSig = IRRootSignatureCreateFromDescriptor(&rootSigDesc, &pRootSigError);
```

* The reflection data in this case will come from azslc. This reflection data is also spit out as a json file in the cache folder for debugging purposes. Essentially we will build iterate over this reflection data and build out an explicit root signature which is fed into shader converter when building out the metal byte code library.

At run time we will need to do the following

* Each sRG will represent a descriptor table and and we will need to use metal 3 api and use functions like IRDescriptorTableSetBuffer(), IRDescriptorTableSetTexture(), and IRDescriptorTableSetSampler(), to encode resource references into descriptor tables. 
* This api allows you to pass more information that may change at runtime
    * For buffers:
        * The buffer length stored in the low 32 bits.
        * A texture buffer view offset stored in 2 bytes left-shifted 32 bits. Metal requires texture buffer views to be aligned to 16 bytes. This offset represents the padding necessary to achieve this alignment.
        * Whether the buffer is a typed buffer (`1`) or not (`0`) in the high bit.
    * For textures:
        * The min LOD clamp in the low 32 bits.
        * 0 in the high 32 bits.
    * For samplers, the LOD bias as a 32-bit floating point number.
* Each PSO contains pipeline layout and we will add a new MTLBuffer within Metal::PipelineLayout. This AB will represent top level Argument Buffer described above.
* At command list submit time we bind the the argument buffers representing descriptor tables per SRG to the render encoder’s top level AB which will be stored within the PSO’s pipeline layout object. 


**Root constants**
If a shader uses following resource

```
struct Root_Constants
{
    float4 m_rootConstants;
};
Root_Constants rootconstantsCB;
ConstantBuffer<ViewSRGConstants> ViewSRG_Constants : register(b0, space0);
StructuredBuffer<LightData> ViewSRG_LightData : register(b1, space0);
```

The layout can look like this

|TopLevelArgumentBufferLayout (Explicit layout)/Root Signature AB	|	|	|
|---	|---	|---	|
|	|	|	|
|float4 (m_rootConstants)	|	|	|
|uint64 (gpu address for ViewSrgArgumentBufer)	|uint64 (gpu address for ViewSRG_Constants) ->	|MTLBuffer for ViewSRG_Constants	|
|	|uint64 (gpu address for ViewSRG_LightData) ->	|MTLBuffer for ViewSRG_LightData	|

So the root constant data can be encoded directly into the top level/root signature AB at the start. So the AB can have root constant at the top and then gpu addresses to bound SRG related Argument buffers. 

**Dual Source Blending**

* By default, Metal shader converter doesn’t inject this capability into its generated Metal IR, but exposes controls to allow you to enable it always, or to defer the decision to pipeline state creation time. To request support for dual source blending, pass `IRDualSourceBlendingConfigurationForceEnabled` to function `IRCompilerSetDualSourceBlendingConfiguration`, or `-dualSourceBlending` via the command-line interface.
* You can alternatively defer the decision to perform dual-source blending to runtime. In that case, use options `IRDualSourceBlendingConfigurationDecideAtRuntime` or `decideAtRuntime` when configuring dual-source blending support. In this case, Metal shader converter injects a function constant `dualSourceEnabled` into your fragment shader that you then provide when retrieving the function from the produced Metal library.


**Open Ended problem**
Due to the fact that we will need Argument Buffer Tier 2 support using the root signature approach via Shader converter is only applicable to Iphone11 and above devices. In theory this wouldn't be a bigger concern if we still preserve the current system of binding resources and at runtime we can switch to new binding capability based on the fact that AB tier 2 is supported. The problem arises from maintaining multiple byte code for the same shader as the layout will be different when using the existing system vs using the root signature. So the shader pipeline will need to create and spit out 2 byte codes for each shader variant/super variant and that is going to be problematic. This will add to shader compilation time and not to mention runtime shader loading code issues. 

* One option is to just build the byte code metal library at run time (for devices that support AB2) but this is not efficient as you are now spending time at runtime during boot up and front end to back end load which could have been moved to offline. 


**Pros**

* Shader pipeline is simplified improving execution time.
* No more hacky code to manage Shader layout with runtime layout. We are now feeding in layout based on reflection data from Azslc and using the same layout at runtime.
* No more relying on spirv-cross for existing or new metal features. For example adding unbounded arrays should be much simpler now and as a result we can add support for Bindless opening doors for terrain and ray Tracing which use Bindless.
* Adding support for RayTracing will become feasible.
* Extremely scalable and flexible for new features like mesh shading for indirect pipeline in future.


**Cons**

* We are introducing one extra indirection to all draw calls when accessing a resource via top level Argument Buffer. This will add a small perf hit.
* Shader source debugging is currently not available as we are going from dxil directly to Apple IR. There is no metalSL source anymore. I have created a ticket with Apple to help with this issue. Maybe they can expose hlsl itself when trying to view shader function in a gpu capture as a start. 
* Min OS version requirements
    * iOS 16 - https://support.apple.com/guide/iphone/supported-models-iphe3fa5df43/ios
    * macOS Ventura - https://support.apple.com/en-us/HT213264
    * Argument Buffer Tier2 - (Iphone11 and above) - This means that we will need to keep the existing solution around for devices that do not support AB tier 2


**Implementation Plan**

* Task 1 - ASV RHI Samples - If we feed in dxil to metal backend we can build root signature at runtime by building the metal bytecode library during PSO creation time. This will allow the most flexibility as we can specify the layout at runtime and submit PSO with the same layout at encoding time. Debugging any issues will become a lot easier.
* Task 2 - ASV RPI Samples with root signature at runtime - Extend the work to RPI samples which uses main render pipeline as well as low end pipeline. 
* Task 3 - Add Shader converter as a 3p package for Mac
* Task 4 - Update dxc 3p package for Mac
* Task 5 - Move building the root layout within the shader pipeline instead of runtime.
* Task 6 - Add support for root constants and dual source blending
* Task 7 - Extend shader pipeline to hold two byte codes, one with the layout of the current system and another with the layout that uses root signature. 

 

