# Editor component Decal Workflow Tests
Testing in this area should focus on the Editor Decal component.

## General Docs
* [Feature Workflows](https://github.com/o3de/sig-graphics-audio/wiki/Feature-Workflows---Atom-Test-Plans)

## Common Issues to Watch For
 - Crashes, soft-locks, hangs when launching Editor.exe or adding a component.
 - Corruption or unexpected distortion of the decal image

## Workflows

### Area: Decal Workflow

**Project Requirements**
1. You have built the O3de project:  
* https://github.com/o3de/o3de
2. You have verified you are set to the AutomatedTesting project
3. Launch AssetProcessor.exe and wait for it to process all assets 


**Editor Platforms:**
* Windows (DX12 and Vulkan)
* Linux (Vulkan only)

**Product:** 
 - You have completed the workflow and logged any issues found.

**Suggested Time Box:** 15 minutes per platform. Note that the workflow below is the basic steps, feel free to experiment with other settings during the remaining time.

| Workflow                     | Requests           | Things to Watch For |
|------------------------------|--------------------|---------------------|
| 1. Launch Editor | Launch Editor.exe, and create a new level with default settings | <ul><li>Editor launches without issue, and new default level can be created </li></ul>  |
| 2. Add two entities with a decal component | Add two new entities to the level, then add a Decal component | <ul><li>Entity created, and components added successfully</li></ul>  |
| 3. Set a material to each component | Assign one of the *.material files to each of the Decal component from: AutomatedTesting\Gem\Sponza\Assets\objects, move the decal entities so they project on the ground plane | <ul><li>Material can be assigned to each decal component, material image projects without distortion. Example:![Two_decals_ground_plane](https://user-images.githubusercontent.com/79114701/198399210-c44941a8-159a-49b0-8c9d-020e1805dfd0.png)</li></ul>  |
| 4. Adjust opacity of each decal | Adjust the opacity slider to between 0 and 1 for each decal, observe results, then set both back to 1 when done | <ul><li>Opacity of decal changes based on setting value (0 is fully transparent 1 is fully opaque), image projects without distortion. Example:![Decal_opactity](https://user-images.githubusercontent.com/79114701/198399249-54eac294-88b9-4b59-b5d8-d9603cfb2b94.png)</li></ul>  |
| 5. Adjust the Attenuation Angle on each decal | Move one decal so it is projecting on the shaderball, and rotate the other decal entity about 20 degrees on the X and/or Y axis. Adjust the Attenuation Angle slider to between 0 and 1 for each decal, observe results, then set both back to 1 when done | <ul><li>Opacity changes based on angle of projection, material image projects without distortion. Example:![Decal_attenuation_angle](https://user-images.githubusercontent.com/79114701/198399375-aa018918-787b-4855-aa14-a05caa8a5962.png)</li></ul>  |
| 6. Adjust the Normal Map Opacity | Select the decal projecting on the shaderball. Adjust the Normal Map Opacity slider to between 0 and 1 for the decal, observe results, then set it back to 1 when done | <ul><li>Opacity changes where the decal projects on the curved surfaces of the shaderball, material image projects without distortion. Example: ![Normal_map_opacity](https://user-images.githubusercontent.com/79114701/198399405-9738e820-11de-4699-b84a-96f0b367719a.png)</li></ul>  |
| 7. Adjust the Sort Key | Select the decal you rotated earlier, set the rotation values back to 0,0,0, then move it so it projects on the same surface as the other decal. Adjust the Sort Key value of the decal to above and below 16 (which is the default value that the other decal should be set to) observe results. | <ul><li> Decal with the higher sort key value appears on top, material image projects without distortion. Example:![Decal_sort_key1](https://user-images.githubusercontent.com/79114701/198399428-f172162d-264f-45d7-9bf3-6688a9b0a8cd.png)![Decal_sort_key2](https://user-images.githubusercontent.com/79114701/198399443-450a7299-41eb-4e84-9693-57cc2e45d942.png)</li></ul>  |
---

# Editor component Cloth Workflow Tests
Testing in this area should focus on the Editor Cloth component.

## General Docs
* [Additional Cloth workflow tests](https://github.com/o3de/sig-graphics-audio/wiki/Additional-Cloth-workflow-tests)
* [Cloth Component](https://www.o3de.org/docs/user-guide/components/reference/physx/cloth/)
* [Cloth for Actor components](https://www.o3de.org/docs/user-guide/interactivity/physics/nvidia-cloth/actors/)
* [Cloth for Mesh components](https://www.o3de.org/docs/user-guide/interactivity/physics/nvidia-cloth/meshes/)

## Common Issues to Watch For
 - Crashes, soft-locks, hangs when launching Editor.exe or adding a component.

## Workflows

### Area: Cloth Workflow

**Project Requirements**
1. You have built the O3de project:  
* https://github.com/o3de/o3de
2. You have verified you are set to the AutomatedTesting project
3. Launch AssetProcessor.exe and wait for it to process all assets 


**Editor Platforms:**
* Windows (DX12 and Vulkan)
* Linux (Vulkan only)

**Product:** 
 - You have completed the workflow and logged any issues found.

**Suggested Time Box:** 25 minutes per platform. Note that the workflow below is the basic steps, feel free to experiment with other settings during the remaining time.

| Workflow                     | Requests           | Things to Watch For |
|------------------------------|--------------------|---------------------|
| 1. Launch Editor, load the cloth mesh test level | Launch Editor.exe, and load the level "NvCloth_AddClothSimulationToMesh" | <ul><li>Editor launches without issue, and test level loads </li></ul>  |
| 2. Adjust cloth mesh prefab settings  | Adjust cloth component properties for mesh prefabs, enter gameplay, and observe results. Repeat several times | <ul><li>Cloth properties change based on the settings changed (see [Cloth Component](https://www.o3de.org/docs/user-guide/components/reference/physx/cloth/) for property descriptions) Example: ![Cloth_mesh](https://user-images.githubusercontent.com/79114701/198399542-86501d72-5e4f-4a3e-8aff-9d0099d1a9f2.png)
 </li></ul>  |
| 3. Launch Editor, load the cloth actor test level | Launch Editor.exe, and load the level "NvCloth_AddClothSimulationToActor" | <ul><li>Editor launches without issue, and test level loads </li></ul>  |
| 4. Adjust cloth prefab settings on chicken actor | Adjust cloth component properties for actor prefabs, enter gameplay, and observe results. Repeat several times | <ul><li>Cloth properties change based on the settings changed (see [Cloth Component](https://www.o3de.org/docs/user-guide/components/reference/physx/cloth/) for property descriptions) Example: ![actor_Cloth](https://user-images.githubusercontent.com/79114701/198399600-4401e7d7-ce80-47ce-a118-6fcff37ac286.png)</li></ul>  |
| 5. Run additional cloth workflows if time allows | Review above general cloth docs and if run the following [Additional Cloth workflow tests](https://github.com/o3de/sig-graphics-audio/wiki/Additional-Cloth-workflow-tests) | <ul><li>You can import meshes from DCC tools and apply cloth properties </li></ul>|
---

