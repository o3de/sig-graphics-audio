# RFC: New Spaces And Actions Architecture For The OpenXRVk Gem
  
## The Problems
1. Regarding `XrSpaces`, the OpenXRVk Gem is hardcoded to support a fixed list of `"Visualized"` spaces. In reality, the OpenXR API is designed to support dynamic creation and destruction of XrSpaces. This RFC addresses the problem by introducing a new interface, `AZ::Interface<IOpenXRReferenceSpaces>`, that is also exposed to the `Behavior Context`, which will provide methods to add, remove and manage new XrSpaces at runtime.
2. Regarding `XrActions`, the OpenXRVk Gem is hardcoded to only support one `XrActionSet` which, in turn, is hardcoded to support only the `Oculus Touch` interaction profile. This RFC addresses the problem by introducing a new data-driven approach, that leverages the O3DE Asset system, to enable developers to customize the list of supported interaction profiles, action sets and actions. This RFC also introduces a new interface, `AZ::Interface<IOpenXRActions>` aka `OpenXRActionsInterface`, that is also exposed to the `Behavior Context`, which will provide methods to poll the state of the different actions as defined in the action-related assets.


## The New OpenXRReferenceSpacesInterface
In OpenXR there are two kinds of **Spaces**, the **Reference Spaces** and the **Action Spaces**. Both kinds of spaces are equally represented by the same handle type known as `XrSpace`.  
Only **Reference Spaces** can be added or removed at runtime, on the other hand, **Action Spaces** are limited by the immutability of the OpenXR ActionSets once they are attached to the session. The **Action Spaces** will be managed by the `OpenXRActionsInterface`, which will be covered later in this RFC.

**Reference Spaces** also work as virtual anchors, and they simply represent a Transform/Pose (Orientation + Position) with respect to another Reference Space. There are three hardcoded Reference Spaces that all OpenXR runtimes must support and they are known as `View`, `Local` and `Stage`. They will be addressable via constant integrals and strings in this new API.

With this new API, the application developer, will address reference spaces by their name, which will be of type `AZStd::string`.  

