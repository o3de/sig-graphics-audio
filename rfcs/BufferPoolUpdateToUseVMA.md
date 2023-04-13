
# RFC: Atom Buffer Memory System Switch to VMA
## Summary:

We’re proposing to replace the GPU BufferPool allocation backend with AMD’s open source Vulkan and DX12 Memory Allocator libraries.
## What is the relevance of this feature?

Experiments using the Vulkan Memory Allocator library have shown large GPU memory savings from the change. With thousands of small meshes loaded it is possible to save several gigabytes of GPU memory. This has been validated by [CyaNinix](https://github.com/o3de/o3de/issues/11446) and myself.
As an additional benefit we get to use a library that is in use by many developers and should be well validated for correctness in it’s current iteration.
## Current Implementation:

The current BufferPool implementations have 2 strategies. The primary strategy is to suballocate buffers from Pool owned large (default 16MB) pages of GPU memory. When requested buffers are larger than a page or of an unsupported class the 2nd strategy is to pass the allocation to the API and allocate a unique buffer. Depending on the API backend various classes of allocation are excluded from the page suballocation strategy. As an example on Vulkan all geometry is stored in uniquely allocated buffers.
## Proposed Change:

The proposed change is to remove the 2 current strategies and forward all allocations to the new libraries for the DX12 and Vulkan RHI implementations. Metal would continue to use the current code.
This removes the need for each BufferPool to keep track of how many pages it has and defragment and release pages over time, those responsibilities are now passed to VMA/DX12MA. Since VMA/DX12MA is now owning the buffer allocations the background heaps are owned by it and shared between the various BufferPools. This leads to much more efficient use of GPU memory.

At a later data the Metal backend would be changed implement something similar to VMA. A layer that allocates heaps shared between all the BufferPools and uses the heaps to placement allocate buffers as needed.

Once Metal has moved over to heaps we can clean out unused allocator RHI code.

The changes to update the Metal RHI implementation and remove unused code are not part of this RFC.
## Technical design description:

AMD has provided two open source (MIT) GPU memory allocation libraries, one for Vulkan and another for DX12

* Vulkan Memory Allocator (VMA) - https://gpuopen.com/vulkan-memory-allocator/
* D3D12 Memory Allocator (D3D12MA) - https://gpuopen.com/d3d12-memory-allocator/

A prototype implementation of the VMA change has been provided by Cyanide [here](https://github.com/cyanide-studio/o3de/commit/cd760faaf1e9b0f70c2b529d0489abc733a3247b).

The DX12::BufferPool and VK::BufferPool classes will be modified to no longer use their internal BufferMemoryAllocator. The (DX12)MemoryAllocation/(Vulkan)BufferMemoryAllocation classes will forward the allocations to VMA/DX12MA, and the BufferPool instance will wrap the allocation in a BufferMemoryView

Atom will continue to track memory allocations by BufferPool, so the pools function as allocation groupings. The BufferPools will no longer record unused space in each page. Both VMA & DX12MA have API’s that report used vs allocated statistics, these will be plumbed in and reported to the user.
## What are the advantages of the feature?

Large GPU memory savings. [Cyanide saw 2-4GB saved on Vulkan for their game](https://github.com/o3de/o3de/issues/11446). In my tests with their change I can see the loft scene’s memory reduced by 600MB. I see a lower amount because the Loft scene uses fewer larger meshes, scene's with many small meshes see larger gains. In addition the gains are likely to be smaller on DX12 because there has already been some work there to address the memory usage inefficiency of Atom.

AMD’s libraries are in use by a large number of developers and likely to be quite well battle-tested.

Acceleration of O3DE’s memory improvements. It would take many developer-months or work to achieve feature parity with the AMD libraries and, by switching to them, O3DE achieves large memory usage improvements now.

## How will this be implemented or integrated into the O3DE environment?

VMA and DX12MA are both small libraries that have reached stable release points. Each will be copied directly into an external subfolder of the Vulkan and DX12 RHI gems. They will then be set up to be referred to as a 3rdParty::library in cmake.
