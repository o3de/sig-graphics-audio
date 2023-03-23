# Maintaining and Authoring Editor Python Bindings Automation Tests In the Atom Code Base

## 1.0 Introduction:

Editor Python Bindings (EPB) are python based hooks for executing automation within the O3DE editor context. These bindings are exposed through modules in the `azlmbr` grouping that are only executable within the O3DE editor. Event busses (ebus) and direct methods are exposed in each `azlmbr` module.

Editor Context properties and busses are not accessible in game launcher; EPB automation doesn't easily translate to game launcher scope. Behavior context busses are similar to editor context but don't cover the same functionality as many EPB.  Behavior context are interfaces (ebus and properties) used by game logic in game mode. More information about [Reflection Context](https://www.o3de.org/docs/user-guide/programming/components/reflection/) and [Event Messaging](https://www.o3de.org/docs/user-guide/programming/messaging/) can be found in the documentation.

In an editor session, using these ebus and method calls found in `azlmbr` modules you can open levels, enter/exit simulation game mode, and perform complex content creation tasks. Ebus calls can create new entities and add new components to those entities. Component properties can be modified and complex relationships can be created between entities and components.

Atom automation primarily uses EPB automation for component verification in the Automated Review (AR) system. Atom components can be instantiated in an editor session using `-rhi=null` so that the basic component can be tested without requiring GPU render. RPI level function will occur when `-rhi=null` so the components are still exercising some code.
We take advantage of helpers to make these ebus calls more object-oriented; specifically [editor_entity_utils.py](https://github.com/o3de/o3de/blob/development/AutomatedTesting/Gem/PythonTests/EditorPythonTestTools/editor_python_test_tools/editor_entity_utils.py).

## 2.0 Structure of Editor Python Test Code

### 2.1 Basic structure

The basic structure of an EPB test as documented in [Parallel Tests](https://www.o3de.org/docs/user-guide/testing/parallel-pattern/#write-individual-editor-tests) can be seen in the following:
```python
def MyTestFunction():
    pass  # do something to test here starting with imports

if __name__ == "__main__":
    from editor_python_test_tools.utils import Report
    Report.start_test(MyTestFunction)
```

### 2.2 Basic Entity and Components

Components are internally referenced with UUID, but we don't expect anyone to know UUID's and use them so helper methods in [editor_entity_utils.py](https://github.com/o3de/o3de/blob/3311941a9204b1593f9406db614f87641f12e1a5/AutomatedTesting/Gem/PythonTests/EditorPythonTestTools/editor_python_test_tools/editor_entity_utils.py#L482) use component names as strings to look up the UUID for you and add components. We utilize atom_constants.py class [AtomComponentProperties](https://github.com/o3de/o3de/blob/3311941a9204b1593f9406db614f87641f12e1a5/AutomatedTesting/Gem/PythonTests/Atom/atom_utils/atom_constants.py#L194) (which has static methods for each component) to store properties for components including the component name and property paths. By writing AtomComponentProperties.mesh() in your code you'll get back the component name string 'Mesh' which you can use for asking for the addition of a Mesh component like this:
```python
def MyTestFunction():
    from editor_python_test_tools.editor_entity_utils import EditorEntity
    from editor_python_test_tools.wait_utils import PrefabWaiter
    from editor_python_test_tools.utils import Report, Tracer, TestHelper
    from Atom.atom_utils.atom_constants import AtomComponentProperties

    # Test setup begins.
    # Setup: Wait for Editor idle loop before executing Editor Python Bindings scripts then open "Base" level.
    TestHelper.init_idle()
    TestHelper.open_level("Graphics", "base_empty")
    
    my_entity = EditorEntity.create_editor_entity('Entity_Mesh')  # creates an entity named 'Entity_Mesh'
    
    # my_mesh_component = my_entity.add_component('Mesh')  # this adds a 'Mesh' component to my_entity.
    # for code maintainability we would use AtomComponentProperties.mesh() which returns 'Mesh' 
    # that way if the name changes our code only needs to update one place in atom_constants.py
    my_mesh_component = my_entity.add_component(AtomComponentProperties.mesh())

if __name__ == "__main__":
    from editor_python_test_tools.utils import Report
    Report.start_test(MyTestFunction)
```
:white_check_mark: Import only the objects you plan to use. In code usage will be cleaner when you `from module import MyClass`, then you don't need to alias it with `import module as alias` you just call MyClass directly.

look at [hydra_AtomEditorComponents_MeshAdded.py line 184](https://github.com/o3de/o3de/blob/4607610bc8e98f70a2a89123fdcb3d6ed5d5be4e/AutomatedTesting/Gem/PythonTests/Atom/tests/hydra_AtomEditorComponents_MeshAdded.py#L184) and you'll find the same basic calls to create an entity named 'Mesh' (we call AtomComponentProperties.mesh() which returns 'Mesh' for the entity name) it then adds a Mesh component to that. Notice that we save references to the entity object and the component object as variables; that allows us to do things with those specific objects. If we wanted to set `my_mesh_component` to not use ray tracing we could do the following:
```python
    my_mesh_component.set_component_property_value(AtomComponentProperties.mesh('Use ray tracing'), value=False)
```
`'Use ray tracing'` in this case is a component property with a property path stored in AtomComponentProperties. The function `set_component_property_value` uses a property path to set a value on the component. In this case the property path looks like `'Controller|Configuration|Use ray tracing'`

### 2.3 Tracer for Collecting AZ_Error, AZ_Assert and Other Messages

Test code is wrapped `with Tracer() as error_tracer:` This allows us to specifically collect messages. Atom tests only collect these, they don't fail or otherwise act on `AZ_Error` or `AZ_Assert`. In part this element of the test is covered as:
```python
def MyTestFunction():
    from editor_python_test_tools.utils import Report, TestHelper, Tracer

    with Tracer() as error_tracer:
        # 1. Test Code here
        # 25. Final test step, testing is concluded

        # 26. Look for errors or asserts.
        TestHelper.wait_for_condition(lambda: error_tracer.has_errors or error_tracer.has_asserts, 1.0)
        for error_info in error_tracer.errors:
            Report.info(f"Error: {error_info.filename} {error_info.function} | {error_info.message}")
        for assert_info in error_tracer.asserts:
            Report.info(f"Assert: {assert_info.filename} {assert_info.function} | {assert_info.message}")
```
`TestHelper.wait_for_condition(lambda: error_tracer.has_errors or error_tracer.has_asserts, 1.0)` returns focus to the editor main thread for up to 1 second timeout or resumes our script if either errors or asserts have occurred. After the wait, we `Report.info` which is essentially like a print statement where we use f-string for detailed summary of the messages that occurred during testing. Other than essentially printing them, we do nothing with messages. If you were testing for specific message outcomes you could search for them with Tracer(). 

### 2.4 Report Manages Tests and Summary

In the super basic example in section 2.1 we import Report because it contains the static method `start_test()` that calls our test function handling the wrap-up of `get_report` which generates a summary of success/failure.

During our test script we conduct each test by calling `Report.result(("pass", "fail"), bool_test_condition)`. `Report.result` will log a True/False outcome of the test selecting a message string from the tuple (pass: str, fail: str). Execution will continue if `Report.result` encounters a fail. To terminate early on fail use `Report.critical_result`. For the equivalent of print we use `Report.info`.

You'll notice at the top of our tests you'll find a class `Tests` which includes test tuples we use in `Report.result`. In the fail strings we specify the priority of the test to clarify the scale of the failure. We format our tuples in PEP8 using newlines.
```python
class Tests:
    creation_undo = (
        "UNDO Entity creation success",
        "P0: UNDO Entity creation failed")
    creation_redo = (
        "REDO Entity creation success",
        "P0: REDO Entity creation failed")
```

### 2.5 AtomComponentProperties in atom_constants.py has Component Properties

Components have common name strings, and they also have property paths which define the editor context path to each property of the component and are used to get/set values as shown in section 2.2 above. We store all these strings in `AtomComponentProperties` which is a class full of static methods for each Atom component.

Each component static method stores a dictionary list of the properties for that component. The default return from the static method is the dictionary key `name` which stores the common name of the component, so calling without argument returns the common name of the component. Each property of the component is stored as a key value pair where value is generally the property path `'property name': 'Controler|Configuration|path|property name'`.

Some components require shapes which we store a list of in the key 'shapes'. Lists in 'shapes' can be used to itterate over required shapes for coverage.

Some components also have required components which we store in a key 'requires' with a list of component references that may be required.

To generate component property paths for a new component we would write code like the block above where we add 'Mesh' component as `my_mesh_component` (replacing Mesh with the component we want) then use a block of code like the following to dump out all the property paths with type for our component:
```python
    properties = my_mesh_component.get_property_type_visibility()
    for path, value in properties.items():
        Report.info(f"'{path}' {value}")
    return
```
`properties` in this example is a dictionary with key being string property_path and value being a tuple (type, visible) where type is the python marshalled type and visible is an indication if the property is UI visible (some properties are concealed in a minimized section of properties). note that get_property_type() uses a regex that makes assumptions about property path content that on rare occasions are not fulfilled when a property has extended characters; if that happens the method will dump the offending property path in the output and return the dictionary without it. By writing each key:value of the dictionary right after the call you should get all the property paths including any that don't match the regex. Some components will crash the editor when `get_property_type_visibility` is called on them. Those components can be more difficult to extract property paths from.

After creating the basic script above that creates 'Mesh' component then calls `get_property_type_visibility` your script executed in the editor will spew the property paths in the console. You can clear the console then run the script to make it easier to find and copy the output.

### 2.6 How to Find Default Values and Ranges for Properties

You can use the tooltips in the editor or look at the C++ source.

Open the O3DE.sln in visual studio and search for the component name in quotes in all files `*.h` and `*.cpp`. You should see among the found lines a file or files "Editor*Name*Component" and the line where your component name in quotes is found will be preceeded by `editContext->Class`. You're at the editor context reflection for this component. You'll find most if not all the properties for your component specified here. Farther down in this file you should find each property. You'll see min, max and unit type specified (unit type isn't generally important). If you peak the variable being set by this you can find deeper typing information. 
```C++
->DataElement(AZ::Edit::UIHandlers::Default, &SkyAtmosphereComponentConfig::m_groundRadius, "Ground radius", "Ground radius")
    ->Attribute(AZ::Edit::Attributes::Suffix, " km")
    ->Attribute(AZ::Edit::Attributes::Min, 0.0f)
    ->Attribute(AZ::Edit::Attributes::Max, 100000.0f)

->DataElement(AZ::Edit::UIHandlers::ComboBox, &SkyAtmosphereComponentConfig::m_originMode, "Origin", "The origin to use for the atmosphere")
```
if you peak the type for `m_originMode` in the above example you'll find its type is `AtmosphereOrigin m_originMode = AtmosphereOrigin::GroundAtWorldOrigin;`. Peak `AtmosphereOrigin` and you'll find that it's an enum;
```C++
enum class AtmosphereOrigin
{
    GroundAtWorldOrigin,
    GroundAtLocalOrigin,
    PlanetCenterAtLocalOrigin
};
```
These enums are used in ComboBox UI to set options. In AtomComponentProperties we represent these as dictionaries:
```python
#Origin type for Sky Atmosphere component
ATMOSPHERE_ORIGIN = {
    'GroundAtWorldOrigin': 0,
    'GroundAtLocalOrigin': 1,
    'PlanetCenterAtLocalOrigin': 2,
}
```
Be aware that some C++ enums don't use default values or present other complexities so getting the integer values these properties expect can be tricky.

### 2.7 Strategy for Testing Component Properties

The objective for Atom component automation is to exercise all properties. When there are a lot of properties we generally set maximum value and move on. When there are fewer properties we set minimum and maximum values. Optionally, we return to default values. Values which are dropdowns and governed by enums are covered using a for loop to exercise all options.

The general flow of each test is as follows:
```text
Test Steps:
1) Create an entity with no components.
2) Add test component to the entity from step 1.
3) Remove the test component.
4) Undo test component removal.
5) Verify test component is enabled.
6) Set all properties of the test component. 
7) Enter/Exit game mode.
8) Test IsHidden.
9) Test IsVisible.
10) Delete entity.
11) UNDO deletion.
12) REDO deletion.
13) Look for errors and asserts.
```

## 3.0 Running an Editor Script in Editor
You run a script in editor by calling the file at the console prompt using `pyRunFile` with a relative path to your script.py file:
```commandline
pyRunFile ../../Gem/PythonTests/Atom/tests/hydra_AtomEditorComponents_MeshAdded.py
```
:white_check_mark: You can right-click the console output window and clear the existing output before running your script to make the output from your script easier to find

When submitting a code review for new or udpated automation the in editor spew of the summary result is what is recommended that you include in "how was this tested".
[Example PR](https://github.com/o3de/o3de/pull/15074)

## 4.0 Running pytest from Command Line

On Windows, from a command prompt in your O3DE repository workspace:
```commandline
.\python\python.cmd -m pytest --build-directory YOUR_BUILD_FOLDER/bin/profile  .\AutomatedTesting\Gem\PythonTests\Atom\TestSuite_Main_Null_Render_Component_03.py
```
This runs a pytest file which specifies multi-test Editor scripts to run [TestSuite_Main_Null_Render_Component_03.py](https://github.com/o3de/o3de/blob/development/AutomatedTesting/Gem/PythonTests/Atom/TestSuite_Main_Null_Render_Component_03.py)
```python
import pytest

import ly_test_tools
from ly_test_tools.o3de.editor_test import EditorBatchedTest, EditorTestSuite


@pytest.mark.parametrize("project", ["AutomatedTesting"])
@pytest.mark.parametrize("launcher_platform", ['windows_editor'])
class TestAutomation(EditorTestSuite):

    class AtomEditorComponents_SkyAtmosphereAdded(EditorBatchedTest):
        from Atom.tests import hydra_AtomEditorComponents_SkyAtmosphereAdded as test_module

    @pytest.mark.test_case_id("C36525666")
    class AtomEditorComponents_SSAOAdded(EditorBatchedTest):
        from Atom.tests import hydra_AtomEditorComponents_SSAOAdded as test_module
```
`@pytest.mark.test_case_id("C36525666")` is a deprecated test case ID notation that isn't exposed in any useful way.

## 5.0 CTest is how Jenkins Automated Review runs this

On Windows, from a command prompt in your O3DE workspace for an existing collection of tests:
```commandline
.\scripts\ctest\ctest_entrypoint.cmd --build-path YOUR_BUILD_FOLDER --suite main --ctest-executable "C:\Program Files\CMake\bin\ctest.exe" --config profile --generate-xml
```
Use the optional parameter to run specific sets of tests `--tests-regex "AutomatedTesting::Atom_Main_Null"`

Tests are defined in [CMakeLists.txt](https://github.com/o3de/o3de/blob/development/AutomatedTesting/Gem/PythonTests/Atom/CMakeLists.txt)

Note the NAME found in CMakeLists.txt is the what's specified at a CTest command line. A PATH property points to the pytest file that collected editor tests in section 4.0.

TIMEOUT is in seconds. Jenkins will terminate jobs that run longer than 1800 seconds so setting this Ctest timeout to values larger would be terminated by Jenkins. `EditorTestSuite` and `EditorBatchedTest` themselves have a timeout default. Cumulative timeout total for EditorTestSuite should be less than CMakeLists.txt TIMEOUT for a given pytest suite module.

## 6.0 Using Notepad++ to Assemble Code for New Components

You can use Notepad++ search replace once you've figured out a components common name and successfully scripting the creation of that component and dump of the property paths with type as shown in section 2.5.

- Starting with the list of property paths with type do some simple replace with extended characters to chop off the spew prefix and extra lines.
- Locate and delete lines for 'Controller' and 'Configuration' since these are actually grouping stubs not properties you would set.
- Your list of property paths with types your starting point will look like
```text
'Controller|Configuration|Stars Asset' ('Asset<StarsAsset>', 'Visible')
'Controller|Configuration|Exposure' ('float', 'Visible')
'Controller|Configuration|Twinkle rate' ('float', 'Visible')
'Controller|Configuration|Radius factor' ('float', 'Visible')
```
- Switching to regex search for the following
```regexp
(?# search field works for 'Controller|Configuration|Group|Property')
'(([A-z09 ]+)?\|([A-z09 ]+)?\|?([A-z09 ]+)?\|?([A-z09 ]+))' \([\'A-z0-9\, \)]+
(?# replace including initial spaces to get the dictionary for AtomComponentProperties)
            '(?{5}${5}:)': '((?{2}${2}|:)(?{3}${3}|:)(?{4}${4}|:)(?{5}${5}:)',
(?# replace including initial spaces to get the docstrings for AtomComponentProperties)
          - '(?{5}${5}:)'
```
Alternatively you can target specifically for your property paths based on characteristics you can see
```regexp
(?# match for literal 'Controller|Configuration|Property' more reliably even with a group included in some)
'(Controller\|Configuration(\|[A-z0-9 ]+)?)\|([A-z09 ]+)' \([\'A-z0-9\, \)]+
(?# replace with initial spaces to get docstring for AtomComponentProperties)
          - '(?{3}${3}:)'
```
Some types may have cleanup from this regex replace, but if you have a component with 20+ properties this can still speed up your work. In the following you would remove `<StarsAsset>', 'Visible')` since it was missed by regex
```text
          - 'Stars Asset'<StarsAsset>', 'Visible')
          - 'Exposure'
          - 'Twinkle rate'
          - 'Radius factor'
```
From your docstring list of properties you can generate the Tests tuples for basic pass/fail strings
```regexp
(?# search for up to three word properties. add more capture groups if your properties are more wordy)
          - '([A-z09]+) ?(\b[A-z09]+)? ?(\b[A-z09]+)?'\r\n
(?# replace)
    \L${1}(?{2}_${2}:)(?{3}_${3}:)\E \= \(\r\n        \"'${1}(?{2} ${2}:)(?{3} ${3}:)' property set\",\r\n        \"P1\: '${1}(?{2} ${2}:)(?{3} ${3}:)' property failed to set\"\)\r\n
```
Your list of properties from docstring now transforms to Tests tuple
```python
class Tests:
    stars_asset = (
        "'Stars Asset' property set",
        "P1: 'Stars Asset' property failed to set")
    exposure = (
        "'Exposure' property set",
        "P1: 'Exposure' property failed to set")
    twinkle_rate = (
        "'Twinkle rate' property set",
        "P1: 'Twinkle rate' property failed to set")
    radius_factor = (
        "'Radius factor' property set",
        "P1: 'Radius factor' property failed to set")
```
Starting with your docstring list of properties you can generate blocks of code and comments
```regexp
(?# search with initial spaces)
          - '([A-z09 ]+)'\r\n
(?# replace with initial spaces and specify your component rather than sky_atmosphere_component)
        # 11. Set a value for '${1}' property\r\n        sky_atmosphere_component.set_component_property_value\(\r\n            AtomComponentProperties.sky_atmosphere\('${1}'\), 0.0\)\r\n        Report.result\(\r\n            Tests.\L${1}\E,\r\n            sky_atmosphere_component.get_component_property_value\(\r\n                AtomComponentProperties.sky_atmosphere\('${1}'\)\) == 0.0\)\r\n\r\n        
```
```python
        # 11. Set a value for 'Exposure' property
        sky_atmosphere_component.set_component_property_value(
            AtomComponentProperties.sky_atmosphere('Exposure'), 0.0)
        Report.result(
            Tests.exposure,
            sky_atmosphere_component.get_component_property_value(
                AtomComponentProperties.sky_atmosphere('Exposure')) == 0.0)
```