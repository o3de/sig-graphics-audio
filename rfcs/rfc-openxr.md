# OpenXR 



**Summary**
OpenXR is an open royalty-free API standard from Khronos, providing engines with native access to a range of devices. It provides access to following API and we will need to add support for it in a modular fashion within O3DE. The api to add support for is as follows

|XrSpace	|A representation of 3D space	|
|---	|---	|
|XrInstance	|A representation of the OpenXR Runtime	|
|XrSystemId	|A representation of the devices	|
|XrActions	|A representation of user inputs	|
|XrSession	|A representation of the session between the app an the user	|
|XrSwapChain	|A representation of XR swapchain 	|

* * *
**Goals**
This document aims to provide framework around setting up XR related functionality that will need to work with Atom (O3de renderer) in an abstract modular manner. As part of setting up this framework we want to adhere to following goals. 


* Clean XR api accessible to all gems that require XR functionality.  
* Iterative development
* Modularity across multiple XR backends
* Debugging support
* Profiling tools
* Supported device - Quest 2. Eventually any device that supports OpenXr with Vk and Dx12 should work. 

* * *
**Proposed Framework**

In order to gain a high level understanding attached are three diagrams related to

* Gem structure
* UML diagram of interaction for XR and Atom gems
* XR render pipeline 


**Gem structure** 

[Image: XR_Gem.jpg]
XR gem will contain the interface which will be used by other gems to provide access to XR related data as well as functions to update the data as needed. This gem will hold common objects which will be extended by the backend gems like OpenXrVk, OpenXrDx12, etc. This will allow the XR gem to contain any common functionality that exists across all XR backends. XR gem will have no idea which backend is running under hood and to do that it will use the Factory pattern. 

At a high level the design will be setup such that only XR gems will be including Atom gems and not vice versa. The RPI will provide XR specific interface which is then implemented by the XR gems. This interface will be specified within XRSystemInterface and will live within RPI gem. This way Atom will not need to have a dependency on XR tech stack and should still work if the concrete implementation for this interface is not provided. With this design if the binaries related to XR gems are modified it will have no impact on binaries for Atom gems. This header file can be setup as a HEADERONLY module called RPI.XR.Interface within RPI’s cmakefile. The XRSystem class within the XR gem will extend from this interface and provide implementation as described below.

For this document we are only focusing on the vulkan implementation via OpenxrVk gem but later on it will be easy to support for DX12 via OpenxrDx12 gem or even other backends for more devices if needed

**UML diagram for XR related functionality**
[Image: image.png]The color for each class dictate they will be in a specific gem. 

* Blue = XR gem
* Orange = OpenXrVk gem
* Green = RPI gem
* Purple = RHI gem
* Pink = RHI::Vk gem


`XRSystemInterface` - Interface related to XR rendering and it lives within RPI. If in future a different gem (other than XR gem discussed here) wants to provide an implementation for this gem it can be done. 

`XRSystem`  (Singleton) - The class implementing all the interface functionality. More details with code provided further in this document. This class will hold all the XR objects and each XR object will be extended and backed by a class that will live in another gem. For example XR::Instance will be extended by OpenXrVk::Instance. Each XR object will also inherit from `AZStd::intrusive_refcount` in order to attain refcount support. The XR version of the objects will contain all the common code plus any validation code related for that object. Further down I have added code with comments to better explain the purpose of each object and how it will interact with other objects.  

`OpenXrVk::SystemComponent` - This will act as a way to register OpenXrVk as a Factory at runtime and we can then use `XR::FactoryManagerSystemComponent` to pick which factory to use (based on which rhi is picked by Atom). This will ensure we do not pick an incorrect factory for a platfor. For example we do not want to register OpenXrVk on Mac. It will provide a way to create objects that are from OpenXrVk namespace. 

`RHI::XR::XXXDescriptors` - The definitions of these descriptors will live in RHI side but it will populated by XR gems. This will ensure that RHI gems will have no dependency on XR gems. The XR gem will include RHI gem headers and OpenXrVk gem will include RHI::Vk gem headers. That way OpenXrVk gem code can cast a RHI::XXXDescriptor object to RHI::Vulkan::XXXDescriptor and populate it accordingly. 

**XR Render Pipeline (No view instancing)**
[Image: image.png]
Above is an example of what a render pipeline for a frame may look like initially. It is small yet big enough to cover most of the use cases we may encounter for a scene on a VR device. It has support for skinning, shadows, direct lighting, PBR shading, sky box, transparent objects (i.e particles), tonemapping and ui. Since the pipeline will be data driven it should be easy to modify or create new ones for a game’s specific needs. 

