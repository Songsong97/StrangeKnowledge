quick notes for ue4 source code walk through

### Useful Code Snippets
\UnrealEngine\Engine\Source\Runtime\RHI\Private\RHIGPUReadback.cpp **从GPU读回数据（例如Buffer, Texture）**
\UnrealEngine\Engine\Source\Runtime\Renderer\Private\DeferredShadingRenderer.cpp
\UnrealEngine\Engine\Source\Runtime\Core\Public\Misc\AssertionMacros.h **Unreal Engine 4 (UE4) provides three different families of assert equivalents: check, verify, and ensure**
\UnrealEngine\Engine\Source\Runtime\Renderer\Private\VelocityRendering.cpp **渲染物体速度，用于TemporalAA，MotionBlur等**
\UnrealEngine\Engine\Source\Runtime\Renderer\Private\PostProcess\PostProcessing.cpp
\UnrealEngine\Engine\Source\Runtime\RHI\Public\RHICommandList.h **Render Hardware Interface，很多Draw Call的接口**

### RenderDoc
一款非常实用的工具。可以查看API的Draw Event等。在我的Windows系统上，Unreal使用的图形API为D3D11。

在Unreal的代码中，```FRDGBuilder& GraphBuilder```提供了加入Render Pass的函数```AddPass()```。结合我们写的shader，就可以将渲染任务发送给GPU。我们可以传入```RDG_EVENT_NAME("Custom Shader")```作为Event Name，这样它就可以在RenderDoc中以Custom Shader的名字出现。我们可以查看其输出的Buffer或Texture。

### 表示渲染scope的class
```cpp
/**
 * Used as the scope for scene rendering functions.
 * It is initialized in the game thread by FSceneViewFamily::BeginRender, and then passed to the rendering thread.
 * The rendering thread calls Render(), and deletes the scene renderer when it returns.
 */
class FSceneRenderer { //... }

/**
 * Scene renderer that implements a deferred shading pipeline and associated features.
 */
class FDeferredShadingSceneRenderer : public FSceneRenderer { //... }

/**
 * Renderer that implements simple forward shading and associated features.
 */
class FMobileSceneRenderer : public FSceneRenderer { //... }
```



### Call Stack
```cpp
FDeferredShadingSceneRenderer::Render(FRHICommandListImmediate & RHICmdList);
RenderViewFamily_RenderThread(FRHICommandListImmediate & RHICmdList, FSceneRenderer * SceneRenderer);
void FRendererModule::BeginRenderingViewFamily(FCanvas* Canvas, FSceneViewFamily* ViewFamily)；
```

其中，
```cpp

/** Renders the view family. */
void FDeferredShadingSceneRenderer::Render(FRHICommandListImmediate& RHICmdList) { //... }

/**
 * Helper function performing actual work in render thread.
 *
 * @param SceneRenderer	Scene renderer to use for rendering.
 */
static void RenderViewFamily_RenderThread(FRHICommandListImmediate& RHICmdList, FSceneRenderer* SceneRenderer) { //... }
```
### 杂乱笔记

The primary method of communication between the two threads is through the ENQUEUE_UNIQUE_RENDER_COMMAND_XXXPARAMETER macro。这类似于FIFO的PIPE进行线程间的通信。

调用关系
```
void RenderViewFamily_RenderThread()
^
TEnqueueUniqueRenderCommandType::DoTask(ENamedThreads::Type CurrentThread, const FGraphEventRef& MyCompletionGraphEvent);
^
TGraphTask::ExecuteTask(TArray<FBaseGraphTask*>& NewTasks, ENamedThreads::Type CurrentThread) override
^
FNamedTaskThread::ProcessTasksNamedThread(int32 QueueIndex, bool bAllowStall)
```

ue的一些类实现了AddRef()和Release()，并且在smart pointer的构造函数中调用AddRef()，如此便统计了对象被引用的次数，用于Garbage Collection。

[这里](https://docs.unrealengine.com/en-US/Programming/Rendering/ThreadedRendering/index.html)给出了更多有关两个线程之间协作的参考。

ProcessTasksNamedThread可以在不同的线程中执行，例如GameThread和RenderThread

### TODO
看RenderViewFamily_RenderThread的render thread是不是主线程开启的

what is a hit proxy?


