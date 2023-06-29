# Summary

The Atom RHI supports instanced draw calls, but there is no support in the MeshFeatureProcessor or in the shaders that ship with Atom for submitting and rendering instanced draw calls.

# What is the relevance of this feature?

Instanced draw calls provide a performance improvement for scenes with lots of objects that share the same model and material.

# Feature Design Description

This RFC describes a proposed approach to instancing any object that is using the same mesh + material combination. The goal is to do this automatically without any user input during content creation. This both simplifies the content creation process, and allows non-o3de applications that use Atom as a standalone renderer to benefit from instancing without needing to add additional tooling.

# Technical Design Description

The main complication that is added when supporting instanced draw calls is that for each view, the instances that are visible can change from frame to frame, but the SV_InstanceID that is visible in the shader is always indexed from 0, 1, ..., N - 1, where N is the number of instances being drawn. You cannot use this index to directly look up data from a stable persistant buffer, because the visible objects are not always the same. Consider this example:

![image](https://github.com/amzn-tommy/sig-graphics-audio/tree/patch-1/rfcs/MeshInstancingRFC_View.PNG)

For an instanced draw call with 2 visible instances, the shader will be executed with SV_InstanceID 0 and 1. But the shader needs to look up the transforms for instances 1 and 3. To manage this the cpu needs to update an instance buffer each frame. This instance buffer could be updated to contain the transforms for instances 1 and 3 directly. However, for each instance there is an object-to-world, object-to-world-inverse-transpose, and object-to-world-history transform. There is also potential for additional per-instance data.

Since the transforms are already managed in a buffer where they can be looked up by object ID, I am proposing that the dynamic per-instance data consist of only the object ID. The shader can then look up the object ID via the SV_InstanceID, and using the object ID, look up the transforms.

## MeshInstanceManager

![image](https://github.com/amzn-tommy/sig-graphics-audio/tree/patch-1/rfcs/MeshInstancingRFC_MeshInstancManager.PNG)

The MeshInstanceManager is a simple map that keeps track of what can be instanced together, which I refer to as an instance group. It returns an InstanceGroupIndex that can be used to access the data that is shared between all instances in an instance group. The MeshFeatureProcessor will then use the results of CPU per-object culling to create and update the per-view buffers that contain the object IDs of all the instances that are visible in that view. The object IDs from each instance group are contiguous, and the draw call for each instance group will have a root constant that contains the offset into this per-view data where the instance group starts. Thus, the object ID for a given instance can be indexed by per-draw offset + SV_InstanceID. 

## Culling

The existing culling system is responsible for culling, lod selection, and adding draw calls to each view. This is not amenable to instancing, because the MeshFeatureProcessor needs to know which instances are visible for each view before preparing the draw call and the per-view instance buffers. To support instancing, we need to take a step in the direction of https://github.com/o3de/sig-graphics-audio/issues/85. To keep the scope of the work down, I propose that we preserve the existing functionality of having the culling system add draw items to the views, but also add a new cullable type where the culling system instead adds the user data to a 'visibility list'. Then, after culling but before draw item sorting, the MeshFeatureProcessor can iterate over these visibility lists, prepare the instanced draw calls, and add them to the views. This will require an additional OnEndCulling stage that the RPI can call for each feature processor, which occurs after culling but before draw item sorting and submission.

The full breadth of the work in the culling RFC requires more deep diving into performance. Iterating over a visibility list has its own overhead, compared to just immediately adding a draw packet to a view once an object is deemed visible. Ideally, that overhead can be compensated for by making culling faster since it will have less data to deal with and can be made more cache coherent, and the systems that iterate over visibility lists can organize their own data to be as efficient as possible. But that requires investigation beyond the scope of instancing, so I suggest keeping the existing path working.

### Shader changes

Ultimately, getting a transform for instancing is a straightforward change in the shader.

```hlsl
ShaderResourceGroup DrawSrg : SRG_PerDraw
{
    //! Returns the matrix for transforming points from Object Space to World Space.
    float4x4 GetWorldMatrix(uint instanceId)
    {
        return SceneSrg::GetObjectToWorldMatrix(ViewSrg::m_instanceData[m_rootConstantInstanceDataOffset + instanceId]);
    }
    //...
}

ShaderResourceGroup ObjectSrg : SRG_PerObject
{
    //! Returns the matrix for transforming points from Object Space to World Space.
    float4x4 GetWorldMatrix()
    {
        return SceneSrg::GetObjectToWorldMatrix(m_objectId);
    }
    //...
}

float4x4 objectToWorldTransform;
if(o_useMeshInstancing)
{
   objectToWorldTransform = DrawSrg::GetWorldMatrix(instanceId);
}
else
{
    objectToWorldTransform = ObjectSrg::GetWorldMatrix();
}
```

Any shader that doesn't support instancing can continue using the ObjectSrg to look up the transforms.

Any shaders utilizing the EvaluateVertexGeometry/EvaluatePixelGeometry macros to share code require more changes, since the world transforms are rarely fetched directly in the vertex shader, but instead go through a series of functions to get to the underlying shared function. Thus the instance ID needs to be passed through these functions to get to the place where GetWorldMatrix is ultimately called. We will send out an Impactful Change notice that describes these changes in further detail, so that developers know how to update their shaders if they are using these macros.

### What are the advantages of the feature?

*   Fewer draw calls results in better runtime performance
*   Fewer draw packets need to be created, which improves spawning/level load/entity activation performance.

### What are the disadvantages of the feature?
==========================================

*   Instance buffers need to be updated every frame based on what is visible in each view. This has its own overhead, which is not necessarily outweighed by fewer draw calls.
*   Adds complexity to the MeshFeatureProcessor, especially maintaining both an instanced and non-instanced path.

### How will this be implemented or integrated into the O3DE environment?
=====================================================================

Instancing support will be added to the shaders that ship with O3DE. The MeshFeatureProcessor will be able to identify if a shader supports instancing by checking for the existance of the o_useMeshInstancing shader option, and shaders that do not include this option can continue using the ObjectSrg to look up the world transforms.

### Are there any alternatives to this feature?
===========================================
#### Offline batching/instancing
Some engines support an offline baking process that determines what objects can be instanced together. This could, for example, determine which objects are visible by static views (e.g. shadow casting lights that never move) and prepare the instance buffer in advance. However, such a system could be subverted by dynamic object spawning or material swapping. It requires additional tooling, awareness from content creators, and an additional bake/offline processing step.

#### Vegetation System Interfacing with the MeshFeatureProcessor Directly
The vegetation system can be used to spawn prefabs, which is flexible from a content creation standpoint, but at a technical level the engine sees these objects as just another entity with components. This can incur some overhead from component activation or otherwise be performed more efficiently by interfacing directly with the MeshFeatureProcessor to spawn many instances of the same model + material at once, instead of the MeshFeatureProcessor handling them one at a time as the components activate.

The vegetation system already supports this, but to leverage it, we would need to add the interface for directly interacting with the MeshFeatureProcessor. This is still viable and is compatable with the system being proposed here, and it may be worthwhile to pursue when there is a customer making heavy use of vegetation. However, it is not part of the work described in this RFC.

#### Mesh Merging
An alternate way to reduce draw calls is to merge meshes together into a single mesh. With this, you do not need the models to be the same. However there are more advanced approaches to instancing that also do not require the models to be the same. Additionally, mesh merging can always be done as part of the content creation process if it is absolutely necessary to reduce draw calls.

#### Advanced Instancing Techniques
There are several more advanced iterations on instancing. Programmable Vertex Pulling replaces the use of InputAssembly with explicitely reading vertex data out of a buffer in a vertex shader. This allows unique geometry to share the same instanced draw call, as long as it has some per-instance data that determines where to read the vertex data from. Bindless materials allow for multiple objects that use different materials but the same pipeline state to be instanced by looking up the material data by a material ID instead of binding it directly to the draw call. These are iterations on the instancing described in this RFC that allow for more objects to be instanced.

#### Indirect Draw, GPU Scene Submission, GPU Culling
There are various techniques that allow for an entire scene to be submitted with one or very few draw calls. This is a long term goal, but due to the substantial amount of work required, instancing can serve as an intermediate step that improves performance in the near term. Instancing is also supported on all platforms, while MultiDrawIndirect is not supported on older mobile hardware.

### How will users learn this feature?
==================================

There should be no need for users to learn this feature, as it is implemented automatically under the hood with no user input required. The exception is the settings registry setting to determine the threshold at which instancing takes place, which can be documented in the documentation, and can otherwise typically be left at the default value.

### Are there any open questions?
=============================

**How do we handle non-transform ObjectSrg data?**

The ObjectSrg becomes invalid as soon as instancing takes place. You have two objects sharing the same draw call, but they can't use the same per-object data, because they are different objects. How is this handled?
For transforms, this is trivial. Every object rendered by the MeshFeatureProcessor, no matter what shader it is using, gets an ObjectID from the TransformServiceFeatureProcessor. Per-object transforms are not stored in the ObjectSrg directly. They live in a buffer in the SceneSrg, and the ObjectSrg indexes into this buffer using the Object ID as the index. The TransformServiceFeatureProcessor manages the object-to-world, object-to-world-inverse-transpose, and the object-to-world-history transforms this way.

However, what happens to data the exists directly in the ObjectSrg, which may or may not exist on a per-shader basis? A few approaches come to mind:
1. Only support shaders that use only transforms in their ObjectSrg
-- For the PBR materials that ship with O3DE, this would mean disabling instancing if the 'Use ForwardPass IBL' setting is enabled, since the ReflectionProbeData in the DefaultObjectSrg is only accessed when that setting is enabled.
-- This would be the simplest approach.
-- This would mean mesh instancing is not supported in the low end pipeline, since the low end pipeline always forces the 'Use ForwardPass IBL' setting on.
2. If the ObjectSrg doesn't match, treat the objects as not able to be instanced together. Make an exception for m_objectId, since that is put in the instance data.
-- This provides some flexibility that allows shaders authored outside of the O3DE repo to still support instancing, even if they have custom data in their ObjectSrg
-- A drawback is that this reduces the number of things that can be instanced together when compared with 3 and 4.
3. Treat the Per-Object data as per-instance data.
-- This simplifies the managing of the data, however as previously explained in this RFC, we benefit from having persistant data with a very small amount of dynamic per-instance data that needs to change for each view based on culling results.
4. Quit managing transforms as a single buffer for everything. Instead, create one buffer per ObjectSrg layout, which contains buffers for each piece of data that otherwise exists in the ObjectSrg. The objectId then becomes an index into a buffer for that specific layout.
-- This can be done in a couple of different ways.
---- Manage it in 'user space'. That is the TransformServiceFeatureProcessor can be modified to take a ShaderResourceGroupLayout as input when acquiring an object ID, and it can manage separate buffers. Instead of going through the ObjectSrg directly to update data, systems would need to go through the TransformServiceFeatureProcessor or some other system to find the correct buffer and update the per-object data in the correct place using the object ID.
---- Modify the ShaderResourceGroup system to handle this implicitly. Effectively, we want the same behavior out of the MaterialSrg as well, so that we can do bindless rendering without needing to modify the shader authoring significantly when accessing per-material data via the MaterialSrg. This is a good long term solution, but we need to determine if this can work on all hardware/platforms we support. It also requires more effort, and ideally we don't want this as a dependency for instancing.
------ This is described in the indirect bindings RFC https://github.com/o3de/sig-graphics-audio/pull/17.

My proposal is to start with 1 in the intial commit, followed by support for 2. Then long term implement option 4.

**Do we need custom per-instance data?**

Currently, the assumption is that anything that varies per-instance is really just the per-object transforms. Is there anything custom that a feature might need that truly varies per-instance and can't be managed on a per-draw or per-object basis?

One example would be adding a base color tint property for material variation, allowing for clumps of grass to be tinted to have variation between each clump. With a fully bindless approach, this would be inherent as different materials that use the same pipeline state can still be instanced together. But prior to support for bindless, or on low end platforms that do not support bindless, we might benefit from exposing the ability for a shader to have custom per-instance data that either lives in the dynamic per-instance buffers that change every frame or in the persistant per-instance data that is looked up via object ID.

**At what threshold should objects be instanced together?**

In other engines, it is common for a group of objects to only be instanced together once they reach a certain threshold. For example, if there is only one or two instances of an object, they will be submitted as separate draw calls. If there are more than 8, they will be instanced together.

**Can we send all meshes through the instanced path to reduce the complexity in the MeshFeatureProcessor?**

Eventually, maybe, but only if the instancing path is optimized to a point where there is no noticeable overhead, or we reach a point where nearly everything is being instanced anyways.

**Would it be better to have many small per-instance-group buffers instead of a few large per-view instance buffers?**

There are a few tradeoffs. My intuition is that on large Map for each view to copy data from the CPU to the GPU will be more performant that many small Maps. This could be potentially be circumvented by using the copy queue to update the data, rather than mapping it directly.

The size of each buffer can change from frame to frame, so that presents another problem. If the buffer is resized, then a new buffer is created and the data is copied to the new buffer. If that buffer is bound to an individual draw item, the draw item would need to be updated to refer to the new buffer, adding overhead any time the number of visible instances for a particular group changes too drastically in any of the views. By limiting it to 1 buffer per-view, there are at most N buffers that need to be resized and rebound, where N is the number of views.
