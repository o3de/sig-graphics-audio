# Editor component Sky Atmosphere/Stars Workflow Tests
Testing in this area should focus on the Sky Atmosphere/Stars component.

## **General Docs**
* [Feature Workflows](https://github.com/o3de/sig-graphics-audio/wiki/Feature-Workflows---Atom-Test-Plans)

## Common Issues to Watch For
 - Crashes, soft-locks, hangs when launching Editor.exe or adding a component.
 - When a component settings value is changed, it has a visual impact on the level. 
   - The change can be subtle, but if no change is present log an issue

## Workflows

- Create a level with a Sky Atmosphere and Stars. 
  - Change various properties of the Sky Atmosphere component and observe the effects in the viewport 
  - Change various properties of the Star component and observe the effects in the viewport 

### Suggested Time Box: 
15 minutes per platform. Note that the workflow below is the basic steps, feel free to experiment with other settings during the remaining time.


Create a Sky Atmosphere with Stars Workflow Steps
-------------------------------------

| Workflow                              | Requests                                                                                                                                                        | Things to Watch For                                                                                  |
|---------------------------------------|-----------------------------------------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------------|
| 1. Launch Editor and create new level | Launch Editor.exe, and create a new level with default settings                                                                                                 | <ul><li>Editor launches without issue, and new default level can be created </li></ul>               |
| 2. Create a Sky Atmosphere Entity     | <ul><li> Create an entity, and name it Sky Atmosphere <li> Add a Sky Atmosphere component to the entity <ul><li> Adjust all property values and observe results | <ul><li>Changes to component properties trigger the correct visual change in the viewport  </li><ul> |
| 3. Create a Stars Entity              | <ul><li> Add another entity to the level <li> Add a Stars component to the entity <ul><li> Adjust all property values and observe results                       | <ul><li>Changes to component properties trigger the correct visual change in the viewport </li><ul>  |
---