### Proposed API
```cpp
namespace OpenXRVk
{
    //! REMARK: This class must be instantiated BEFORE OpenXRActionsInterface.
    //! To know more about OpenXR Reference Spaces, see the spec:
    //! https://registry.khronos.org/OpenXR/specs/1.0/html/xrspec.html#spaces
    //! This interface consolidates the API that works with definition of Reference Spaces
    //! and retreival of their Poses as AZ::Transforms.
    class IOpenXRReferenceSpaces
    {
    public:
        // Identifiers for all reference spaces that all OpenXR implementations
        // must support. These are the default system spaces. Most OpenXR implementations
        // support additional system-provided reference spaces, and you can try to use their id to attempt
        // to create an space of them at runtime. This explains why this is an ordinary integral
        // instead of an enum.
        using ReferenceSpaceId = uint32_t;
        static constexpr ReferenceSpaceId ReferenceSpaceIdView = 1;
        static constexpr ReferenceSpaceId ReferenceSpaceIdLocal = 2;
        static constexpr ReferenceSpaceId ReferenceSpaceIdStage = 3;
        // With these names you can refer to the default system spaces.
        static constexpr char ReferenceSpaceNameView[] = "View"; // Typically represents the user's head centroid.
        static constexpr char ReferenceSpaceNameLocal[] = "Local"; 
        static constexpr char ReferenceSpaceNameStage[] = "Stage";

        //! @returns a list of strings, Where each string is the name of each active reference space.
        //! The session will always instantiate the three reference spaces mentioned above: View, Local and Stage.
        //! So this list will always contain at least three strings.
        virtual AZStd::vector<AZStd::string> GetReferenceSpaceNames() const = 0;

        //! Creates a new Reference Space, and the developer will define what name they want to use to address the new space.
        //! @param referenceSpaceType An integral, that identifies a system reference space, that will be used as anchor reference
        //!        for the new space that will be created. The caller can use other integrals, different than the three constants
        //!        mentioned above, but the caller would need to verify on their own that said system reference space is supported
        //!        by the OpenXR Runtime they are working with. OpenXR only guarantess that View, Local and Stage are always supported.
        //! @param spaceName Name of the new space, and must be different than any other reference space previously created.
        //! @param poseInReferenceSpace A transform that defines the default orientation and position of the new space, relative
        //!        to the system reference space defined by @referenceSpaceType.
        //! @returns Success if the space name is unique, and the @referenceSpaceType space id is supported by the runtime.
        //!          In case of failure, an error description is provided.
        virtual AZ::Outcome<bool, AZStd::string> AddReferenceSpace(ReferenceSpaceId referenceSpaceType,
            const AZStd::string& spaceName, const AZ::Transform& poseInReferenceSpace) = 0;

        //! Removes a previously user-created reference space.
        //! @param spaceName The name of the reference space to remove.
        //! @returns Success if a reference space with said name exists and it is NOT one of "View", "Local" or "Stage".
        virtual AZ::Outcome<bool, AZStd::string> RemoveReferenceSpace(const AZStd::string& spaceName) = 0;

        //! REMARK: Maybe it's a good idea to move this into a private interface for the OpenXRVk Gem,
        //! and privately use the native handle (XrSpace)
        virtual const void * GetReferenceSpaceNativeHandle(const AZStd::string& spaceName) const = 0;

        //! Returns the Pose of @spaceName relative to @baseSpaceName.
        //! This function is typically called within the game loop to poll/query the current pose
        //! of a particular reference space, relative to another reference space.
        //! @param spaceName The name of the reference space whose Transform the caller needs to know.
        //! @param baseSpaceName The name of the base reference space that the OpenXR runtime will use to calculate
        //!                      a relative Transform.
        //! @returns If successful, a Transform that defines the pose of @spaceName, relative to @baseSpaceName.
        virtual AZ::Outcome<AZ::Transform, AZStd::string> GetReferenceSpacePose(const AZStd::string& spaceName, const AZStd::string& baseSpaceName) const = 0;

        //! With this method the caller defines the base reference space that will be used each frame
        //! to locate the "View" reference space. Each frame, The "View" reference space is the only
        //! reference space that will be automatically located, and its pose will be internally cached.
        //! The located pose will always be relative to the @spaceName defined the last time this method
        //! was called.
        //! If this method is never called, the runtime will default to the "Local" reference spaces as the base reference space
        //! that will be used to locate the "View" reference space.
        //! This is a stateful  API, you only need to call this method once, or as needed.
        //! Each time GetViewSpacePose() is called, the returned Transform will be relative to @spaceName
        //! @param spaceName The name of the base reference space that will be used to caculate the pose of the View space.
        //! @remark Typically, the "View" reference space represents the User's head centroid.
        //!         Internally, the Pose for each Eye View will be calculated relative to the View Space centroid
        //!         and it will be reported via the notification bus: AZ::RPI::XRSpaceNotificationBus 
        virtual AZ::Outcome<bool, AZStd::string> SetBaseSpaceForViewSpacePose(const AZStd::string& spaceName) = 0;

        //! Returns the name of the current Reference Space that is used as base reference space
        //! when locating the "View" reference space.
        virtual const AZStd::string& GetBaseSpaceForViewSpacePose() const = 0;

        //! @returns The per-frame "calculated and cached" Pose of the "View" Reference Space.
        //! @remark This same Pose is also reported automatically, each frame, via the AZ::RPI::XRSpaceNotificationBus.
        //!         Also the returned Pose is a Transform relative to the base Reference Space defined via
        //!         SetBaseSpaceForViewSpacePose().
        virtual const AZ::Transform& GetViewSpacePose() const = 0;

        //! Useful constants for OpenXR Stereo systems.
        static constexpr uint32_t LeftEyeView = 0;
        static constexpr uint32_t RightEyeView = 1;

        //! For an AR application running on a Phone (Mono) the count will be 1.
        //! For an AR/VR application running on a typical headset (Stereo) the count will be 2.
        //! Some headsets like the Varjo support Quad Views and the eye (view) counts will be 4. 
        virtual uint32_t GetViewCount() const = 0;

        //! Pose of a view(aka eye) relative to the View Space (aka Head) Pose.
        //! For AR applications running on a Phone (aka Mono view configuration) the Eye View Pose
        //! is centered exactly where the View Space Centroid is located, so calling this function wouldn't
        //! make much sense because it'll return an Identity transform.
        //! @remark DO NOT CONFUSE View Pose with "View" Space Pose. This is a confusing aspect
        //!         of the OpenXR API. It is tempting to call this function as GetEyePose.
        virtual const AZ::Transform& GetViewPose(uint32_t eyeIndex) const = 0;

        //! Returns the FovData for a particular View (aka Eye).
        virtual const AZ::RPI::FovData& GetViewFovData(uint32_t eyeIndex) const = 0;

        //! A convenient method that returns the relative pose of all Views (aka Eyes)
        //! relative to the "View" Reference Space, which typically represents the user's head centroid.
        virtual const AZStd::vector<AZ::Transform>& GetViewPoses() const = 0;

        //! Forces updating the cached pose and projection data for all Views (Eyes).
        //! Each frame, all view (aka eye) poses and projections are updated automatically, making this function optional to use.
        //! By default the caller simply gets the per-frame cached version, which should be fine for most applications.
        //! This API was added because according to OpenXR xrLocateViews spec:
        //! "Repeatedly calling xrLocateViews with the same time may not necessarily return the same result.
        //!  Instead the prediction gets increasingly accurate as the function is called closer to the
        //!  given time for which a prediction is made".
        virtual void ForceViewPosesCacheUpdate() = 0;
    };

    using OpenXRReferenceSpacesInterface = AZ::Interface<IOpenXRReferenceSpaces>;

} // namespace OpenXRVk
```

