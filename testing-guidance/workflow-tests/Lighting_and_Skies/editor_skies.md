# Editor component Skies Workflow Tests
Testing in this area should focus on the Editor HDRi Skybox, and physical sky components.

## **General Docs**
* [Feature Workflows](https://github.com/o3de/sig-graphics-audio/wiki/Feature-Workflows---Atom-Test-Plans)

## Common Issues to Watch For
 - Crashes, soft-locks, hangs when launching Editor.exe or adding a component.

## Workflows

- Create an entity and attach an HDRi Skybox Component. 
  - Change various aspects of the skybox and ensure they show up properly in editor.
- Create an entity with the Physical Sky component.
  - Change various aspects of the Physical Sky component and ensure they show up properly in editor.


##### Preparation

- Editor launched with New Level opened for use.
- Delete the existing "GlobalSky" Entity present in the level


HDRi Skybox Workflow Steps
-------------------------------------

Global Skylight Image-Based Lighting (IBL) - This rendering technique lights a scene using by capturing and omnidirectional light information from an image. This is then projected on a dome to simulate lighting for objects present in the scene.

High dynamic range image (HDRi) Skybox - An HDR image of the sky, horizon, and distant background objects placed into a cube map texture. This is then projected onto a large cube that surrounds the level geometry.

*   Create new level or open an existing level
*   Delete existing "GlobalSky" Entity
*   Add new child Skytest entity 
*   Add Global Skylight (IBL) and HDRi Skybox components
*   Navigate to Global Skylight within entity inspector
*   Set Diffuse image to: "default\_iblskyboxcm\_ibldiffuse.exr.streamingimage"
*   Set Specular to: "default\_iblskyboxcm\_iblspecular.exr.streamingimage"
*   Switch to HDRi Skybox entity
*   Set Cubemap texture to default\_iblskyboxcm.exr.streamingimage
*   Navigate to the ground plane component 
*   Change the material to: metal\_gold\_matte
*   Observe reflections and Highlights
*   Adjust focus value on  HDRi Skybox
*   Adjust focus value on  Global Skylight  

(repeat steps with different cubemaps, and ground plane mats)



Physical Sky component Workflow Steps
-------------------------------------
* Create a new level
* Select the Shaderball entity
* Set the material component to "basic_m10_r-2" 
* Add three new entities to the level, add a mesh asset to each (such as: "lucy_high.fbx", "bunny.fbx", and "suzanne.fbx")
* Add a different material to each, try to vary the reflective and roughness of the materials 
* Place one object close to the shadershphere and on the same plane (about 1.5-2.5 meters), one at medium (about 3-4 meters), one further away (about 8-9 meters)
* Delete the Global Sky element
* Add a new entity to the level, give it a Physical Sky component
* Change the Sky Intensity slider and observe change in level 
  * Sky Intensity 0.65
  * Sky Intensity 4.5
  * Change Sky back to Intensity to 4.0
* Change the Sun Intensity slider and observe change in level
  * Sun Intensity 3.5
  * Sun Intensity -4
  * Change Sun Intensity back to 8
* Adjust Turbidity value from 1-10, observe color change in sky
  * Set Turbidity back to 1
* Adjust Sun Radius Factor from 0.1 - 2.0, observe change in sun
  * Change Sun Radius Factor back to 1.0
* Enable Fog
  * Change Fog color, increase top height, and bottom height

Steps to create a modular Physical Sky.
* Create Entity
* Zero out transforms
* Add Physical Sky Component
* Add Deferred Fog Component
* Add PostFX Layer
* Add the Sun under this Entity (Make the new Physical Sky Entity the parent)
* Zero out the Sun Transforms
Now you will have a modular Physical Sky



