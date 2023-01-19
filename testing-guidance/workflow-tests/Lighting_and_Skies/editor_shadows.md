# Editor component Shadows (Spotlight / Cascaded Shadow Maps)  Workflow Tests
Testing in this area should focus on the Shadows (Spotlight / Cascaded Shadow Maps) components.

## **General Docs**
* [O3DE Learn - Light Documentation](https://www.o3de.org/docs/user-guide/components/reference/atom/light/)
* [O3DE Learn - Directional Light Documentation](https://www.o3de.org/docs/user-guide/components/reference/atom/directional-light/)

## Common Issues to Watch For
 - Crashes, soft-locks, hangs when launching Editor.exe or adding a component.
 - When a component settings value is changed, it has a visual impact on the level. 
   - The change can be subtle, but if no change is present log an issue


## Workflows

- Observe shadows with various lights and intensities
  - Ensure that different types and styles of lights create shadows as you would intuitively expect
  - Try to overlap a few different lights. 
  - Check shadow directions and intensities.

### Suggested Time Box: 
15 minutes per platform. Note that the workflow below is the basic steps, feel free to experiment with extra time to ensure that different types and styles of lights perform as you would intuitively expect, and try to overlap a few different lights. Check shadow directions and intensities.


Observe Shadows (Spotlight / Cascaded Shadow Maps) with various lights Workflow Steps
-------------------------------------

| Workflow                                          | Requests                                                                                                                                                                                                                                                                                                                                             | Things to Watch For                                                                                                                            |
|---------------------------------------------------|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------------------------------------------------------|
| 1. Launch Editor and create new level             | Launch Editor.exe, and create a new level with default settings                                                                                                                                                                                                                                                                                      | <ul><li>Editor launches without issue, and new default level can be created </li></ul>                                                         |
| 2. Focus on Shader Ball default settings          | Select Shader Ball from the Entity Outliner, Atom Default Environment and press Z to focus in the viewport                                                                                                                                                                                                                                           | <ul><li>Observe shadows with the default level setup ![ShadowsDefaultLighting](https://user-images.githubusercontent.com/41299597/213578511-e92e2644-b939-49d7-9ede-fd6761bb9767.png)                                    |
| 3. Create a Spot Light entitiy and enable shadows | <li> Add an entity for Spot Light <li> In the Entity Inspector, add the Light component <li> Select one of the "Spot (disk)" for the Light Type, <li> In the entity outliner disable or remove the sun <li> In the viewport, position the light where you can see more Shader Ball shadows. <li> Within the Light settings, toggle ON Enable shadows | <ul><li>Observe changes in the shader ball shadows when Sun is disabled, and Spot light is added <li> Observe changes when Shadows are enabled |
| 4. Experiment with Spot Light Settings            | <li> Experiment with the various settings in the Light component. Observe changes in the shadows <li> disable Spot light entity when finished                                                                                                                                                                                                        | <ul><li> Observe changes to shadows as parameters are adjusted. The lights should create shadows as you would intuitively expect <li> example of two pointed lights with different parameters casting shadows from shader ball    ![ShadowsTwoSpotLightsNoSun](https://user-images.githubusercontent.com/41299597/213579133-5127ccc5-24c5-4006-8f6b-d8372d04e78d.png)     |
| 5. Add a Directional Light and observe shadows    | <li> Add an entity for Directional Light <li> In the Entity Inspector, add the Directional Light component <li> Move the Directional Light entity above the ground plane <li> Rotate the light to point at the shaderball                                                                                                                            | <ul><li> Example of Default settings with Directional Light  ![Directional_light](https://user-images.githubusercontent.com/41299597/213578710-4afa1e16-9c99-4483-a30a-3a7ae3277525.png)   |
| 6. Experiment with Directional Light Settings     | <ul><li> Experiment with the various settings in the Light component.  <ul><li> Change the Color <li> Intensity Type <li> Intensity <li> Angular Diameter                                                                                                                                                                                            | <ul><li> Observe changes in the shadows.                                                                                                       |
--- 