`BeginXRViewPass/EndXRViewPass `- These passes are setup so that RHI can call into XR (as part of Execution phase of the framegraph) and synchronize XR swapchain images. Part of this synchronization would contain calls like xrAcquireSwapchainImage, xrWaitSwapchainImage and xrReleaseSwapchainImage for multiple views. 

`CopyToXRSwapChain` - This pass will be used to copy the final image to the XR swapchain image for a given view.

Initially we will duplicate all the CPU Pass work for multiple views as that will be easy to add and will not require core changes to any of the current rendering features. Further down the line we can look into adding MultiView/View instancing support whereby we just have one pipeline and it will be able to do the work for multiple views and write out to multiple render target textures. This will require much bigger changes within RPI and RHI space and hence should be considered as a separate feature to be added later on. 

**XRSystemInterface (pseudocode)**
As part of writing this document it was easier for me to just write pseudo code in order to explain how all the code within XR gems will look like. I have added comments within the pseudocode to provide more context. 

Once XR gem is activated it will initialize the XRSystem and then register iteself with RPI gem . The initialization may involve checking some criteria which may evolve later on. As a start it can just be as simple as checking a command line parameter (-xr=openxr). We should add an enum to the XRSystemInterface header which will allow us to capture the result of all XRSystem calls. The backend XR gems implementing the XR interface will return this ResultCode for most of the calls and allow the XR gem to take action based on the return code. 

```
namespace RPI::XR
{
    enum class ResultCode : uint32_t
    {
        // The operation succeeded.
        Success = 0,
        
        // The operation failed with an unknown error.
        Fail,
       
        // The operation failed because the feature is unimplemented on the particular platform.
        Unimplemented,
       
        // The operation failed due to invalid arguments.
        InvalidArgument,
    }
}
```




**XR::XRSystemInterface pseudo code -** Here is a possible starting interface used by the XR gem. This is mostly based on OpenXR functionality. 


```
namespace RPI::XR
{
    class XRSystemInterface
    {
        static XRSystemInterface* Get();

        XRSystemInterface() = default;
        virtual ~XRSystemInterface() = default;
        
        // Creates the XR::Instance which is responsible for managing 
        // XrInstance (amongst other things) for OpenXR backend
        // Also initializes the XR::Device
        virtual ResultCode InitializeSystem() = 0;
        
        // Create a Session and other basic session-level initialization.
        virtual XR::ResultCode InitializeSession() = 0;
        
        // Start of the frame related XR work
        virtual void BeginFrame() = 0;
        
        // End of the frame related XR work
        virtual void EndFrame() = 0;
        
        // Start of the XR view related work
        virtual void BeginXRView() = 0;
        
        // End of the XR view related work
        virtual void EndXRView() = 0;

        // Manage session lifecycle to track if RenderFrame should be called.
        virtual bool IsSessionRunning() const = 0;
    
        // Create a Swapchain which will responsible for managing
        // multiple XR swapchains and multiple swapchain images within it
        virtual void CreateSwapchain() = 0;
        
        // This will alow XR gem to provde device related data to RHI
        RHI::XR::DeviceDescriptor* GetDeviceDescriptor() = 0;
        
        // Provide access to instance specific data to RHI
        RHI::XR::InstanceDescriptor* GetInstanceDescriptor() = 0;
        
        // Provide Swapchain specific data to RHI
        RHI::XR::SwapChainImageDescriptor* GetSwapChainImageDescriptor(uint_32 swapchainIndex) = 0;
        
        // Provide access to Graphics Binding specific data that RHI can populate
`        RHI::XR::GraphicsBindingDescriptor* GetGraphicsBindingDescriptor() = 0;`
    }
}
```

**XR::XRSystem pseudo code -** This will be implementing the Interface described above. It is a singleton so it can be accessed by other gems for anything related to XR.