The API mentioned above will be implemented by the class `OpenXRVk::ReferenceSpacesManager`, which will be instantiated and owned by the OpenXRVkSession.

Regarding Behavior Context reflection, which enables scripting with Lua or Python, the API looks like:
```cpp
context.Class<OpenXRReferenceSpaces>("OpenXRReferenceSpaces")
    ->Attribute(AZ::Script::Attributes::Scope, AZ::Script::Attributes::ScopeFlags::Common)
    ->Attribute(AZ::Script::Attributes::Module, "openxr")
    ->Constant("ReferenceSpaceIdView", BehaviorConstant(IOpenXRReferenceSpaces::ReferenceSpaceIdView))
    ->Constant("ReferenceSpaceIdLocal", BehaviorConstant(IOpenXRReferenceSpaces::ReferenceSpaceIdLocal))
    ->Constant("ReferenceSpaceIdStage", BehaviorConstant(IOpenXRReferenceSpaces::ReferenceSpaceIdStage))
    ->Constant("ReferenceSpaceNameView", BehaviorConstant(AZStd::string(IOpenXRReferenceSpaces::ReferenceSpaceNameView)))
    ->Constant("ReferenceSpaceNameLocal", BehaviorConstant(AZStd::string(IOpenXRReferenceSpaces::ReferenceSpaceNameLocal)))
    ->Constant("ReferenceSpaceNameStage", BehaviorConstant(AZStd::string(IOpenXRReferenceSpaces::ReferenceSpaceNameStage)))
    ->Constant("LeftEyeViewId", BehaviorConstant(IOpenXRReferenceSpaces::LeftEyeView))
    ->Constant("RightEyeViewId", BehaviorConstant(IOpenXRReferenceSpaces::RightEyeView))
    ->Method("GetReferenceSpaceNames", &OpenXRReferenceSpaces::GetReferenceSpaceNames)
    ->Method("AddReferenceSpace", &OpenXRReferenceSpaces::AddReferenceSpace)
    ->Method("RemoveReferenceSpace", &OpenXRReferenceSpaces::RemoveReferenceSpace)
    ->Method("GetReferenceSpacePose", &OpenXRReferenceSpaces::GetReferenceSpacePose)
    ->Method("SetBaseSpaceForViewSpacePose", &OpenXRReferenceSpaces::SetBaseSpaceForViewSpacePose)
    ->Method("GetBaseSpaceForViewSpacePose", &OpenXRReferenceSpaces::GetBaseSpaceForViewSpacePose)
    ->Method("GetViewSpacePose", &OpenXRReferenceSpaces::GetViewSpacePose)
    ->Method("GetViewCount", &OpenXRReferenceSpaces::GetViewCount)
    ->Method("GetViewPose", &OpenXRReferenceSpaces::GetViewPose)
    //TODO: Serialize AZ::RPI::FovData
    //->Method("GetViewFovData", &OpenXRReferenceSpaces::GetViewFovData)
    ->Method("GetViewPoses", &OpenXRReferenceSpaces::GetViewPoses)
    ->Method("ForceViewPosesCacheUpdate", &OpenXRReferenceSpaces::ForceViewPosesCacheUpdate)
     ;
```
  

### Practical API Usage of OpenXRReferenceSpacesInterface
A basic application would not need to create new Reference Spaces, but most likely would need to know where is the user's head located in the virtual world relative to the `Local` Reference Space. In c++ would look like (No error checking, for simplicity):
```cpp
auto spacesInterface = OpenXRReferenceSpacesInterface::Get();
// headTm will have the pose of the user's head relative to the `Local` reference space.
const AZ::Transform& headTm = spacesInterface->GetViewSpacePose();
// Let's say the user will shoot rays that come straight from their eyes,
// we'll need the world orientation+location of both eyes:
// 1st, let's get the eyes orientation relative to the user's head.
const AZ::Transform& leftEyeTm = spacesInterface->GetViewPose(IOpenXRReferenceSpaces::LeftEyeView);
const AZ::Transform& rightEyeTm = spacesInterface->GetViewPose(IOpenXRReferenceSpaces::RightEyeView);
// With simple Transform multiplication we can get each eye's transfrom relative
// to the `Local` space:
const AZ::Transform leftEyeLocalTm = headTm * leftEyeTm;
const AZ::Transform rightEyeLocalTm = headTm * rightEyeTm;
// And the rest will be typical math to extract the forward vector, etc...
``` 
This is how the same code will look like in Lua:
```lua
local headTm = OpenXRReferenceSpaces.GetViewSpacePose()
local leftEyeTm = OpenXRReferenceSpaces.GetViewPose(OpenXRReferenceSpaces.LeftEyeViewId)
local rightEyeTm = OpenXRReferenceSpaces.GetViewPose(OpenXRReferenceSpaces.RightEyeViewId)
local leftEyeLocalTm = headTm * leftEyeTm;
local rightEyeLocalTm = headTm * rightEyeTm;
```
  
## The New OpenXRActionsInterface Part 1
And now, let's move on to the second part of the RFC, which is related to new APIs to help application developers work with Inputs and Haptics (aka XrActions).  
  
It is tempting to retrofit the OpenXR Input & Haptics APIs into the O3DE Input framework, BUT the approach OpenXR takes regarding user I/O is so different that standard Input Event Mappings won't work in this case. The key difference is that OpenXR was designed with hardware compatibility and extensibility in mind.  
In summary, the application defines a list of actions, identifiable with strings, like "jump", "grab", etc and for each action it defines a list of all the Interaction Profiles and their respective Component I/O Paths that the application plans to support. For details, see this chapter from the OpenXR spec:  
https://registry.khronos.org/OpenXR/specs/1.0/html/xrspec.html#input  
  
The most important thing to notice, is that with just plain text data, an OpenXR application can describe its Input and Haptic needs. And this RFC proposes the definition of two types of Assets to deal with this.  

### The OpenXRInteractionProfilesAsset
This new asset type contains a list of all the interaction profiles and their supported component paths. the content of this asset will be just a set of strings, organized per the following chapter in the OpenXR Spec:  
https://registry.khronos.org/OpenXR/specs/1.0/html/xrspec.html#semantic-path-interaction-profiles  
  
All of that data will be represented with the help of the following C++ classes:  

1. The OpenXRInteractionProfilesAsset:
```cpp
//! This asset defines a list of Interaction Profile Descriptors.
//! The Engine only needs one of these assets, which is used to express
//! all the different interaction profiles (aka XR Headset Devices) that
//! are supported by OpenXR.
//! Basically this asset contains data as listed here:
//! https://registry.khronos.org/OpenXR/specs/1.0/html/xrspec.html#semantic-path-interaction-profiles
class OpenXRInteractionProfilesAsset final
    : public AZ::Data::AssetData
{
public:
    static constexpr char s_assetTypeName[] = "OpenXR Interaction Profiles";
    static constexpr char s_assetExtension[] = "xrprofiles";
    //! The asset is just a list of Interaction Profile descriptors.
    AZStd::vector<OpenXRInteractionProfileDescriptor> m_interactionProfileDescriptors;
};
```  
The asset is just a list of Interaction Profile Descriptors, each descriptor will be represented by the `OpenXRInteractionProfileDescriptor` class.  

2. The OpenXRInteractionProfileDescriptor class:  
```cpp
//! An Interaction Profile descriptor describes all the User Paths and Component Paths that
//! a particular Vendor Equipment supports.  
class OpenXRInteractionProfileDescriptor final
{
public:
    //! Unique name across all OpenXRInteractionProfileDescriptor.
    //! It serves also as user friendly display name, and because
    //! it is unique it can be used in a dictionary.
    AZStd::string m_name;
    //! A string convertible to XrPath like:
    //! "/interaction_profiles/khr/simple_controller", or
    //! "/interaction_profiles/oculus/touch_controller"
    AZStd::string m_path;
    //! All the User Paths that this equipment supports.
    AZStd::vector<OpenXRInteractionUserPathDescriptor> m_userPathDescriptors;
    //! Common Component Paths that are supported by all User Paths listed in @m_userPathDescriptors
    AZStd::vector<OpenXRInteractionComponentPathDescriptor> m_commonComponentPathDescriptors;
};
```
In turn, an Interaction Profile descriptor, is just a list of User Path descriptors and common Component Path descriptors, which are defined as:  

3. The OpenXRInteractionUserPathDescriptor class:
```cpp
//! A User Path descriptor describes the XrPath (as a string) that will be
//! use to identify a Left or Right hand controller, or a Game pad controller. 
class OpenXRInteractionUserPathDescriptor final
{
public:
    //! A user friendly name.
    AZStd::string m_name;
    //! For OpenXR a User Path string would look like:
    //! "/user/hand/left", or "/user/hand/right", etc
    AZStd::string m_path;
    //! Component Paths that are only supported under this User Path.
    //! This list can be empty. In case it is empty, it means that all component Paths
    //! are listed under the Interaction Profile Descriptor that owns this User Path Descriptor.
    AZStd::vector<OpenXRInteractionComponentPathDescriptor> m_componentPathDescriptors;
};
```

4. The OpenXRInteractionComponentPathDescriptor class:
```cpp
//! A Component Path Descriptor identifies Inputs or Haptics
//! available in a particular controller Like the 'X' or 'Y' Buttons
//! or the ability to vibrate (Haptic)
class OpenXRInteractionComponentPathDescriptor final
{
public:
    //! A user friendly name.
    AZStd::string m_name;
    //! For OpenXR a Component Path string would look like:
    //! "/input/x/click", or "/input/trigger/value", etc
    AZStd::string m_path;
    //! Whether this is a boolean, float, vector2 or pose.
    //! The user will be presented with a combo box to avoid
    //! chances for error.
    AZStd::string m_actionTypeStr;
    //! List of values that @m_actionTypeStr can take.
    static constexpr AZStd::string_view s_TypeBoolStr = "Boolean";
    static constexpr AZStd::string_view s_TypeFloatStr = "Float";
    static constexpr AZStd::string_view s_TypeVector2Str = "Vector2";
    static constexpr AZStd::string_view s_TypePoseStr = "Pose";
    static constexpr AZStd::string_view s_TypeVibrationStr = "Vibration";
};
```

In terms of UX, the Interaction Profiles Asset will be exposed to the `Asset Editor` and here is a screenshot
on how it will look like:  
![Editing Interaction Profiles in the Asset Editor](asset_editor_interaction_profiles.png)  
  
There will be an asset builder and content validators to make sure the developer doesn't incur in typograhic mistakes.  
  
There will be a default Interaction Profiles Asset located at:
`Gems/OpenXRVk/Assets/OpenXRVk/system.xrprofiles`.
  
Application developers will be able to replace it into their own Gems or game projects, just like any other asset.  


### The New OpenXRActionSetsAsset
The Action Sets Asset is what most applications will have to define and customize. In this asset, the developer will define the Action Sets, and all the Actions that relate to an applications gameplay.  
  
This asset will have a dependency on a developer defined Interaction Profiles Asset, and thanks to this dependency, the user will be prevented from making action definition errors, because all I/O data will be limited by what the Interaction Profiles Asset provides.  
  
To learn more about Action Sets and Actions, please take a look at this section of the OpenXR spec:  
https://registry.khronos.org/OpenXR/specs/1.0/html/xrspec.html#input  
  
All of that data will be represented with the help of the following C++ classes:  
  
1. The OpenXRActionSetsAsset class:
```cpp
//! This asset defines a list of  OpenXR Action Sets that an application supports
//! regarding inputs and haptics.
class OpenXRActionSetsAsset final
    : public AZ::Data::AssetData
{
public:
    static constexpr char s_assetTypeName[] = "OpenXR Action Sets Asset";
    static constexpr char s_assetExtension[] = "xractions";

    //! By referencing a particular Interaction Profiles asset, the actions
    //! exposed in this Action Sets asset will be limited to the vendor support
    //! profiles listed in the Interaction Profiles asset.
    AZ::Data::Asset<OpenXRInteractionProfilesAsset> m_interactionProfilesAsset;

    //! List of all Action Sets the application will work with.
    AZStd::vector<OpenXRActionSetDescriptor> m_actionSetDescriptors;
};
```
  
2. The OpenXRActionSetDescriptor class:
```cpp
    //! Describes a custom Action Set. All applications
    //! will have custom Action Sets because that's how developers define
    //! the gameplay I/O.
    class OpenXRActionSetDescriptor final
    {
    public:
        //! This name must be unique across all Action Sets listed in an Action Sets Asset.
        //! The content of this string is limited to the characters listed here:
        //! https://registry.khronos.org/OpenXR/specs/1.0/html/xrspec.html#well-formed-path-strings 
        AZStd::string m_name;

        //! User friendly name. 
        AZStd::string m_localizedName; // UTF-8 string.
        
        //! Higher values mean higher priority.
        //! The priority is used by the OpenXR runtime in case several action sets
        //! use identical action paths and the highest priority will win the event.
        uint32_t m_priority = 0;
        
        //! Free form comment about this Action Set.
        AZStd::string m_comment;

        //! List of all actions under this Action Set.
        AZStd::vector<OpenXRActionDescriptor> m_actionDescriptors;
    };
```
  
