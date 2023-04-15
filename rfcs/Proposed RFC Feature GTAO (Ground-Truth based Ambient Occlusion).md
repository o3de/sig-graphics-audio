# Proposed RFC Feature GTAO (Ground-Truth based Ambient Occlusion)

### Summary:
Ambient occlusion (AO) is an important feature in photo-realistic rendering. However, O3DE only provides the SSAO (Screen Space Ambient Occlusion) feature which can not meet all our needs. To make this gap, we developed  GTAO (Ground-Truth based Ambient Occlusion) feature for O3DE. The GTAO algorithm, which is first proposed by Activision Blizzard in Siggraph 2016 (FYI, one can refer to [this paper](https://www.activision.com/cdn/research/PracticalRealtimeStrategiesTRfinal.pdf) and [this slides](https://blog.selfshadow.com/publications/s2016-shading-course/activision/s2016_pbs_activision_occlusion.pdf) for more details about the algorithm.), can be seen as an enhanced version of SSAO. In a word, the GTAO can achieve better quality with comparable performance cost to the SSAO.  Our GTAO implementation is not a full version of Activision’s version, for example, we didn’t support colored occlusion and temporal denoising for now. Despite that, it works well according to our test. Now, we open an RFC feature request here and intend to commit our GTAO implementation to the O3DE community.

### What is the relevance of this feature?
In photo-realistic/physically-based rendering, a mesh point is shaded by calculating direct and indirect illumination with diffuse or/and glossy BRDFs according to the rendering equation. The diffuse indirect illumination we talk about here is usually known as ambient light. Ambient occlusion can be interpreted as the visibility for the ambient light, which is an essential feature to improve the realism of rendered images. 

In real-time rendering, accurate ambient occlusion is impractical to calculate. For instance, the SSAO is a cheap but coarse approximation to accurate ambient occlusion. Thus it can not always meet the users' needs. To enhance the capability of O3DE in AO, more advanced features are necessary. 

As one of the candidates, though GTAO is also a kind of approximation, it is theoretically closer to the Monte Carlo ground truth in algorithm designing. And compared to the SSAO, the GTAO can achieve better quality with a comparable performance cost. By integrating the GTAO into O3DE, users can make a choice between the SSAO and the GTAO according to their needs.

### Feature design description:
#### Architecture

The GTAO is integrated into Atom Gem and AtomLyIntegration Gem following the current post-process pipeline.  Based on the SSAO component that O3DE already has, a new AO component that contains both SSAO and GTAO is developed. In overview, the architecture of the new AO component is as follows:
![AO-architecture](https://user-images.githubusercontent.com/126558516/221829155-28ab92b2-cd94-4a43-af84-fc2df8f89891.png)

#### Component Panel

The user can specify an `AO type` in the `Ambient Occlusion` panel to enable one of SSAO and GTAO.

**SSAO:**
![image-20230227143410906](https://user-images.githubusercontent.com/126558516/221830173-30ed629d-3820-4612-8f2a-6e9b4a722398.png)


**GTAO:**
![image-20230227143347518](https://user-images.githubusercontent.com/126558516/221830212-aef00a03-1a97-4bc6-89f2-8804cbe326ac.png)


| **Parameters**        | **Description**                                              |
| :-------------------- | :----------------------------------------------------------- |
| `AO type `            | Can be selected in `SSAO`and `GTAO` in this drop box.        |
| `GTAO Strength`       | Control the amount of ambient occlusion.  The range from 0 to 1, the bigger the occlusion. |
| `Quality`             | Quality level of GTAO, 5 levels to adjust. It actually determines the number of sample directions in the GTAO algorithm. The more samples the better quality. |
| `Radius`              | Sample sphere radius, the bigger the more obvious occlusion effect. |
| `Thickness`           | Thickness parameter on object edges,  the bigger the less occlusion on the thin object area. |
| `MaxDepth`            | Maximum depth for computing AO. Depth of objects exceeding this value stop computing AO to prevent distortion at a far distance. |
| `Enable Blur`         | Enable blur to denoise the output AO map.                    |
| `Blur Strength`       | Blur parameter                                               |
| `Blur Edge Threshold` | Blur parameter                                               |
| `Blur Sharpness`      | Blur parameter                                               |

#### Usage

The `AOParentPass` will control his children, enable the corresponding pass and disable the others, according to the `AO Type` from `AOSettings`. For instance, the user adds an `Ambient Occlusion` component to the scene. Then he/her selects`GTAO` from the `AO Type` drop box. This attribute will be first saved in the `AOComponentConfig`, and further passed to `AOSettings` by `AOComponentController`. In runtime, the `AOParentPas` gets `AOSettings` from the `PostProcessingFeatureProcessor`. The `GTAOPass` will be enabled because the `AO Type` matches `GTAO`. Correspondingly, the `SSAOPass` will be disabled automatically.


### Technical design description:
#### Recap: HBAO & GTAO

##### Baseline

Before giving implementation details, we make a brief introduction to the GTAO algorithm. The GTAO is first proposed by Activision Blizzard in Siggraph 2016. GTAO is highly related to HBAO (Horizon-based Ambient Occlusion) in algorithm. HBAO assumes a height field around the shading point, in which case the visibility is continuous on the hemisphere (simplify computing). Then they calculate integration on the hemisphere between two horizon lines, which are determined by tracing the height field (depth buffer) in screen space. But its integration equation doesn't correctly match $k_A$ which is deduced from the render equation and ensured to be physically correct. Thus, HBAO cannot promise the same results as the Monte Carlo based ray traced results in theory. To implement ground-truth based ambient occlusion, the GTAO integrates cosine weight term to their integration equation :
$$V_d=\frac{1}{\pi}\int_\Omega V(\omega_i)(n\cdot\omega_i)\,\mathrm{d}\omega_i=\frac{1}{\pi}\int_o^\pi\int_{-\pi/2}^{\pi/2}V(\theta, \phi)(n\cdot\omega_i)|\sin(\theta)|\mathrm{d}\theta\mathrm{d}\phi$$
where the inner integration
$$\int_{-\pi/2}^{\pi/2}V(\theta, \phi)(n\cdot\omega_i)|\sin(\theta)|\mathrm{d}\theta=IntergrateArc(h_1, h_2, n)$$
can be solved analytically.

![image-20230227192552869](https://user-images.githubusercontent.com/126558516/221832314-dda394e1-0730-4822-8b9a-6bfdf0f43e78.png)


Horizon lines $h_1$ and $h_2$ can be found by searching the height field (depth buffer) in screen space.

![image-20230227194134493](https://user-images.githubusercontent.com/126558516/221832359-d017d408-67f7-4c9e-9f0b-602b6b4f9195.png)

Similar to the HBAO,  the inner integration is solved analytically, and the outer one can be numerically solved by sampling a number of directions around the shading pixel in screen space. 

##### Multi-bounce approximation

To approximate multi-bounce reflection, GTAO models the multi-bounced visibility $V_d^\prime$ as a function of the single-bounced visibility $V_d$ and (neighboring) albedo $\rho$. 
$$V_d^\prime=f(V_d,\rho)$$
There exists an assumption that neighboring albedo can be approximated with the albedo of the current point being shaded. Then they fit $f$ from data of Monte Carlo ray-traced results with a cubic polynomial function under various albedo:
$$V_d^\prime=f(V_d)=((aV_d+b)V_d+c)V_d$$

#### Implementation details

We implement GTAO following the **architecture** described previously in O3DE. `Atom` and `AtomLyIntegration` Gems are involved. The code tree including both SSAO and GTAO is as follows:

**Code tree**

```cpp
// Atom Gem
Atom
|--Feature
	|--Common
		|--Assets
			|--Passes
    			|--AOparent.pass // newly added
    			|--GTAOParent.pass // newly added
    			|--GTAOCompute.pass // newly added
    			|——SsaoParent.pass
			    |--SsaoCompute.pass
    		|--Shaders
    			|--PostProcessing
    				|--SsaoCompute.shader
    				|--SsaoCompute.azsl
    				|--GTAOCompute.shader // newly added
    				|--GTAOCompute.azsl // newly added
    	|--Code
    		|--Include
    			|--Atom
    				|--Feature
    					|--PostProcess
    						|--AmbientOcclusion // renamed
    							|--SsaoConstants.h
    							|--SsaoParams.inl
    							|--GTAOConstants.h // newly added
    							|--GTAOParams.inl // newly added
    							|--AOSettingsInterface.h // modified
			|--Source
    			|--PostProcess
    				|--AmbientOcclusion // renamed
    					|--AOSettings.h // modified
    					|--AOSettings.cpp // modified
    			|——PostProcessing
    				|--SsaoPasses.h
    				|--SsaoPasses.cpp
    				|--GTAOPasses.h // newly added
    				|--GTAOPasses.cpp // newly added
```

```cpp
// AtomLyIntegration Gem
AtomLyIntegration
    |--CommonFeatures
    	|--Code
    		|--Include
    			|--AtomLyIntegration
    				|--CommonFeatures
    					|--PostProcess
    						|--AmbientOcclusion // renamed
    							|--AOBus.h // modified
    							|--AOComponentConfig.h // modified
    		|--Source
    			|--PostProcess
    				|--AmbientOcclustion // renamed
    					|--AOComponent.h // modified
    					|——AOComponent.cpp // modified
    					|——AOEditorComponent.h // modified
    					|——AOEditorComponent.cpp // modified
    					|——AOComponentConfig.cpp // modified
    					|——AOComponentController.h // modified
    					|——AOComponentController.cpp // modified
```

Note that, for brief reasons, only core codes are listed.

The component is designed as same as SSAO. `AOParent.pass`,  `GTAOParent.pass`,`GTAOCompute.pass`, `GTAOCompute.shader`, `GTAOCompute.azsl`, `GTAOConstant.h`, `GTAOParams.inl`, `GTAOPasses.h` and `GTAOPasses.cpp` are newly added file. In addition, `SsaoXXX` files are renamed to `AOXXX` respectively.

**Pass design**

```json
{
    "Type": "JsonSerialization",
    "Version": 1,
    "ClassName": "PassAsset",
    "ClassData": {
        "PassTemplate": {
            "Name": "GTAOParentTemplate",
            "PassClass": "GTAOParentPass",
            "Slots": [
                {
                    "Name": "LinearDepth",
                    "SlotType": "Input",
                    "ScopeAttachmentUsage": "Shader"
                },
                {
                    "Name": "Modulate",
                    "SlotType": "InputOutput",
                    "ScopeAttachmentUsage": "Shader"
                }
            ],
            "PassRequests": [
                // downsample pass
                {
                    "Name": "DepthDownsample",
                    "TemplateName": "DepthDownsampleTemplate",
                    "Enabled": true,
                    "Connections": [
                        {
                            "LocalSlot": "FullResDepth",
                            "AttachmentRef": {
                                "Pass": "Parent",
                                "Attachment": "LinearDepth"
                            }
                        }
                    ]
                },
                // compute pass
                {
                    "Name": "GTAOCompute",
                    "TemplateName": "GTAOComputeTemplate",
                    "Connections": [
                        {
                            "LocalSlot": "LinearDepth",
                            "AttachmentRef": {
                                "Pass": "DepthDownsample",
                                "Attachment": "HalfResDepth"
                            }
                        }
                    ]
                },
                // blur pass
                {
                    "Name": "GTAOBlur",
                    "TemplateName": "FastDepthAwareBlurTemplate",
                    "Enabled": true,
                    "Connections": [
                        {
                            "LocalSlot": "LinearDepth",
                            "AttachmentRef": {
                                "Pass": "DepthDownsample",
                                "Attachment": "HalfResDepth"
                            }
                        },
                        {
                            "LocalSlot": "BlurSource",
                            "AttachmentRef": {
                                "Pass": "GTAOCompute",
                                "Attachment": "Output"
                            }
                        }
                    ]
                },
                // upsample pass
                {
                    "Name": "Upsample",
                    "TemplateName": "DepthUpsampleTemplate",
                    "Enabled": true,
                    "Connections": [
                        {
                            "LocalSlot": "FullResDepth",
                            "AttachmentRef": {
                                "Pass": "Parent",
                                "Attachment": "LinearDepth"
                            }
                        },
                        {
                            "LocalSlot": "HalfResDepth",
                            "AttachmentRef": {
                                "Pass": "DepthDownsample",
                                "Attachment": "HalfResDepth"
                            }
                        },
                        {
                            "LocalSlot": "HalfResSource",
                            "AttachmentRef": {
                                "Pass": "GTAOBlur",
                                "Attachment": "Output"
                            }
                        }
                    ]
                },
                // modulate pass
                {
                    "Name": "ModulateWithGTAO",
                    "TemplateName": "ModulateTextureTemplate",
                    "Enabled": true,
                    "Connections": [
                        {
                            "LocalSlot": "Input",
                            "AttachmentRef": {
                                "Pass": "Upsample",
                                "Attachment": "Output"
                            }
                        },
                        {
                            "LocalSlot": "InputOutput",
                            "AttachmentRef": {
                                "Pass": "Parent",
                                "Attachment": "Modulate"
                            }
                        }
                    ],
                    "PassData": {
                        "$type": "ComputePassData",
                        "ShaderAsset": {
                            "FilePath": "Shaders/PostProcessing/ModulateTexture.shader"
                        },
                        "Make Fullscreen Pass": true
                    }
                }
            ]
        }
    }
}

```

Similar to the SSAO pass, the pass for GTAO is:  `downsample`->`compute`->`bluring`->`upsample`->`modulate`.

**Shader snippets** 

```C
// compute the inner integration given azimuth.
float ComputeInnerIntegral(float2 Angles, float2 ScreenDir, float3 ViewDir, float3 ViewSpaceNormal)
{
    float PI_HALF = 3.1415926535 / 2;
    // Given the angles found in the search plane we need to project the View Space Normal onto the plane defined by the search axis and the View Direction and perform the inner integrate
    float3 PlaneNormal = normalize(cross(float3(ScreenDir.xy,0) ,ViewDir));
    float3 Perp = cross(ViewDir, PlaneNormal);
    float3 ProjNormal = ViewSpaceNormal - PlaneNormal * dot(ViewSpaceNormal, PlaneNormal);

    float LenProjNormal = length(ProjNormal) + 0.000001f;
    float RecipMag = 1.0f / (LenProjNormal);

    float CosAng = dot(ProjNormal, Perp) * RecipMag;    
    float Gamma = acosFast(CosAng) - PI_HALF;                
    float CosGamma = dot(ProjNormal, ViewDir) * RecipMag;
    float SinGamma = CosAng * -2.0f;                    

    // clamp to normal hemisphere 
    Angles.x = Gamma + max(-Angles.x - Gamma, -(PI_HALF) );
    Angles.y = Gamma + min( Angles.y - Gamma,  (PI_HALF) );

    float AO = ( (LenProjNormal) *  0.25 * 
                        ( (Angles.x * SinGamma + CosGamma - cos((2.0 * Angles.x) - Gamma)) +
                            (Angles.y * SinGamma + CosGamma - cos((2.0 * Angles.y) - Gamma)) ));
    return AO;
}
```

```c
// search for horizon lines in screen space
float2 SearchForLargestAngleDual(float2 BaseUV, float2 ScreenDir, float pixelRadius, float InitialOffset, float3 ViewPos, float3 ViewDir,float AttenFactor)
{
    float SceneDepth, LenSq, OOLen, Ang, FallOff;
    float3 V;
    float2 SceneDepths = 0;

    float2 BestAng = float2(-1,-1);
    float Thickness = PassSrg::m_constantsGTAO.m_enabledThincknessAttenFactorMaxDepth.y;

    for(uint i = 0; i < GTAO_NUMTAPS; i++)
    {
        float fi = (float) i;
        float s = (fi + InitialOffset) / (GTAO_NUMTAPS + 1);
        s = s * s;
        float2 sampleOffset = ScreenDir * max(pixelRadius * s, (fi+1));
        float2 UVOffset = GetPixelSize() * sampleOffset;

        UVOffset.y *= -1;
        float4 UV2 = BaseUV.xyxy + float4( UVOffset.xy, -UVOffset.xy);

        // Positive Direction
        SceneDepths.x = PassSrg::m_linearDepth.SampleLevel(PassSrg::PointSampler, UV2.xy, 0).r;
        SceneDepths.y = PassSrg::m_linearDepth.SampleLevel(PassSrg::PointSampler, UV2.zw, 0).r;

        V = ViewSrg::GetViewSpacePosition(UV2.xy, SceneDepths.x) - ViewPos;
        LenSq = dot(V,V);
        OOLen = rsqrt(LenSq + 0.0001);
        Ang = dot(V,ViewDir) * OOLen;

        FallOff = saturate(LenSq * AttenFactor);  
        Ang = lerp(Ang, BestAng.x, FallOff);

        BestAng.x = ( Ang > BestAng.x ) ? Ang : lerp( Ang, BestAng.x, Thickness );  

        // Negative Direction
        V = ViewSrg::GetViewSpacePosition(UV2.zw, SceneDepths.y) - ViewPos;
        LenSq = dot(V,V);
        OOLen = rsqrt(LenSq + 0.0001);
        Ang = dot(V,ViewDir) * OOLen;

        FallOff = saturate(LenSq * AttenFactor);  
        Ang = lerp(Ang, BestAng.y, FallOff);

        BestAng.y = ( Ang > BestAng.y ) ? Ang : lerp( Ang, BestAng.y, Thickness );  
    }

    BestAng.x = acosFast(clamp(BestAng.x, -1.0,  1.0));
    BestAng.y = acosFast(clamp(BestAng.y, -1.0,  1.0));

    return BestAng;
}
```

```c
// multi-bounce approximating 
float MultiBounce(float AO,float3 albedo)
{
    float3 a = 2.0404 * albedo - 0.3324;
    float3 b = -4.7951 * albedo + 0.6417;
    float3 c = 2.7552 * albedo + 0.6903;

    return max(AO, ((AO * a + b) * AO + c) * AO);
}
```

### What are the advantages of the feature?
- As described in previous sections, compared to SSAO and HBAO, GTAO can produce comparable results to that of Monte Carlo ray tracing.

- Approximate multi-bounce visibility.

- GTAO with a medium quality level performs comparably to SSAO

![image-20230228101129927](https://user-images.githubusercontent.com/126558516/221832614-8933949b-99af-41bb-8827-9a9d2edb755c.png)

GTAO time spent 0.41ms in Sponza scene.
	

**Platform**:

- CPU: Intel i9 11900K @ 3.5GHz
- GPU: Nvidia Geforce RTX3090 (24GB)
- Mem: 64GB
- OS: Window 10
- Resolution: $1920\times1080$
- Test Scene: Sponza

**SSAO:**
![SSAO](https://user-images.githubusercontent.com/126558516/222020300-2bc98665-3cb2-4c48-aad2-075f4d1f0d96.PNG)

**GTAO (Low):**
![GTAO-low](https://user-images.githubusercontent.com/126558516/222020331-6dbbb2c4-1641-4e70-90f4-699bd53c6428.PNG)



- Compare to SSAO, no obvious stripes in complex structures, fewer defects at a far distance. Some comparison results are list blow.

**Test Scene1: Spaonza**

**O3DE SSAO:**
![image](https://user-images.githubusercontent.com/126558516/222305990-873f5f83-cfa9-4ccf-95b6-5148560844a9.png)
**O3DE GTAO:**
![image](https://user-images.githubusercontent.com/126558516/222305960-f2123072-85e7-4ddd-908c-a5f536ad9327.png)
**O3DE RTAO:**
![image](https://user-images.githubusercontent.com/126558516/222306019-17af3f4a-b32a-4e9e-b167-a0130cbaa1e8.png)
**UE SSAO:**
![image](https://user-images.githubusercontent.com/126558516/222306149-5d0c43a4-7c50-41d7-9a5c-2a46a99f545e.png)
**UE RTAO:**
![image](https://user-images.githubusercontent.com/126558516/222306167-03becb7f-1813-4b47-981d-ec7cebca0f39.png)



**Test Scene2: Simple Blocks**

**O3DE SSAO:**
![image](https://user-images.githubusercontent.com/126558516/222306296-2e1bf398-7ea6-40e6-87c8-d77ddf43eafa.png)
**O3DE GTAO:**
![image](https://user-images.githubusercontent.com/126558516/222306311-b0213658-e2a6-4311-a325-56c69acd4af0.png)
**O3DE RTAO:**
![image](https://user-images.githubusercontent.com/126558516/222306324-f644c043-c422-4257-b457-296f94c7a57b.png)
**UE SSAO:**
![image](https://user-images.githubusercontent.com/126558516/222306356-f547b1bc-480c-455a-b7ec-fa32f5af5906.png)
**UE RTAO:**
![image](https://user-images.githubusercontent.com/126558516/222306403-759d8e36-b157-4bf8-b64f-17a027559983.png)



### What are the disadvantages of the feature?
- In the current version, there are also some problems with GTAO we implemented that I should point out.

Problem1: Artifacts in some areas.
![image-20230228101254426](https://user-images.githubusercontent.com/126558516/221833167-797850c8-f506-41d4-8cf7-6cfc3e915d17.png)

Problem2: Noise in some areas.
![image-20230228101320705](https://user-images.githubusercontent.com/126558516/221833211-5bf63665-3eea-4f7a-b062-bab488a5941e.png)

- Our Test Conclusions
	- Compare to the SSAO and RTAO of UE, insufficient occlusion in details textures, exists in obvious noises in some local areas.
	- Occlusion is heavy in small gaps, but insufficient when gaps are bigger. The transition between small and big gaps is hard and unnatural.
	- Insufficient occlusion at a far distance, makes the scene lack stereoscopic.

### How will this be implemented or integrated into the O3DE environment?
- GTAO is integrated into the SSAO component that we already have. `Atom` and `AtomLyIntegration` Gems are involved, as described above.

### Are there any alternatives to this feature?
- HBAO is a similar feature to GTAO. Compared to it, GTAO can produce better results and are closer to those of Monte Carlo based ray tracing.
- RTAO (Ray tracing based Ambient Occlusion) can produce more accurate results. However, in real-time rendering, the spp. (sample per pixel) will be largely restricted. Thus further spatial or temporal denoising is also necessary for RTAO. GTAO can produce similar results to RTAO but with a lower performance cost.

### How will users learn this feature?
- Please refer to [this paper](https://www.activision.com/cdn/research/PracticalRealtimeStrategiesTRfinal.pdf) and [this slides](https://blog.selfshadow.com/publications/s2016-shading-course/activision/s2016_pbs_activision_occlusion.pdf) for more details about the GTAO.
- This feature is platform-independent, no document should be changed or recognized.
- Users can use the GTAO in the same way as the SSAO.

### Are there any open questions?
- We didn't implement colored occlusion in the current version, which can be done by fit $f$ with the respective RGB albedo.
- Similar to SSAO, we only adopt spatial denoising now, additional temporal denoising could further decrease noises thus improving effects.
- Some details are incomplete now, for example, lack of thickness heuristic for thin features.
- We didn't implement specular occlusion, which is known as GTSO.