```

    // XRSystem will be the singleton that will act as a frontend to everything XR
    // related. It will have API to allow creation of XR specific objects and 
    // provide access to it's data. This class will also inherit from  SystemTickBus
    // in order to tick xr input
    class XRSystem: public XRSystemInterface
            , public AZ::SystemTickBus::Handler
    {
    public:
    
        virtual ~XRInterface() = default;
   
        // Accessor functions for RHI objects that are populated by backend XR gems
        // This will alow XR gem to provide device related data to RHI
        RHI::XR::DeviceDescriptor* GetDeviceDescriptor() override
        {
            return m_deviceDesc.get();
        }
        
        // Provide access to instance specific data to RHI
        RHI::XR::InstanceDescriptor* GetInstanceDescriptor() override
        {
            return m_instanceDesc.get();
        }
        
        // Provide Swapchain specific data to RHI
        RHI::XR::SwapChainImageDescriptor* GetSwapChainImageDescriptor(uint_32 swapchainIndex) override
        {
            return m_swapchainDesc.get();
        }
        
        // Provide access to Graphics Binding specific data that RHI can populate
        RHI::XR::GraphicsBindingDescriptor* GetGraphicsBindingDescriptor() override
        {
            return m_gbDesc.get();
        }
        
        // Access supported Layers and extension names
        const AZStd::vector<AZStd::string>& GetXRLayerNames(){..}
        const AZStd::vector<AZStd::string>& GetXRExtensionNames(){..}
         
        // Create XR instance object and initialize it            
        ResultCode InitInstance() 
        {
            m_instance = Factory::Get()->CreateXRInstance();

            if(m_instance)
            {
                return m_instance->InitInstanceInternal();
            }
            return ResultCode::Fail;
        }
        
        // Create XR device object and initialize it 
        ResultCode InitDevice() 
        {
            m_device = Factory::Get()->CreateXRDevice();

            //Get a list of XR compatible devices
            AZStd::vector<AZStd::intrusive_ptr<PhysicalDevice>> physicalDeviceList =
                  Factory::Get()->EnumerateDeviceList();
            
            //Code to pick the correct device. 
            //For now we can just pick the first device in the list
            
            if(m_device)
            {
                return m_device->InitDeviceInternal();
            }
            return ResultCode::Fail;
        }
        
        // Initialize XR instance and device 
        ResultCode InitializeSystem() override
        {
            ResultCode instResult = InitInstance();
            if(instResult != ResultCode::Success)
            {
               AZ_Assert(false, "XR Instance creation failed");
               return instResult;
            }
            
            ResultCode deviceResult = InitDevice();
            if(deviceResult != ResultCode::Success)
            {
               AZ_Assert(false, "XR device creation failed");
               return deviceResult;
            }
            return ResultCode::Success;
        }
        
        // Initialize a XR session 
        ResultCode InitializeSession(AZStd::intrusive_ptr<GraphicsBinding> graphicsBinding) override
        {
            m_session = Factory::Get()->CreateXRSession();
            
            if(m_session)
            {
                Session::SessionDescriptor sessionDesc;
                m_gbDesc = Factory::Get()->CreateGraphicsBindingDescriptor();
                sessionDesc.m_graphicsBinding = RPISystem::Get()->PopulateGrapicsBinding(m_gbDesc);
                ResultCode sessionResult = m_session->Init(sessionDesc);
                AZ_Assert(sessionResult==ResultCode::Success, "Session init failed");
                
                m_xrInput = Factory::Get()->CreateXRInput();
                return m_xrInput->InitializeActions(); 
            } 
            return ResultCode::Fail;  
        }
   
        // Manage session lifecycle to track if RenderFrame should be called.
        bool IsSessionRunning() const override
        {
            return m_session->IsSessionRunning();
        }
   
        // Create a Swapchain which will responsible for managing
        // multiple XR swapchains and multiple swapchain images within it
        ResultCode CreateSwapchain() override
        {
            m_swapChain = Factory::Get()->CreateSwapchain();
            
            if(m_swapChain)
            {
                ResultCode swapchainCreationResult = m_swapChain->Init(sessionDesc);
                AZ_Assert(sessionResult==ResultCode::Success, "Swapchain init failed");
                return swapchainCreationResult;
            }
            return ResultCode::Fail; 
        }
        
        // Indicate start of a frame
        void BeginFrame() override
        {
            ..
        }
        
        // Indicate end of a frame
        virtual void EndFrame() override
        {
            ..
        }
        
        // Indicate start of a XR view to help with synchronizing XR swapchain
        virtual void BeginXRView() override
        {
            ..
        }
        
        // Indicate end of a XR view to help with synchronizing XR swapchain
        virtual void EndXRView() override
        {
            ..
        }
        
    private:
        ResultCode InitInstance();
        
        //System Tick to poll input data
        void OnSystemTick() override
        {
            m_input->PollEvents();
            if (exitRenderLoop) 
            {
                break;
            }

            if (IsSessionRunning()) 
            {
                m_input->PollActions();
            }
        }
        
        AZStd::intrusive_ptr<Instance> m_instance;
        AZStd::intrusive_ptr<Device> m_device;
        AZStd::intrusive_ptr<Session> m_session;
        AZStd::intrusive_ptr<Input> m_input;
        AZStd::intrusive_ptr<Input> m_swapChain;
        bool m_requestRestart = false;
        bool m_exitRenderLoop = false;
        AZStd::intrusive_ptr<RHI::XR::DeviceDescriptor> m_deviceDesc;
        AZStd::intrusive_ptr<RHI::XR::InstanceDescriptor> m_instanceDesc;
        AZStd::intrusive_ptr<RHI::XR::SwapChainDescriptor> m_swapchainDesc;
        AZStd::intrusive_ptr<RHI::XR::GraphicsBindingDescriptor> m_gbDesc;
    };
    
    XRSystemInterface* XRSystemInterface::Get()
    {
         return Interface<XRSystemInterface>::Get();
    }
}
```

