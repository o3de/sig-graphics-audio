# Editor component Sky Atmosphere/Stars Workflow Tests
Testing in this area should focus on the Sky Atmosphere and Stars components.

## **General Docs**
* [Feature Workflows](https://github.com/o3de/sig-graphics-audio/wiki/Feature-Workflows---Atom-Test-Plans)

## Common Issues to Watch For
 - Crashes, soft-locks, hangs when launching Editor.exe or adding a component.
 - When a component settings value is changed, it has a visual impact on the level. 
   - The change can be subtle, but if no change is present log an issue

## Required Gems
- Before launching the editor, open the Project Manager and confirm that the Stars gem is toggled ON, rebuild project after changing any Gem settings.

## Workflows

- Create a level with a Sky Atmosphere and Stars. 
  - Change various properties of the Sky Atmosphere component and observe the effects in the viewport 
  - Change various properties of the Star component and observe the effects in the viewport 

### Suggested Time Box: 
15 minutes per platform. Note that the workflow below is the basic steps, feel free to experiment with other settings during the remaining time.



Create a Sky Atmosphere with Stars Workflow Steps
-------------------------------------

| Workflow                              | Requests                                                                                                                                                                                             | Things to Watch For                                                                                  |
|---------------------------------------|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------------|
| 1. Launch Editor and create new level | Launch Editor.exe, and create a new level with default settings                                                                                                                                      | <ul><li>Editor launches without issue, and new default level can be created </li></ul>               |
| 2. Disable the Global Sky             | <ul><li> In the Entity Outliner, select the Atom Default Environment. <li> Remove or disable Global Sky from the Atom Default Enviornment                                                            | <ul><li> Example of Global Sky Disabled: <ul>                                                        |
| 3. Create a Sky Atmosphere Entity     | <ul><li> Create an entity, and name it Sky Atmosphere <li> Add a Sky Atmosphere component to the entity <ul><li> Adjust all property values and observe results<li> Disable Sky Atmosphere when done | <ul><li>Changes to component properties trigger the correct visual change in the viewport  </li><ul> |
| 4. Create a Stars Entity              | <ul><li> Add another entity to the level <li> Add a Stars component to the entity <ul><li> Adjust all property values and observe results                                                            | <ul><li>Changes to component properties trigger the correct visual change in the viewport </li><ul>  |
---




