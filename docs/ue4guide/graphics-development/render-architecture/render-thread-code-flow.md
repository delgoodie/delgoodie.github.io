---
sortIndex: 4
sidebar: ue4guide
---

# Threading Model

[Scalability for All: Unreal Engine 4 with Intel](https://www.slideshare.net/IntelSoftware/scalability-for-all-unreal-engine-4-with-intel) gives a good overview on the UE4 render arch

- **Render Thread vs. RHIThread vs. RenderTaskgraph threads**
-

# How Unreal Renders A Frame (~4.17): Great General high level overview breakdowns

<https://interplayoflight.wordpress.com/2017/10/25/how-unreal-renders-a-frame>

<https://interplayoflight.wordpress.com/2017/10/25/how-unreal-renders-a-frame-part-2>

<https://interplayoflight.wordpress.com/2017/10/25/how-unreal-renders-a-frame-part-3>

<https://medium.com/@lordned/unreal-engine-4-rendering-overview-part-1-c47f2da65346>

Render thread runs a frame or two behind

Game thread blocks at end of Tick() to allow render thread to catch up

On D3D12, there's a separate RHIThread that's just responsible for submitting to the driver

- On D3D11, the renderthread is what submits to the driver

Asynchronous communication between two threads through ENQUEUE RENDER CMD Macro

(basically a command buffer that render thread executes)

- CreateRenderState_Concurrent creates initial render state (not dynamic data). This happens on the Game Thread

- Can create fences

- Can do deferred deletion (BeginDestroy, FinishDestroy for objects derived from (FDeferredCleanupInterface). Necessary bc can't GC objects while render thread is still using them

- Gamethread objects own memory init & deletion of RHI resources, even though they're used in render thread

- Allows for asynchronous, centralized, deterministic memory allocation

- MarkRenderStateDirty()

Triggered by MarkRenderDynamicDataDirty() or MarkRenderStateDirty()

Ex: Dynamic Resource interaction with SkinnedMeshComponent

Gamethread

1. Initializes RHI

1. Overrides CreateRenderState_Concurrent so that it can create renderstate

   - SkeletalMeshObject manages sending skinned mesh bone xforms, vertex anim state, etc to the render thread

1. TickComponent() updates animations, calls MarkRenderDynamicDataDirty()

1. MarkRenderDynamicDataDirty() flags this component to be updated at the end of the frame on a thread (can override RequiresGameThreadEndOfFrameUpdates() to specify game thread)

1. At the end of the frame, a task job processor calls DoDeferredRenderUpdates_Concurrent() on each actor that needs updates

1. DoDeferredRenderUpdates_Concurrent ->

   - (Renderstate_Dirty)=>RecreateRenderState_Concurrent

   - (bRenderTransformDirty) => SendRenderTransform_Concurrent

   - (bRenderDynamicDataDirty) => SendRenderDynamicData_Concurrent

1. SkinnedMeshComponent::SendRenderDynamicData_Concurrent() then is responsible for sending updated render data to the render thread (through Enqueue_Unique_Render_Command)
   - Uses SkeletalMeshObject as the helper class to manage that
   - Payload is a struct of FDynamicSkelMeshObjectData

1. On Component detachment/destruction, game thread enqueues commands to release all RHI FRenderResources

   - a. DestroyRenderState_Concurrent () called when component is unregistered

   - b. FSkeletalMeshObject::ReleaseResources() is called to free up RHIs owned by this object
     - Calls BeginReleaseResource() on all RHI objects
     - Enqueues Render Command: Resource::ReleaseResource()
     - ReleaseResource() is responsible for deallocating the RHI Resource (e.g. calls VertexBuffer:ReleaseResource->ReleaseRHI/ReleaseDynamicRHI)

   - c. BeginCleanup(MeshObject) is called to do a deferred deletion of the object

   - d. After the render command buffer has been flushed, FSkeletalMeshObject::FinishCleanup() ( which is a delete this; for FSkeletalMeshObject)

Ex: Static Resource interaction with USkeletalMesh

1. On Component detachment/destruction, game thread enqueues commands to release all RHI FRenderResources

   a. GC Calls MeshObject::ReleaseResources(). This doesn't immediately destroy anything.

   1. Calls BeginReleaseResource() to enqueue command on render thread to release RHI resources (e.g. BeginReleaseResource(&ColorVertexBuffer);)

   1. Game thread creates a fence so it can continue so we don't block on render thread processing the delete

   1. GC calls USkeletalMesh::IsReadyForFinishDestroy() which checks if the fence is set or not

   1. GC calls UObject::FinishDestroy()

*Reference From <https://docs.unrealengine.com/latest/INT/Programming/Rendering/ThreadedRendering/index.html>*

FSceneView is per Eye scene view. Calculates offsets & HMD rotation

Depth Priority Groups are deprecated even though they don't show up that way in the API

PlayerController is where you handle keyboard inputs for map/game/hud

Creating HUD:

- Powerful but complicated: Derive AHUD. Ex: Shooter Game

- Blueprint way:

  The high level outline would be that your map needs a GameMode blueprint (see here) which will allow you to specify blueprints that will likely control your HUD--PlayerController Class and HUD Class.

Your PlayerController Class would ideally be where you want to handle any keyboard inputs for your map/game/HUD. For example, in Blueprint_HUD map, the BP_PlayerController_HUD blueprint is where we have our Input M to open the menu.

Your HUD Class is where all the magic happens with drawing your HUD. Event Receive Draw HUD is a Tick event, meaning it will fire off every frame of your game to draw your HUD given the instructions you connect to it. In the Blueprint_HUD map, our BP_HUD_Example has a branch early on that is toggled based on if the Menu should be drawn or not. If not, the main gameplay HUD is drawn, if yes, the Menu HUD is drawn (and gameplay HUD is not).

The HUD Class also contains the events for when you click the Hitboxes that are drawn by the Receive Draw HUD event.

Hopefully this enough to keep you going on working with Blueprint HUD. We're working hard to get more official documentation for both Blueprint HUD and C++ HUD and hope to be able to provide it soon!

*Reference From <https://wiki.unrealengine.com/Content_example_blueprint_HUD>*
