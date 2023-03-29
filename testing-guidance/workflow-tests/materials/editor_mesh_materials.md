# Editor Mesh and Materials Workflow Tests

Testing in this area should focus on the Mesh and Material components, settings, overrides, etc. within various editor applications.

## General Docs
* [Mesh Component](https://www.o3de.org/docs/user-guide/components/reference/atom/mesh/)
* [Material Component](https://www.o3de.org/docs/user-guide/components/reference/atom/material/)
* [Scene Settings Meshes Tab](https://www.o3de.org/docs/user-guide/assets/scene-settings/meshes-tab/)

## Common Issues to Watch For

Test guidance will sometimes note specific issues to watch for. The common issues below should be watched for through all testing, even if unrelated to the current workflow being tested.
- Material preview might not update in all scenarios

## Workflows

### Area: In Editor Mesh and Material edit, update and instance modify

**Project Requirements**
* Automated Testing project is used with the MeshBase.prefab in this workflow. If you ignore the prefab any project can be used for exploratory testing.
* Editor applications launched with default or desired RHI options.
* GPU with recent driver updates.
* Windows PC default RHI is DX12 (create a shortcut with additional parameter `-rhi=vulkan` or use other setreg methods for specifying Vulkan on Windows).
* Linux default is Vulkan.


**Editor Platforms:**
* Supported Editor Platforms (see documentation for more information regarding platforms)

**Game Launcher Supported Platforms:**
* See documentation for more information regarding platform support
* Windows (DX12, Vulkan)
* Linux (Vulkan)
* Android (Vulkan)

**Product:** Mesh component and Material component have been exercised in editor

**Suggested Time Box:** 20 minutes per platform.

| Workflow                     | Requests           | Things to Watch For |
|------------------------------|--------------------|---------------------|
| Invoke Mesh test prefab      | <ol><li>Open level `Graphics/base_empty`</li><li>Aquire and invoke prefab [MeshBase.prefab](https://github.com/o3de/sig-graphics-audio/blob/main/testing-guidance/workflow-tests/materials/prefabs/MeshBase.prefab)</li><li>Locate the entity named `Probe` and on the Reflection Probe component entity inspector click Bake Reflection Probe</li><li>Enter game mode and examine the viewport</li></ol> | <ul><li>Level of Detail (LOD) spheres show a progression from blue foreground, yellow, red and finally tan with least polygons</li><li>Reflective sphere does not show the red shaderball in reflection</li><li>Directly viewed 4 shaderballs including the red one are visible</li>![mesh_prefab](https://user-images.githubusercontent.com/26234397/194175057-7a0afa3b-89c3-4a91-b673-0ba4ab1bbe28.png)</ul>  |
| Add/edit Material on Mesh    | <ol><li>Create a new level</li><li>Add Material Component to the shaderball entity</li><li>generate all editable materials using default names</li><li>generate all editable materials but use at least one custom name</li><li>generate a single editable material using default name</li><li>generate a single using a custom name</li><li>generate all editable materials, then remove one or more material assignments</li><li>generate all editable materials, then change some slots to assign other existing materials</li><li>open the material editor from the hamburger menu to edit a material</li><li>edit an assigned material instance with the pop-out editor</li></ol> | <ul><li>All Materials are created, utilized, and applied as expected.</li></ul>  |
---


## Additional Coverage: New Features, Feature Improvements, Areas of Concern for Current LKG
This section should change for each LKG cycle to target new features, feature area improvements, or an area that has been presenting issues and can use additional coverage in the LKG cycle.

Execute the following Workflow Docs:


