# Atom Renderer Automation Strategy

The objective of automation is the robust detection of regression early, allowing correction before issues integrate with other systems. If we allow regressions to persist, consumers of our code may inadvertently code to them, causing a negative outcome to the quality of the product. An ancillary objective of automation is the detection of previously unknown defects during the initial creation of automation.

Automation supports higher confidence in the quality of our product. Code coverage is a measurable proxy for confidence in code quality. While code coverage does not equate to code quality, the level of coverage correlates to perceived confidence in quality. Our atom editor unit test code coverage is below 42% with some RHI builder dll's being below 12%; the common industry objective being greater than 80%. All software has defects; given that we are not covering our code, it is likely that defects exist in the uncovered areas.

*   1 [Priority of Implementation](#AtomAutomationStrategy-PriorityofImplementation)
    *   1.1 [Initiatives](#AtomAutomationStrategy-Initiatives)
        *   1.1.1 [Increased unit test code coverage](#AtomAutomationStrategy-Increasedunittestcodecoverage)
        *   1.1.2 [Implement robust performance metric collection as an automated process](#AtomAutomationStrategy-Implementrobustperformancemetriccollectionasanautomatedprocess)
        *   1.1.3 [Expand support for batched test case patterns to include more tooling than just Editor](#AtomAutomationStrategy-ExpandsupportforbatchedtestcasepatternstoincludemoretoolingthanjustEditor)
        *   1.1.4 [Generate test results for inclusion in pull requests](#AtomAutomationStrategy-Generatetestresultsforinclusioninpullrequests)
        *   1.1.5 [Create sample prefabs that cover the test scenarios of major render components](#AtomAutomationStrategy-Createsampleprefabsthatcoverthetestscenariosofmajorrendercomponents)
        *   1.1.6 [Convert AtomSampleViewer feature coverage to AutomatedTesting editor project](#AtomAutomationStrategy-ConvertAtomSampleViewerfeaturecoveragetoAutomatedTestingeditorproject)
        *   1.1.7 [Review rev the engine and other workflow scenarios for additional coverage targets](#AtomAutomationStrategy-Reviewrevtheengineandotherworkflowscenariosforadditionalcoveragetargets)
        *   1.1.8 [Static Code Analysis](#AtomAutomationStrategy-StaticCodeAnalysis)
        *   1.1.9 [Dynamic Code Analysis as part of AR](#AtomAutomationStrategy-DynamicCodeAnalysisaspartofAR)
        *   1.1.10 [Expose Material Editor and Material Canvas to Multi-Test Framework](#AtomAutomationStrategy-ExposeMaterialEditorandMaterialCanvastoMulti-TestFramework)
*   2 [Test Automation Pyramid](#AtomAutomationStrategy-TestAutomationPyramid)
*   3 [Scope](#AtomAutomationStrategy-Scope)
*   4 [Risks](#AtomAutomationStrategy-Risks)
*   5 [Dependencies](#AtomAutomationStrategy-Dependencies)
*   6 [Code Coverage](#AtomAutomationStrategy-CodeCoverage)
*   7 [Performance Testing](#AtomAutomationStrategy-PerformanceTesting)
*   8 [Soak Testing](#AtomAutomationStrategy-SoakTesting)
*   9 [Fuzz Testing](#AtomAutomationStrategy-FuzzTesting)
*   10 [API Testing](#AtomAutomationStrategy-APITesting)
*   11 [Fault Injection Testing](#AtomAutomationStrategy-FaultInjectionTesting)
*   12 [Platform Testing](#AtomAutomationStrategy-PlatformTesting)
    *   12.1 [General Requirements](#AtomAutomationStrategy-GeneralRequirements)
    *   12.2 [Windows PC](#AtomAutomationStrategy-WindowsPC)
    *   12.3 [macOS](#AtomAutomationStrategy-macOS)
    *   12.4 [Linux (dedicated server and GUI based)](#AtomAutomationStrategy-Linux(dedicatedserverandGUIbased))
    *   12.5 [iOS](#AtomAutomationStrategy-iOS)
    *   12.6 [Android](#AtomAutomationStrategy-Android)
    *   12.7 [OpenXR](#AtomAutomationStrategy-OpenXR)
*   13 [Supporting Systems](#AtomAutomationStrategy-SupportingSystems)
    *   13.1 [Automated Review (AR)](#AtomAutomationStrategy-AutomatedReview(AR))
    *   13.2 [Unit/Integration Testing Using AZTest](#AtomAutomationStrategy-Unit/IntegrationTestingUsingAZTest)
    *   13.3 [CMake/Ctest](#AtomAutomationStrategy-CMake/Ctest)
    *   13.4 [Python LYTestTools (LYTT)](#AtomAutomationStrategy-PythonLYTestTools(LYTT))
    *   13.5 [Multitest Framework](#Multitest Framework (pytest related))
    *   13.6 [Editor Python Binding (EPB aka Hydra)](#AtomAutomationStrategy-EditorPythonBinding(EPBakaHydra))
    *   13.7 [PySide2](#AtomAutomationStrategy-PySide2)
    *   13.8 [AtomSampleViewer (ASV) lua driven integration tests](#AtomAutomationStrategy-AtomSampleViewer(ASV)luadrivenintegrationtests)
    *   13.9 [WARP (Windows Advanced Rasterization Platform)](#AtomAutomationStrategy-WARP(WindowsAdvancedRasterizationPlatform))
*   14 [Process Considerations](#AtomAutomationStrategy-ProcessConsiderations)
    *   14.1 [Test Qualities](#AtomAutomationStrategy-TestQualities)
    *   14.2 [Image Comparison (screenshot) Tests](#AtomAutomationStrategy-ImageComparison(screenshot)Tests)
    *   14.3 [Timeouts](#AtomAutomationStrategy-Timeouts)
    *   14.4 [Github automated tests workflow](#AtomAutomationStrategy-Githubautomatedtestsworkflow)
    *   14.5 [Credentials and Secrets](#AtomAutomationStrategy-CredentialsandSecrets)
    *   14.6 [Documentation](#AtomAutomationStrategy-Documentation)
*   15 [FAQ](#AtomAutomationStrategy-FAQ)
    *   15.1 [Q: Is there more information about this fascinating testing pyramid?](#AtomAutomationStrategy-Q:Istheremoreinformationaboutthisfascinatingtestingpyramid?)

  

Priority of Implementation
--------------------------

The three pillars of general availability are stability, performance, and ease of use. Automation will be scoped exclusively on stability and performance. Priority is not an assignment of ownership simply a balance of precedence.

This section provides an overview of our prioritization logic based on the [Test Automation Pyramid](#AtomAutomationStrategy-TestAutomationPyramid) methodology.

Focus will be mostly on Windows with some emphasis on Linux, but automation will be written to be platform agnostic. We will work to introduce higher level cross-platform automation eventually. Unit tests will always be the most abundant contributor to Atom automation and should be platform-agnostic.

We will accomplish this by relying as much as we can on unit tests and utilize Automated Review (AR) through LYTestTools (LYTT) for integration tests. By focusing on O3DE wide tooling, we will avoid reinventing work that has already been done, and we will increase visibility to the wider project, since reporting and understanding will be standardized. Another benefit being simpler integration with non-Atom teams processes; our tests will be runnable without novel configuration. Regression automation will allow contributors to maintain Atom render code quality while contributing to O3DE.

A code coverage assessment will be used to identify significant gaps in coverage and prioritize future work to fill those gaps with emphasis on unit tests as a priority.

### Initiatives

#### Increased unit test code coverage  

Unit tests are designed to be quick to run and can provide rapid verification of changes by any contributor to O3DE both as a local development verification and as a gated AR element. This represents the lowest cost solution in both implementation and ongoing maintenance. Emphasis should be placed on both increasing coverage and designing meaningful tests. Low feature sprints that focus on delivering tech debt unit tests are one way to address this priority.

*   Unit test creation training
*   O3DE community documentation of how to and expectations of inclusion with contributions
*   Unit test code coverage documentation and guidance

#### Implement robust performance metric collection as an automated process

This effort is already underway in service of goals established by EOY 2022. New metrics goals should be established with automatable means of collecting reproducible numbers to monitor progress.

#### Expand support for batched test case patterns to include more tooling than just Editor

With the addition of batch support for Material Editor we will seek to expand testing on tools outside of Editor.exe. Material Canvas and other tools will be built with the intention that python binding automation will be written for them utilizing behavior context and other ebus systems of interaction through python bindings. This will enable functional integration testing and also provide an entry point for content creation automation tasks. Exposure of function that doesn't require pyside2 UI automation will make efforts more robust and allow for more intuitive process automation.

#### Generate test results for inclusion in pull requests

Coordinating work with other teams, create a system to generate concise test results from local runs of GPU tests for the inclusion with pull requests. This addresses the lack of GPU nodes for gated AR. Ideally this would be similar to the AR check in that it is a separate requirement for merge approval. Inclusion of results in pull requests follows a model used by Blender to avoid maintaining a fleet of GPU enabled nodes.

*   Specify requirements which could vary from simple pass/fail text string, to generating some sort of hashed result string, to local execution tool having a pull request field that directly submits pass/fail data to github.
*   Implement solution
*   PR could include a test artifact file that we create a precheck github action for, although this could make Pull Requests more complicated to undertake.

#### Create sample prefabs that cover the test scenarios of major render components

Creating sample prefabs which lay out major render components with various options enabled will permit more rapid regression testing and provide a target for future screen compare testing.

#### Convert AtomSampleViewer feature coverage to AutomatedTesting editor project

Existing samples and test content in ASV should transition to AutomatedTesting where it is more accessible to customers and where it can be added to periodic or gated AR.

#### Review rev the engine and other workflow scenarios for additional coverage targets

Coverage should support tech artist user workflows as they are the primary use case customer.

#### Static Code Analysis

Inclusion of static code analysis as part of the pull request system would have developers either fixing issues or declaring them false positive during a pull request review. Developers will also likely run static code analysis locally before making a pull request to reduce the amount of churn in their request.

*   [https://www.incredibuild.com/blog/top-9-c-static-code-analysis-tools](https://www.incredibuild.com/blog/top-9-c-static-code-analysis-tools)

#### Dynamic Code Analysis as part of AR

since AR has built and executed code, the application of dynamic code analysis is a possibility. This represents a nice to have priority that depends on other teams to implement.

#### Expose Material Editor and Material Canvas to Multi-Test Framework

The batched execution pytest system, [Multitest Framework](#Multitest Framework (pytest related)), should have ability to execute tests against tools outside of Editor.exe. New tests should be composed using the Multitest Framework.

  

* * *

Test Automation Pyramid
-----------------------

The industry standard approach is the test automation pyramid because it decreases costs and speeds up detection of defects. The test automation pyramid philosophy is required for a successful agile CI/CD environment.

The transition from full manual to automation software testing typically starts by directly replacing the manual steps with front end UI automation since this seems like a easy win (human clicking button now replaced by automation clicking button). These front end tests are slower, expensive, and occur at the "full stack" end to end level of a program. This is why a lot of software developers, moving to a more agile process, question the effectiveness of automated tests; it is because they are running too many of the wrong type of tests. This is known as the “test automation ice cream cone” or “inverted pyramid” and it is the inefficient method of running automated tests in an agile CI/CD environment.

Assuming you have the expected minimum 80% of code coverage for test automation, then the amount of tests you should have are 60% unit tests, 30% integration tests, and 10% GUI tests. This can fluctuate a bit depending on project, but shouldn't be that much different than these baseline % numbers. So let's say you have 1000 tests that achieve 80% coverage of your features and code, the breakdown would be: 600 unit tests, 300 integration tests, and 100 GUI tests. This ideal distribution of test types is rarely achieved in practice, but acts as a guide for where to shift emphasis for additional tests. Ultimately the execution time and reliability constraints will govern the total number of tests possible. Given that unit tests are designed to execute quickly and unlikely to have reliability issues, we will always benefit more from adding more unit tests. Even with parallelized test execution, time to complete a run becomes a constraint on the number and type of tests you can include. The test pyramid also motivates developers to write more unit tests because more unit tests = more integration and GUI tests as well, since the bottom part of the pyramid feeds to the upper parts. Of course, the risk here would be higher costs, but we can evaluate that as we add tests and observe the pipeline. Higher costs generally come from maintaining and creating complex full stack tests which are difficult to debug and determine root cause of issues with. Higher costs can also arise from expenses of running tests on limited resources like GPU enabled EC2. Whenever possible expensive UI tests should be shifted to integration tests by exposing controls to underlying systems for automation hooks (using PySide2 is a last resort when developers fail to expose adequate hooks). Integration tests should be bolstered by many unit tests. Eventually constraints may lead to pruning of UI and integration tests inclusion; if unit tests are not present to cover the same area, gaps may form.

We want to insist on highest standards, but balance that with frugality and deliver results since unlimited time and resources are not available. Investing in the most cost efficient unit tests will reap the highest return. Maintaining a healthy level of integration tests will achieve confidence in full stack systems but should not be at the expense of unit tests.



![test_automation_pyramid_inverted](https://user-images.githubusercontent.com/26234397/194173309-f95f0797-8b72-4e24-b680-ddceec08e169.png)

**The Inverted Pyramid or "ice cream cone" - Inefficient approach to test automation.**


![test_automation_pyramid](https://user-images.githubusercontent.com/26234397/194173519-c42b4178-887e-41d0-892d-9f2c753c02da.png)

**Test Automation Pyramid - the preferred way to plan test automation in the pipeline.**

Scope
-----

All Atom render stack components and tooling. This includes functionality within Editor, Material Editor, Material Canvas, AtomSampleViewer and various subsystems for shader

Atom is not responsible for establishing a build pipeline code coverage solution, but when one becomes available we will adopt it. In the meantime, we expect developers to use available workstation solutions such as OpenCPPCoverage. 

Risks
-----

Limited GPU capacity for test automation. Fewer tests can run on null renderer since actually rendering output is the ultimate objective of a render pipeline.

Editor Python Bindings (Hydra) tests are dependent on UX consistency for may of their functions. Changes like the removal of non-uniform scaling can introduce failures in existing automation.

Platform automation has limited solutions available.

Finding owners and driving progress in an open source project can be difficult

Dependencies
------------

Some testing will require GPU resources and will be marked for this with cmake requires entries.

O3DE frameworks including LYTT and Editor Python Bindings.

Code Coverage
-------------

A code coverage target of 80% for new code is a standard followed by most teams. Code coverage is a helpful indication for confidence in quality, however, unit tests do not tell you if the function of your code is correct from an end user perspective only if the units of code are as asserted. It is possible to have 95% coverage and yet still have significant bug count once units of code integrate with other systems. Conversely, we can conclude that with low coverage numbers we cannot have confidence that the units of code are correct beyond compilation, it is an unknown quality state.

Ideally we would have this integrated with build pipeline.

OpenCPPCoverage can be used by individual engineers.

Code Coverage obtained manually is a critical source of gap analysis to identify where to add tests.

Developers should install OpenCPPCoverage visual studio plugin and check coverage for their tests to identify and augment gaps. Developers should include information about code coverage in code reviews.

![OpenCPPCoverage](https://user-images.githubusercontent.com/26234397/194173598-2c17dc87-a98b-449b-a96a-b1a49e37d3cd.png)


Performance Testing
-------------------

Specific performance testing plans are covered elsewhere.

Soak Testing
------------

Soak tests represent a very long duration run intended to identify stability and memory leak issues. We could seek to run something like this on a jenkins node, or simply run it locally on a machine. Since the primary objective is identifying stability issues we might run this within visual studio using debug so that detailed call stacks are available. We would script a recurring execution of the same tests over and over and leave it running for more than 12 hours possibly over a weekend.

Fuzz Testing
------------

Early incarnations of this engine have historically assumed all data inputs are without issue. Fuzz testing is a technique that alters input data to generate variations that may be invalid in order to harden systems such as file I/O to increase stability. Many systems in Atom are ingesting json content some are streaming textures and models; all of these file assets may introduce corruption or invalid content. Atom should remain stable or at least identify the non-valid content and gracefully exit.

Fuzz Testing involves the use of tools with an iterative modification of content external to the executable. Issues uncovered may be revealed in asset processing or when resources are loaded.

API Testing
-----------

This is primarily accomplished with unit tests and reflects both valid and invalid inputs against API. Using constructs of unit testing you can reach error conditions with intentionally invalid inputs and then mark the test for the expected error condition to validate that the API fails correctly as well as functioning with valid inputs. This is how you test API error handling like testing for null pointers if done in API.

Fault Injection Testing
-----------------------

Often referred to as chaos monkey or gremlin testing, this involves using test versions of low level structures or API that inject random failure or invalid data in unplanned intervals. This is more common with software as a service and may not apply easily to engine stack. Dependency injection design patterns would be more suitable for this since a fault injection dependency can be injected instead of the regular version. we could look at introducing alternate versions of low level classes for type or other systems. The key here is that the consumer of the fault injection has to be designed with a recovery handling system that can deal with the injected faults. A highly performant render engine stack may not be designed to be fault tolerant intentionally.

Platform Testing
----------------

#### General Requirements

Platforms should be reached through common mechanisms like jenkins nodes and LYTT. With this in mind it is important that python used in LYTT be platform-agnostic. Unit tests may be the only tests run on many platforms.

#### Windows PC

This is the primary platform for almost all automation. Automation tests will run locally on developer PC and will run on jenkins nodes. Jenkins nodes will primarily be without GPU, automation must specify a GPU requirement if it needs GPU jenkins nodes. GPU required tests will be included in the nightly or periodic runs due to very limited resources available (far fewer GPU nodes than would be required for frequent runs). Create tests that can function without GPU whenever possible. Any test that involves a GPU screenshot should also have a companion test that can examine similar aspects without the GPU requirement (test component properties for valid content separately from a screenshot)

PC will cover both DirectX 12.1 and Vulkan Render Hardware Interface (RHI)

#### macOS

Automation will run locally on developer macOS machine. Jenkins nodes for macOS would be required, build nodes exist but GPU resources are to be determined. This will use METAL RHI.

#### Linux (dedicated server and GUI based)

Automation should run locally on a developers linux physical or EC2 instance (GPU required for some tests). EC2 jenkins nodes would support linux automation.

This utilizes the Vulkan RHI.

#### iOS

Manual testing pending more information.

#### Android

Manual testing pending more information.

A mobile integration testing solution will be needed for both Android and iOS.

#### OpenXR

Rendering to VR/AR using OpenXR with primarily an Android system. Listed here even though this is more a feature than a platform since it has unique test considerations. This is primarily a manual testing coverage option, although we can pursue running this on PC with multiple render outputs and screen compare automation.

  

Supporting Systems
------------------

### Automated Review (AR)

Owner: O3DE outside of Atom

A required check for merge of github PR. This system builds and runs an automation suite that gates code. On the PR page this is labeled `continuous-integration/jenkins/pr-merge` 

*   [Git Workflow](https://www.o3de.org/docs/contributing/to-code/git-workflow/)

### Unit/Integration Testing Using AZTest

True unit tests are needed for platform testing. Integration "unit" tests are also using these unit test frameworks, but calling multiple modules and using live external dependencies.

It is expected that unit tests will achieve 80% code coverage. Preference is for true unit tests because platform testing may only include true unit tests meaning that integration tests may never be run on non-windows.

*   [Using AzTest](https://www.o3de.org/docs/user-guide/testing/aztest/aztest/)

### CMake/Ctest

As part of AR, LYTT tests suites are declared in CmakeLists.txt files for discovery by `ctest\_entrypoint.cmd`. The ctest suite will specify a python file containing the suite of tests to run. Additional metadata for resources and dependencies are specified in the CmakeLists.txt.

*   [Getting Started with Test Automation](https://www.o3de.org/docs/user-guide/testing/getting-started/)

### Python LYTestTools (LYTT pytest)

Owner: O3DE outside of Atom

LyTestTools is a pytest based framework for running tests. This can manage fixtures and launch application under test. This provides common functionality for path resolution and other utilities used by other systems.

*   [Getting Started with LyTestTools](https://www.o3de.org/docs/user-guide/testing/lytesttools/getting-started/)

### Multitest Framework (pytest related)

This is a custom framework over top of pytest that implements a collection routine that allows for batched and prallel execution of EPB in editor tests. More specialized than LYTT.

Multitest Framework includes support for Material Editor, Canvas Editor, and expansion potential for other tools beyond editor.

*   [Parallel Pattern & Batching Test-Writing](https://www.o3de.org/docs/user-guide/testing/parallel-pattern/)
*   [Multitest Framework (Running multiple tests in one executable)](https://github.com/o3de/o3de/wiki/Multitest-Framework-(Running-multiple-tests-in-one-executable))


### Editor Python Binding (EPB aka Hydra)

Cons:

*   Only supports in editor testing on platforms that will run editor. Will not work on mobile or game launcher only platforms.
*   Project dependency on Editor Python Bindings Gem as well as others

Owner: O3DE outside of Atom

This is a python interpreter within editor that uses built in interfaces to execute editor functions and call through to ebus for newer features. This allows an Editor session to create levels and content on the fly and dynamically reference content by entityId and component to achieve functional tests within the editor. This python context has no exposure to the LYTT framework. These are distinct python scripts that cannot be run outside the editor. They are invoked by launching the editor with specific script execution arguments. This was originally for task automation for content creators.

Useful for UI level testing but is Windows Editor limited so we cannot benefit from any rendering related tests across platforms. Tests utilizing this should be focused on UI functionality or features which have ebus hooks for EPB.

This is close to but not strictly UI testing since it typically uses ebus interfaces rather than UI controls. This still represents a somewhat fragile high maintenance cost solution. Breaking changes to interfaces happen from time to time.

*   [Automating the O3DE Editor with the Python Editor Bindings gem](https://www.o3de.org/docs/user-guide/editor/editor-automation/)

### PySide2 and QtPy

Owner: O3DE outside of Atom

Qt solution for python to reach editor controls not exposed to EPB. This should be used as a last resort since it is straight up UI automation which can be fragile and has a high cost of maintenance.

### AtomSampleViewer (ASV) lua driven integration tests

Owner: Atom

Cons: not all imgui functionality in ASV is exposed to lua automation

Lua exposed functionality to run AtomSampleViewer samples with built-in image comparison functionality and some ability to modify sample options in lua.

Because this is a separate project repository, the entry requirement means this may not be utilized by developers. An effort is planned to move testing from ASV to AutomatedTesting project in core O3DE. ASV will continue to be a platform for developing new features but automation will transition to core O3DE.

### WARP (Windows Advanced Rasterization Platform)

Pros:

*   Can run on any Windows platform
*   cheaper automation nodes utilized

Cons:

*   100x slower in framerate than GPU typically. Run times may be impacted.
*   Only emulates Windows DirectX up to 12.0 (Vulkan and non-DX platforms will be unable to use this)
*   Windows DX reference adapter, however may require special code options to avoid failures that are unique to WARP (just as Nvidia and AMD GPU may require special code for hardware issues)
*   Since this is a different DX adaptor than the standard one, there are a number of compatibility issues that require fixing (in theory there shouldn't be, but there are crashes)
*   Microsoft has no clear intention to support this with required fixes to enable its use.

Owner: Microsoft but this appears to be inactive

Cost of GPU test instances means that if we want to avoid bottlenecks of instance availability we must utilize software rasterized tests wherever possible. Because of limitations an inactive support we should not spend time on this solution.

Running with WARP command line argument

`-forceAdapter="Microsoft Basic Render Driver"`

*   [https://docs.microsoft.com/en-us/windows/win32/direct3darticles/directx-warp](https://docs.microsoft.com/en-us/windows/win32/direct3darticles/directx-warp)

  

Process Considerations
----------------------

### Test Qualities

Tests will be deterministic (they will have consistent results based on inputs)

Tests will be atomic (they will have all fixtures and creation steps handled within their steps and will not rely on order of execution)

Tests will isolate resource requirements so that non-render aspects are tested separately from GPU requirements

Tests will minimize reliance on AssetProcessor for success conditions. File I/O is not reliable enough to gate AR passing.

### Image Comparison (screenshot) Tests

Atom has established image comparison systems that we will utilize. Atom is a render pipeline so it can be difficult to establish abstract pass/fail without visual inspection; therefore, we use image comparison for automation.

Where possible we will set comparison thresholds tightly so that render system drift over time will be detected

Screenshot tests should be composed to focus on the component under test limiting inclusion of extra content like background skybox or ground textures. specific tests should be separately address components like skybox, grid and ground surface. Mixing multiple components in one test composition risks causing large portions of tests to fail for one common component issue. As a distinct test for integration purposes we may have screenshot compositions that include complex content with interactions of many components.

### Timeouts

cmakelists.txt defines a timeout and additionally LYTT calls to launch an application under test also have a timeout. if multiple RHI providers are called in a test ensure that the cmakelists timeout is a multiple of RHI providers timeouts and some additional. For example if you are testing DX12 and Vulkan and the LYTT editor launch needs 60 seconds to complete the timeout in LYTT should be greater (say 100 seconds) and then the cmakelists.txt timeout should be double that or at least 200 seconds (maybe 300 with buffer for overhead).

cmakelists.txt changes require a cmake reconfigure to be taken up for testing.

Ultimately there are test performance limits expected so timeouts should reflect the length required for the test, but if the test requires a longer duration, it may need to move to a periodic suite.

### Github automated tests workflow

New AR tests should go to development branch. Pull requests should include results that show the touched suite continues to pass reliably.

Only bug fixes to existing AR tests would go to stabilization.

No AR changes will go directly to main unless extremely special circumstances like a security hotfix and then by way of a hotfix branch.

Existing AR tests which are deemed unreliable would follow established procedures for demotion to sandbox.

### Credentials and Secrets

No credentials or secrets will be stored in script files or source controlled files. Coordinate with infrastructure for the best solution to your credentials and secrets needs. This may include jenkins secrets, environment variables, AWS SecretsManager or AWS Systems Manager Parameter Store or other approved solutions.

### Documentation

Automated tests will include detailed comments which document purpose and function sufficiently for external contributors to understand these aspects of the test.

Comments will not reference internal resources since access will be limited.

FAQ
---

#### Q: Is there more information about this fascinating testing pyramid?

A: For more reading about this concept from other SDETs & automation engineers, check these links:

*   [https://www.devopsgroup.com/insights/resources/diagrams/all/the-testing-pyramid/](https://www.devopsgroup.com/insights/resources/diagrams/all/the-testing-pyramid/)
*   [https://www.intentsg.com/test-pyramid/](https://www.intentsg.com/test-pyramid/)
*   [https://www.mountaingoatsoftware.com/blog/the-forgotten-layer-of-the-test-automation-pyramid](https://www.mountaingoatsoftware.com/blog/the-forgotten-layer-of-the-test-automation-pyramid)
*   [https://martinfowler.com/bliki/TestPyramid.html](https://martinfowler.com/bliki/TestPyramid.html)
*   [https://www.james-willett.com/the-evolution-of-the-testing-pyramid/](https://www.james-willett.com/the-evolution-of-the-testing-pyramid/)
*   [https://www.lambdatest.com/blog/how-agile-teams-use-test-automation-pyramid/](https://www.lambdatest.com/blog/how-agile-teams-use-test-automation-pyramid/)
*   [https://www.agilecoachjournal.com/2014-01-28/the-agile-testing-pyramid](https://www.agilecoachjournal.com/2014-01-28/the-agile-testing-pyramid)
*   [https://en.wikipedia.org/wiki/Project\_management\_triangle](https://en.wikipedia.org/wiki/Project_management_triangle)