3. The OpenXRActionDescriptor class:
```cpp
    //! Describes a custom Action I/O that will be queried/driven
    //! by the application gameplay.
    class OpenXRActionDescriptor final
    {
    public:
        //! This name must be unique across all Actions listed in an Action Set.
        //! The content of this string is limited to the characters listed here:
        //! https://registry.khronos.org/OpenXR/specs/1.0/html/xrspec.html#well-formed-path-strings 
        AZStd::string m_name; // Regular char*

        //! User friendly name.
        AZStd::string m_localizedName; // UTF-8 string.

        //! Free form comment about this Action.
        AZStd::string m_comment;

        //! List of I/O action paths that will be bound to this action.
        //! The first action path in this list, determines what type of action paths
        //! can be added to the list. For example:
        //! If the first action path happens to be a boolean, then subsequent action paths
        //! can only be added if they can map to a boolean.
        //! Another important case is if the this is a haptic feedback action (Output), then
        //! subsequent action paths can only be of type haptic feedback actions.
        AZStd::vector<OpenXRActionPathDescriptor> m_actionPathDescriptors;
    };
```
  
4. The OpenXRActionPathDescriptor class:
```cpp
    //! An Action Path Descriptor is nothing more than a tuple of three
    //! strings that identify a unique Input or Haptic control for a particular
    //! vendor equipment. The interesting point is that these strings MUST be limited
    //! to the unique names provided by an OpenXRInteractionProfileAsset.
    class OpenXRActionPathDescriptor final
    {
    public:
        //! Should match an OpenXRInteractionProfileDescriptor::m_name
        AZStd::string m_interactionProfileName;
        
        //! Should match an OpenXRInteractionUserPathDescriptor::m_name
        AZStd::string m_userPathName;

        //! Should match an OpenXRInteractionComponentPathDescriptor::m_name
        AZStd::string m_componentPathName;
    };
```
  
Regarding UX, the OpenXRActionSetsAsset will be exposed to the `Asset Editor`, and the developer will be able to create an Action Sets in a guided fashion that will mitigate content errors. The user will have to select an Interaction Profiles asset (*.xrprofiles), which will limit the hardware support for the new Action Sets.  
  
Here is a snapshot of the UX:  
![Editing Action Sets in the Asset Editor](asset_editor_action_sets.png)  
  
#### REMARK: About Action Sets Immutability
The OpenXR spec states that Action Sets become immutable once they get attached to the current Session.  
If an application needs to change the Action Sets, which is almost never needed, the Session will have to be destroyed and recreated before being able to attach the new Action Sets.

The key implication for O3DE is that, althought the user will be able to create as many Action Sets asset as needed, only one Action Sets Asset will be active, and the Editor or the Application will have to be killed and restarted if a different Action Sets Asset is required.  
  
Because of this, a registry key will be provided so the developer can customize the active Action Sets Asset upon application initialization:
**"/O3DE/Atom/OpenXR/ActionSetsAsset"**. If this key is not defined, the application will default to:  
**"Assets/OpenXRVk/default.xractions"**.  
Here is an example of an application named `AdventureVR` that customizes the Action Sets asset:
- ActionSet Asset Location: \<AdventureVR\>/Assets/adventurevr.xractions
- Registry File Location: \<AdventureVR\>/Registry/adventurevr.setreg, with the following content:  
```json
{
    "O3DE": {
        "Atom": {
            "OpenXR": {
                "ActionSetsAsset": "assets/adventurevr.xractions"
            }
        }
    }
}
```

## The New OpenXRActionsInterface Part 2
Now that we got a tour off the new asset types, it is time to explain the runtime APIs that will be available to the applications developer to read the state of Input Actions or drive signals into Haptic Actions.  