Since XRSystem inherits from AZ::SystemTickBus::Handler it will be able to use OnSystemTick to poll input as shown in the pseudocode above. 

**XR::Factory pseudo code**
As explained above we will have XR objects which are implemented by Openxr gems like OpenXrVk. In order to help get this working we can setup a XR factory that is extended by SystemComponent within OpenXrVk

The factory will act as an interface for creating XR objects. It will be a singleton and be accessed by a call to XR::Factory::Get(). The OpenXrVk::SystemComponent will be responsible creating OpenXrVk objects by calling the static function Create that resides in all the OpenXrVk objects. 

```
namespace XR
{
    //! Interface responsible for creating all the XR objects which are 
    //! internally backed by concrete objects
    class Factory
    {
    public:
        Factory();
        virtual ~Factory() = default;

         AZ_DISABLE_COPY_MOVE(Factory);
      
         //! Registers the global factory instance.
         static void Register(Factory* instance);

         //! Unregisters the global factory instance.
         static void Unregister(Factory* instance);

         //! Access the global factory instance.
         static Factory& Get();
         
         //Create XR::Instance object
         virtual AZStd::intrusive_ptr<Instance> CreateXRInstance() = 0;
         
         //Create XR::Device object
         virtual AZStd::intrusive_ptr<Device> CreateXRDevice() = 0;
         
         //Return a list of XR::PhysicalDevice 
         AZStd::vector<AZStd::intrusive_ptr<PhysicalDevice>> EnumerateDeviceList() = 0
         
         //Create XR::Session object
         virtual AZStd::intrusive_ptr<Session> CreateXRSession() = 0;
         
         //Create XR::Input object
         virtual AZStd::intrusive_ptr<Input> CreateXRInput() = 0;
         
         //Create XR::SwapChain object
         virtual AZStd::intrusive_ptr<SwapChain> CreateSwapchain() = 0; 
         
         //Create XR::ViewSwapChain object
         virtual AZStd::intrusive_ptr<ViewSwapChain> CreateViewSwapchain() = 0;  
         
         //Create RHI::XR::GraphicsBindingDescriptor that will contain
         //renderer information needed to start a session
         virtual AZStd::intrusive_ptr<RHI::XR::GraphicsBindingDescriptor> CreateGraphicsBindingDescriptor() = 0;  
    }
}
```

Below are all the XR objects which will define higher level common functionality. XR objects have no idea which gem will be implementing the XR backend. These objects should hopefully encapsulate most of the XR specific data and the API around these objects would be subject to change in future. Most of these objects are self explanatory and does not require further clarification around why they are needed. For deeper insight look at the code for OpenXrVk version of these objects. 



