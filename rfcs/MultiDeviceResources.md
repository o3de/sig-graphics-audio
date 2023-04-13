# RFC: Multi-device Resources

## Summary

This proposal outlines how resources (buffers, images, etc.) and related classes and structs are prepared for multi-device support in Atom. This is the first necessary step towards multi-device support and already comprises of a significant amount of changes. The main idea here is to introduce a new layer of data structures in the RHI, which support multiple devices and then forward calls to the RHI backends for each device. A non-goal of this proposal is to already support multi-device execution, although changes to the frame graph already have to be made for this to work as well as the transition from multi-device objects to single-device execution, which currently happens when advancing from frame graph compilation to execution. A side goal is to keep the backend and RPI code as unaffected as possible to (1) avoid having to implement multi-device features multiple times within every backend and (2) avoid all higher-level code (RPI and above, as well as any third-party Gems) having to change much. However, this is not completely possible to avoid.

## What is the relevance of this feature?

The ultimate goal is to support multi-device execution within Atom. This could for example be used to render stereoscopic images for virtualy reality headsets on two devices simultaneously, i.e., one device per eye. The first step towards this goal is removing the globally accessible device. When looking into this, one quickly finds that in most places, one does not know on which device a resource should then be allocated on, since there is not enough context for the execution. For example, when mesh data is loaded and uploaded to the device, one does not know which device to use, since one doesn't know which render passes will use that mesh and where they will be executed. It is actually likely, that the data is required on multiple devices. Therefore, support for having resources on multiple devices is necessary and this needs to be established before any execution on multiple devices can be accomplished.

## Feature/Technical Design Description

The basic idea of the technical solution is to introduce new classes and structs in the RHI which support multi-device resources. This includes most of the classes derived from `DeviceObject` with few exceptions. The base class for these new classes will be a `MultiDeviceObject`, which stores a device mask, denoting on which devices this resource will be present. This device mask is a 32-bit field, where each bit represents one device. The `RHISystem` will support an array of multiple devices where each device can be accessed through an index (corresponding to the bit position in the device mask, e.g., the least significant bit corresponds to the device with index 0). Internally, each multi-device object will then contain a map, which maps from the device index to the corresponding `DeviceObject`-derived object. An example is given in the following UML diagram, where we show this for the buffer resources.

![UML Diagram showing DeviceObject <|- DeviceResource <|- DeviceBuffer ----<> MultiDeviceBuffer -|> MultiDeviceResource -|> DeviceObject](MultiDeviceResourcesUML.png)

Any API calls on a multi-device object will be forwarded to all managed device objects, unless otherwise specified, e.g., through providing a mask, selecting which devices should apply the call. The masks may also contain bits for devices that are not actually available, which will be ignored. This allows us to have an `AllDevicesMask` mask, where all bits are set independently of how many are actually available. The default device has device index 0 and will be also available through a `DefaultDeviceMask`, where only the least significant bit is set. The following is a list of all the classes, where both a single-device and a multi-device version will be available. All "final" classes, i.e., those that are not further derived from, contain a map in the respective multi-device class, which maps to the corresponding single-device class objects.

    Object
        Fence
        IndirectBufferSignature
        PipelineLibrary
        PipelineState
        RayTracingBlas
        RayTracingBufferPools
        RayTracingPipelineState
        RayTracingShaderTable
        RayTracingTlas
        Resource
            Buffer
            Image
            Query
            ShaderResourceGroup
        ResourcePool
            BufferPoolBase
                BufferPool
            ImagePoolBase
                ImagePool
                StreamingImagePool
                SwapChain
            QueryPool
            ShaderResourceGroupPool
        TransientAttachmentPool

### DeviceObject classes without multi-device version

There are a few classes in the `DeviceObject` hierarchy, which do not get a multi-device class version. The first few classes of these are `ResourceView`-derived classes, i.e., `BufferView` and `ImageView`. Outside their RHI backend derivatives, these two only have two members each, the corresponding resource (buffer or image) and a descriptor, which is device independent. Since the memory management regarding views is quite complicated as is, as the resources store references to their views and a layer of multi-device objects would complicate this even more, we opt to change the higher-level interface at this point. Instead of a multi-device view, we use the combination of the resource and descriptor instead of binding them together.

The classes `AliasedHeap`, `AliasedAttachmentAllocator` and `CommandQueue` seem to not need a multi-device version, since they are never used outside of a single-device context. However, the class `CommandQueue` will actually handle multi-device classes and translate them to single-device objects, without needing a separate multi-device version, as we do not currently plan to be able to submit work to multiple devices at the same time. More details will be discussed further below.

### Other classes and structs with both a single-device and multi-device version

Other classes and structs, which do not themselves derive from `DeviceObject`, need multi-device support as well. Most of them are descriptors, requests and items (for copy, dispatch and draw). They need a multi-device version, because one or more of their members reference some `DeviceObject` derivative and are used both in the higher-level API, where we now want to use multi-device objects, as well as in the RHI backends, where we need their single-device version. In most cases, those classes and structs are passed as parameters to some methods of the resource classes. Therefore, the conversion happens mostly inside the multi-device classes, when they loop over their single-device object map and forward the call.

Some of the multi-device classes need a map as well in order to be able to be used properly or more efficiently. These classes are `ImageSubresourceLayoutPlaced`, `IndirectBufferWriter`, `PipelineLibraryDescriptor`, `ShaderResourceGroupData` and the `*Item` classes. The derivation of `ImageSubresourceLayoutPlaced` from `ImageSubresourceLayout` and their polymorphic usage actually makes it difficult to have a multi-device version of them without using unnecessarily performance-impacting dynamic casts to figure out which one is actually behind a pointer. Therefore, we propose to merge the two and, if necessary at all, add a boolean, whether the offset is used or not.

