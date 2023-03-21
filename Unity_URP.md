

### **SRP 中的部分定义**

###### **Render Pipeline **

说明：决定了场景中哪些对象需要现实

 分为三个步骤：

1.  **culing**：筛选出需要渲染的对象，最好是对相机可见的对象 (截体剔除)和未被其他对象遮挡(遮挡剔除) 。
2. **rendering**：将这些具有正确照明和某些属性的对象绘制到基于像素的缓冲区中。
3. **post-processing**：在buffers上进行后处理操作。

<img src="https://p.ipic.vip/xxx5uc.png" alt="image-20230321153055828" style="zoom:50%;" /> 

###### **Shader**

说明：运行在GPU上的程序

###### **Direct lighting** 直接光

说明：指的是来自自发光光源(如灯泡)的照明，而不是光从表面反射的结果。根据光源的大小和它到接收器的距离，这样的照明通常会产生清晰明显的阴影

notice：与方向光（directional lighting）的区别，方向光没有距离上的衰减。

###### **Indirect lighting** 间接光

说明：间接照明是由于光从表面反射，并通过介质(如大气或半透明材料)传输和散射而产生的。在这种情况下，遮挡器通常会投射出柔和或难以分辨的阴影。

###### **Global illumination(GI)**

说明：是一组模拟直接和间接照明的技术，以提供真实的照明结果。

GI有几种方法，比如烘烤/动态光照图 、辐照度体积、光传播体积、烘烤/动态光探针、基于体素的GI和基于距离场的GI。开箱即用，Unity支持烘烤/动态光映射和光探针。

###### **Lightmapper**

说明：它通过发射光线、计算光线反弹并将结果照明应用到纹理中来生成光线映射和光探针的数据。因此，不同的灯光映射器通常会产生不同的灯光外观，因为它们可能依赖不同的技术来产生灯光数据。

###### ScriptableRenderContext

说明：充当c#渲染管道代码和Unity低级图形代码之间的接口。

###### FilteringSettings

说明：描述了如何过滤剔除结果

###### DrawingSettings

说明：用于描述绘制哪个几何图形以及如何绘制它

###### 光照选择流程

<img src="https://p.ipic.vip/m9k16g.png" alt="image-20230321155949335" style="zoom:50%;" />

###### PipelineAsset

<img src="https://p.ipic.vip/ez718e.png" alt="image-20230321141939101" style="zoom:50%;" />

