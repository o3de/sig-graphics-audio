# Editor component PostFX Layer and Modifiers Workflow Tests
Testing in this area should focus on the Punctual/Area Lights components.

## **General Docs**
* [O3DE Learn: PostFX Components](https://www.o3de.org/docs/user-guide/components/reference/atom/post-processing-modifiers/)
* [Post-processing Effects Tutorial](https://www.o3de.org/docs/learning-guide/tutorials/postfx/)

## Common Issues to Watch For
 - Crashes, soft-locks, hangs when launching Editor.exe or adding a component.
 - When a component settings value is changed, it has a visual impact on the level. 
   - The change can be subtle, but if no change is present log an issue


## Workflows
- Create various entities with meshes and materials and place them in the space. 
  - Add PostFX Layer and Exposure components to an entity and see how it controls render brightness 
    - Utilize PostFX modifier components to see how they change the Exposure component effects 

## Prepare the level
Prepare a simple level for all workflows to be run against
- Launch the editor, and create a new level with default settings 

### Suggested Time Box: 
5 minutes per workflow per platform. Note that the workflow below is the basic steps, feel free to experiment with other settings during the remaining time.
- With extra time try adding other PostFX components replacing Exposure (Bloom, Depth of Field, etc)


---
### Exposure Control

| Workflow                                   | Requests                                                                                                                                                                                                                                                                       | Things to Watch For                                                                                                                                                                                                                                                                                                                                                                                                |
|--------------------------------------------|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| Exposure Control | <ul><li>Create a new entity<li>Add a `PostFX Layer` component<li>Add a `Exposure Control` component<li>Locate the default `Camera` component on the level or add one, then add a `Tag` component to that entity and set a string value in the elements list to "Exposure"<li>Modify the `PostFX Layer` component property `Select Camera Tags Only` adding a new element with the string "Exposure"<li>Set the `Exposure Control` component's Manual Compensation to `-1.0`<li>Enter game mode `ctrl+g`<li>Exit game mode `esc`<li>Modify properties and enter/exit game mode | <ul><li>Overall brightness of render is impacted by Manual Compensation property |

---
### Bloom

| Workflow                                   | Requests                                                                                                                                                                                                                                                                       | Things to Watch For                                                                                                                                                                                                                                                                                                                                                                                                |
|--------------------------------------------|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| Bloom | <ul><li>Create a new entity<li>Add a `PostFX Layer` component<li>Add a `Bloom` component<li>Toggle the Enable Bloom property to enabled<li>Locate the Shader Ball entity and add a `Material` component setting the Default Material property to `011_emissive`<li>Locate the `Tag` component on the Camera entity remove previous tag string elements and set a string value in the elements list to "Bloom"<li>Modify the `PostFX Layer` component property `Select Camera Tags Only` adding a new element with the string "Bloom"<li>On the `Bloom` component set the Intensity property to `1.0`<li>Enter game mode `ctrl+g`<li>Exit game mode `esc`<li>Modify properties and enter/exit game mode; toggle the Enable comparing the game mode render enabled/disabled | <ul><li>Tags should prevent the Exposure Control from impacting in game mode for Bloom<li>The emissive material should have a visible glow beyond the surface of the Shader Ball |

---
### Depth of Field

| Workflow                                   | Requests                                                                                                                                                                                                                                                                       | Things to Watch For                                                                                                                                                                                                                                                                                                                                                                                                |
|--------------------------------------------|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| Depth of Field | <ul><li>Create a new entity<li>Add a `PostFX Layer` component<li>Add a `Depth Of Field` component<li>Set the Camera Entity property of the `Depth of Field` component using the crosshair picker to select the Camera Enitity from the Entity Outliner<li>Toggle the Enable Depth of Field property to enabled<li>Locate the `Tag` component on the Camera entity remove previous tag string elements and set a string value in the elements list to "Depth"<li>Modify the `PostFX Layer` component property `Select Camera Tags Only` adding a new element with the string "Depth"<li>Enter game mode `ctrl+g`<li>Exit game mode `esc`<li>Modify properties and enter/exit game mode | <ul><li>Near and far distance should be less focused |

---
### HDR Color Grading and Look Modification

| Workflow                                   | Requests                                                                                                                                                                                                                                                                       | Things to Watch For                                                                                                                                                                                                                                                                                                                                                                                                |
|--------------------------------------------|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| HDR Color Grading | <ul><li>Create a new entity<li>Add a `PostFX Layer` component<li>Add a `HDR Color Grading` component<li>Toggle Enable HDR Color Grading property to enabled<li>Locate the `Tag` component on the Camera entity remove previous tag string elements and set a string value in the elements list to "Color"<li>Modify the `PostFX Layer` component property `Select Camera Tags Only` adding a new element with the string "Color"<li>Modify properties<li>Press the Generate LUT button on the `HDR Color Grading` component in Entity Inspector and wait for it to finish<li>Press the Activate LUT button on the `HDR Color Grading` component | <ul><li>Console spew should indicate that a LUT is generating a file with several lines `(python_test) - [ColorGrading`<li>Activate LUT should add a `Look Modification` component and disable the `HDR Color Grading` component |
| Look Modification | <ul><li>Starting with the entity used for `HDR Color Grading` component<li>Having clicked the Generate LUT button and the Activate LUT you should now see the `HDR Color Grading` component disabled by scripts and a `Look Modification` component added and enabled<li>Enter game mode `ctrl+g`<li>Exit game mode `esc`<li>Modify properties and enter/exit game mode | <ul><li>The enabled look modification uses the generated LUT to alter render |

---
### SSAO

| Workflow                                   | Requests                                  | Things to Watch For                                    |
|--------------------------------------------|-------------------------------------------|--------------------------------------------------------|
| SSAO | <ul><li>Use AtomSampleViewer to test SSAO | Consider this covered in the AtomSampleViewer workflow |

---
### Deferred Fog
This is covered in another workflow already

---
### PostFX Radius Weight Modification

| Workflow                                   | Requests                                  | Things to Watch For                                    |
|--------------------------------------------|-------------------------------------------|--------------------------------------------------------|
| Radius Weight Exposure Control | <ul><li>Locate the Exposure Control entity you worked with or create a new entity with `PostFX Layer` and `Exposure Control`<li>Add `PostFX Radius Weight Modifier` component<li>Locate the default `Camera` component on the level or add one, then update a `Tag` component to that entity and set a string value in the elements list to "Exposure"<li>Modify the `PostFX Layer` component property `Select Camera Tags Only` adding a new element with the string "Exposure"<li>Set the `Exposure Control` component's Manual Compensation to `-1.0`<li>Enter game mode `ctrl+g` and use the fly camera `ASDW and mouse` to move towards and away from the Shader Ball mesh<li>Exit game mode `esc`<li>Modify properties and enter/exit game mode | <ul><li>Brightness of render is impacted by Manual Compensation property<li>Exposure effect is localized to the radius around the PostFX Radius Weight Modifier |

---
### PostFX Shape Weight Modification

| Workflow                                   | Requests                                  | Things to Watch For                                    |
|--------------------------------------------|-------------------------------------------|--------------------------------------------------------|
| Shape Weight Exposure Control | <ul><li>Locate the Exposure Control entity you worked with or create a new entity with `PostFX Layer` and `Exposure Control`<li>Delete or Disable `PostFX Radius Weight Modifier` component<li>Add a `PostFX Shape Weight Modifier` component<li>Add a required shape object<li>Locate the default `Camera` component on the ensure that it still has the tag element "Exposure"<li>Modify the `PostFX Layer` component property `Select Camera Tags Only` adding a new element with the string "Exposure" if it is not already set<li>Set the `Exposure Control` component's Manual Compensation to `-1.0`<li>Enter game mode `ctrl+g` and use the fly camera `ASDW and mouse` to move towards and away from the Shader Ball mesh<li>Exit game mode `esc`<li>Modify properties and enter/exit game mode | <ul><li>Brightness of render is impacted by Manual Compensation property<li>Exposure effect is localized to the area covered by the PostFX Shape Weight Modifier |

---
### PostFX Gradient Weight Modfication

| Workflow                                   | Requests                                  | Things to Watch For                                    |
|--------------------------------------------|-------------------------------------------|--------------------------------------------------------|
| Radius Weight Exposure Control | <ul><li>Locate the Exposure Control entity you worked with or create a new entity with `PostFX Layer` and `Exposure Control`<li>Add `PostFX Gradient Weight Modifier` component and remove any previous PostFX Weight Modifier components<li>Add a new entity and to that entity add the `Perlin Noise Gradient`, `Gradient Transform Modifier`, and `Sphere Shape` (radius 5) components<li>Locate the default `Camera` component on the level or add one, then update a `Tag` component to that entity and set a string value in the elements list to "Exposure"<li>Modify the `PostFX Layer` component property `Select Camera Tags Only` adding a new element with the string "Exposure"<li>Set the `Exposure Control` component's Manual Compensation to `-1.0`<li>Enter game mode `ctrl+g` and use the fly camera `ASDW and mouse` to move towards and away from the Shader Ball mesh<li>Exit game mode `esc`<li>Modify properties and enter/exit game mode | <ul><li>Brightness of render is impacted by Manual Compensation property<li>Exposure effect is localized to the noise gradient around the PostFX Gradient Weight Modifier |



