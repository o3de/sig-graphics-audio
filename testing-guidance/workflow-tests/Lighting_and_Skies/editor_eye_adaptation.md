# Editor component Eye Adaptation Workflow Tests
Testing in this area should focus on the Eye Adaptation components.

## **General Docs**
* [O3DE Learn: PostFx Layer](https://www.o3de.org/docs/user-guide/components/reference/atom/postfx-layer/)
* [O3DE Learn: Exposure Control](https://www.o3de.org/docs/user-guide/components/reference/atom/exposure-control/) 

## Common Issues to Watch For
 - Crashes, soft-locks, hangs when launching Editor.exe or adding a component.
 - When a component settings value is changed, it has a visual impact on the level. 
   - The change can be subtle, but if no change is present log an issue


## Workflows
- Create an entity with the Exposure Control and enable Eye Adaptation
  - Move the view around in the level and observe how Eye Adaptation adjusts the exposure when looking at a bright light


### Suggested Time Box: 
15 minutes per platform. Note that the workflow below is the basic steps, feel free to experiment with other settings during the remaining time.



Create an entity with Eye Adaptation activated
-------------------------------------

| Workflow                              | Requests                                                                                                                                                                                                                                                                                                                  | Things to Watch For                                                                                                                                                          |
|---------------------------------------|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| 1. Launch Editor and create new level | Launch Editor.exe, and create a new level with default settings                                                                                                                                                                                                                                                           | <ul><li>Editor launches without issue, and new default level can be created                                                                                                  |
| 2. Set up Exposure Control Entity     | <ul><li> Create an entity, name it something like Exposure Control<li> Add the following components <ul><li> PostFX Layer <li> Exposure Control                                                                                                                                                                           | <ul><li>Entity can be created and components added                                                                                                                           |
| 3. Turn on Eye Adaptation             | <ul><li>In the Viewport, adjust the view focus on the sun <li> Within the Exposure Control component switch the Control Type from Manual Only to Eye Adaptation<li> In the viewport, move the camera around to see how the Eye Adapation feature effects the exposure when facing a bright light compared to away from it | <ul><li>View the exposure in the viewport fades to be darker when the active camera is facing a bright light. When the view turns away from the light the exposure brightens.|
---




