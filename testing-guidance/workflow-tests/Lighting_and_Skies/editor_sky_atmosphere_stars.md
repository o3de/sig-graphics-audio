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
![StarsGemToggle](https://user-images.githubusercontent.com/41299597/213521028-08346558-d759-480d-903d-d26896340b6e.png)


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
| 2. Disable the Global Sky             | <ul><li> In the Entity Outliner, select the Atom Default Environment. <li> Remove or disable Global Sky from the Atom Default Enviornment                                                            | <ul><li> Example of Global Sky Disabled: ![Disabled Global Sky](https://user-images.githubusercontent.com/41299597/213521115-92333155-5f3e-496e-ae70-a191d19ed5b3.png)  |
| 3. Create a Sky Atmosphere Entity     | <ul><li> Create an entity, and name it Sky Atmosphere <li> Add a Sky Atmosphere component to the entity <ul><li> Adjust all property values and observe results<li> Disable Sky Atmosphere when done | <ul><li>Changes to component properties trigger the correct visual change in the viewport <ul><li> example of Default settings ![SkyAtmoDefault](https://user-images.githubusercontent.com/41299597/213521445-f0324d8f-6190-48dc-a2d0-0f512ceba881.png) <li> example of various settings changed ![SkyAtmoSettingsChanged](https://user-images.githubusercontent.com/41299597/213521580-c26fcbde-ab22-4232-aa79-03f807cd37d8.png) </li><ul> |
| 4. Create a Stars Entity              | <ul><li> Add another entity to the level <li> Add a Stars component to the entity <ul><li> Adjust all property values and observe results    | <ul><li>Changes to component properties trigger the correct visual change in the viewport <ul><li> Example of default stars settings with no Sky Atmosphere ![StarsDefaultNoAtmo](https://user-images.githubusercontent.com/41299597/213521790-bc0faffa-d08f-4035-8a63-375d5a7a6791.png) <li> Example of Stars with various settings changed and Sky Atmosphere enabled.![StarsSettingsChangedWithAtmo](https://user-images.githubusercontent.com/41299597/213521954-31f2cb04-e5f7-4585-b820-a54e0d84040c.png) </li><ul>  |
---