This is the new API:
```cpp
    //! Interface used to query the state of actions and also
    //! to drive the state of haptic feedback actions.
    //! The implementation is encouraged to expose each method
    //! of this interface as global functions in the behavior context.
    //! REMARK: This class must be instantiated AFTER OpenXRReferenceSpacesInterface.
    class IOpenXRActions
    {
    public:
        //! An opaque handle that will provide efficient
        //! access to an Action State data. 
        using ActionHandle = AZ::RHI::Handle<uint16_t, IOpenXRActions>;

        //! Returns a list with the names of all Action Sets that are attached to the
        //! current active Session.
        virtual AZStd::vector<AZStd::string> GetAllActionSets() const = 0;

        //! @returns:
        //!     If successful:
        //!     - true means the action set exists and is active.
        //!     - false means the action set exists and is inactive.
        virtual AZ::Outcome<bool, AZStd::string> GetActionSetState(const AZStd::string& actionSetName) const = 0;

        //! @returns:
        //!     if successful:
        //!     returns the current state of the action set before changing it.
        virtual AZ::Outcome<bool, AZStd::string> SetActionSetState(const AZStd::string& actionSetName, bool activate) = 0;

        //! @returns If an action with the given name exists, returns a successful outcome that contains
        //!     a handle that the caller will further utilize when calling any of the GetActionState*** methods. 
        virtual AZ::Outcome<ActionHandle, AZStd::string> GetActionHandle(const AZStd::string& actionSetName, const AZStd::string& actionName) const = 0;

        //! @returns A successful outcome if the action is representable by the given data type. In particular,
        //!     if an action is of type Vector2, and it is attempted to be read with GetActionStateBoolean() then
        //!     most likely the returned outcome will be an error.
        virtual AZ::Outcome<bool, AZStd::string> GetActionStateBoolean(ActionHandle actionHandle) const = 0;
        virtual AZ::Outcome<float, AZStd::string> GetActionStateFloat(ActionHandle actionHandle) const = 0;
        virtual AZ::Outcome<AZ::Vector2, AZStd::string> GetActionStateVector2(ActionHandle actionHandle) const = 0;

        //! Pose actions are also known as Action Spaces, and their Pose/Transform is always calculated
        //! using a reference/base visualized space. This is a stateful API that can be called at anytime.
        //! See OpenXRReferenceSpacesInterface to get details about Reference Spaces.
        //! By default the base Reference Space is the "View" space, which represents the user's head centroid.
        virtual AZ::Outcome<bool, AZStd::string> SetBaseReferenceSpaceForPoseActions(const AZStd::string& visualizedSpaceName) = 0;
        virtual const AZStd::string& GetBaseReferenceSpaceForPoseActions() const = 0;
        
        //! The returned transform is relative to the base Reference Space.
        virtual AZ::Outcome<AZ::Transform, AZStd::string> GetActionStatePose(ActionHandle actionHandle) const = 0;
        
        //! Same as above, but also queries Linear and Angular velocities.
        virtual AZ::Outcome<PoseWithVelocities, AZStd::string> GetActionStatePoseWithVelocities(ActionHandle actionHandle) const = 0;
        
        //! @param amplitude Will be clamped between 0.0f and 1.0f.
        virtual AZ::Outcome<bool, AZStd::string> ApplyHapticVibrationAction(ActionHandle actionHandle, uint64_t durationNanos, float frequencyHz, float amplitude) = 0;
        virtual AZ::Outcome<bool, AZStd::string> StopHapticVibrationAction(ActionHandle actionHandle) = 0;
    };

    using OpenXRActionsInterface = AZ::Interface<IOpenXRActions>;

    //! In addition to reading the current Pose of a particular Action Space
    //! it is also possible to read both the linear and angular Velocity
    //! of an Action Space. When calling IOpenXRActions::GetActionStatePoseWithVelocities()
    //! This is the returned data.
    struct PoseWithVelocities final
    {
        AZ_TYPE_INFO(PoseWithVelocities, "{AF5B9FF7-FB02-4DA4-89FB-66E605F728E2}");
        static void Reflect(AZ::ReflectContext* context);

        //! The current pose of the Action Space relative to the Reference Space
        //! defined by IOpenXRActions::SetBaseReferenceSpaceForPoseActions().
        //! If IOpenXRActions::SetBaseReferenceSpaceForPoseActions() is never called
        //! then this transform will be relative to the "View" Reference Space.
        AZ::Transform m_pose;

        //! To know more about how to interpret Linear and Angular Velocity see:
        //! https://registry.khronos.org/OpenXR/specs/1.0/man/html/XrSpaceVelocity.html  
        AZ::Vector3 m_linearVelocity;
        AZ::Vector3 m_angularVelocity;
    };
```

The API mentioned above will be implemented by the class `OpenXRVk::ActionsManager`, which will be instantiated and owned by the OpenXRVkSession.

Regarding Behavior Context reflection, which enables scripting with Lua or Python, the API looks like:
```cpp
context.Class<OpenXRActions>("OpenXRActions")
    ->Attribute(AZ::Script::Attributes::Scope, AZ::Script::Attributes::ScopeFlags::Common)
    ->Attribute(AZ::Script::Attributes::Module, "openxr")
    ->Method("GetAllActionSets", &OpenXRActions::GetAllActionSets)
    ->Method("GetActionSetState", &OpenXRActions::GetActionSetState)
    ->Method("SetActionSetState", &OpenXRActions::SetActionSetState)
    ->Method("GetActionHandle", &OpenXRActions::GetActionHandle)
    ->Method("GetActionStateBoolean", &OpenXRActions::GetActionStateBoolean)
    ->Method("GetActionStateFloat", &OpenXRActions::GetActionStateFloat)
    ->Method("GetActionStateVector2", &OpenXRActions::GetActionStateVector2)
    ->Method("SetBaseReferenceSpaceForPoseActions", &OpenXRActions::SetBaseReferenceSpaceForPoseActions)
    ->Method("GetBaseReferenceSpaceForPoseActions", &OpenXRActions::GetBaseReferenceSpaceForPoseActions)
    ->Method("GetActionStatePose", &OpenXRActions::GetActionStatePose)
    ->Method("GetActionStatePoseWithVelocities", &OpenXRActions::GetActionStatePoseWithVelocities)
    ->Method("ApplyHapticVibrationAction", &OpenXRActions::ApplyHapticVibrationAction)
    ->Method("StopHapticVibrationAction", &OpenXRActions::StopHapticVibrationAction)
    ;
```
  
