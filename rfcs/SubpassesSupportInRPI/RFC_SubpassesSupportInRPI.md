# A Solution To Expose Subpasses To The RPI

**Why is this important?**  
In terms of getting maximum graphics performance for the least amount of bandwidth on GPUs
architected as Tile Based Rasterizers(TBR, for short) you need to group as many Raster Passes
as possible into a series of Subpasses.

Most Mobile and XR devices have TBR GPUs. Both bandwidth and power consumption should be minimized.
Merging several Raster Passes as a sequence of Subpasses is one option available that typically achieves this goal. To learn more about Subpasses (In Vulkan):  
1. https://arm-software.github.io/vulkan_best_practice_for_mobile_developers/samples/performance/render_subpasses/render_subpasses_tutorial.html  
2. https://developer.qualcomm.com/sites/default/files/docs/adreno-gpu/snapdragon-game-toolkit/gdg/gpu/best_practices_tiling.html  
3. https://www.saschawillems.de/blog/2018/07/19/vulkan-input-attachments-and-sub-passes/


# O3DE Background History - Vulkan RHI
In order to understand the proposed solution, it is very important to go over a few touch points on how the RPI and RHI work together to define the Frame Graph and how it gets executed.  
  
Each time the RPI initializes a Shader, it also instantiates a Pipeline State Object (aka PSO) for the given Shader (for a given Shader Variant to be precise). Vulkan requires a VkRenderPass to instantiate a PSO:  
```
// Internally, the RHI creates or re-uses a VkRenderPass to create ShaderPSO.
ShaderPSO = Shader->AcquirePipelineState()
```
In the pseudo code mentioned above let's assume the Vulkan RHI created a private `VkRenderPass0`.

 Later, when the Vulkan RHI compiles the Scopes, it creates a new `VkRenderPass1` and when submitting Draw commands the following pseudo code happens:
```
CmdBeginRenderPass(VkRenderPass1)
    CmdBindPSO(ShaderPSO)
    CmdDraw(...)
CmdEndRenderPass() 
```
And here comes the first lesson, Vulkan does NOT require `VkRenderPass0` and `VkRenderPass1` to be identical, BUT they have to be **compatible**. Compatible usually means that both VkRenderPasses must be created by referring to the exact same type and amount of Frame Attachments, and also means that **if there are Subpass Dependency declarations, those declarations must be identical**, otherwise validation errors or crashes may occur.  

In the example above, `VkRenderPass0` is created with help of the API `AZ::RHI::Vulkan::RenderPass::ConvertLayoutAttachment()`, while `VkRenderPass1` is created by a different code path: `AZ::RHI::Vulkan::RenderPassBuilder`. The key takeaway is that because there was no subpass support exposed to the RPI it was relatively easy that two different code paths end up creating compatible VkRenderPasses.  

Let's go over a slightly more complicated scenario where two consecutive Raster Passes are executed by the RHI.
Also let's assume each Raster Pass have their own shader:  
```
// Somewhere in the RPI at initialization time:

// Shader used under Raster Pass A
ShaderPSOA = ShaderA->AcquirePipelineState() // Internally VkRenderPassA is created.

// Shader used under Raster Pass B
ShaderPSOB = ShaderB->AcquirePipelineState() // Internally VkRenderPassB is created.
... 
A few seconds later
...
CmdBeginRenderPass(VkRenderPassC) //Scope of Raster Pass A
    CmdBindPSO(ShaderPSOA)
    CmdDraw(...)
CmdEndRenderPass() 
CmdBeginRenderPass(VkRenderPassD) //Scope of Raster Pass B
    CmdBindPSO(ShaderPSOB)
    CmdDraw(...)
CmdEndRenderPass() 
```
The example above worked fine because `VkRenderPassC` was compatible with `VkRenderPassA`, and `VkRenderPassD` was compatible with `ShaderPSOB`.  
  
Let's go over the final example, where we assume that `Raster Pass A` & `Raster Pass B` are mergeable as subpasses:  
```
// Somewhere in the RPI at initialization time:

// Shader used under Raster Pass A (as Subpass 0)
ShaderPSOA = ShaderA->AcquirePipelineState() // Internally VkRenderPassA is created.

// Shader used under Raster Pass B (as Subpass 1)
ShaderPSOB = ShaderB->AcquirePipelineState() // Internally VkRenderPassB is created.
... 
A few seconds later
...
CmdBeginRenderPass(VkRenderPassC) //Merged Scope of Raster Pass A & B
    CmdBindPSO(ShaderPSOA)
    CmdDraw(...)
CmdNextSubpass() //Scope of Raster Pass B
    CmdBindPSO(ShaderPSOB)
    CmdDraw(...)
CmdEndRenderPass() 
```
For this final example to work well, all VkRenderPasses must be compatible: VkRenderPassA, VkRenderPassB and VkRenderPassC, and **most importantly** the VkSubpassDependency's must be identical. The challenge is that VkSubpassDependency is nothing more than a combination of Bit Flags and it can be tedious to write two different code paths that generate the exact same Bit Flags.  