Here is a list of the affected data structures:

Classes:

    IndexBufferView
    IndirectBufferView
    IndirectBufferWriter
    RayTracingBlasDescriptor
    RayTracingTlasDescriptor
    RayTracingPipelineStateDescriptor
    RayTracingShaderTableDescriptor
    ResourceEventInterface
    ResourceInvalidateBus
    ShaderResourceGroupData
    StreamBufferView

Structs:

    BufferInitRequest
    BufferMapRequest
    BufferMapResponse
    BufferStreamRequest
    CopyBufferDescriptor
    CopyImageDescriptor
    CopyBufferToImageDescriptor
    CopyImageToBufferDescriptor
    CopyQueryToBufferDescriptor
    CopyItem
    DispatchArguments
    DispatchItem
    DispatchRaysArguments
    DispatchRaysItem
    DrawArguments
    DrawItem
    DrawItemProperties
    ImageInitRequest
    ImageSubresourceLayoutPlaced
    ImageUpdateRequest
    IndirectArguments
    IndirectBufferSignatureDescriptor
    PipelineLibraryDescriptor
    RayTracingGeometry
    RayTracingTlasInstance
    RayTracingShaderTableRecord
    StreamingImageInitRequest
    StreamingImageExpandRequest
    TransientAttachmentPoolDescriptor

### Switching from multi-device to single-device for execution

At some point we have to switch from multi-device resources to single-device resources. The last chance for that is when submitting work to a specific device. Therefore, we propose (at least for now) to use a single frame graph for multiple devices. Doing this should help to synchronize the work between multiple devices, by being able to use already present synchronization mechanisms in the frame graph, just extended for multiple devices. This is not fleshed out in detail yet and not the goal of this proposal, but in order to get this code to work at all, we need to have some point of transition. For now, we basically transition between frame graph compilation and execution. Since we are not actually submitting work to multiple devices yet, our plan is to always transition to the default device at the moment.

As mentioned before, the multi-device versions of `*Item` classes contain a map to their single device version. This is not strictly necessary, since a conversion could simply be done when necessary, but since they are potentially kept for multiple submissions and even multiple frames, it is more efficient to cache the single-device structs as well. The conversion from the multi-device to single-device version happens in the `RHI::CommandList::Submit` methods, which forward the calls to the corresponding backend.

### Planned git history

We intend to divide the pull request for this proposal into four commits, which should simplify the review. Also, looking at the changes as a whole, git will not detect renaming of files properly, which it still does, when looking at the first commit only for example.

1. The first commit is "simply" a rename of all relevant structs and classes to `Device*`. Since this can mostly be done by search and replace with little manual intervention, there should not be much that could fail here.

2. In a smaller, second commit we actually initialize multiple devices in the `RHISystem`. We introduce a new command line switch (`--rhi-multiple-devices`) to initialize all available devices, otherwise only one device is initialized as before. Note again that we will allocate resources on these devices, but only submit commands to the default device.

3. The next commit introduces the new multi-device classes with the original naming of the classes that were renamed in the first commit.

4. Finally, in the last commit, we will rewrite everything to use the new multi-device RHI classes rather than the `Device*` classes. This concerns mainly the RPI classes of those resources, but also direct uses of the RHI classes, which constitute a considerable amount of the changeset. This will be the critical commit to check, if everything is working.

## What are the advantages of the feature?

* The `RHI::Device` is not a global anymore and it is possible to use multiple devices.
* It is not necessary to know in advance, which devices a resource will be used on, since it can be easily allocated on multiple devices.
* The API of how to use the RHI resources hardly changes and only few changes in the RHI backends are necessary.

## What are the disadvantages of the feature?

* Adding a layer in the RHI will have some minimal performance impact.
* Some complexity is added to the RHI layer that future developments have to consider.
* Some changes at higher levels (RPI, FeatureProcessors, Gems) are necessary.

## How will this be implemented or integrated into the O3DE environment?

The feature will be developed in a branch parallel to the development branch, until properly tested and ready for merging. When merged, third-party projects will have to make some smaller adjustments to the new API changes. During the testing and development phase, we will need help for MacOS/Metal and mobile plattforms, since we do not have the necessary hardware and software available to test this.

## Are there any alternatives to this feature?

We tried an approach to introduce multi-device support in the RPI resources, which turned out not to be possible due to some objects needing multi-device support in the RHI. Another approach would be to support multi-device resources directly within the RHI resources without an additional layer at the cost of having to implement everything for each backend which, would probably add unnecessary code duplication at hardly any performance benefit. It would also be possible not to have multi-device resources at all, but that would complicate any further multi-device developments, since resources then have to be held on multiple devices at a much higher level in the code base, leading to the inconvenient case that every feature that should support multiple devices has to keep track of resources on each device manually. This would likely also lead to a lot of code duplication.

## How will users learn this feature?

Most of these changes should not affect users at all. Any changes necessary to higher level code should be straightforward and easy to implement. The required changes will be documented properly and changes in o3de itself and the atom-sampleviewer will provide plenty examples.

## Are there any open questions?

The naming of the classes and structs is still something that is open for debate. The least amount of changes in the higher level code base would happen if the multi-device versions would take over the names of the single-device versions and single-device versions get a new name. I.e., the current `Buffer` class will be renamed to `DeviceBuffer` and the multi-device version of it will simply be named `Buffer`. An alternative would be to call the classes `DeviceBuffer` and `MultiDeviceBuffer` respectively.