```
namespace XR
{
    // XR::Instance class. It will be responsible for collecting all the data like 
    // form factor, physical device etc that will be needed to initialize an instance
    class Instance
    {
        class InstanceDescriptor
        {
            //Form Factor enum
            //XR::PhysicalDevice* physicalDevice
        };
        
        virtual XR::ResultCode InitInstanceInternal() = 0; 
    }
    
    // This class will be responsible for iterating over all the compatible physical
    // devices and picking one that will be used for the app
    class PhysicalDevice
    {
        struct PhysicalDeviceDescriptor
        {
            AZStd::string m_description;
            uint32_t m_deviceId = 0;
            //Other data related to device
        };
        PhysicalDeviceDescriptor m_descriptor;
    }
    
    // This class will be responsible for creating XR::Device instance 
    // which will then be passed to the renderer to be used as needed. 
    class Device
    {
        struct DeviceDescriptor
        {
            //XR::PhysicalDevice* physicalDevice
        };
        
        virtual XR::ResultCode InitDeviceInternal(DeviceDescriptor descriptor) = 0;
        DeviceDescriptor m_descriptor;
    }
    
    // This class will be responsible for creating XR::Session and 
    // all the code around managing the session state
    class Session
    {
    public:    
        struct SessionDescriptor
        {
            // Graphics Binding will cntain renderer related data to start a xr session
            GraphicsBinding* m_graphicsBinding
        };
        
        void Init(SessionDescriptor sessionDesc)
        {
            return InitSessionInternal(sessionDesc);
        }
        
        bool IsSessionRunning() const 
        {
            return m_sessionRunning;
        }
        
        bool IsSessionFocused() const = 0;
        virtual ResultCode InitSessionInternal(SessionDescriptor descriptor) = 0;
    private:
        
        SessionDescriptor m_descriptor;
        bool m_sessionRunning = false;
    }
   
    // This class will be responsible for creating XR::Input
    // which manage event queue or poll actions
    Class Input
    {
    public:    
        struct InputDescriptor
        {
            Session* m_session;
        };
    
         ResultCode Init(InputDescriptor descriptor)
         {
            m_session = descriptor.m_session;
            return InitInternal();
         }
         
         virtual void PollActions() = 0;         
         virtual ResultCode InitInternal() = 0;
    private:
         AZStd::intrusive_ptr<Session> m_session;
    }
    
    // This class will be responsible for creating multiple XR::SwapChain::ViewSwapchains
    // (one per view). Each XR::SwapChain::ViewSwapchain will then be responsible 
    // for manging and synchronizing multiple swapchain images 
    class SwapChain
    {
    public:
    
         class Image
         {
            struct ImageDescriptor
            {
                uint16_t m_width;
                uint16_t m_height;
                uint16_t m_arraySize;
            }
            ImageDescriptor m_descriptor;
         }
         class ViewSwapChain
         {
              //! All the images associated with this ViewSwapChain
              AZStd::vector<AZStd::intrusive_ptr<Image>> m_images;
              
              //! The current image index.
              uint32_t m_currentImageIndex = 0;
         }    
         
         //! Returns the view swapchain related to the index
         ViewSwapChain* GetViewSwapChain(const uint32_t swapchainIndex) const;

         //! Returns the image associated with the provided image 
         //! index and view swapchain index
         Image* GetImage(uint32_t imageIndex, uint32_t swapchainIndex) const;
         
         ResultCode Init()
         {
            return InitInternal();
         }
         
         virtual ResultCode InitInternal() = 0;
    private:
        
          AZStd::vector<AZStd::intrusive_ptr<ViewSwapChain>> m_viewSwapchains;    
    }
    
    // This class will be responsible for managing XR Space
    class Space
    {
    public:
        virtual ResultCode InitInternal() = 0;
    }
    
}
```


**OpenXRVk::XX pseudo code -** This code will be part of OpenXrVk gem. Below is an example of one possible backend implementation for XR functionality

