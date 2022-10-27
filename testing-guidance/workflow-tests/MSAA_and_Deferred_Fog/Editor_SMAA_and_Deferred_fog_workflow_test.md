
# Editor component SMAA Workflow Tests
There are currently two types of Anti-Aliasing supported by O3de: MSAA ("Multi Sample Anti-Aliasing"), and SMAA ("Subpixel Morphological Anti-Aliasing"). MSAA is enabled by default in Editor, while SMAA requires a change to a configuration file to enable. This test will be focused on SMAA, MSAA functionality is covered by the MSAA samples in AtomSampleViewer. 

## General Docs
* [Feature Workflows](https://github.com/o3de/sig-graphics-audio/wiki/Feature-Workflows---Atom-Test-Plans)

## Common Issues to Watch For
 - Crashes, soft-locks, hangs when launching Editor.exe or loading a level.
 - Angular or pixelated appearance on geometry edges, especially round edges.

## Workflow

### Area: SMAA workflow

**Project Requirements**
1. You have built the O3de project:  
* https://github.com/o3de/o3de
2. You have verified you are set to the AutomatedTesting project
2. Launch AssetProcessor.exe and wait for it to process all assets 


**Editor Platforms:**
* Windows (DX12 and Vulkan)
* Linux (Vulkan only)

**Product:** 
 - You have completed the workflow and logged any issues found.

**Suggested Time Box:** 10 minutes per platform. Note that the workflow below is the basic steps, feel free to experiment with other settings during the remaining time.

| Workflow                     | Requests           | Things to Watch For |
|------------------------------|--------------------|---------------------|
| 1. Change configuration file to turn on SMAA | Navigate to \Gems\Atom\Feature\Common\Assets\Passes, open "SMAAConfiguration.azasset" in a text editor, change "Enable" from 0 to 1 and save | <ul><li>File is present, and "Enable" value can be changed</li></ul>  |
| 2. Launch Editor | Launch Editor.exe, or close and re-launch Editor if it is already running | <ul><li>Editor launches successfully</li></ul>  |
| 3. Create new level | Create a new level in Editor with default assets | <ul><li>New level creation successfully </li></ul>  |
| 4. Observe default shaderball | Zoom in on the default shaderball and observe geometry edges | <ul><li>No angular or pixelated appearance on geometry edges, especially round edges</li></ul>  |
| 5. Add additional geometry | Add 5 new entities, add a mesh component to each, assigned different models to each  | <ul><li>Zoom in and observe geometry edges, no angular or pixelated appearance on geometry edges</li></ul>  |

---

# Editor component Deferred Fog Workflow Tests

## General Docs
* [Environment World Building](https://github.com/o3de/sig-graphics-audio/wiki/Environment-%7C-World-Building---Atom-Workflow-Test-Plan)

## Common Issues to Watch For
 - Crashes, soft-locks, hangs when launching Editor.exe or loading a level.
 - Abrupt changes in appearance that do not appear match settings, such as abrupt falloff when value it set to a slightly different value.

## Workflow

### Area: Deferred Fog workflow

**Project Requirements**
1. You have built the O3de project:  
* https://github.com/o3de/o3de
2. You have verified you are set to the AutomatedTesting project
2. Launch AssetProcessor.exe and wait for it to process all assets 


**Editor Platforms:**
* Windows (DX12 and Vulkan)
* Linux (Vulkan only)

**Product:** 
 - You have completed the workflow and logged any issues found.

**Suggested Time Box:** 20 minutes per platform. Note that the workflow below is the basic steps, feel free to experiment with other settings during the remaining time.

| Workflow                     | Requests           | Things to Watch For |
|------------------------------|--------------------|---------------------|
| 1. Launch Editor | Launch Editor.exe, and create a new level with default settings | <ul><li>Editor launches without issue, and new default level can be created </li></ul>  |
| 2. Add a new entity with a deferred fog component | Add a new entity to the level, then add a Deferred Fog component, and Post FX Layer to it | <ul><li>Entity created, and components added successfully</li></ul>  |
| 3. Change fog color | Click on the color swatch, then change the for color several times  | <ul><li>Fog color changes each time you click on a color in the Select Color dialog </li></ul>  |
| 4. Enable Fog Layer | Toggle the Enable Fog Layer on | <ul><li>Fog height and distance value are applied to fog in viewport</li></ul>  |
| 5. Adjust fog height and distance | Change Fog Start Distance, Fog End Distance, Fog Bottom Height, and Fog top height to different sets of values | <ul><li>Changes fog changes in viewport appropriately, and reflect the settings you adjusted![Height_Start_end](https://user-images.githubusercontent.com/79114701/197056079-dc70bbb9-c3ed-4b97-84ca-136da3ed0ecc.png)</li></ul>  |
| 6. Enable Turbulence Properties | Toggle Turbulence Properties on | <ul><li>Turbulence noise values are applied to fog in viewport</li></ul>  |
| 7. Adjust Noise Texture settings, and Octaves blend factor | Adjust Noise Texture First Octave Scale, Noise Texture First Octave Velocity, Noise Texture Second Octave Scale, Noise Texture Second Octave Velocity, and Octaves Blend Factor | <ul><li>The appearance of the fog turbulence is changed in the viewport with each setting change, no abrupt falloff.![Turbulance](https://user-images.githubusercontent.com/79114701/197068388-4b56438e-8baf-43a5-b62c-9de4ffe049f2.png)</li></ul>  |

---

Note: Right now "Noise Texture" can not be easily changed, this will need to be added to the workflow once this feature has been fixed.


