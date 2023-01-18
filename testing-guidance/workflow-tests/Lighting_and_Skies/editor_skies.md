# Editor component Skies Workflow Tests
Testing in this area should focus on the Editor HDRi Skybox, and physical sky components.

## **General Docs**
* [Feature Workflows](https://github.com/o3de/sig-graphics-audio/wiki/Feature-Workflows---Atom-Test-Plans)

## Common Issues to Watch For
 - Crashes, soft-locks, hangs when launching Editor.exe or adding a component.
 - All materials and assets load properly and has the expected visual impact.
   - Examples: 
     - no asset load errors when selecting materials or assets
     - no assets that appear completely black or obviously distorted when applied to a component 
 - When a component settings value is changed, it has a visual impact on the level. 
   - The change can be subtle, but if no change is present log an issue

## Workflows

- Create an entity and attach an HDRi Skybox Component. 
  - Change various aspects of the skybox and ensure they show up properly in editor.
- Create an entity with the Physical Sky component.
  - Change various aspects of the Physical Sky component and ensure they show up properly in editor.

### Suggested Time Box: 
15 minutes per workflow per platform. Note that the workflow below is the basic steps, feel free to experiment with other settings during the remaining time.


HDRi Skybox Workflow Steps
-------------------------------------

Global Skylight Image-Based Lighting (IBL) - This rendering technique lights a scene using by capturing and omnidirectional light information from an image. This is then projected on a dome to simulate lighting for objects present in the scene.

High dynamic range image (HDRi) Skybox - An HDR image of the sky, horizon, and distant background objects placed into a cube map texture. This is then projected onto a large cube that surrounds the level geometry.

| Workflow                                                           | Requests                                                                                                                                                                                                     | Things to Watch For                                                                                       |
|--------------------------------------------------------------------|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-----------------------------------------------------------------------------------------------------------|
| 1. Launch Editor and create new level                              | Launch Editor.exe, and create a new level with default settings                                                                                                                                              | <ul><li>Editor launches without issue, and new default level can be created </li></ul>                    |
| 2. Delete "GlobalSky" Entity                                       | Delete the existing "GlobalSky" Entity present in the level                                                                                                                                                  | <ul><li>Entity is removed from the level</li></ul>                                                        |
| 3. Create a new child Skytest with Global Skylight and HDRi Skybox | Add new child entity and name it Skytest, then add the Global Skylight (IBL) and HDRi Skybox components                                                                                                      | <ul><li>Components can be added to the newly created entity</li></ul>                                     |
| 4. Adjust Global Skylight within entity inspector                  | Navigate to Global Skylight within entity inspector. Set Diffuse image to: "default\_iblskyboxcm\_ibldiffuse.exr.streamingimage", and set Specular to: "default\_iblskyboxcm\_iblspecular.exr.streamingimage | <ul><li>Diffues and Specular settings can be updated and changes are reflected in the viewport. </li></ul> |
| 5. Adjust HDRi Skybox entity settings within the entity inspector. | Navigate to HDRi Skybox within the entity inspector. Set Cubemap texture to default\_iblskyboxcm.exr.streamingimage                                                                                          | <ul><li>Cubemap Texture can be set and changes are reflected in the viewport. </li></ul>                  |
| 6. Adjust ground plane component material                          | Navigate to the ground plane component in entity inspector. Change the material to: metal\_gold\_matte                                                                                                       | <ul><li>Observe reflections and Highlights updated in the viewport</li></ul>                              |
| 7. Adjust focus value on HDRi SKybox and Global Skylight           | Within the entity inspector, adjust the focus value on the HDRi Skybox and the Global Skylight. For further testing, repeat steps with different cubemaps, and ground plane mats, and try other approaches.  | <ul><li>Observe changes in the viewport before and after focus value is updated.</li></ul>                |
---





Physical Sky component Workflow Steps
-------------------------------------

| Workflow                                            | Requests                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                     | Things to Watch For                                                                                                   |
|-----------------------------------------------------|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-----------------------------------------------------------------------------------------------------------------------|
| 1. Launch Editor and create new level               | Launch Editor.exe, and create a new level with default settings                                                                                                                                                                                                                                                                                                                                                                                                                                                              | <ul><li>Editor launches without issue, and new default level can be created </li></ul>                                |
| 2. Set up Shaderball and add 3 entities with meshes | <ul><li>Navigate to the Shaderball Entity within the entity inspector. <li>Set the material component to "basic_m10_r-2" <li> Add three new entities to the level, add a mesh asset to each (such as: "lucy_high.fbx", "bunny.fbx", and "suzanne.fbx") <li> Add a different material to each, try to vary the reflective and roughness of the materials <li> Place one object close to the shadershphere and on the same plane (about 1.5-2.5 meters), one at medium (about 3-4 meters), one further away (about 8-9 meters) | <ul><li>User is able to populate the level with entities with different materials and placements on plane. </li></ul> |
| 3. Set up Physical Sky Component                    | First delete the Global Sky element, and then add a new entity to the level, give it a Physical Sky component                                                                                                                                                                                                                                                                                                                                                                                                                | <ul><li>Global sky element can be removed and replaced with Physical Sky component.<li> example:![Phys_sky](https://user-images.githubusercontent.com/41299597/213257757-7c0a171b-b35b-4444-82f8-089616beb663.png)</li></ul>                          |
| 4. Adjust the Sky Intensity slider                  | <ul><li> Change the Sky Intensity slider and observe change in level <ul><li>Change Sky back to Intensity to 4.0                                                                                                                                                                                                                                                                                                                                                              | <ul><li>Diffues and Specular settings can be adjusted and changes are reflected in the viewport. <li> Examples: <li> Sky Intensity 0.65  ![Sky_intensity_065](https://user-images.githubusercontent.com/41299597/213291494-b9a0545c-07b7-4bd0-b32b-c1bd9456c716.png)<li> Sky Intensity 4.5  ![Sky_intensity_4 5](https://user-images.githubusercontent.com/41299597/213291573-0765a859-4004-417d-9b37-6b795f1bcd3d.png) </li></ul>           |
| 5. Adjust the Sun Intensity slider                  | <ul><li>Change the Sun Intensity slider and observe change in level<ul><li>Sun Intensity 3.5 <li> Sun Intensity -4 <li> Change Sun Intensity back to 8                                                                                                                                                                                                                                                                                                                                                                       | <ul><li>Cubemap Sun intensity can be adjusted and changes are reflected in the viewport. </li></ul>                   |
| 6. Adjust Turbidity value                           | <ul><li>Adjust Turbidity value from 1-10, observe color change in sky<ul><li>Set Turbidity back to 1                                                                                                                                                                                                                                                                                                                                                                                                                         | <ul><li>Turbidity value can be adjusted and the color change in sky are reflected in the viewport                     |
| 7. Adjust Sun Radius Factor                         | <ul><li> Adjust Sun Radius Factor from 0.1 - 2.0, observe change in sun <ul><li> Change Sun Radius Factor back to 1.0                                                                                                                                                                                                                                                                                                                                                                                                        | <ul><li>Sun Radius Factor can be adjusted and the changes are reflected in the viewport.                              |
| 8. Enable and Adjust Fog values                     | <ul><li> Enable Fog in the entity inspector settings and observe changes when adjusting settings <ul><li>Change fog color <li> increase top height <li> increase bottom height                                                                                                                                                                                                                                                                                                                                               | <ul><li> Fog can be enabled, and changes to color or the height are reflected in the viewport.                         |
---





Create a Modular Physical Sky Workflow Steps
-------------------------------------

| Workflow                                             | Requests                                                                                                                                                                  | Things to Watch For                                                                    |
|------------------------------------------------------|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------|
| 1. Launch Editor and create new level                | Launch Editor.exe, and create a new level with default settings                                                                                                           | <ul><li>Editor launches without issue, and new default level can be created </li></ul> |
| 2. Create Entity for the Physical sky                | <ul><li> Create an entity, and name it Physical Sky <ul><li> zero out the transforms<li> Add Physical Sky Component<li> Add Deferred Fog Component <li>  Add PostFX Layer | <ul><li>All three components can be added to the Physical Sky Entity </li><ul> |
| 3. Add the Sun as a child to the Physical sky Entity | <ul><li>Add the Sun under the Physical sky, making the sun the child of the Physical Sky <ul><li> Zero out the Sun Transforms                                             | <ul><li>Now you will have a modular Physical Sky                                       |
---