| 参数                  | 说明                                                         |
| :-------------------- | :----------------------------------------------------------- |
| **Render  Scale**     | 此滑动条用于缩放渲染目标分辨率（而不是当前设备的分辨率）。如果出于性能原因要以较小的分辨率进行渲染或需要升级渲染来提高质量，请使用此属性。这只会缩放游戏渲染。UI 渲染保留采用设备的原始分辨率。 |
| **Shadow.Depth Bias** | 使用此设置可减轻[阴影暗斑](https://docs.unity3d.com/Manual/ShadowPerformance.html)。[说明](https://learn.microsoft.com/en-us/windows/win32/direct3d11/d3d10-graphics-programming-guide-output-merger-stage-depth-bias) |
| **SRP Batcher**       | 选中此复选框可启用 SRP Batcher。如果有许多不同材质使用相同的着色器，这将非常有用。SRP Batcher 是一个内部循环，可以在不影响 GPU 性能的情况下加速 CPU 渲染。使用 SRP Batcher 时，它将替代 SRP 渲染代码内部循环。 |
| **Dynamic Batching**  | 启用 [Dynamic Batching](https://docs.unity3d.com/Manual/DrawCallBatching.html) 可以使渲染管线自动批处理一系列共享相同材质的小动态对象。这对于不支持 GPU 实例化的平台和图形 API 非常有用。如果目标硬件确实支持 GPU 实例化，请禁用 **Dynamic Batching**。可以在运行时更改此设置。 |



###### Universal Renderer Data

<img src="https://p.ipic.vip/5v0ase.png" alt="image-20230321141827208" style="zoom:50%;" />

|                       |                                                              |
| --------------------- | ------------------------------------------------------------ |
| **Native RenderPass** | 指示是否使用 URP 的 Native RenderPass API。启用此属性后，URP 使用此 API 来构建渲染通道。因此，可以在自定义 URP 着色器中使用[可编程混合](https://docs.unity3d.com/Manual/SL-PlatformDifferences.html#using-shader-framebuffer-fetch)。有关 RenderPass API 的更多信息，请参阅 [ScriptableRenderContext.BeginRenderPass](https://docs.unity3d.com/ScriptReference/Rendering.ScriptableRenderContext.BeginRenderPass.html)。  **注意**：启用此属性对 OpenGL ES 没有影响。 |



### 代码

- **RenderPipelineManager**

  ```c#
  DoRenderLoop_Internal(){……}//当前的pipeline.Render()
  ```

- **UniversalRenderPipeline**（**RenderPipeline**）

  ```C#
  /*
  		Culing:	context.Cull
  		Rendeing:
  						renderer.Setup(context, ref renderingData);
      				renderer.Execute(context, ref renderingData);
  */
  Render(ScriptableRenderContext renderContext, List<Camera> cameras)
  {
      ……
      //根据相机的depath 排序
      SortCameras(cameras);
      //渲染CameraStack中camera,对所有相机 RenderSingleCamera
      RenderCameraStack(ScriptableRenderContext context, Camera baseCamera)
  }
  
  //剪裁、设置、执行渲染
  RenderSingleCamera(ScriptableRenderContext context, Camera camera)
  {
      Camera camera = cameraData.camera;
    	//renderer is ScriptableRenderer
      var renderer = cameraData.renderer;
      //省略部分代码
      CommandBuffer cmd = CommandBufferPool.Get();
  	  //设置剪裁参数
      renderer.SetupCullingParameters(ref cullingParameters, ref cameraData);
      //提交执行渲染命令
      context.ExecuteCommandBuffer(cmd); 
      //进行裁剪
      var cullResults = context.Cull(ref cullingParameters);
      //对lighting,shadows,postprocessing 设置
      InitializeRenderingData(asset, ref cameraData, ref cullResults, anyPostProcessingEnabled, out var 		  renderingData);
      //每帧执行，设置渲染参数 
      renderer.Setup(context, ref renderingData);
      //执行渲染
      renderer.Execute(context, ref renderingData);
      CleanupLightData(ref renderingData.lightData);
     // Sends to ScriptableRenderContext all the commands enqueued since cmd.Clear, i.e the "EndSample" command
      context.ExecuteCommandBuffer(cmd); 
      CommandBufferPool.Release(cmd);
      context.Submit();
  }
  
  
  ```

- **ScriptableRenderer**

  ​     

  ```c#
  //执行渲染 
  public void Execute(ScriptableRenderContext context, ref RenderingData renderingData)
  {
  	//pass.OnCameraSetup(cmd, ref renderingData);
    //在渲染camera之前执行，
    InternalStartRendering(context, ref renderingData);
    //重置每个相机着色器的关键字
    //省略代码
  	//对pass 进行排序
    SortStable(m_ActiveRenderPassQueue);
    
     //依据 RenderPassEvent 进行pass数据整理
     using var renderBlocks = new RenderBlocks(m_ActiveRenderPassQueue);
     //设置光照参数
     SetupLights(context, ref renderingData);
     //省略
  
     //渲染不透明 Opaque blocks...
     if (renderBlocks.GetLength(RenderPassBlock.MainRenderingOpaque) > 0)
     {
        // TODO: Separate command buffers per pass break the profiling scope order/hierarchy.
        // If a single buffer is used (passed as a param) for passes,
        // put all of the "block" scopes back into the command buffer. (i.e. null -> cmd)
        using var profScope = new ProfilingScope(null, Profiling.RenderBlock.mainRenderingOpaque);
        ExecuteBlock(RenderPassBlock.MainRenderingOpaque, in renderBlocks, context, ref renderingData);
      }
      
     //渲染透明Transparent blocks...
     if (renderBlocks.GetLength(RenderPassBlock.MainRenderingTransparent) > 0)
     {
          using var profScope = new ProfilingScope(null, Profiling.RenderBlock.mainRenderingTransparent);
        ExecuteBlock(RenderPassBlock.MainRenderingTransparent, in renderBlocks, context, ref renderingData);
     } 
     //渲染绘图发生后，例如后处理，视频播放器捕获。
     if (renderBlocks.GetLength(RenderPassBlock.AfterRendering) > 0)
     {
         using var profScope = new ProfilingScope(null, Profiling.RenderBlock.afterRendering);
         ExecuteBlock(RenderPassBlock.AfterRendering, in renderBlocks, context, ref renderingData);
     }
     //释放在render过程中创建的资源
     InternalFinishRendering(context, cameraData.resolveFinalTarget);
  
     //提交执行渲染命令
     context.ExecuteCommandBuffer(cmd);
     CommandBufferPool.Release(cmd);
   } //同一个渲染队列的pass 执行渲染
   void ExecuteBlock(int blockIndex, in RenderBlocks renderBlocks,ScriptableRenderContext context, ref    									RenderingData renderingData, bool submit = false)
   {
     //省略代码……
     ExecuteRenderPass();
   }
   void ExecuteRenderPass(ScriptableRenderContext context, ScriptableRenderPass renderPass,ref         RenderingData renderingData)
   {
       //nativeRenderPass 不支持 OpenGL ES.
       //ConfigureNativeRenderPass(cmd, renderPass, cameraData);
       //设置参数
     	 renderPass.Configure(cmd, cameraData.cameraTargetDescriptor);
       //执行渲染
     	 //ExecuteNativeRenderPass(context, renderPass, cameraData, ref renderingData);
       renderPass.Execute(context, ref renderingData);
   }
  ```

- **ScriptableRendererFeature**

  ```C#
  //pass的管理类 
  public abstract class ScriptableRendererFeature : ScriptableObject, IDisposable
  {
    //pass初始化设置
     public abstract void Create();
     //主要是把pass添加到渲染的queue中
     public abstract void AddRenderPasses(ScriptableRenderer renderer, ref RenderingData renderingData);
    
     //省略其它代码
  }
  ```

  

- **ScriptableRenderPass**

  ```c#
  //pass 实现类
  public abstract partial class ScriptableRenderPass
  {
     //为这个pass 设置渲染对象
     public void ConfigureTarget(RenderTargetIdentifier[] colorAttachments, RenderTargetIdentifier depthAttachment){ }
     
     public virtual void Configure(CommandBuffer cmd, RenderTextureDescriptor cameraTextureDescriptor){}
   
     public abstract void Execute(ScriptableRenderContext context, ref RenderingData renderingData);
    
     //省略其它代码
  }
  ```

### Shader

```
Simple Lit
```

```HLSL
Unlit
```

```HLSL
Lit
```