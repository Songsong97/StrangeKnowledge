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



### call stack
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

//TODO 看RenderViewFamily_RenderThread的render thread是不是主线程开启的
