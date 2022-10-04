# AtomSampleViewer Debug and Profiling Tools Workflow Tests
This workflow test will focus on verifying that the AtomSampleViewer Debug and Profiling Tools samples are functioning correctly.

The tools that are covered by this test are:  
 - CPU Profiler
 - Culling Debug Window
 - GPU Profiler
 - PassTree
 - TransientAttachmentProfiler

## General Docs
* [Debug and Profiling Tools Samples Documentation](https://github.com/o3de/o3de-atom-sampleviewer/wiki/Debug-and-Profiling-Tools)

## Common Issues to Watch For
- Crashes, soft-locks, hangs when switching between samples or adjusting sample specific Debug and Profiling Tools

## Workflows

### Area: AtomSampleViewer RPI WORKFLOW

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
 - 5 to 10 minutes per tool.

###Test procedure:
In order to execute this test, you will need to run each debug and profiling tool in the above list though the following workflow test twice.
 - First run uses DX12 API, to do this you just need to launch AtomSampleViewerStandalone.exe directly
 - Second run will use Vulkan API, which means you need to run AtomSampleViewerStandalone.exe with the --Debug and Profiling Tools=vulkan parameter.
    - To do this open a cmd window within the folder containing the executable and add the parameter. For example "AtomSampleViewerStandalone.exe --Debug and Profiling Tools=vulkan". Another option is to create a shortcut to the AtomSampleViewerStandalone.exe and add --Debug and Profiling Tools=vulkan to it that way.

| Workflow                     | Requests           | Things to Watch For |
|------------------------------|--------------------|---------------------|
| 1. Load any Feature sample             | Go to the "Samples > Features" load any feature sample.  | <ul><li>Prep for tool testing, some tools need specific samples. For example Culling Debug Window needs the ShadowSponza sample. </li></ul>  |
| 2. Load tool you are testing              | Go to the "Pass" menu for PassTree, "Culling" menu for Culling Debug Menu, and "Profile" for the rest of the tools.  | <ul><li>Tool loads without issue, no crashes, hangs, or other stability issues. </li></ul>  |
| 3. Test tool based on documentation              | Review documentation and run any test cases specified. | <ul><li>Tool behaves as expected, compare to screenshots and information. in the documentation for expected result: [Debug and Profiling Tools Samples Documentation](https://github.com/o3de/o3de-atom-sampleviewer/wiki/Debug-and-Profiling-Tools) </li></ul>  |
| 4. Test tool with other samples if applicable             | Open 3-4 more RPI or Feature samples with the tool open.  | <ul><li> Tool updates when the new sample loads, application remains stable. </li></ul>  |
---




