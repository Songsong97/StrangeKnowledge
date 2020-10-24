quick notes for ue4 source code walk through

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

ue的一些类实现了AddRef()和Release()，并且在构造函数中调用AddRef()，如此便统计了对象被引用的次数，用于Garbage Collection。

[这里](https://docs.unrealengine.com/en-US/Programming/Rendering/ThreadedRendering/index.html)给出了更多有关两个线程之间协作的参考。

ProcessTasksNamedThread可以在不同的线程中执行，例如GameThread和RenderThread

### TODO
看RenderViewFamily_RenderThread的render thread是不是主线程开启的

what is a hit proxy?