```
namespace OpenXR
{
    // Class that will help manage XrInstance
    class Instance : public XR::Instance
    {
    
    public:
        static AZStd::intrusive_ptr<Instance> Create();
        XR::ResultCode InitInstanceInternal() override
        {
            ....
            // xrCreateInstance(m_xrInstance);
            // xrGetSystem(m_systemId)
            // vkCreateInstance(m_instance)
            ...
        }
        
        
    private:
        XrInstance m_xrInstance{ XR_NULL_HANDLE };
        AZStd::vector<XrApiLayerProperties> m_layers;
        AZStd::vector<XrExtensionProperties> m_extensions;
        XrFormFactor m_formFactor{ XR_FORM_FACTOR_HEAD_MOUNTED_DISPLAY };
        XrSystemId m_systemId{ XR_NULL_SYSTEM_ID };
        VkInstance m_instance = VK_NULL_HANDLE;
    }
    
    
    // Class that will help manage VkPhysicalDevice
    class PhysicalDevice: public XR::PhysicalDevice
    {
    
    public:
        static AZStd::intrusive_ptr<PhysicalDevice> Create();
        XR::ResultCode InitInstanceInternal() override
        {
            ....
            //xrGetVulkanGraphicsDeviceKHR
            ...
        }
    private:
        VkPhysicalDevice m_physicalDevice;  
    }
    
    
    // Class that will help manage VkDevice
    class Device: public XR::Device
    {
    
    public:
        static AZStd::intrusive_ptr<Device> Create();
        XR::ResultCode InitDeviceInternal(XR::PhysicalDevice& physicalDevice) override
        {
            ....
            // Create Vulkan Device
            
            ...
        }
    private:
        VkDevice m_nativeDevice;
        
    }
   
    
    // Class that will help manage XrSession
    class Session: public XR::Session
    {   
     public: 
        static AZStd::intrusive_ptr<Session> Create();  
        XR::ResultCode InitSessionInternal(SessionDescriptor descriptor) override
        {
            ..
            //AZStd::intrusive_ptr<GraphicsBinding> gBinding = static_cast<GraphicsBinding>(descriptor.m_graphicsBinding);
            //xrCreateSession(..m_session,gBinding,..)
            ..
        }
        void LogReferenceSpaces()
        {
            //..xrEnumerateReferenceSpaces/
        }
        void HandleSessionStateChangedEvent(const XrEventDataSessionStateChanged& stateChangedEvent, bool* exitRenderLoop, bool* requestRestart)
        {
            //Handle Session state changes
        }
        
        XrSession GetSession()
        {
            return m_session;
        }
        
        bool IsSessionFocused() const override
        {
            return m_sessionState == XR_SESSION_STATE_FOCUSED;
        }
        
        ResultCode InitInternal()
        {
            //Init specific code
        }
     private:
        
        XrSession m_session{ XR_NULL_HANDLE };
        // Application's current lifecycle state according to the runtime
        XrSessionState m_sessionState{XR_SESSION_STATE_UNKNOWN};
        XrFrameState m_frameState{ XR_TYPE_FRAME_STATE };
    }
    
    // Class that will help manage XrSpace's'
    class Space: public XR::Space
    {
    public:
        static AZStd::intrusive_ptr<Space> Create(); 
    
        XrSpaceLocation GetSpace(XrSpace space)
        {
          .. 
          //xrLocateSpace
          ..
        }
        ResultCode InitInternal()
        {
            //Init specific code
        }
    private:
        
        
        XrSpace m_baseSpace{ XR_NULL_HANDLE };
    }
    
    // Class that will help manage XrSwapchain
    class SwapChain: public XR::SwapChain
    {
    public:
        static AZStd::intrusive_ptr<SwapChain> Create();
        
        class Image : public XR::SwapChain::Image
        {
        public:
            static AZStd::intrusive_ptr<Image> Create();
        private:
            VkImage m_image;
            XrSwapchainImageBaseHeader* m_swapChainImageHeader;    
        }
        
        class ViewSwapChain : public XR::SwapChain::Image
        {
        public:
            static AZStd::intrusive_ptr<ViewSwapChain> Create();
            ResultCode Init(XrSwapchain handle, uint32_t width, uint32_t height);
        private:
            XrSwapchain m_handle;
            int32_t m_width;
            int32_t m_height;      
        }
        
        ResultCode InitInternal() override
        {
             ..
             // xrEnumerateViewConfigurationViews
             ..
             for(int i = 0 ; i < views ; i++
             {
                xrCreateSwapchain
                AZStd::intrusive_ptr<ViewSwapChain> vSwapChain = Factory::Get()->ViewSwapChain();
                
                if(vSwapChain)
                {
                    xrCreateSwapchain(.., xrSwapchainHandle, .)
                    vSwapChain->Init(xrSwapchainHandle,..);
                    m_viewSwapchains.push_back(vSwapChain);
                }
             }
             ..
        }
    private:
        AZStd::vector<XrViewConfigurationView> m_configViews;
        AZStd::vector<XrView> m_views;
        int64_t m_colorSwapchainFormat{ -1 };
    }
    
    // Class that will help manage XrActionSet/XrAction
    Class Input: public XR::Input
    {
    public:   
        static AZStd::intrusive_ptr<Input> Create();
        
        ResultCode Init() override
        {
            InitializeActions();
        }
        
        void InitializeActions() override
        {
            ..
            // Code to populate m_input
            // xrCreateActionSet
            // xrCreateAction
            // xrCreateActionSpace
            // xrAttachSessionActionSets
        }
        
        void PollActions() override
        {
            ..
            // xrSyncActions
            ..
            
        }
        
        void PollEvents() override
        {
            ..
            // .. m_session->HandleSessionStateChangedEvent
            ..
        }
   
       
    private:
       struct InputState
        {
            XrActionSet actionSet{ XR_NULL_HANDLE };
            XrAction grabAction{ XR_NULL_HANDLE };
            XrAction poseAction{ XR_NULL_HANDLE };
            XrAction vibrateAction{ XR_NULL_HANDLE };
            XrAction quitAction{ XR_NULL_HANDLE };
           AZStd::array<XrPath, Side::COUNT> handSubactionPath;
            AZStd::array<XrSpace, Side::COUNT> handSpace;
            AZStd::array<float, Side::COUNT> handScale = { { 1.0f, 1.0f } };
            AZStd::array<XrBool32, Side::COUNT> handActive;
        };
        InputState m_input;
    }
}
```

