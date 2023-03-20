### PipelineAsset

|   **Render  Scale**   | 此滑动条用于缩放渲染目标分辨率（而不是当前设备的分辨率）。如果出于性能原因要以较小的分辨率进行渲染或需要升级渲染来提高质量，请使用此属性。这只会缩放游戏渲染。UI 渲染保留采用设备的原始分辨率。 |
| :-------------------: | ------------------------------------------------------------ |
| **Shadow.Depth Bias** | 使用此设置可减轻[阴影暗斑](https://docs.unity3d.com/Manual/ShadowPerformance.html)。[说明](https://learn.microsoft.com/en-us/windows/win32/direct3d11/d3d10-graphics-programming-guide-output-merger-stage-depth-bias) |
|    **SRP Batcher**    | *选中此复选框可启用 SRP Batcher。如果有许多不同材质使用相同的着色器，这将非常有用。SRP Batcher 是一个内部循环，可以在不影响 GPU 性能的情况下加速 CPU 渲染。使用 SRP Batcher 时，它将替代 SRP 渲染代码内部循环。 |
| **Dynamic Batching**  | 启用 [Dynamic Batching](https://docs.unity3d.com/Manual/DrawCallBatching.html) 可以使渲染管线自动批处理一系列共享相同材质的小动态对象。这对于不支持 GPU 实例化的平台和图形 API 非常有用。如果目标硬件确实支持 GPU 实例化，请禁用 **Dynamic Batching**。可以在运行时更改此设置。 |



### Universal Renderer Data

| **Native RenderPass** | 指示是否使用 URP 的 Native RenderPass API。启用此属性后，URP 使用此 API 来构建渲染通道。因此，可以在自定义 URP 着色器中使用[可编程混合](https://docs.unity3d.com/Manual/SL-PlatformDifferences.html#using-shader-framebuffer-fetch)。有关 RenderPass API 的更多信息，请参阅 [ScriptableRenderContext.BeginRenderPass](https://docs.unity3d.com/ScriptReference/Rendering.ScriptableRenderContext.BeginRenderPass.html)。  **注意**：启用此属性对 OpenGL ES 没有影响。 |
| --------------------- | ------------------------------------------------------------ |
|                       |                                                              |



### 代码

- **RenderPipelineManager**

  ```
  DoRenderLoop_Internal
  ```

- **RenderPipeline**（**UniversalRenderPipeline**）

  ```C#
  Render(ScriptableRenderContext renderContext, List<Camera> cameras){……}
  RenderSingleCamera(ScriptableRenderContext context, Camera camera){……}
  ```

- **ScriptableRenderer**

- **ScriptableRendererFeature**

- **ScriptableRenderPass**