To summarize: During application startup, the **OpenXRVkSession** will instantiate the `OpenXRVk::ActionsManager` which will load the OpenXRActionSetsAsset, either by trying first from the O3DE Registry or the default asset located at:  `o3de-extras/Gems/OpenXRVk/Assets/OpenXRVk/default.xractions`.  
  
After loading the OpenXRActionSetsAsset, the `OpenXRVk::ActionsManager` takes the responsibility of the `OpenXRActionsInterface`, and the application will be able to query the state of all actions.  
  
### Practical API Usage of OpenXRActionsInterface
Let's assume an application has defined one Action Set named **"milkshake"**, and this Action Set contains two Actions:  
1. The first action is an Input Boolean called **"runshake"**, which is mapped to one or more interaction profiles.
2. The second action is an Output Haptic called **"earthquake"**. And the idea is that if the **"runshake"** action boolean state is **true**, the application will drive a haptic signal into the **"earthquake"** Output Action.  
Here is a C++ snippet (No error checking, for simplicity)
```cpp
// The Action Handles can be re-queried every loop or cached during initialization:
auto actionsInterface = OpenXRActionsInterface::Get();
IOpenXRActions::ActionHandle runshakeHandle = actionsInterface->GetActionHandle("milkshake", "runshake");
IOpenXRActions::ActionHandle earthquakeHandle = actionsInterface->GetActionHandle("milkshake", "earthquake");
// Read the state of the "runshake" Input as a boolean.
auto outcome = actionsInterface->GetActionStateBoolean(runshakeHandle);
if (!outcome.IsSuccess())
{
    // Nothing to do, as most likely the controller is inactive.
    return;
}
// The user seems to be holding the controller at the moment.
bool buttonIsPressed = outcome.GetValue();
if (!buttonIsPressed)
{
    return;
}
// Start the earthquake by driving a vibration signal that last 250 milliseconds, with frequency of 20Hz.
uint64_t durationNanos = MillisToNanos(250);
float twentyHz = 20;
float mediumStrength = 0.5;
actionsInterface->ApplyHapticVibrationAction(earthquakeHandle, durationNanos, twentyHz, mediumStrength);
```
  
Here is the same gameplay written in Lua:  
```lua
-- The Action Handles can be re-queried every loop or cached during initialization:
local runshakeHandle = OpenXRActions.GetActionHandle("milkshake", "runshake")
local earthquakeHandle = OpenXRActions.GetActionHandle("milkshake", "earthquake")
-- Read the state of the "runshake" Input as a boolean.
local outcome = OpenXRActions.GetActionStateBoolean(runshakeHandle)
if not outcome:IsSuccess() then 
    -- Nothing to do, as most likely the controller is inactive.
    return
end
-- The user seems to be holding the controller at the moment.
local buttonIsPressed = outcome:GetValue()
if not buttonIsPressed then
    return
end
// Start the earthquake by driving a vibration signal that last 250 milliseconds, with frequency of 20Hz.
local durationNanos = MillisToNanos(250)
local twentyHz = 20
local mediumStrength = 0.5
OpenXRActions.ApplyHapticVibrationAction(earthquakeHandle, durationNanos, twentyHz, mediumStrength)
```

# What are the advantages of the feature?
1. This is a data driven approach to OpenXR Actions management. Application developers may only modify, extend or add new InteractionProfiles and ActionSets Assets, and won't have to modify nor recompile the engine to use the new actions. In particular, if the application developer is simply using Script Canvas or Lua, it won't be necessary to re-compile C++ code at all.  
2. Because this approach leverages the `Asset Editor`, the developer will get a familiar user interface for asset creation oor modification.
3. In addition to using the `Asset Editor`, other user interfaces can be further developed in C++ or Python to work with InteractionProfiles and ActionSets Assets. 
4. The proposed API can be easily extended in the near future for an event driven approach where action states are notified only whenever there are Input changes.
5. This proposal exposes the new interfaces in the Behavior Context, allowing developers to create OpenXR applications with only Lua or Script Canvas.

# What are the disadvantages of the feature?
1. The only disadvantage in this proposal is that the O3DE Input Framework won't be used. This introduces a new cognitive load for O3DE application developers, BUT it will be a very familiar API for those experienced in developing OpenXR applications. 

# Are there any alternatives to this feature?
1. It **may** be possible to retro-fit some OpenXR Actions into the O3DE Input Framework, but most likely will have to be done in a hard-coded C++ fashion which would defeat the extensibility of OpenXR.

# How will users learn this feature?


# Open questions?