We will also need to add support for proper validation logging.  This will allow us to better debug error codes across XR api. For example a function like this will be needed to log errors in case of a non successful XR error code. 

```

bool IsSuccess(XrResult result)
{
    if (result != XR_SUCCESS)
    {
        AZ_Error("XR", false, "ERROR: XR API method failed: %s", GetResultString(result));
        return false;
    }
    return true;
}
```


**RPI** 
Following stuff will need to be done at RPI level

* RPI core gem will be responsible for initializing the XRSystem. Once the device is created it will also work with the RHI backend to populate GraphicsBindingDescriptor which is used to create a XR session.  The pseudocode for this work looks like this

```
class RPISystem
{
    void RPISystem::RegisterXRSystem(XRSystemInterface* xrSystem);
    {
        m_xrSystem = xrSystem;
    }

    void RPISystem::Initialize()
    {
        ..
        ..
        //RHI::Vulkan::XR::DeviceDescriptor* deviceDesc = m_xrSystem->GetDeviceDescriptor();
        //pass the devicedscriptor to RHI for device creation
        ..
        
        //Session creation
        if(m_xrSystem)
        {
            RHI::XR::GraphicsBindingDescriptor* gb_desc = m_xrSystem->`GetGraphicsBindingDescriptor`();
            m_rhiSystem->PopulateGrapicsBinding(gb_desc);
            m_xrSystem->InitializeSession(graphicsBinding);
        }
        ..
        
    }
    
    AZStd::intrusive_ptr<XRSystem> m_xrSystem;
}

```



* We will also need to setup a special XRSwapchain pass that is able to call BeginXRView/EndXRView during framegraph execution phase. 
* A separate XR pipeline will need to be created that will be duplicated for left and right eye
* ViewSRG and View c++ code will need to be modified to support multiple views. FOV/Orientation data will need to be queried via XRSystem and passed to the view SRG. We could add View data to the bindless heap and index into it from the shader. That way view indices can just be added as root constants. 
* Swapchain specific RHI image views will need to be created with the Image object extracted from XR.


**RHI** 


* FrameScheduler - Modify RHISystem::FrameUpdate to check if FrameScheduler::BeginFrame succeeded. If not skip calling compile or execute on the FrameScheduler (i.e framegraph).
* Add API to accept a Device from XR gem
* Add API to generate  a RHI::ImageView from XR::Image passed in from RPI.
    * We could modify RHI::Image api to skip creation of native object and have it accept the one from XR::Image.  
    * RHI:;ImageView would then just be created as normal. Any issues related to this will be caught in Step 1 of dev plan below. 

**Debugging**


* PC support - In order to facilitate debugging while working on XR we should still maintain a window and swapchain on PC. The idea is that when someone removes the VR device the app should automatically switch to using the PC app. It can do that by switching to PC swapchain instead of XR swapchain. The app can check every frame if the XR session is still active and if it is we will use the XR swapchains but as soon as the device is removed from the head RPI should be able to detect this and switch to PC swapchain making it seameless. The PC swapchain support could be added behind a macro so we dont ship this code as part of a Ocullus app. 
* DX12 vs Vulkan - We should consider adding XR support to both DX12 and Vulkan instead of just Vulkan. Given that DX12 is more used, it is usually more stable and and more performant and hence it will provide stable builds for XR dev work day to day. Having said that it will add to capacity which is a concern at the moment. Hence we can consider just dropping dx12 initially
* Having PC builds across DX12, Vulkan and Android will also help isolate issues which are rhi backend specific vs platform specific, etc. Definitely for dev purposes PC builds will be the most used in this case. 
* Oculus provides Renderdoc support so we should be able to do gpu captures natively on device to help debug gpu specific rendering artifacts.
* Add support for debug text specific to VR rendering. 


