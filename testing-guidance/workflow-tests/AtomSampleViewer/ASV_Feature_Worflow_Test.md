# AtomSampleViewer Features Workflow Tests
This workflow test will focus on verifying that the AtomSampleViewer Features samples are functioning correctly.

The specific samples that are covered by this test are:  
 - AreaLight
 - Bloom
 - Checkerboard
 - DepthOFField
 - DiffuseGI
 - Exposure
 - EyeMaterial
 - LightCulling
 - Parallax
 - Shadow
 - ShadowSponza
 - SkinnedMesh
 - SSAO
 - SSR
 - Tonemapping
 - Transparency

## General Docs
* [Features Samples Documentation](https://github.com/o3de/o3de-atom-sampleviewer/wiki/Feature-Samples)

## Common Issues to Watch For
- Crashes, soft-locks, hangs when switching between samples or adjusting sample specific debug menus
- Black squares that appear on rendered image (sometimes known as NaN errors)
- All graphic assets render as expected, compare to screenshots in the documentation for expected result.

## Workflows

### Area: AtomSampleViewer RPI workflow

**Project Requirements**  
1. You have built both O3de and AtomSampleViewer projects:  
* https://github.com/o3de/o3de 
* https://github.com/o3de/o3de-atom-sampleviewer  
2. Switch your project to AtomSampleViewer
3. Launch AssetProcessor.exe and wait for it to process all assets


**Supported Editor Platforms:**
* Windows (DX12 and Vulkan)
* Linux (Vulkan only)

**Product:** 
  - You have completed the workflow and logged any issues found. 

**Suggested Time Box:** 
 - 3 Minutes to 10 minutes per sample depending on sample complexity.

###Test procedure:
In order to execute this test, you will need to run each Features sample in the above list though the following workflow test twice.
 - First run uses DX12 API, to do this you just need to launch AtomSampleViewerStandalone.exe directly
 - Second run will use Vulkan API, which means you need to run AtomSampleViewerStandalone.exe with the --Features=vulkan parameter.
    - To do this open a cmd window within the folder containing the executable and add the parameter. For example "AtomSampleViewerStandalone.exe --Features=vulkan". Another option is to create a shortcut to the AtomSampleViewerStandalone.exe and add --Features=vulkan to it that way.

| Workflow                     | Requests           | Things to Watch For |
|------------------------------|--------------------|---------------------|
| 1. Load Features sample              | Go to the "Samples > Features" menu and select the next sample on the list | <ul><li>Sample loads without issue, no crashes, hangs, or other stability issues </li></ul>  |
| 2. Test Features sample              | Observe sample and test all debug option on the sidebar if applicable (not all samples have a debug menu options) | <ul><li>All graphic assets render as expected, compare to screenshots in the documentation for expected result: [Features Samples Documentation](https://github.com/o3de/o3de-atom-sampleviewer/wiki/Features-Samples) </li></ul>  |
---




