# Test Plan: Mesh Instancing of Draw Calls
### **1.0 Introduction:**

Hardware Instancing is a method of reducing draw calls by bundling repeated instances of mesh and material within a scene or view. This plan covers high level risk mitigation plans for this feature as a two phase implementation.

##### **1.1 References:**

*   RFC [https://github.com/o3de/sig-graphics-audio/pull/114](https://github.com/o3de/sig-graphics-audio/pull/114)

### **2.0 Defect Tracking Issues:**

Bugs will be tracked with Github Issues (GHI) primarily entered in [https://github.com/o3de/o3de](https://github.com/o3de/o3de).
Labels will include `kind/bug`, `feature/graphics/mesh`, `sig/graphics-audio`, and initially `needs-triage`.

### **3.0 Test Schedule and Entry Criteria:**

##### **3.1 Remaining Test Tasks:**

###### **Phase 1**
- Configuration flag(s) and any console variables (cvar) used for testing, enabling and disabling the feature

###### **Phase 2**
- Creation of a mobile scaled level 1K_instanced_mesh or equivalent.
- Hooks for fetching instanced draw call count that work with `-rhi=null`
- Mobile devices ready to test
- Creation of an Editor Python Bindings test automation case in AutomatedTesting project
- Profiling and benchmarking on Android and iOS devices

##### **3.2 Test Entry Criteria**

###### **Phase 1**
- Build is free of errors for both DLY_UNITY_BUILD ON and OFF
- Unit tests have been written and pass consistently
- Configuration and cvars are available

###### **Phase 2**
- Mobile devices are available for Android and iOS
- Any blocking platform issues are addressed or scope of testing is changed to address open issues
- Hooks for feature testing are available

### **4.0 Risks:**

Risks are scored using a risk assessment matrix of impact and likelihood https://en.wikipedia.org/wiki/Risk_matrix

| Risk                                                                                                                                        | Mitigation                                                                                                                                                                           | Detection Method                                                                                                                                                                 | Notes                          | Impact        | Likelihood            | Risk Level     |
|---------------------------------------------------------------------------------------------------------------------------------------------|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|--------------------------------|---------------|-----------------------|----------------|
| Splitting implementation into two phases will mean that we don't see full scope of impact to other platforms and performance                | Mesh instancing will be disabled by default, and can be enabled/disabled on a per-project basis.                                                                                     | community feedback that they have a use case that needs instancing on mobile.                                                                                                    | Phase 1 initial implementation | insignificant | certain               | 5 - low        |
| Mobile performance may be degraded by mesh instancing. Disabled by default and not testing in Phase 1 may only hide any issues              | Mesh instancing will be disabled by default, and can be enabled/disabled on a per-project basis.  <br>Use profiling to isolate causes of performance impact and seek to address them | We will use deployed test levels on mobile devices to measure frame time impact and draw call count to ensure performance is not impacted. Contract testers will conduct testing | Phase 2                        | significant   | possible / eliminated | 14 - medium / 0 - none |
| iOS may be untestable                                                                                                                       | we will test only on android if iOS is untestable and accept risk that android and iOS are comparable for impact to performance                                                      | existing GHI blocking iOS and contract testers attempting to deploy                                                                                                              | Phase 2                        | marginal      | eliminated            | 0 - none       |
| Getting draw call counts may not be possible from RPI when `-rhi=null` which would mean additional work to get it from MeshFeatureProcessor | Worst case we can focus automation on stability when mesh instancing is active. Take the cost of implementing counters in MeshFeatureProcessor                                       | Prototype case review. Exploratory automation                                                                                                                                    | Phase 2                        | marginal      | unlikely              | 2 - negligible |
|                                                                                                                                             |                                                                                                                                                                                      |                                                                                                                                                                                  |                                |               |                       |                |
|                                                                                                                                             |                                                                                                                                                                                      |                                                                                                                                                                                  |                                |               |                       |                |

##### **4.1 Assumptions and Dependencies**

- Meaningful test content can be created
- Frame time measurements are a suitable proxy for performance impact in manual testing
- Draw call counts can be measured in stable scenarios to determine integrated functionality

### **5.0 In Scope:**

###### **Phase 1**
*   Unit tests
*   Mesh instancing on PC and Linux
*   RHI Vulkan and DX12

###### **Phase 2**
*   Mesh instancing on mobile (iOS, Android)
*   Editor, game launcher, dedicated server
*   Editor Python Bindings test

##### **5.1 Test Items:**

###### **Phase 1**
*   Configuration flag enables/disables mesh instancing

###### **Phase 2**
*   Performance impact on mobile
*   Impact on out-of-scope objects such as terrain and white box is zero (draw calls are the same)

##### **5.2 Developer Test Items:**

###### **Phase 1**
*   `r_meshInstancingEnabled` enabled and disabled
*   10KEntity and 10KVegInstances levels with mesh instancing enabled
  
###### **Phase 2**
*   Profile testing

### **6.0 Out of Scope:**

###### **Phase 1**
*   Mobile

###### **Phases 1 and 2**
*   Dynamic mesh
*   Skinned mesh
*   Terrain
*   White box
*   Shape components
*   AuxGeom

### **7.0 Test Approach:**

##### **7.1 Test Method**

This is a listing of the test strategies that will be used for test execution. Examples of common test strategies include but are not limited to:

**Dynamic:** Exploratory testing will be used at both early and late stages of a feature development, test approach, test cases and risk analysis will adapt to new information as this testing is completed.

**Consulted:** Input from the developer of the feature, and potentially from customers, will be used to help determine areas to test.

**Automation:** Automated tests will be run to cover specific instancing scenarios with expected draw call counts.

**Scene Testing:** System integration tests will be run against predetermined scenes with known content to determine performance impact from frame time measurements.

##### **7.2 Automation Approach**

###### **Phase 1**
*   Unit tests

###### **Phase 2**
*   Python Editor Bindings automation case to ensure instanced draw calls are correct

##### **7.3 Test Data and Environments**  
###### **Phase 2**
*   Mobile scaled test level 1K entity scenario with mixed batches of mesh required for mobile testing. Existing 10K levels are showing 333ms to 500ms frame times; we would want the test level to demonstrate frame times on mobile that are shorter and easier to gauge impact with. <br>A 1K level should contain the following:<br>A level with 3 mesh components that each have the same model + material combination, meaning they are instance-able. The should each have the 'Use Forward Pass IBL' setting set to true on the mesh component (or test on mobile where forward pass IBL is used for all meshes, regardless of the component setting). There should be two reflection probes in the level. 2 of the 3 mesh components should be near one probe, and the other should be near the other probe. This should result in 2 draw calls in the main view, since the 3rd mesh that is near a different reflection probe cannot be instanced with the other two. Functionally, the reflections should be the same with and without instancing enabled.

### **8.0 Test Exit Criteria:**

- `priority/critical` and `priority/blocker` triage labeled github issues are closed appropriately
- Specified functionality is confirmed to exist and function.
- All unit tests for the feature are consistently passing