**Profiling** 

* CPU perf - Usually cpu perf is not that hard to analysis. On PC we can use Pix but on Android we can use O3de’s in-built cpu profiler. It should already have support to export out serialized cpu profiling data that we can load on PC later for analysis. 
* GPU - There is a mention of a performance profiler here but I have doubts if it will be able to provide detailed data based on a gpu capture - https://developer.oculus.com/documentation/native/pc/dg-performance-profiler/
* GPU perf analysis in general is a very tricky thing on android (due to tiled GPUS) but hopefully native device tools would serve best for perf analysis.


**Performance**

**1. Multi View / View instancing support**
As part of the perf work the first big change we should do is to implement “Multi View” support within Atom. This will essentially allow us to not duplicate CPU work related to all the passes for left and the right eye. We should be able to add support that will allow the passes to write out to texture arrays. This will be a fairly significant change to Atom. 
DX12 - https://microsoft.github.io/DirectX-Specs/d3d/ViewInstancing.html
VK - https://www.khronos.org/registry/vulkan/specs/1.3-extensions/man/html/VK_KHR_multiview.html
Metal - https://developer.apple.com/documentation/metal/render_passes/rendering_to_multiple_viewports_in_a_draw_command?language=objc

**XR Render Pipeline (MultiView support)**
[Image: image.png]High level RHI changes

* PSO api changes support view instance declaration
* Commandlist api changes to support view instance mask
* Shader changes to now support SV_ViewID . This will impact shaders for any pass that is processing view specific data
    *  Azslc support for SV_ViewID
* Investigate tier approach to view instancing. 
* This will require deeper prototyping in order to better understand all the requirements. 


High level RPI

* A new pipeline as shown above which will be much more performant.
* This will require changes to all the passes and how the render target for each pass that deals with view data is setup.



**2. VRS (Foveated rendering) support** - This will require a separate doc on VRS abstraction

**3. Mobile specific optimizations (via O3de scalability work)** - We can use it to disable features within the XR pipeline or lower configuration settings per feature that can help yield better perf. For example

* Lower Rendering scale
* Half Float support in the forward pass. Huawei was asking about this as well.
* Faster Shadow filtering  
* Reduced Shadow sample count
* Only allow directional shadows
* Baked shadows?
* Baked lighting (Direct)?
* Disable Area lights
* XR specific pipeline optimizations **-** This can be a variant off the low end render pipeline. Deeper analysis of perf using native gpu tools.
* Experiment with cheaper BRDF (optimized for mobile) - (Ref -> **siggraph2015  Optimizing PBR**) 
* Filament also provides optimized BRDF when considering half floats - https://google.github.io/filament/Filament.md.html#materialsystem/specularbrdf
* Investigate replacing Roughness angle LUT texture (for env IBL) with the analytical function for DFG approximation mentioned here - https://knarkowicz.wordpress.com/2014/12/27/analytical-dfg-term-for-ibl/


**Recommended Development Plan in iterative steps**

1. RHI OpenXR sample with Vk renderer (using Oculus link) 
    1. Add support for XR and OpenXrVk gems. 
    2. RHI OpenXR sample within AtomSampleViewer 
        1. XR initialization support
        2. Setup a simple XR pipeline (in c++) that renders a cube per xr Space 
        3. Extract view data per view and create a framegraph that will execute a simple XR pipeline twice. It could look like BeginXRView(Left eye)→RenderCube→EndXRView(Left eye)->BeginXRView(Right eye)→RenderCube→EndXRView(Right eye)
        4. Extract Positional data per space from XR gem and feed that to the RenderCube pass
2. RHI sample support on Android 
    1. Add support for the sample to run on Android so we do not need to use Oculus link.
3. Dx12 support (based on capacity). Drop this if needed  
4. RPI support on PC 
    1. Add a the proper XR pipeline using the pass data (described in the doc above)
    2. Setup usage of this pipeline when XR is enabled
    3. Setup multi view support within shaders + code
5. RPI support on Android
6. Multi View support on RHI sample (VK only at first) 
7. Multi View support integrated within RPI 
8. Profile + optimizations as described und the perf section 

