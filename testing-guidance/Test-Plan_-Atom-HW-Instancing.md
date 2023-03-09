# Test Plan: Hardware Instancing of Mesh Draw Calls
### **1.0 Introduction:**

Hardware Instancing is a method of reducing draw calls by bundling repeated instances of mesh and material within a scene or view. This plan covers high level risk mitigation plans for this feature as a two phase implementation.

##### **1.1 References:**

*   RFC [https://github.com/o3de/sig-graphics-audio/pull/114](https://github.com/o3de/sig-graphics-audio/pull/114)

### **2.0 Defect Tracking Issues:**

Bugs will be tracked with Github Issues (GHI) primarily entered in [https://github.com/o3de/o3de](https://github.com/o3de/o3de).
Labels will include `kind/bug`, `feature/graphics`, `sig/graphics-audio`, and initially `needs-triage`.

### **3.0 Test Schedule and Entry Criteria:**

##### **3.1 Remaining Test Tasks:**

- Creation of a mobile scaled level 1K_instanced_mesh or equivalent.
- Hooks for fetching instanced draw call count that work with `-rhi=null`
- Mobile devices ready to test
- Configuration flag(s) and any console variables (cvar) used for testing, enabling and disabling the feature
- Creation of an Editor Python Bindings test automation case in AutomatedTesting project
- Profiling and benchmarking on Android and iOS devices

##### **3.2 Test Entry Criteria**

This section should be used to determine the entry criteria for accepting the software into test. These should be measurable items, each with an assigned owner. Examples include:

- Build is free of errors for both DLY_UNITY_BUILD ON and OFF
- Unit tests have been written and pass consistently
- Configuration and cvars are available
- Mobile devices are available for Android and iOS
- Any blocking platform issues are addressed or scope of testing is changed to address open issues
- Hooks for feature testing are available

### **4.0 Risks:**

Identify what the critical risk areas are as identified through the Risk Analysis process defined by the QA team, such as:

1.  Delivery of a third party product.
2.  Ability to use and understand a new package/tool, etc.
3.  Extremely complex functions
4.  Modifications to components with a past history of failure
5.  Poorly documented modules or change requests
6.  Misunderstanding of original requirements
7.  Complexity of integration
8.  Dependency on other high risk modules
9.  Defects introduced from outside of this effort
10.  Bottlenecks or disruptions related to Automation framework

Risks are scored using a risk assessment matrix of impact and likelihood https://en.wikipedia.org/wiki/Risk_matrix

| Risk                                                                                                                                   | Mitigation                                                                                                                                        | Detection Method                                                                                                                                                                 | Notes                          | Impact      | Likelihood | Risk Level     |
|----------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|--------------------------------|-------------|------------|----------------|
| Splitting implementation may mean that we don't see full scope of impact to other platforms and performance                            | Consider disabling HW instancing on platforms not specifically profiled or tested                                                                 | hard to detect what we don't look for                                                                                                                                            | Phase 1 initial implementation |             |            |                |
| Mobile performance may be degraded by HW instancing                                                                                    | Possibly disable HW instancing on mobile and other platforms.  <br>Use profiling to isolate causes of performance impact and seek to address them | We will use deployed test levels on mobile devices to measure frame time impact and draw call count to ensure performance is not impacted. Contract testers will conduct testing | Phase 2                        | significant | possible   | 14 - medium    |
| iOS may be untestable                                                                                                                  | we will test only on android if iOS is untestable and accept risk that android and iOS are comparable for impact to performance                   | existing GHI blocking iOS and contract testers attempting to deploy                                                                                                              | Phase 2                        | marginal    | eliminated | 0 - none       |
| Getting draw call counts may not be possible when `-rhi=null` which would mean automation would depend on GPU nodes that are expensive | Worst case we can focus automation on stability when HW instancing is active. Essentially not testing integration functionality.                  | Prototype case review. Exploratory automation                                                                                                                                    | Phase 2                        | marginal    | unlikely   | 2 - negligible |
|                                                                                                                                        |                                                                                                                                                   |                                                                                                                                                                                  |                                |             |            |                |
|                                                                                                                                        |                                                                                                                                                   |                                                                                                                                                                                  |                                |             |            |                |

##### **4.1 Assumptions and Dependencies**

- Meaningful test content can be created
- Frame time measurements are a suitable proxy for performance impact in manual testing
- Draw call counts can be measured in stable scenarios to determine integrated functionality

### **5.0 In Scope:**

*   HW instancing on various platforms (PC, Linux, iOS, Android)
*   Editor, game launcher, dedicated server
*   Editor Python Bindings test
*   Unit tests

##### **5.1 Test Items:**

*   Configuration flag enables/disables HW instancing
*   Performance impact on mobile (this may be delayed)
*   Impact on out-of-scope objects such as terrain and white box is zero (draw calls are the same)

##### **5.2 Developer Test Items:**

*   Profile testing

### **6.0 Out of Scope:**

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

*   Unit tests
*   Python Editor Bindings automation case to ensure instanced draw calls are correct

##### **7.3 Test Data and Environments**  

*   Mobile scaled test level 1K entity scenario with mixed batches of mesh required for mobile testing. Existing 10K levels are showing 333ms to 500ms frame times; we would want 

### **8.0 Test Exit Criteria:**

- `priority/critical` and `priority/blocker` triage labeled github issues are closed appropriately
- Specified functionality is confirmed to exist and function.
- All unit tests for the feature are consistently passing

