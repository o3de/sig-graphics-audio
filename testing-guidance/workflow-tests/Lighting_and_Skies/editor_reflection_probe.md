# Editor component Reflection Probe Workflow Tests
Testing in this area should focus on the Reflection Probe component.

## **General Docs**
* [O3DE Learn: Reflection Probe](https://www.o3de.org/docs/user-guide/components/reference/atom/reflection-probe/)

## Common Issues to Watch For
 - Crashes, soft-locks, hangs when launching Editor.exe or adding a component.
 - When a component settings value is changed, it has a visual impact on the level. 
   - The change can be subtle, but if no change is present log an issue


## Workflows
- Set up a scene with various reflective materials on meshes 
  - Add an entity with the Reflection Probe
  - Adjust settings and observe changes to the shaderball's reflections


### Suggested Time Box: 
15 minutes per platform. Note that the workflow below is the basic steps, feel free to experiment with other settings during the remaining time.


Create a scene with reflective meshes and test the reflection probe
-------------------------------------

| Workflow                                                       | Requests                                                                                                                                                                                                                                                                                                                                                        | Things to Watch For                                                                                                                        |
|----------------------------------------------------------------|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|--------------------------------------------------------------------------------------------------------------------------------------------|
| 1. Launch Editor and create new level                          | Launch Editor.exe, and create a new level with default settings                                                                                                                                                                                                                                                                                                 | <ul><li>Editor launches without issue, and new default level can be created                                                                |
| 2. Set up Shaderball                                           | <ul><li> Select the Shaderball entity <ul><li> Set the material component to a highly reflective material (Example: groundplanemirror) <li>  Verify that 'Exclude from reflection cubemaps' is disabled on the mesh components.                                                                                                                                 | <ul><li>Shaderball can be set up with a reflective material<li> Example                                                                    |
| 3. Set up 3 assets with various levels of reflectiveness       | <ul><li> Add three new entities to the level<ul><li> add a mesh asset to each (example: "hermanubis_high.fbx", "bunny.fbx", and "suzanne.fbx")<li> Give each mesh a different material that is matte or has low reflectiveness (less than 80%)<li> Place one object close to the shaderball and on the same plane, one a medium distance away, one further away | <ul><li>Assets can be added to the level and materials can be assigned<li> all entities are visible in the level <li> Example              |
| 3. Create the reflection probe entity                          | <ul><li> Add a new entity and add the Reflection Probe Component <li> add the required Box Shape component and ensure that all assets in the level fit inside of the box shape <li> Move the Reflection Probe Entity so it is right above the shaderball                                                                                                        | <ul><li>Reflection Probe can be added, and all 4 assets fit within the Reflection Probe's box shape                                        |
| 4. Bake Reflection Probe and observe changes                   | <ul><li> In the Reflection Probe component, click on the "Bake Reflection Probe" button <ls> Wait for it to finish baking <ls> Observe changes to the shaderball                                                                                                                                                                                                | <ul><li>The shaderball's reflections update and should now be reflecting the other meshes in the level.                                    |
| 5. Adjust Settings of the inner extents and observe changes    | <ul><li> In the Reflection Probe component, change inner extents to Height, Length, Width to 9, then back to 1, while viewing changes in the shaderball                                                                                                                                                                                                         | <ul><li> Changes to inner extents should effect the appearance of shaderball reflection, blending should be much more visible when set to 1 |
| 6. Adjust settings in the Reflection Probe and observe changes | <ul><li> In the Reflection Probe component, toggle Parallax Correction on and off while viewing changes in the shaderball                                                                                                                                                                                                                                       | <ul><li> Example of Parallax Correction on <li> Example of Parallax Correction off                                                         |
| 7. Toggle Show Visualization on and off                        | <ul><li> In the Reflection Probe component, toggle Show Visualization on and off                                                                                                                                                                                                                                                                                | <ul><li> Confirm that preview sphere is present when turned on, and disappears when turned off                                             |
---