# The Solution
  
There are many considerations in this solution:  

## Solution to Shareable Render Attachment Layouts
So far each RPI::Pass has been constructing their Render Attachment Layouts in isolation (See `AZ::RPI::RenderPass::GetRenderAttachmentConfiguration()`) and it is precisely the Render Attachment Layout what contains all the data that describes Render Attachments and dependecies for each subpass.  
The new change in the Pass asset is that `PassData` contains a new field called `"MergeChildrenAsSubpasses"`, which the `RPI::ParentPass` class can use to merge/combine a list of `RPI::RasterPass` as subpasses. The benefit of delegating the responbility to `RPI::ParentPass` is that it can sequentially build a single AZ::RHI::RenderAttachmentLayout for all the child passes that it is supposed to merge.  

The major piece of work is encompassed by the new function:   `AZ::RPI::ParentPass::CreateRenderAttachmentConfigurationForSubpasses()`.  
This function is called just after BuildInternal() is called for all Child passes.  
The reason this is the best time to call this function is because the RPI has not called `AZ::RPI::RenderPass::GetRenderAttachmentConfiguration()` yet for any of the PSOs.  
  
Also  `AZ::RPI::RenderPass::GetRenderAttachmentConfiguration()` was changed to `virtual`. This allows RPI::RasterPass to override `GetRenderAttachmentConfiguration()` and returned the Render Attachment Configuration that was built with help of `RPI::ParentPass`.

## Solution to Subpass Dependencies
As mentioned already, it is a burden to have two different code paths, and guarantee that they both will create the exact same set of bitflags that make a set of VkSubpassDepency's. Another problem is that We need to have APIs that the RPI can use, which should decouple from the intricacies of the Vulkan RHI.  

A helper class called `AZ::Vulkan::SubpassDependencyHelper`, which is only visible to `AZ::Vulkan::RenderPass`, will run the algorithm that keeps track
for *Source* and *Destination* stage flags for each attachment. This algorithm follows the same ideas implemented in `AZ::Vulkan::FrameGraphCompiler::QueueResourceBarrier(...)`
and `RenderPassBuilder::AddScopeAttachments(...)` to keep track of the proper pipeline stage flags. This helper class is used inside `AZ::Vulkan::RenderPass::ConvertRenderAttachmentLayout(...)` to define all the subpass dependencies.  

Additionally, the following classes were extended to accept `AZ::RHI::ScopeAttachmentAccess` and `AZ::RHI::ScopeAttachmentStage` flags because they are necessary to accurately
define the pipeline stage flags in `VkSubpassDependency`:
- AZ::RHI::RenderAttachmentLayoutBuilder
- AZ::RHI::RenderAttachmentLayoutBuilder::RenderAttachmentEntry
- AZ::RHI::RenderAttachmentLayoutBuilder::SubpassAttachmentEntry
- AZ::RHI::RenderAttachmentDescriptor
- AZ::RHI::SubpassInputDescriptor

  
The following timeline diagram illustrates how the `RHI::RenderAttachmentLayoutNotificationsInterface` interface is used during `RPI::ParentPass::BuildInternal()`:  

![RPI::ParentPass Subpass Layout constrution](ParentPass_SubpassLayout.PNG)
  
It was also found that even if two independent arrays of `VkSubpassDependency` are identical, except for the order of the items in each list, it will trigger validation errors
from the Vulkan Validation layer. So ultimately a new function was added to `AZ::Vulkan::RenderPass::Descriptor` called `MergeSubpassDependencies()`:  
```cpp
    struct AZ::Vulkan::RenderPass::Descriptor
    {
        ...
        //! Necessary to avoid validation errors when Vulkan compares
        //! the VkRenderPass of the PipelineStateObject vs the VkRenderPass of the VkCmdBeginRenderPass.
        //! Even if the subpass dependencies are identical but they differ in order, it'd trigger validation errors.
        //! To make the order irrelevant, the solution is to merge the bitflags. 
        void MergeSubpassDependencies();
    };
```

Ultimately `MergeSubpassDependencies()` will be called by the following two code paths, producing identical `VkSubpassDependency` arrays:
```cpp
...\Gems\Atom\RHI\Vulkan\Code\Source\RHI\RenderPass.cpp
AZ::Vulkan::RenderPass::ConvertRenderAttachmentLayout(Device& device,
            const RHI::RenderAttachmentLayout& layout,
            const RHI::MultisampleState& multisampleState)
{
    ...
    renderPassDesc.MergeSubpassDependencies();
    return renderPassDesc;
}

...\Gems\Atom\RHI\Vulkan\Code\Source\RHI\RenderPassBuilder.cpp
RHI::ResultCode AZ::Vulkan::RenderPassBuilder::End(RenderPassContext& builtContext)
{
    ...   
    m_renderpassDesc.MergeSubpassDependencies();
    builtContext.m_renderPass = m_device.AcquireRenderPass(m_renderpassDesc);
    ...
    return RHI::ResultCode::Success;
}
```
  
