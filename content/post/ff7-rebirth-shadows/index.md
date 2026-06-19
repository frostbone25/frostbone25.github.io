---
title: Final Fantasy 7 Rebirth Contact Shadows Mod
description: Sharing my personal notes and experimentation with modding contact shadows into FF7 Rebirth gameplay.
slug: ff7-rebirth-contact-shadows
date: 2026-06-19 00:00:00+0000
image: content/contact-shadows-result.jpg
categories:
    - Graphics
tags:
    - Modding
    - Reverse Engineering
    - DirectX
    - DX12
    - Shadows
weight: 1       # You can add weight to some posts to override the default sorting (date descending)
---

*By: David Matos*

Sharing my personal notes and experimentation with modding contact shadows into one of my most favorite games in recent memory, Final Fantasy 7 Rebirth, into actual run-time gameplay *[(at the end)](#video-preview)*

*NOTE: This will be updated as time goes on with any new or incorrect information.*

![contact-shadows-result](content/contact-shadows-result.jpg)
[*Final Fantasy 7 Rebirth, Square Enix*](https://www.square-enix.com/ffvii/en-us/games/rebirth/)

#### Table of Contents
- [Preface](#preface)
    - [Context](#context)
    - [Lighting Solutions - Ray Tracing?](#lighting-solutions---ray-tracing)
    - [Lighting Solutions - Increase Shadow Map Resolution?](#lighting-solutions---increase-shadow-map-resolution)
    - [Lighting Solutions - Increase Shadow Map Cascades?](#lighting-solutions---increase-shadow-map-cascades)
    - [Lighting Solutions - Shadow Map Caching?](#lighting-solutions---shadow-map-caching)
    - [Lighting Solutions - Contact Shadows](#lighting-solutions---contact-shadows)
- [Brief RenderDoc Breakdown](#brief-renderdoc-breakdown)
- [Implementation](#implementation)
- [Micro Shadows](#micro-shadows)
- [Contact Shadows](#contact-shadows)
    - [Limitations](#limitations)
    - [Getting Good Results](#getting-good-results)
    - [Naive Contact Shadows (World Space)](#naive-contact-shadows-world-space)
    - [Optimization: Reduce Sample Counts](#optimization-reduce-sample-counts)
    - [Quality Boost: Introduce Noise (Blue Noise)](#quality-boost-introduce-noise-blue-noise)
    - [Optimization: Switching to Interleaved Gradient Noise (IGN)](#optimization-switching-to-interleaved-gradient-noise-ign)
    - [Optimization: Clip Space](#optimization-clip-space)
    - [Optimization: Early Sky Out](#optimization-early-sky-out)
    - [Optimization: Further Sample Reduction](#optimization-further-sample-reduction)
    - [Bonus: Dual Depth Sampling](#bonus-dual-depth-sampling)
    - [Bonus: Depth Point Sampling](#bonus-depth-point-sampling)
    - [Bonus: Checkerboard Rendering Optimization](#bonus-checkerboard-rendering-optimization)
- [Final Results](#final-results)
- [Video Preview](#video-preview)
- [Timings on 1920x1080 and 3840x2160](#timings-on-1920x1080-and-3840x2160)
- [References / Sources](#references--sources)

## Preface

#### Context

After waiting a little over a year to play a sequel to one of my favorite franchises on PC in 2024, I finally got to play the game. I was taken a back by the sheer amount of content, and variety that was in the game. The story was fantastic, the gameplay was stellar, full of depth, and it was a genuinely awesome experience that was well worth the wait...

But... the back of my mind, there is a little graphics gremlin that was nagging me during some segments of the game. I couldn't hold him at bay so unfortunately I let him speak... and his words led to some realizations about the game's rendering.

Rebirth unfortunately has some short-comings in regards to it's lighting quality during gameplay. Now during cutscenes or other more curated segments the game looks great, asset quality and art direction is fantastic. However in some areas during gameplay, it's very clear that there are cutbacks that were done to the lighting for performance sake. Especially when you consider how the previous game in the series, Final Fantasy 7 Remake, looked in similar areas.

<p float="left">
    <img src="content/ff7remakeNight.jpg" width="49%" />
    <img src="content/ff7remakeDay.jpg" width="49%" />
</p>

[*Final Fantasy 7 Remake Intergrade, Square Enix*](https://www.square-enix.com/ffvii/en-us/games/remake/)

<p float="left">
    <img src="content/ff7rebirthDark.jpg" width="49%" />
    <img src="content/ff7rebirthDay.jpg" width="49%" />
</p>

[*Final Fantasy 7 Rebirth, Square Enix*](https://www.square-enix.com/ffvii/en-us/games/rebirth/)

When you compare similar areas to Rebirth *(ignoring art direction differences)*, the generalized cutbacks made become quite apparent. From Remake to Rebirth one of the biggest switches they did was to go from pre-computed lightmaps to pre-computed light probes. 

<p float="left">
    <img src="content/unity-lightmap.png" width="49%" />
    <img src="content/unity-probe.png" width="49%" />
</p>

*(Unity Example) Left Side: Pre-Computed Lightmaps | Right Side: Pre-Computed Probe Volumes*

From a practicality standpoint, this is the right call considering the size of Rebirth's world, and even I would have done the same *(although one could argue that games like [Horizon Zero Dawn](https://80.lv/articles/horizon-zero-dawn-interview-with-the-team) managed it)*. From a visuals standpoint the quality of global illumination especially in ambient areas will suffer the most because of that, as going from pre-computed lightmaps to pre-computed light probes means you lose out on alot of spatial resolution, so things because very blurry and blobby like you see above.

Now given the constraints, you really can't do a whole lot without resorting to some more complex rasterization effects or elaborate GI setups to get half decent results. Doing all of that would definetly eat into your frametime budget more than you would probably want for a game that is centered around fast action. So... honestly for the most part I can forgive the quality of the global illumination that we ended up with.

*Side Note: Just for the record I know it's not impossible to make such a open world look great in ambient areas. Games like [Far Cry 3](https://www.ubisoft.com/en-us/game/far-cry/far-cry-3), [Assassin's Creed Unity](https://www.ubisoft.com/en-us/game/assassins-creed/unity), and even [The Divsion](https://www.ubisoft.com/en-us/game/the-division) managed to get very impressive results in their time with tigher constraints than today's hardware... but those are different teams, with different technology bases, different production pipelines, different timelines, and constraints.*

So let's be reasonable here... let's assume that we are under very tight constraints that were imposed on the FF7R team *(prioritizing high framerate and fast action with good clarity)*... so within these constraints, what can we do to improve the quality of our game lighting?

#### Lighting Solutions - Ray Tracing?

I want to knock this one off first because it's usually the one that is on everyones minds... generally Ray-Tracing is amazing and when done well can solve so many lighting problems in one go, but even with today's hardware it is still incredibly expensive and creates it's own large set of problems that need solving before you get something usable.

First off you'd need to have an acceleration structure that essentially stores all of the triangles within your scene, but Rebirth is a huge open world so that means billions of triangles that need to be stored and held in memory. Rebuilding that every frame is out of the queston unless you have some crazy cascade like scheme. Precomputation would be a valid option but then you'd have to hold all of that data in memory, so you'd have deal with it in smaller chunks. All of that even with precomputation would most certainly introduce a lot of overhead to just managing all of that data, and also dealing with dynamic meshes into the mix.

Ontop of that, once you have the acceleration structure you still have the issue of needing to calculate your lighting. For most path-traced games *(Cyberpunk and RE9 come to mind)* you can only manage a small amount of rays for every pixel for a decent playable frame rate, sometimes maybe even a ray for every 1/2th pixel or even 1/4th! With a small amount of rays for every pixel that also means lots of noise! That noise needs to be piped through many layers of filtering *(Temporal Filtering, ReSTIR, ATrous Wavelet, etc.)* just to get something half decent at the end that often is not sharp enough to resolve good detail in motion. To add insult to injury that is usually just for diffuse shading, for specular shading it needs it's own pipeline and set of filters/solutions which just piles on more costs and problems.

There are different techniques for shading with Ray-Tracing, and I could go on... but I think you get the picture. That isn't to say I'm not a fan of Ray-Tracing, I am but we need to be realistic and reasonable here given the constraints, and unfortunately Ray-Tracing is a no go because there are just too many problems to solve with it... So what else can we do?

#### Lighting Solutions - Increase Shadow Map Resolution?

The first and obvious thing would be to bump the resolution of the Directional Light Shadowmap. In the case of FF7 Rebirth, using something like [FFVIIHook](https://www.nexusmods.com/finalfantasy7rebirth/mods/4?tab=description) can let you do this even at runtime. Most other UE-based games are similar to a degree either with a console or .ini tweaks.

<p float="left">
    <img src="content/unity-shadow-low.png" width="49%" />
    <img src="content/unity-shadow-high.png" width="49%" />
</p>

*Unity Example: Left is Hard Shadows with high distance, right side is the same but doubled resolution.*

It's simple, effective... but costly! There are actually multiple caveat's with doing this...
- Increasing the resolution means more memory for the shadowmap, and more pixel invocations...  
- Shadowmaps can only resolve so much detail across an area, and this scales with distance. 

Rebirth is an open world... so we need to be able to see shadows really far away, so shadow distances need to be high, but that means less resolution with more distance! Again we could try bumping the resolution to compensate but we can hit VRAM/Memory problems as we can only hold so much data. This only works up to a certain point but due to the scale of the world this is just not enough.

Well that sucks... what else can we do?

#### Lighting Solutions - Increase Shadow Map Cascades?

Building on the increased shadow map resolution idea, if we can't cover the entire world with one large single shadow map texture... why not cover it with multiple shadow maps instead?

This is effectively what [Cascaded Shadow Maps (CSM)](https://dev.epicgames.com/documentation/unreal-engine/use-cascaded-shadows?application_version=4.27) are, and fortunately Rebirth is already using this *(along with most of the industry, it's very common)*. It works like this...

- Cascade 0 (1024x1024): Render the scene depth 100 units across from perspective of directional light
- Cascade 1 (1024x1024): Render the scene depth 250 units across from perspective of directional light
- Cascade 2 (1024x1024): Render the scene depth 500 units across from perspective of directional light
- Cascade 3 (1024x1024): Render the scene depth 1000 units across from perspective of directional light

<p float="left">
    <img src="content/unity-shadow-csm.png" width="49%" />
    <img src="content/unity-shadow-csm-visual.png" width="49%" />
</p>

*Unity Example: Left side is 4 cascade shadowmaps, same resolution as before but note how much better looking shadows close to the camera are. The Right visualizes the 4 cascades with colors.*

The more cascades we do, the better coverage we get of our scene, and because each cascade is at a different scale we can allocate far more pixels for certain areas closer to the player camera. With some manual fine tuning of the distances for each cascade we can get some exceptionally good looking results with this. Awesome!

Except... there are problems with this technique as well.

For each Shadowmap Cascade, you need to re-render the scene *(because it's just another shadowmap)*. That means for shadow 4 cascades *(which is usually the standard at most)* you need to re-render the scene 4 different times again! That is alot of draw calls that will eventually tank your performance. Even worse, the greater the scene complexity gets *(i.e. the bigger the world / more objects on screen to draw)* the heavier this gets!

Because of this, Rebirth sticks to 2 Shadow cascades. 1 Shadow cascade fairly close to the camera, and the other a reasonable distance away. The rest just remains unshadowed.

<p float="left">
    <img src="content/directionalshadowmap.jpg" width="100%" />
</p>

*FF7 Rebirth's two cascaded directional light shadowmap.*

#### Lighting Solutions - Shadow Map Caching?

*(NOTE: I believe the game is already using this, though I haven't been able to verify just yet at the moment)*

Shadowmap Caching? Usually Shadowmaps are rendered every frame, but turns out we actually don't have to do that!

Shadowmap Caching is simple, for certain lights or maybe even cascades we can actually update them at a slower rate. This gives us performance gains as we don't have to re-render them every frame, great! We can even take this idea further and "pre-compute" a shadowmap that we can just simply pull whenever we want and not have to redraw the scene at all, even better!

Except... just like shadowmaps you still have to deal with the problem of memory. Holding too many shadowmap textures in VRAM/memory can lead to problems, and again with the scale of Rebirth's world there wouldn't be enough to reasonably hold all of the needed data for the quality level we'd want to hit.

In addition also rendering shadows at slower rates can introduce new problems. Moving objects rendered in a delayed shadowmap can create "false" occlusion issues, and visual problems where shadows will appear to "stutter" with sparse updates update.

Well darn... things are looking grim... is there anything else we can do?

#### Lighting Solutions - Contact Shadows

Instead of utilizing shadow maps, we can calculate shadows somewhat accurately just by using the camera depth buffer and a given light direction. If games can calculate screen-space ambient occlusion with half decent results along with other similar techniques, then we should be able to calculate shadows with screen-space information as well!

Now we get into the whole reason why this article exists, turns out there is a technique here thats been around for a while and has been used by many games in the industry. [Screen-Space Contact Shadows!](https://panoskarabelas.com/posts/screen_space_shadows/)

<p float="left">
    <img src="content/unity-no-contact.png" width="49%" />
    <img src="content/unity-contact.png" width="49%" />
</p>

*Unity Example: Left side is soft shadows only. Right side adds screen-space contact shadows onto the soft shadows during light shading.*

The biggest win of this technique is that there is no need for re-drawing the scene for a shadowmap, and dealing with those kinds of issues. All we need is a depth buffer and and a light direction *(or position)*, and during shading time for drawing a light we just simply trace shadows in screen space at the same time. Requires minimal setup, costs roughly as much as SSAO, and scales far better than most of the other rasterization techniques *(that I've seen anyway)*

Now, just like the lighting solutions before, there are issues with this technique as well, but considering the context that we are in this effect is actually perfectly suited for this kind of situation! Rebirth is built on UE4, and it already has quite the feature set when it comes to graphical effects. Among them are [contact shadows that are already built-in to the engine by default!](https://dev.epicgames.com/documentation/unreal-engine/contact-shadows-in-unreal-engine?lang=en-US) 

But heres the real head-scratcher... Rebirth does not use them at all? Unfortunately with modding using FFVII Hook and many commands it appears that many of those graphical features including contact shadows have been stripped out. Why?

This single problem has led me down a spiral *(hence this mod and article :D)* and on a journey to be able to see and play the game with this effect integrated. So... let's get into it!

## Brief RenderDoc Breakdown

Here is a breakdown of how a frame is rendered in Rebirth using [RenderDoc](https://renderdoc.org/) for this whole process. Unfortunately I am not [Muhammad](https://mamoniem.com/author/muhammad/) or [Adrian Courreges](https://www.adriancourreges.com/blog/2020/12/29/graphics-studies-compilation/) so this will be a very brief breakdown of Rebirth's rendering. *(And that would veer the focus of this article off too far)*. This should be just enough to give you some idea of what's going on under the hood.

Both FF7 Remake and FF7 Rebirth are both built on a modified UE4 technology base. UE4 uses a [Deferred Renderer Pipeline](https://en.wikipedia.org/wiki/Deferred_shading), in english this means that any shading is done later in rendering process. Before shading we draw the scene multiple times into multiple different graphics buffers *(GBuffers)* that can then be used later to do our actual shading. *Hence the name deferred, the shading is deferred to a later point in time.*

In the case of rebirth, the renderer draws to **7 different GBuffers + Depth**...

<p float="left">
    <img src="content/gbuffer-rt0.jpg" width="49%" />
    <img src="content/gbuffer-rt1.jpg" width="49%" />
</p>

*Left Side: RT0 | R16G16B16A16_FLOAT | Full black except for alpha channel, I'm not sure what this is used for yet...*
*Right Side: RT1 | R10G10B10A2_UNORM | WorldNormal (RGB), SelectiveOutputMask (A)*

<p float="left">
    <img src="content/gbuffer-rt2.jpg" width="49%" />
    <img src="content/gbuffer-rt3.jpg" width="49%" />
</p>

*Left Side: RT2 | B8G8R8A8_TYPELESS | Metallic (R) Specular (G) Roughness (B) ShadingModelID (A)*
*Right Side: RT3 | B8G8R8A8_TYPELESS | BaseColor (RGB) GBufferAO (A)*

<p float="left">
    <img src="content/gbuffer-rt4.jpg" width="49%" />
    <img src="content/gbuffer-rt5.jpg" width="49%" />
</p>

*Left Side: RT4 | B8G8R8A8_TYPELESS | Custom Data, This is quite multi-purpose for different parts of the image depending on the ShadingModelID.*
*Right Side: RT5 | B8G8R8A8_TYPELESS | WorldNormals repeated again, not sure what the purpose of this is.*

<p float="left">
    <img src="content/gbuffer-rt6.jpg" width="49%" />
    <img src="content/gbuffer-depth.jpg" width="49%" />
</p>

*Left Side: RT6 | R16G16_UNorm | Velocity Buffer*
*Right Side: Depth | D32S8_TYPELESS | Scene Depth*

Just to also show you the rendering timeline...

<p float="left">
    <img src="content/render-doc-timeline.png" width="100%" />
</p>

*[RenderDoc](https://renderdoc.org/) Timeline*

This occurs almost near the end of the pipeline *(the flag on the timeline is where the Deferred GBuffer finishes drawing)*, it looks like Rebirth has a vast amount of events up before the actual GBuffer drawing which I won't go into, but most of them boil down to just...

<p float="left">
    <img src="content/hzb-mip0.jpg" width="100%" />
</p>

*Hierarchical-Z Buffer (HZB) Occlusion: Rendering the world to determine what objects to occlude later (with mips).*

<p float="left">
    <img src="content/directionalshadowmap.jpg" width="100%" />
</p>

*Shadow Map Draws: Rendering of the world through the perspective of some lights for shadowmaps. In this case the sun/directional light*

Among other miscellaneous tasks necessary for the game to render a frame that you will eventually see, but by this point after the GBuffer drawing we essentially have almost everything we need to start doing actual shading with the game. Fast forwarding just a few small rendering events and we stumble upon our Directional Light *(sunlight)* draw pass which will shade the scene with the sunlight! This is what we will modify...

<p float="left">
    <img src="content/base-naked.jpg" width="100%" />
</p>

## Implementation

Going forward we will be using [RenderDoc](https://renderdoc.org/) not only for timings, but [RenderDoc](https://renderdoc.org/) also allows us to modify shaders and be able to replay them in the pipeline to see how they work *(not during runtime though)*

Now just before we get into the implementation of these effects we need to get baseline timings before we start adding things that could slow us down later. These baselines will help keep ourselves in check later so we don't build something too expensive.

[RenderDoc](https://renderdoc.org/) has a very useful feature where for a given capture, you can actually replay it and get timing values from it to see generally how much time each induvidual draw or operation takes. Granted the timings will not match the original game/application 1:1. You have to run it multiple times to get a good average, and even then it will be off by some factors, but for the most part it's relatively accurate and will show you if one or more draws are more expensive than the other.

It's important do this so we can orient ourselves later to check how expensive things have gotten. To start let's look at the total frame time of the original game capture here. This is on an RTX 3080, at 3840x2160, and the settings of the game are all maxed out.

<p float="left">
    <img src="content/timing-entire-frame.png" width="100%" />
</p>

Roughly ```34.5ms```. To help orient ourselves with the timing values, heres a quick reference...
- 30 FPS Frame-Time Budget: 33.33ms
- 60 FPS Frame-Time Budget: 16.67ms
- 120 FPS Frame-Time Budget: 8.33ms

For the timing value that we have, it's effectively 30 FPS, which is actually accurate given that I played the game at **native 4K** *(No DLSS, upscaling, or dynamic resolution)* and maxed out the visual settings. Sometimes during certain areas it increases or decreases depending on the complexity of the scene.

Now lets go to the draw call that is responsible for shading the scene with sunlight, and here is the original timings for the Directional Pixel Light Shader that the game is using within [RenderDoc](https://renderdoc.org/)...

<p float="left">
    <img src="content/timing-original-shader.png" width="100%" />
</p>

General timings for it are ```0.38ms - 0.40ms```, this is an important baseline to keep in mind so we can check if we are slower, or faster before we move forward. Now heres the thing... that is a compiled shader, I cannot accurately decompile it into a shader that I can edit and make changes to. I don't have the original source code, so I'll have to reverse engineer the shader.

Fortunately Unreal shader source files are public for UE4, and with some searching plus trial and error I managed to mostly rebuild Unreal's Deferred Directional Pixel Light shader. It's not match as there are some modifications that have been done for the game. I've done my best to match the original game shading as close as possible. For the most part it's almost indiscernable visually minus some specualrity differences that I have yet to fully iron out.

The timings for the reverse engineered shader are as follows...

<p float="left">
    <img src="content/timing-base.png" width="100%" />
</p>

General timings are roughly ```0.2ms - 0.21ms```, that's definetly quite a bit faster than the original but I'm sure I'm probably missing some terms here and there that the original shader is doing. Again the shader is almost identical to the in-game one visually but for sanity and simplicity sake let's just say that we are working with the original shader here. Importantly, we have an actual baseline timing now which is ```0.2ms```. When we start adding more and more effects we should see the timing values rise, and we'll know how far off we are from baseline. Now ideally for most post effects we want to be as far below 1 milisecond as we can, the smaller the timing the better our performance by the end will be. Less is always better!

So... with our baseline set now at roughly ```0.209ms```, let's get to work!

### Micro Shadows

This was the first thing that I wanted to try, as it's a very simple and cheap *(but very effective)* technique that was introduced in [Uncharted 4](https://advances.realtimerendering.com/other/2016/naughty_dog/). 

[Micro Shadows](https://advances.realtimerendering.com/other/2016/naughty_dog/) are a very low cost way of adding detail to a surface or object in direct light by simulating micro shadows using the material ambient occlusion.

```HLSL
float ComputeMicroShadowing(float AO, float NdotL, float opacity)
{
    float aperture = 2.0 * AO * AO;
    float microshadow = saturate(NdotL + aperture - 1.0);
    return lerp(1.0, microshadow, opacity);
}
```

To apply it to our shading all we need to do is just call the function, pass in the necessary terms and multiply it against our final light.

```HLSL
//pseudo code!
//lightAttenuation will factor in other light terms, like shadows
//if it's a local light it should also factor in distance falloffs and whatnot
//...

lightAttenuation *= ComputeMicroShadowing(GBufferData.GBufferAO, NdotL, 1.0f);

//add diffuse and specular
finalColor += diffuseLight;
finalColor += specularLight;

//apply attenuation
//NOTE: theroetically the micro shadowing and other shadowing terms we do should always be affecting both the diffuse and specular.
finalColor *= lightAttenuation;
```

Now let's see how it looks in the game, looking at the original first then with microshadows...

<p float="left">
    <img src="content/base.jpg" width="100%" />
</p>

*No Micro Shadows*

<p float="left">
    <img src="content/micro-shadows.jpg" width="100%" />
</p>

*With Micro Shadows*

You will need to compare both images side by side and flip between them to see the full effect on the final rendered image, but to help with that I will show the micro-shadow effect in pure isolation.

<p float="left">
    <img src="content/micro-shadows-only.jpg" width="100%" />
</p>

The beauty of the technique is in it's simplicity. It is a bit of a hack, but it's physically plausible and the result's are quite effective at adding detail just by leveraging the authored material ambient occlusion maps. Holes, crevices, and other details that are in the occlusion map are amplified.

Checking our timings...

<p float="left">
    <img src="content/timing-micro-shadow.png" width="100%" />
</p>

It's difficult to pinpoint the exact values due to some noise, but generally they fall in the same exact ```0.2ms - 0.21ms``` range just like before. Like I said, super cheap! That certainly helped a bit... but I'm not fully satisfied yet! I want to go even further and beyond!

### Contact Shadows

Now just before we get into the meat of things, a brief explanation on what Contact Shadows actually are...

<p float="left">
    <img src="content/contact_shadows_screen_space.svg" width="100%" />
</p>

**Contact Shadows** is a screen-space shadowing technique that works by raymarching from a shaded point toward a light source and testing whether the ray intersects scene geometry reconstructed from the depth buffer.
- If we are hitting scene geometry on our way to the light, that point is occluded *(in-shadow)*
- If the ray reaches the light without hitting geometry, the point receives direct lighting.

Now unlike traditional shadow maps, Contact Shadows operate entirely in screen-space and use information already available in the depth buffer. This makes them relatively inexpensive while still providing small-scale shadow detail. Ok... sounds plausible enough, but there are drawbacks with this technique...

#### Limitations

**We only trace visible geometry** 

Contact Shadows only have access to geometry that was visible when the depth buffer was generated. In simple terms that means if an object isn't visible to the camera, it basically doesn't exist. Due to this there a few artifacts that can happen...

- Off-screen objects cannot cast contact shadows.
- Objects entering or leaving the screen can cause shadows to appear or disappear *(same for edges of screen)*
- Occlusion information may be incomplete when geometry is hidden behind other geometry.

**We are testing against limited geometry information**.

Contact Shadows do not test against actual scene geometry. Instead, they reconstruct geometry from the depth buffer. The depth buffer only stores the closest visible surface at each pixel and contains no information about things like surface thickness, backfaces, hidden or interior geometry and so on.

Due to this issue, most implementations introduce a thickness value when performing intersection tests. Choosing a value that is too small can cause rays to miss thin objects, while values that are too large can create false self-shadowing and other artifacts.

#### Getting Good Results

I probably did a bad job of selling the effect, but that's ok! It's important to learn about the problems first so we can setup this effect in a way where we can maximize the best results from it... so how do we do that?

- **Shadow ray lengths should generally be kept short.** Longer rays increase the likelihood of missing occluders, encountering missing geometry, and producing visible artifacts. *(NOTE: This is also where it gets it's name, because it excels at adding small-scale shadows where objects make contact with nearby surfaces)*
- **Thickness values should remain relatively small.** Excessive thickness can lead to over-occlusion and funky self-shadowing artifacts, while values that are too small may allow thin geometry to leak light through.

If we keep all of that in mind, then our contact shadows can provide us a significant amount of additional shadow detail! So... with all of that yapping done... let's go ahead and try it out!

#### Naive Contact Shadows (World Space)

Starting out, we will do things in world-space to keep things simple. Remembering the concept... given a point in 3D space *(vector_worldPosition)*, and the direction of the light *(vector_worldLightDirection)*, we want to march our ray for *(CONTACT_SHADOWS_SAMPLES)* amount of samples towards the light. Within each sample we will read the depth texture and check if our ray is inside geometry or not.

First some configuration variables to define how we do this effect, these eventually will be tweaked later as well.

```HLSL
//how many marches we do against the scene depth buffer
#define CONTACT_SHADOWS_SAMPLES 64

//how far out do the rays go
#define CONTACT_SHADOWS_RAY_LENGTH 100.0

//assumed thickness of the depth
#define CONTACT_SHADOWS_THICKNESS 0.35

//offset to our starting position so we avoid self-occlusion artifacts
#define CONTACT_SHADOWS_BIAS 1e-04

//offset to our starting position using the surface normal so we avoid self-occlusion artifacts
#define CONTACT_SHADOWS_NORMAL_BIAS 1e-04
```

Now heres the actual function.

```HLSL
float ContactShadowWorldSpace(
    float3 vector_worldPosition,
    float3 vector_worldNormal, //this isn't required, but this is used as an offset for our starting position
    float3 vector_worldLightDirection)
{
    //based on the amount of samples, and the ray length, calculate how spaced out each march should be
    float raymarchStepSize = CONTACT_SHADOWS_RAY_LENGTH / CONTACT_SHADOWS_SAMPLES;

    float3 rayOrigin = vector_worldPosition;

    //NOTE: using the geometry normal, we offset our starting position so that we don't occlude ourselves
    rayOrigin += vector_worldNormal * CONTACT_SHADOWS_NORMAL_BIAS; //normal bias

    //our step interval
    float3 rayStep = vector_worldLightDirection * raymarchStepSize;

    //advance the ray once
    rayOrigin += rayStep; 

    [loop]
    for (int i = 0; i < CONTACT_SHADOWS_SAMPLES; i++)
    {
        float3 vector_samplePosition = rayOrigin + rayStep * i;
        float2 vector_sampleUV = WorldToScreenUV(vector_samplePosition);

        //clip the edges
        if (vector_sampleUV.x < 0.0 || vector_sampleUV.x > 1.0 || vector_sampleUV.y < 0.0 || vector_sampleUV.y > 1.0)
            break;

        float sampledDepth = SceneTexturesStruct_SceneDepthTexture.SampleLevel(View_SharedBilinearClampedSampler, vector_sampleUV, 0).r;
        float3 vector_sampleWorldPosition = ReconstructWorldPosition(vector_sampleUV, sampledDepth);
        float rayDepth = distance(vector_samplePosition, View_WorldCameraOrigin);
        float sceneDepth = distance(vector_sampleWorldPosition, View_WorldCameraOrigin);
        float depthDiff = rayDepth - sceneDepth;

        if (depthDiff > CONTACT_SHADOWS_BIAS && depthDiff < CONTACT_SHADOWS_THICKNESS * 100)
            return 0.0;
    }

    return 1.0;
}
```

Just like the [Micro Shadows](#micro-shadows), we apply this to the light attenuation as such.

```HLSL
//pseudo code!
//you'll need to calculate vector_worldPosition, vector_normalDirection and access to vector_lightDirection
//lightAttenuation will factor in other light terms, like shadows
//if it's a local light it should also factor in distance falloffs and whatnot
//...

//micro shadowing from before
lightAttenuation *= ComputeMicroShadowing(GBufferData.GBufferAO, NdotL, 1.0f);

//NEW: our contact shadows!
lightAttenuation *= ContactShadowWorldSpace(
    vector_worldPosition.xyz, //vector_worldPosition
    vector_normalDirection, //vector_worldNormal
    vector_lightDirection); //vector_worldLightDirection

//add diffuse and specular
finalColor += diffuseLight;
finalColor += specularLight;

//apply attenuation
//NOTE: again just like micro shadowing, the terms should always be affecting both the diffuse and specular.
finalColor *= lightAttenuation;
```

Ok sweet, it's all setup, now lets check the results!

<p float="left">
    <img src="content/micro-shadows.jpg" width="100%" />
</p>

*Micro Shadows Only*

<p float="left">
    <img src="content/contact-shadows-no-random-world-space.jpg" width="100%" />
</p>

*Micro Shadows + Contact Shadows*

Wow! What a difference, so much detail in the scene gets revealed. All of the foliage also that was excluded from the shadowmaps now are casting shadows again. Near contact points also the shadows sharpen up and the overall shadow resolution appears much higher!

If we take a closer peek...

<p float="left">
    <img src="content/closeup1-contact.png" width="100%" />
</p>

*Micro Shadows + Contact Shadows*

<p float="left">
    <img src="content/closeup1-og.png" width="100%" />
</p>

*Micro Shadows Only*

Look at that! Cloud recieves shadows from his hair, that kind of detail would be too small scale to be resolved by the main shadowmaps. The shadow underneath his shoulder-pad also became sharper, and even behind him many foliage details start casting their shadows onto the scene.

<p float="left">
    <img src="content/contact-shadows-no-random-world-space-naked.jpg" width="100%" />
</p>

*Contact Shadows Term*

Well... I geuss we're done right? What do the timings look like...

<p float="left">
    <img src="content/timing-naive-contact-shadows-64-worldspace.png" width="100%" />
</p>

``3.02ms`` Oh my... that is not good... the shader with the added effect is really slow! *(baseline timings were roughly ```0.2ms - 0.21```)* 

Well... time to really dig in and optimize things, atleast now we got it working so lets see what we can do.

#### Optimization: Reduce Sample Counts

So the first obvious thing we can reduce the sample count, initally I had the sample count set to 64 *(which is pretty high)* so lets knock it down by half.

```HLSL
//previously was 64
#define CONTACT_SHADOWS_SAMPLES 32
```

Now what do the timings look like?

<p float="left">
    <img src="content/timing-naive-contact-shadows-32-worldspace.png" width="100%" />
</p>

``1.63ms``, Ok that is much better! Still far from ideal *(we want to be sub 1 milisecond)*, but that makes things a little more managable. However we reduced the sample count though... so that means our quality is worse right?

<p float="left">
    <img src="content/stepping-32-no-random.png" width="49%" />
    <img src="content/stepping-64-no-random.png" width="49%" />
</p>

*Left: 32 Samples | Right: 64 Samples*

Yeah it's definetly worse, and what is wierd is that we're seeing this wierd stepping like artifact *(which is a side effect of raymarching, we march in steps)*. If we reduce samples even more those steps will become even more visible. We could try to reduce the ray length to shorten the step distance between samples but that means we won't be tracing rays as far anymore. 

What else can we do?

#### Quality Boost: Introduce Noise (Blue Noise)

Fortunately we are in luck, as there is a way that we can mitigate these raymarch stepping artifacts. The solution is noise! *(or jitter, or dithering)*

To explain quickly how this works, currently we are stepping uniformly into the depth buffer for every pixel. The uniformity is why we are seeing the stepping. If we introduce randomness into the step, then this will actually allow us to march at different intervals to cover broader areas without increasing sample counts. Seems sound... lets try it!

First let's adjust the function with random in mind...

```HLSL
float ContactShadowWorldSpace(
    float3 vector_worldPosition,
    float3 vector_worldNormal,
    float3 vector_worldLightDirection,
    float random) //added field for random values that we provide
{
    float raymarchStepSize = CONTACT_SHADOWS_RAY_LENGTH / CONTACT_SHADOWS_SAMPLES;

    float3 rayOrigin = vector_worldPosition;
    rayOrigin += vector_worldNormal * CONTACT_SHADOWS_NORMAL_BIAS;

    //our step interval
    float3 rayStep = vector_worldLightDirection * raymarchStepSize;

    //advance the ray once with jitter
    rayOrigin += rayStep * random; 

    [loop]
    for (int i = 0; i < CONTACT_SHADOWS_SAMPLES; i++)
    {
        float3 vector_samplePosition = rayOrigin + rayStep * i;
        float2 vector_sampleUV = WorldToScreenUV(vector_samplePosition);

        if (vector_sampleUV.x < 0.0 || vector_sampleUV.x > 1.0 || vector_sampleUV.y < 0.0 || vector_sampleUV.y > 1.0)
            break;

        float sampledDepth = SceneTexturesStruct_SceneDepthTexture.SampleLevel(View_SharedBilinearClampedSampler, vector_sampleUV, 0).r;
        float3 vector_sampleWorldPosition = ReconstructWorldPosition(vector_sampleUV, sampledDepth);
        float rayDepth = distance(vector_samplePosition, View_WorldCameraOrigin);
        float sceneDepth = distance(vector_sampleWorldPosition, View_WorldCameraOrigin);
        float depthDiff = rayDepth - sceneDepth;

        if (depthDiff > CONTACT_SHADOWS_BIAS && depthDiff < CONTACT_SHADOWS_THICKNESS * 100)
            return 0.0;
    }

    return 1.0;
}
```

Then just before we call our function, we can either calculate a random noise pattern or sample one from a texture. In my case I will sample a noise pattern texture, mostly because it's already being provided into the shader.

```HLSL
//pseudo code!
float random = //... calculate noise here, or sample it from a texture (needs to be per pixel!)

lightAttenuation *= ContactShadowWorldSpace(
    vector_worldPosition.xyz, //vector_worldPosition
    vector_normalDirection, //vector_worldNormal
    vector_lightDirection,
    random); //plug in our random value

```

Just for visuals, this is the noise texture given to the shader already. It's [Blue Noise](https://blog.demofox.org/2018/01/30/what-the-heck-is-blue-noise/).

<p float="left">
    <img src="content/renderdoc-blue-noise-tex.png" width="100%" />
</p>

Alright, now that we have noise within our contact shadows, how does it look now?

<p float="left">
    <img src="content/stepping-32-random-blue.png" width="100%" />
</p>

Ok that actually looks pretty good! I'm actually seeing some areas that before were not even visible with the 64 Samples non-jittered result. The only drawback is now we have noise everywhere... but fortunately we have TAA later in the rendering pipeline so we can rely on it to clean up the noise and blend results temporally.

<p float="left">
    <img src="content/stepping-32-random-blue.png" width="49%" />
    <img src="content/stepping-32-no-random.png" width="49%" />
</p>

*Left: 32 samples with Blue Noise | Right: 32 samples with no noise*

<p float="left">
    <img src="content/stepping-32-random-blue.png" width="49%" />
    <img src="content/stepping-64-no-random.png" width="49%" />
</p>

*Left: 32 samples with Blue Noise | Right: 64 samples with no noise*

Anyway... how are the timings now, they should be the same right?

<p float="left">
    <img src="content/timing-with-blue-noise.png" width="49%" />
</p>

Roughly ```1.67ms```, huh? 

It's actually more expensive to use noise? Granted it's not by alot *```(~0.04ms)```*, but we are at the same sample count! How is this happening? 

Well the answer comes down to coherence. To keep the explanation as simple as I can, heres the quick context. We are working with shaders here, and shaders run on GPUs. GPUs are essentially big multi-core processors. They are extremely good at processing large groups of pixels that perform similar work. 

Now before we introduced jitter, neighboring pixels were marching through the scene using nearly identical sample locations. This allowed the GPU to access memory efficiently and reuse cached data. Great for performance!

However... once we added noise, every pixel now follows a slightly different path. The sample count is the same, but memory accesses become less coherent, cache efficiency decreases, and performance becomes worse! Arghhhhh!

Fortunately... we're not out of options yet!

For jitter we're currently using [Blue Noise](https://blog.demofox.org/2018/01/30/what-the-heck-is-blue-noise/). Compared to White Noise, [Blue Noise](https://blog.demofox.org/2018/01/30/what-the-heck-is-blue-noise/) produces a more visually pleasing distribution of samples and tends to hide artifacts much better. Now whether it's white or blue noise it doesn't really matter here because both are very random *(white noise more so technically)* and due to that nature that actually is contributing to a loss of coherence. So now we have an interesting question... can we find a 

 pattern that sits somewhere between perfectly uniform and completely random?

#### Optimization: Switching to Interleaved Gradient Noise (IGN)

Enter [Interleaved Gradient Noise](https://blog.demofox.org/2022/01/01/interleaved-gradient-noise-a-different-kind-of-low-discrepancy-sequence/), this is a noise pattern that comes from [Jorge Jimenez](https://www.iryoku.com/next-generation-post-processing-in-call-of-duty-advanced-warfare/). Unlike Blue Noise or White Noise, [Interleaved Gradient Noise](https://blog.demofox.org/2022/01/01/interleaved-gradient-noise-a-different-kind-of-low-discrepancy-sequence/) is designed to be low-discrepancy, which just means that neighboring pixels receive values that are distributed more evenly and predictably *(as opposed to uneven, random distribution)*. 

The goal isn't to be perfectly random, but about providing good sample coverage while maintaining some degree of coherence. Introducing enough variation to break up visible stepping artifacts, while remaining structured enough to be more GPU-friendly than purely random sampling patterns.

<p float="left">
    <img src="content/noisesbig.png" width="100%" />
</p>

*[Image from Demofox]((https://blog.demofox.org/2022/01/01/interleaved-gradient-noise-a-different-kind-of-low-discrepancy-sequence/))*

Ok, I geuss I'm sold... lets try it out! This one we calculate rather than sample from a texture.

```HLSL
float InterleavedGradientNoise(float2 pixCoord, int frameCount)
{
	const float3 magic = float3(0.06711056f, 0.00583715f, 52.9829189f);
	const float2 frameMagicScale = float2(2.083f, 4.867f);
	pixCoord += frameCount * frameMagicScale;
	return frac(magic.z * frac(dot(pixCoord, magic.xy)));
}
```

Plugging this in place of the Blue Noise that we were using earlier...

<p float="left">
    <img src="content/stepping-32-ign.png" width="100%" />
</p>

*32 Samples with Interleaved Gradient Noise*

Interesting... the noise seems reduced by quite a bit which is great for quality. The stepping artifacts also arent that visible either! 

<p float="left">
    <img src="content/stepping-32-ign.png" width="49%" />
    <img src="content/stepping-32-random-blue.png" width="49%" />
</p>

*Left: 32 Samples Interleaved Gradient Noise | Right: 32 Samples Blue Noise*

This is definetly looking better, but the real question is... is the GPU happier with this?

<p float="left">
    <img src="content/timing-with-ign.png" width="100%" />
</p>

Roughly ```1.63ms```, the timings went back down! The GPU is happier with that, good!

But again we still are not done, like I said earlier while knocking the sample count down further led to much better timings, ```1.6ms``` is still not great and not where I want it to be. Remember that the original shader timings without contact shadows were roughly ```0.2ms```, so we still have about ```~1.3ms``` to shave off at the most *(it's a little unrealistic but thats where we started from)*. What are some other optimizations we can actually do to improve performance?

#### Optimization: Clip Space

Well other than just toying with the sample counts and ray lengths... we can actually change our raymarching strategy! 

Our original function traces contact shadows in world space, which seems fine and easy enough... but as it's turns out we are actually doing quite a bit more work than we need to.

For each raymarch step we advance a world-space position, project that position into clip/screen space, sample the depth buffer, and then perform our intersection tests. Since these operations occur inside the raymarch loop, their cost quickly adds up especially with the large amount of samples.

```HLSL
float2 vector_sampleUV = WorldToScreenUV(vector_samplePosition); //<------

//...

float3 vector_sampleWorldPosition = ReconstructWorldPosition(vector_sampleUV, sampledDepth); //<------
float rayDepth = distance(vector_samplePosition, View_WorldCameraOrigin);
float sceneDepth = distance(vector_sampleWorldPosition, View_WorldCameraOrigin);
```

Recall that Contact Shadows ultimately is just screen-space effect, not world space. The depth buffer already exists in screen space, and every occlusion test eventually happens against that screen-space data. So instead of marching a ray through world space and repeatedly projecting it back to the screen, we can perform the raymarch directly in clip space. This removes much of the per-step transformation work and allows the shader to operate in the same space as the depth buffer it is testing against.

The end result should be the same shadowing solution, but with significantly less work performed inside the raymarch loop. Awesome, lets try it!

```HLSL
float ContactShadowClipSpace(
    float3 vector_worldPosition,
    float3 vector_worldNormal,
    float3 vector_worldLightDirection,
    float random)
{
    //minor micro optimization, precompute 1.0f / CONTACT_SHADOWS_SAMPLES so we only end up doing a mul
    float invSamples = rcp(CONTACT_SHADOWS_SAMPLES);

    float3 rayOrigin = vector_worldPosition;

    float4 clipStart = mul(View_WorldToClip, float4(rayOrigin, 1));
    float4 clipEnd = mul(View_WorldToClip, float4(rayOrigin + vector_worldLightDirection * CONTACT_SHADOWS_RAY_LENGTH, 1));

    float3 ndcStart = clipStart.xyz / clipStart.w;
    float3 ndcEnd = clipEnd.xyz / clipEnd.w;

    float rayLinearStart = LinearEyeDepth(ndcStart.z);
    float rayLinearEnd   = LinearEyeDepth(ndcEnd.z);

    float rayLinearDepth = lerp(rayLinearStart, rayLinearEnd, random * invSamples);
    float rayLinearStep = (rayLinearEnd - rayLinearStart) * invSamples;

    const float2 uvScale = float2(0.5, -0.5);
    float2 uvStep = (ndcEnd.xy - ndcStart.xy) * (invSamples * uvScale);
    float2 uv = mad(ndcStart.xy, uvScale, 0.5) + uvStep * random;

    [loop]
    for (int i = 0; i < CONTACT_SHADOWS_SAMPLES; i++)
    {
        if (any(uv < 0.0) || any(uv > 1.0))
            break;

        float depth = SceneTexturesStruct_SceneDepthTexture.SampleLevel(View_SharedBilinearClampedSampler, uv, 0).r;
        float linearDepth = View_InvDeviceZToWorldZTransform.z / (depth - View_InvDeviceZToWorldZTransform.w);
        float penetration = rayLinearDepth - linearDepth;

        if (penetration > CONTACT_SHADOWS_BIAS && penetration < CONTACT_SHADOWS_THICKNESS)
            return 0.0;

        rayLinearDepth += rayLinearStep;
        uv += uvStep;
    }

    return 1.0;
}
```

It's worth mentioning that moving from world space to clip space changes some of the ray traversal math. The important takeaway is that we eliminate the space conversions that were previously happening inside the raymarch loop.

Now because our depth comparisons now occur in projected space rather than world space, parameters such as CONTACT_SHADOWS_THICKNESS will need to be adjusted. Aside from that though it still functions the same. 

So! how do our contact shadows look now? If we did everything right it should look mostly the same...

<p float="left">
    <img src="content/contact-shadows-random-world-32-naked.jpg" width="49%" />
    <img src="content/contact-shadows-random-clip-32-naked.jpg" width="49%" />
</p>

*Left: World Space Version | Right: Clip Space Version*

Ok good, it does. Thats a good sign, means our math all works as it should... now what about the timings?

<p float="left">
    <img src="content/timings-clip-space-32.png" width="100%" />
</p>

Roughly ```0.95ms```, oh wow, that killed effectively ```~0.6ms```, Awesome! Not only do things now run much better, but they still look the same, fantastic! But there is still a little more that we can do...

#### Optimization: Early Sky Out

One very small and simple thing we can do for optimization, is just before we do contact shadows we can check if our point is in the sky. This is important as we wouldn't want to do contact shadow tracing on sky pixels as it'd just be a waste. We do this just by checking our depth buffer, and the distance value. If the distance relative to camera is insanely high, it's probably a sky pixel so we don't calculate shadows. 

```HLSL
//optimization: early out for sky
if(GBufferData.Depth < 1000000)
{
    //do contact shadows
}
```

Visually there is no difference, but timing wise there is a small bit of improvement *(hard to fully pin down from the noise within RenderDocs replay timings)*

<p float="left">
    <img src="content/timing-early-sky-out.png" width="100%" />
</p>

Roughly ```0.94ms```, It's really small but every little bit can help. Granted this will vary depending on how much of the screen is the sky vs. ground, in this example quite a large portion of the screen is ground.

So just to simulate that example of where we have a frame with less ground coverage and more sky, I've reduced the depth distance check value from ```1000000``` to ```10000```. Checking the timing values then...

<p float="left">
    <img src="content/timing-early-sky-out-aggressive.png" width="100%" />
</p>

Roughly ```0.76ms```, which is not bad actually. However this can vary because the player is often times looking at the ground and never upwards toward the sky. So atleast for those that do... saved you some frames :D

#### Optimization: Further Sample Reduction

Now with the different techniques in place, especially with the noise, our contact shadows is as good as I can get it. However it's still too many samples, texture reads seem to be the main performance bottleneck so I will reduce the quality all the way down 8 samples, which is actually usually the common sample count even in unreal's own contact-shadow implementation.

```HLSL
//dropped from 32 to 8
#define CONTACT_SHADOWS_SAMPLES 8
```

<p float="left">
    <img src="content/contact-shadows-random-clip-8-naked.jpg" width="100%" />
</p>

*Contact Shadows with Interleaved Gradient Noise 8 Samples*

<p float="left">
    <img src="content/contact-shadows-random-clip-8-naked.jpg" width="49%" />
    <img src="content/contact-shadows-random-clip-32-naked.jpg" width="49%" />
</p>

*Left: 8 samples | Right: 32 samples*

It still looks suprisingly good! Now definetly when you compare the shadows near the contacts of objects are much weaker, and this is because with 8 samples now, the step size is much larger now than it was with 32. To compensate for this, we can knock down the max ray length so the step size is a little closer together. This does mean that we will lose out on some slightly longer shadows, but in turn we will regain our sharper contact shadows.

```HLSL
//dropped from 100 to 50
#define CONTACT_SHADOWS_RAY_LENGTH 50.0
```

<p float="left">
    <img src="content/contact-shadows-random-clip-8-half-naked.jpg" width="100%" />
</p>

Still looking fantastic! The shadows near geometry contacts regained their sharpness/intensity. Importantly I can now see the finer details of clouds hair casting shadows on his face more clearly, and same for all of the foliage on the ground. 

<p float="left">
    <img src="content/contact-shadows-random-clip-8-half-naked.jpg" width="49%" />
    <img src="content/contact-shadows-random-clip-8-naked.jpg" width="49%" />
</p>

*Left: 8 samples 50 ray length | Right: 8 samples 100 ray length*

Now... we dropped both the sample count and ray length quite a bit, so... what are the final timings now?

<p float="left">
    <img src="content/timing-final.png" width="100%" />
</p>

Roughly ```0.44ms```, that's wweet! Our contact shadows now is way more optimized than where it started out, and still can pack quite a punch to the final image all for a reasonable cost! 


Now as a refresher let's do some timing comparisons, remember that the original game shader in it's entirety was the following...

<p float="left">
    <img src="content/timing-original-shader.png" width="100%" />
</p>

Roughly ```0.40ms```, so actually funny enough our reverse engineered shader + the contact shadows are almost at the same timing value now ```+ 0.04ms - 0.05ms``` but let's not get ahead of ourselves. The reverse engineered shader timings without contact shadows was the following...

<p float="left">
    <img src="content/timing-base.png" width="100%" />
</p>

Roughly ```0.2ms - 0.21ms```, and now with the optimized contact shadows putting us at ```0.44ms``` this means that the contact shadows we added is roughly ```0.23ms`` which is actually not too bad considering we are running at 4k (3840x2160). That's good, and now we have something that radically transforms the look of the game specially in these areas that adds a large amount of detail for a relatively small cost!

[*NOTE: Click here if you want to see 1080p timings.*](#timings-on-1920x1080-and-3840x2160)

#### Bonus: Dual Depth Sampling

I left this for last as a bonus, but also in "[Rendering Tiny Glades With Entirely Too Much Ray Marching](https://www.youtube.com/watch?v=jusWW2pPnA0)" the main plus they did for improving the results of their contact shadows was actually sampling the depth textures twice. One with point sampling, the other with bilinear filtering.

The result was a significant reduction in contact shadow self-occlusion issues, the only reason I'm leaving this for last is while this does improve the quality, it means 2 depth texture samples now in the loop which makes it quite a bit more expensive.

```HLSL
float ContactShadowClipSpaceDualDepth(
    float3 vector_worldPosition,
    float3 vector_worldNormal,
    float3 vector_worldLightDirection,
    float random)
{
    float invSamples = rcp(CONTACT_SHADOWS_SAMPLES);

    float3 rayOrigin = vector_worldPosition;
    float4 clipStart = mul(View_WorldToClip, float4(rayOrigin, 1));
    float4 clipEnd = mul(View_WorldToClip, float4(rayOrigin + vector_worldLightDirection * CONTACT_SHADOWS_RAY_LENGTH, 1));

    float3 ndcStart = clipStart.xyz / clipStart.w;
    float3 ndcEnd = clipEnd.xyz / clipEnd.w;

    float rayLinearStart = LinearEyeDepth(ndcStart.z);
    float rayLinearEnd   = LinearEyeDepth(ndcEnd.z);

    float rayLinearDepth = lerp(rayLinearStart, rayLinearEnd, random * invSamples);
    float rayLinearStep = (rayLinearEnd - rayLinearStart) * invSamples;

    const float2 uvScale = float2(0.5, -0.5);
    float2 uvStep = (ndcEnd.xy - ndcStart.xy) * (invSamples * uvScale);
    float2 uv = mad(ndcStart.xy, uvScale, 0.5) + uvStep * random;

    [loop]
    for (int i = 0; i < CONTACT_SHADOWS_SAMPLES; i++)
    {
        if (any(uv < 0.0) || any(uv > 1.0))
            break;

        float pointDepth = SceneTexturesStruct_SceneDepthTexture.SampleLevel(View_SharedPointClampedSampler, uv, 0).r;
        float bilinearDepth = SceneTexturesStruct_SceneDepthTexture.SampleLevel(View_SharedBilinearClampedSampler, uv, 0).r;

        float2 depths = float2(pointDepth, bilinearDepth);
        float2 linearDepths = View_InvDeviceZToWorldZTransform.z / (depths - View_InvDeviceZToWorldZTransform.w);

        float minDepth = min(linearDepths.x, linearDepths.y);
        float maxDepth = max(linearDepths.x, linearDepths.y);

        float penetration = rayLinearDepth - minDepth;
        float depthDistance = maxDepth - rayLinearDepth;

        if (depthDistance < 0.0 && penetration < CONTACT_SHADOWS_THICKNESS)
            return 0.0;

        rayLinearDepth += rayLinearStep;
        uv += uvStep;
    }

    return 1.0;
}
```

<p float="left">
    <img src="content/singleDepth.png" width="49%" />
    <img src="content/dualDepth.png" width="49%" />
</p>

*Left: Single Depth Sample | Right: Point and Bilinear Depth Sampling*

Checking the timings...

<p float="left">
    <img src="content/timing-dual-depth.png" width="100%" />
</p>

```0.59ms```, quite a bit more than just singular depth sampling. For that cost to quality ratio I honestly would probably rather just increase sample counts, but you can see the options. 

But there is an alternative to this!

#### Bonus: Depth Point Sampling

Another intresting idea from the same place "[Rendering Tiny Glades With Entirely Too Much Ray Marching](https://www.youtube.com/watch?v=jusWW2pPnA0)" for improving the results of their contact shadows was actually sampling the depth texture using point sampling rather than bilinear filtering. While for the most part the point sampled depth was intended to be used in tandem with the bilinear sampled one, using it by itself actually can lead to better results in some cases.

```HLSL
//bilinear depth
//float sampledDepth = SceneTexturesStruct_SceneDepthTexture.SampleLevel(View_SharedBilinearClampedSampler, vector_sampleUV, 0).r;

//point depth
float sampledDepth = SceneTexturesStruct_SceneDepthTexture.SampleLevel(View_SharedPointClampedSampler, vector_sampleUV, 0).r;
```

These screenshots are from a different capture but it's one case where the benifits of using point sampling were quite obvious.

<p float="left">
    <img src="content/bilinear-sampling.png" width="49%" />
    <img src="content/point-sampling.png" width="49%" />
</p>

*Left: Single Bilinear Depth Sampling | Right: Single Point Depth Sampling*

No timings on this because it's effectively free, all we are doing is just sampling the same depth texture that we did before but with point sampling instead of bilinear. It helps with reducing self-occlusion.

#### Bonus: Checkerboard Rendering Optimization

One other small optimization idea we can do if we are really trying to squeeze as much as possible out of this, is to do checkerboard rendering. This will definetly drop the quality of the shadows a bit but if we are trying to squeeze every ounce that we possibly can this is something that we can do.

The idea is simple, we only do contact shadows for every other pixel within a 2x2 grid. For the other pixels within the same grid we skip doing contact shadows. So out of the 2x2 pixel block which has 4 pixels total, only 2 pixels do contact shadows, the other 2 are skipped entirely.

Now because Rebirth is using TAA later in the pipeline, and we are already relying on it to help cleanup the resolve of our shadows, for the checkerboard pattern it would also be wise to flip the pattern every other frame so we can average results over time within the 2x2 pixel grid.

The code for it is simple, we calculate the 2x2 checkerboard pattern and then wrap our contact shadows call within a conditional.

```HLSL
float contactShadow = 1.0f; //start off with full visibility

//do a simple 2x2 checkerboard
uint2 pixel = uint2(vector_svPosition.xy);
bool checkerboardTest = (pixel.x + pixel.y + View_TemporalAAParams.x) % 2;

//do contact shadows only for 2 pixels within the 2x2 block
//on the next frame the pattern will shift and we will do the other 2 pixels...
if(checkerboardTest)
{
    contactShadow = ContactShadowClipSpace(
        vector_worldPosition.xyz,
        vector_normalDirection,
        vector_lightDirection,
        random);
}
```

<p float="left">
    <img src="content/checkerboard-off.png" width="49%" />
    <img src="content/checkerboard-on.png" width="49%" />
</p>

*Left: Checkerboard Off | Right: Checkerboard On*

Intresting... but now the shadows look half as intense now and there is a bunch of holes in the final image! TAA could potentially fill it in but the final look may appear much softer than it's intended to be... can we somehow fill in those holes?

The answer is yes we can! Using an HLSL built-in function called [QuadReadLaneAt](https://learn.microsoft.com/en-us/windows/win32/direct3dhlsl/quadreadlaneat). What is that? Well, time for some quick context...

Modern GPUs execute pixel shaders in small 2×2 pixel groups known as quads. In [Shader Model 6](https://learn.microsoft.com/en-us/windows/win32/direct3dhlsl/hlsl-shader-model-6-0-features-for-direct3d-12) we actually get access to this hardware behavior, allowing us to read values from neighboring lanes within the current pixel quad.

Sweet! Let's do it and get the values that are in the other lanes.

*NOTE: QuadReadLaneAt and it's sibling functions are apart of [Shader Model 6](https://learn.microsoft.com/en-us/windows/win32/direct3dhlsl/hlsl-shader-model-6-0-features-for-direct3d-12), which is essentially hardware that supports DX12, and for the most part this game is already on DX12.*

```HLSL
//NOTE: after the if(checkerboardTest) block we do a quad read

//read other pixel lanes, there should be a shadow in one of them
float lane0 = QuadReadLaneAt(contactShadow, 0);
float lane1 = QuadReadLaneAt(contactShadow, 1);
float lane2 = QuadReadLaneAt(contactShadow, 2);
float lane3 = QuadReadLaneAt(contactShadow, 3);
```

Each lane contains the value of contactShadow from one of the four pixels in the current 2×2 quad. This means a pixel can directly access results that were calculated by its neighbors without requiring any additional texture lookups or passes. In this case with the checkerboard pattern we already have, which is in a 2x2 pattern already, turns out this a perfect use case for these functions!

Now because all four pixels belong to the same quad, the skipped pixels can read the results from the pixels that performed the raymarch, and we can reconstruct a pretty decent reasonable approximation of the missing shadow information.

That sounds great! Let's try it out!

```HLSL
//NOTE: another conditional after the quad reads
if(!checkerboardTest)
{
    //reconstruction style 2:
    //this is simple, if any of the pixels are 0 (shadow), USE IT!
    contactShadow = min(min(lane0, lane1), min(lane2, lane3));

    //reconstruction style 2:
    //averaging all 4 lane values (and pumping contrast a bit)
    //the apperance is softer than doing min
    contactShadow = mad(lane0 + lane1 + lane2 + lane3, 0.5f, -1.0f);
}
```

<p float="left">
    <img src="content/checkerboard-on-min-reconstruction.png" width="49%" />
    <img src="content/checkerboard-on-avg-reconstruction.png" width="49%" />
</p>

*Left: Checkerboard with Min Reconstruction | Right: Checkerboard with Averaged Reconstruction*

Doesn't look too bad actually! Considering also that we are using TAA later in the pipeline *(and the noise changes every frame, same with the checkerboard pattern)* during motion this should resolve into something that looks pretty clean. It won't be as good as the full per-pixel quality without checkerboarding, but this isn't that big of a degredation either, now is it worth it though?

Here are the final timings now...

<p float="left">
    <img src="content/timing-checkerboard.png" width="100%" />
</p>

Roughly ```0.43ms```, not as big as I was expecting. It's definetly a net positive for performance but it's only a small improvement. Like I said earlier if you really want to squeeze every ounce out of this this is something that you can do that doesn't mangle your final effect too bad.

## Final Results

With that we have sucessfully managed to implement contact shadows into FF7 Rebirth!

For showcase let's show the raw light attenuation term that the game was originally using, and progressively add the techniques that we introduced.

<p float="left">
    <img src="content/original-attenuation.jpg" width="100%" />
</p>

*Light Attenuation: Shadowmaps * NdotL (Original)*

<p float="left">
    <img src="content/attenuation-micro.jpg" width="100%" />
</p>

*Light Attenuation: Shadowmaps * NdotL * Micro Shadows*

<p float="left">
    <img src="content/attenuation-micro-contact.jpg" width="100%" />
</p>

*Light Attenuation: Shadowmaps * NdotL * Micro Shadows * Contact Shadows*

Now with our final optimized results...

<p float="left">
    <img src="content/contact-shadows-optimized-result.jpg" width="100%" />
</p>

It's looking better than ever! Now I do want to point out something very important, because in deferred rendering lights are shaded in full screen pixel draws usually. It means every light is shaded this way. So contact shadows can also be used for local lights for a similar cost as well to the directional lights, and arguably in some areas within rebirth this is the one place that would definetly give you the most bang for your buck.

<p float="left">
    <img src="content/local-light-original.jpg" width="49%" />
    <img src="content/local-light-contacts.jpg" width="49%" />
</p>

*Left: Original | Right: Micro Shadows + Contact Shadowing for local Light passes (point/spot/area lights)*

## Video Preview

Now this was done just through RenderDoc but there is a whole other part of this story...

I built a Shader Injector mod, that allows me to actually take this modified RenderDoc shader, compile it and replace the original shader bytecode at runtime within the actual game so I could play with this modified shader during gameplay... and it's awesome!

[Shader Injector Mod Preview Video](https://www.youtube.com/watch?v=ta1WpIeoP1s)

## Timings on 1920x1080 and 3840x2160

This is an extra bonus but I wanted to see how the timings scaled when going from native 3840x2160 that we were working with, down to 1920x1080. Again using an RTX 3080, playing in the same game area.

***Frame Budget Guide***
- 30 FPS Frame-Time Budget: 33.33ms
- 60 FPS Frame-Time Budget: 16.67ms
- 120 FPS Frame-Time Budget: 8.33ms

First looking at the total frametime of the game, the graphics settings are maxed out but the resolution is now native 1920x1080 instead of 3840x2160 like before.

<p float="left">
    <img src="content/timing-original-full-1080.png" width="100%" />
</p>

Roughly ```14.23ms```, which is accurate. When dropping the game down to 1080 the game runs much better and can hit a 60 FPS+ framerate, now let's check the timing on the original shader.

It's worth pointing out here that because things are much "faster" the milisecond timings values are much smaller, and given the fact that RenderDoc has quite a bit of noise when doing a replay timings, some of these values may not be 100% accurate. But anyway, lets check the original shader!

<p float="left">
    <img src="content/timing-original-shader-1080.png" width="100%" />
</p>

Roughly ```0.081ms```, that's pretty small, which is what you want, now let's plug in the reverse engineered shader with it's main effects *(micro shadows, contact shadows)* disabled...

<p float="left">
    <img src="content/timing-base-1080.png" width="100%" />
</p>

Roughly ```0.055ms```, just like we saw before the reverse engineered shader is lighter. I take this with a grain of salt as I'm sure there are some shading terms I'm missing, even though visually it looks almost identical. Now let's enable micro-shadows and contact-shadows *(the main effect)* and check the final timings...

<p float="left">
    <img src="content/timing-final-1080.png" width="100%" />
</p>

Roughly ```0.088ms```, yep it lines up with what we were seeing prior. Doing the quick subtraction math that means that at native resolution 1920x1080 the contact shadows we are adding is roughly ```0.033ms```, which is not bad at all!

For this frame we are still in excess of roughly ```2.5ms``` before we hit the 60 FPS frame-time budget. If one desired that is plenty of room to scale up the effect quality wise to get some more oompf!

# References / Sources
List of references that helped with my implementations and understanding.

- [Unreal Engine Contact Shadows documentation](https://dev.epicgames.com/documentation/unreal-engine/contact-shadows-in-unreal-engine?lang=en-US)

- [Unreal Engine CSM documentation](https://dev.epicgames.com/documentation/unreal-engine/use-cascaded-shadows?application_version=4.27)

- [Rendering Tiny Glades With Entirely Too Much Ray Marching](https://www.youtube.com/watch?v=jusWW2pPnA0)

- [Technical Art of Uncharted 4](https://advances.realtimerendering.com/other/2016/naughty_dog/)

- [Panos Karabelas / Screen space shadows](https://panoskarabelas.com/posts/screen_space_shadows/)

- [demofox / Interleaved Gradient Noise: A Different Kind of Low Discrepancy Sequence](https://blog.demofox.org/2022/01/01/interleaved-gradient-noise-a-different-kind-of-low-discrepancy-sequence/)

- [demofox / What the Heck is Blue Noise?](https://blog.demofox.org/2018/01/30/what-the-heck-is-blue-noise/)

- [RenderDoc](https://renderdoc.org/)

*By: David Matos*