## URP14+ 获取深度+法线贴图 (_CameraDepthNormalsTexture) 的姿势？

叠甲部分：本人也才刚学习 Shader 没多久，网上关于 URP 高版本的资料又少，很多都是东拼西凑，各种踩坑零碎的使出来的。大部分原理细节部分可能不一定对甚至漏洞百出，也可能不严谨。希望如果有真的知道的大佬可以改正纠错一下，真的非常非常感谢！！！大家一起共同学习！

---

事出有因，在学习 《Unity Shader 入门精要》的时候，本着与现代接轨的想法，使用 URP 改写本书中的各个案例。但是在URP中获取深度法线贴图的时候遇到极大的困难，先简单写一下一种网上常见的思路：

1. 开启默认渲染管线的 **SSAO** ，将源设置为 **Depth+Normal**，之后即可获取 `_CameraDepthTexture` 和 `_CameraNormalTexture`，不过需要注意的是这有2个需要注意的地方：

   第一个是 **SSAO** 只会对拥有 `"LightMode" = "DepthNormals"` Pass 材质所渲染的物体起作用（但根据实验显示貌似只有 `Lit` 材质才能显示出正确结果，其他的就算有了 DepthNormals Pass，渲染时也会出点问题，比如不知道为什么法线直接变白了）。

   第二个就是 `_CameraNormalTexture` 采样出来的值直接就是 `[-1, 1]` ，并没有经过映射。

---

其实注意到上面方法就已经存在诸多局限性，因此我们打算还是自己实现一个获取深度+法线贴图的轮子？这里我参考了诸多大佬的思路，但在 URP14+ 上亲自实现的时候遇到了 **非常非常非常多的坑**，不禁一边感叹新资料少的可怜，又没办法。

参考思路来源：

[【Unity UPR】造个获取深度法线纹理的轮子_urp获取buffer中的法线-CSDN博客](https://blog.csdn.net/qq_41835314/article/details/130151062)

[urp管线的自学hlsl之路 第二十四篇 科幻扫描效果后篇 - 哔哩哔哩](https://www.bilibili.com/opus/409222774573820232)

[URP深度法线纹理 - 简书](https://www.jianshu.com/p/6c75bb64e8a0)

这里我就会从零开始写一个深度+法线贴图的轮子，并且避开诸多的坑和踩雷点。

先说一下大体思路，我们最后希望我们可以和在 `Built-in` 渲染管线里一样方便的使用类似于 `_CameraDepthNormalsTexture` 的全局纹理，因此我们直接利用 `Built-in` 的内置着色器来帮我们干这个事，该着色器位置处于 `Internal-DepthNormalsTexture` ,可以在 [Unity-Built-in-Shaders/DefaultResourcesExtra/Internal-DepthNormalsTexture.shader at master · TwoTailsGames/Unity-Built-in-Shaders](https://github.com/TwoTailsGames/Unity-Built-in-Shaders/blob/master/DefaultResourcesExtra/Internal-DepthNormalsTexture.shader) 翻阅到该 Shader 的源码，具体原理会在笔记末尾进行补充。

首先我们既然要在渲染的过程中插入一个我们自己的渲染点，那势必需要借助 `ScriptableRendererFeature` 这个东西，它允许我们定制和扩展我们自己的渲染管线，这样我们就可以随意所欲插入我们自己的自定义渲染 Pass，写法如下：

```c#
using System;
using Unity.VisualScripting;
using UnityEngine;
using UnityEngine.Rendering;
using UnityEngine.Rendering.Universal;

namespace qjklw.Core
{
    public class DepthNormalsFeature : ScriptableRendererFeature
    {
        // 我们的自定义 Pass
        private DepthNormalsPass _depthNormalsPass;

        // 渲染时所有的材质
        private Material _depthNormalsMaterial;

        // 给pass传递变量，并加入渲染管线中
        public override void Create() {
            // 关键点: 使用 Built-in 内置的 深度+法线贴图 Shader
            _depthNormalsMaterial = CoreUtils.CreateEngineMaterial("Hidden/Internal-DepthNormalsTexture");
            /*
            	构造函数我们传入了 渲染队列范围(RenderQueueRange.opaque), 和渲染层级
            	这是给 FilteringSettings 使用的, 它决定了我们那个 Pass 需要渲染哪些对象
            	
            	RenderQueueRange.opaque 表示只筛选所有不透明的物体
            	layerMask 为 -1 则是渲染所有 layer, 我们也可以指定层级
            */
            _depthNormalsPass = new DepthNormalsPass(RenderQueueRange.opaque, -1, _depthNormalsMaterial) {
                // 指定该 Pass 会在哪个阶段发生, 我们让它在 AfterRenderingPrePasses 发生, 因为深度和法线贴图
                // 数据是我们需要提前准备的有用的数据
                renderPassEvent = RenderPassEvent.AfterRenderingPrePasses
            };
        }

        public override void AddRenderPasses(ScriptableRenderer renderer, ref RenderingData renderingData) {
            // 我们拦截 UI 的渲染 (之后会重点解释)
            if (renderingData.cameraData.camera.cameraType == CameraType.Preview) return;
            renderer.EnqueuePass(_depthNormalsPass);
        }

        // 不要忘了记得销毁
        protected override void Dispose(bool disposing) {
            base.Dispose(disposing);
            if (_depthNormalsPass != null) CoreUtils.Destroy(_depthNormalsMaterial);
        }
    }
    
}
```

接着我们再来看 `DepthNormalsPass` 的内容：

```c#
using UnityEngine;
using UnityEngine.Experimental.Rendering;
using UnityEngine.Rendering;
using UnityEngine.Rendering.Universal;

namespace qjklw.Core
{
    public class DepthNormalsPass : ScriptableRenderPass
    {
        private Material _material; // 要渲染的材质
        private FilteringSettings _filteringSettings; // 渲染物体的筛选

        // 我们希望可以在分析器里看到这个 Pass 渲染的一些细节和分析, 因此我们需要借助
        // Profiler, 这里 _profilerTag 是我们给他定义的名字
        private static readonly string _profilerTag = "Depth Normals Pre Pass"; 

        /*
        	当我们对物体进行渲染的时候, 我们需要指定带有特定 "LightMode" = "DepthNormals" 的 Pass。
        	只有拥有这样的 Tags 的 Pass 才会得到渲染效果。该技术叫 "着色器替换", 感兴趣可以自行搜索
        */
        private ShaderTagId _shaderTagId = new("DepthNormals"); 

        // 分析采样器
        private ProfilingSampler _profilingSampler;

        /*
        	RTHandle 实质是对一个 RenderTexture 的包装器, 它提供了更好的生命周期管理, 以及支持动态分辨率等等
        */
        private RTHandle _depthNormalsRT;
        private RTHandle _depthRT;
        private readonly string _depthNormalsTextureName = "_CameraDepthNormalsTexture";
        private readonly string _depthTextureName = "_CameraDepthTexture_Q";

        private RTHandle _lazyRThandle;
        
        public DepthNormalsPass(RenderQueueRange renderQueueRange, LayerMask layerMask, Material material) {
            // 渲染对象筛选器
            _filteringSettings = new FilteringSettings(renderQueueRange, layerMask);
            _material = material;
            // 初始化分析器
            _profilingSampler = new ProfilingSampler("Depth-Normals Render");
            
            /*
            	注意这里的 Alloc 在当你没有明确给出一些细节数据例如, RenderTexture 的宽高时，这里的 Alloc
            	只是起到一个类似 "声明" 的作用, 它的内部所维护的 rt (renderTexture) 并不会实质性的分配到内存
            	拥有内存的只是 RTHandle 这个包装器本身
            */
            _depthNormalsRT = RTHandles.Alloc(_depthNormalsTextureName, name: _depthNormalsTextureName);
            _depthRT = RTHandles.Alloc(_depthTextureName, name: _depthTextureName);
        }
        
        // 在渲染 (Execute) 之前进行的一些初始化和配置操作写在这里
        public override void Configure(CommandBuffer cmd, RenderTextureDescriptor cameraTextureDescriptor) {
            // 渲染需要用的一些数据
            var descriptor = cameraTextureDescriptor;
            // 这里我们必须让它的深度缓冲区位数归 0, 不然的话渲染的时候我们的 _CameraDepthNormalsTexture 会被
            // 当成一张单纯的深度贴图, 只会写入深度信息, 这里我们不给它深度缓存, 让它只写入颜色缓冲中, 这样一来
            // 就会获得 深度+法线 的双重数据了
            descriptor.depthBufferBits = 0;	
            descriptor.colorFormat = RenderTextureFormat.ARGB32;
            descriptor.msaaSamples = 1;
            
            // 但注意这里必须颜色和深度分开放, 如果没有深度缓冲会出现深度排序错误，我们可以把这里的深度放在
            // 另一个 RTHandle 中, 在帧调试器中就可以看到每一个过程除了 "RT0" 的颜色缓存, 还有一个单独
            // 的 "深度", 里面存放的就深度缓冲区的深度贴图, 我们也用类似的做法
            var descriptorDepth = cameraTextureDescriptor;
            descriptorDepth.depthBufferBits = 32;
            descriptorDepth.colorFormat = RenderTextureFormat.ARGB32;
            descriptorDepth.msaaSamples = 1;
            
            // 我们先用 _lazyRThandle 提前记录一下 _depthNormalsRT, 下面解释了为什么
            _lazyRThandle = _depthNormalsRT;
            /*
            	这里的 ReAllocateIfNeeded 有一个很大的坑点, 这个函数在每次进行重分配之前, 会事先检查
            	当前的 RTHandle 里的 rt 数据是不是和已经有的一样, 如果符合它的标准, 它会判断为 "不需要再分配"
            	因此 rt 就会依然沿用旧的, 这也是为什么函数名叫 ReAllocate If Needed (如果需要才重分配)
            	
            	它的返回结果就是是否进行了重分配, 因此我们提前用 _lazyRThandle 记录下 _depthNormalsRT 的
            	引用, 如果重分配成功, 此时的 _depthNormalsRT 已经有了新的内存,我们再释放 _lazyRThandle 的数据, 
            	也就是旧的 RTHandle 的数据。
            	
            	注意虽然 RTHandle 本身没有对象引用之后, 会被 C# 的垃圾
            	回收机制干掉, 但 RTHandle 里面维护的 RenderTexture 属于 GPU 资源, 是需要我们手动释放的
            	不然的话就等着电脑死机吧 (
            */
            if (RenderingUtils.ReAllocateIfNeeded(ref _depthNormalsRT, descriptor, name: _depthNormalsTextureName)) {
                _lazyRThandle?.Release();
            }
            
            // 深度也做一样的处理
            _lazyRThandle = _depthRT;
            if (RenderingUtils.ReAllocateIfNeeded(ref _depthRT, descriptorDepth, name: _depthTextureName)) {
                _lazyRThandle?.Release();
            }
            
            // 配置我们当前所渲染到的颜色对象和深度对象
            ConfigureTarget(_depthNormalsRT, _depthRT);
            // 在每帧渲染之前, 我们选择用黑色先清空所有的数据
            ConfigureClear(ClearFlag.All, Color.black);
        }

		// 真正的渲染时的操作
        public override void Execute(ScriptableRenderContext context, ref RenderingData renderingData) {
            if (_material == null) return;
            
            // 从命令缓存池获取一个命令缓存对象, 命令缓存本质就是一个存放命令的数组, 我们把他写进去
            // 最后让 CPU 将这些数据发送到 GPU 进行执行
            var cmd = CommandBufferPool.Get(_profilerTag);
            context.ExecuteCommandBuffer(cmd);
            cmd.Clear();

            // 在分析器中进行操作
            using (new ProfilingScope(cmd, _profilingSampler)) {
                // 定义渲染的顺序, 我们采取默认的不透明物体的渲染顺序, 也就是根据深度排序
                var sortFlags = renderingData.cameraData.defaultOpaqueSortFlags;
                // 渲染的一些配置数据
                var drawSettings = CreateDrawingSettings(_shaderTagId, ref renderingData, sortFlags);
                drawSettings.perObjectData = PerObjectData.None;
                // 定义我们需要覆盖的材质
                drawSettings.overrideMaterial = _material;
                
                // 进行绘制, renderingData.cullResults 是经过剔除后的对象, 剩下的都是传入数据就行
                context.DrawRenderers(renderingData.cullResults, ref drawSettings, ref _filteringSettings);
                
                // 最后不要忘了把渲染之后的结果, 设置为我们的 全局纹理
                cmd.SetGlobalTexture(_depthNormalsTextureName, _depthNormalsRT);
            }
            
            // 执行命令
            context.ExecuteCommandBuffer(cmd);
            // 记得销毁
            CommandBufferPool.Release(cmd);
        }
    }
}
```

那么我再说一下我遇到的坑，第一个坑是内存分配的。由于我一开始觉得 `_CameraDepthNormalsTexture` 这种算全局的持久化资源，因此我想着 Alloc 一次就行了。但我首先 Alloc 一次，再去使用，就发现我得到一个错误。它提示我用了一个空的 RenderTexture 去渲染，搜索相关资料才发现，RTHandle 如果没有给足够的信息那 Alloc 就仅仅会是一个声明，内部 rt 不会得到空间。因此我在每帧进行了 ReAllocate，但一开始由于粗心，并未进行 Release，起因是我觉得哪怕我不去管，由于自动回收，没有引用的旧对象也会自己销毁。事实确实如此，但是其内部的 rt 是需要我们手动释放的。

但是由于 ReAllocateIfNeed 会自动检测是否需要重分配的特殊机制，导致我一开始没发现这个问题，即使没释放内存也没炸。直到我每次在 ReAllocateIfNeed 之前，我总是先把它设置为一个新的 RTHandle ( 类似初始化 )，然后再 ReAllocateIfNeed，然后我电脑就......炸了。这就是因为 ReAllocateIfNeed 每帧都判断你是一个新的 RTHandle，和我已经有的 rt 差别很大，需要再分配。因此一直累计，半秒就可以卡出 4个多G 的内存。

ReAllocateIfNeed 参考源码：

```c#
public static bool ReAllocateIfNeeded(
    ref RTHandle handle,
    in RenderTextureDescriptor descriptor,
    FilterMode filterMode = FilterMode.Point,
    TextureWrapMode wrapMode = TextureWrapMode.Repeat,
    bool isShadowMap = false,
    int anisoLevel = 1,
    float mipMapBias = 0,
    string name = "")
{
    TextureDesc requestRTDesc = RTHandleResourcePool.CreateTextureDesc(descriptor, TextureSizeMode.Explicit, anisoLevel, 0, filterMode, wrapMode, name);
    // 可以清楚的看到先进行了判断是否需要重分配
    if (RTHandleNeedsReAlloc(handle, requestRTDesc, false))
    {
        if (handle != null && handle.rt != null)
        {
            TextureDesc currentRTDesc = RTHandleResourcePool.CreateTextureDesc(handle.rt.descriptor, TextureSizeMode.Explicit, handle.rt.anisoLevel, handle.rt.mipMapBias, handle.rt.filterMode, handle.rt.wrapMode, handle.name);
            AddStaleResourceToPoolOrRelease(currentRTDesc, handle);
        }

        if (UniversalRenderPipeline.s_RTHandlePool.TryGetResource(requestRTDesc, out handle))
        {
            return true;
        }
        else
        {
            handle = RTHandles.Alloc(descriptor, filterMode, wrapMode, isShadowMap, anisoLevel, mipMapBias, name);
            return true;
        }
    }
    return false;
}
```



所以一定要记得 Release 啊！但是紧接着下一个问题就来了。一开始我在 `OnCameraCleanup` 中进行释放，但最后缺导致了我编辑器的 UI 花屏，原因是因为其实 `CameraCleanup` 仅仅是在相机渲染完毕之后执行，如果此时我们把 RTHandle 提前释放，不要忘了我们才刚刚把它设置为 `Target`，就在 `Configure` 函数里。那么在这之后的渲染，就会因为我们提前释放了 Target 而出现错误。所以我最后改成了在 `Configure` 函数里的自动检测是否释放，如果需要释放上一帧，直到这一步其实就已经差不多了。

还有一点就是不要使用 `cmd.SetRenderTarget` 函数，虽然你要用也行，但是你最后会在帧调试器里发现，你的深度+法线贴图并没有写入 `_CameraDepthNormalsTexture` 里，而是写到别的地方去了。具体原因我也不太清楚，大概是 URP 在渲染的时候只认 `ConfigureTarget` 设置的渲染对象，就算你在命令缓存里再指定，也会被 URP 的某些默认行为所覆盖掉。

最后是这一句：

```c#
public override void AddRenderPasses(ScriptableRenderer renderer, ref RenderingData renderingData) {
    // 我们跳过 Preview 的渲染 (之后会解释)
    if (renderingData.cameraData.camera.cameraType == CameraType.Preview) return;
    renderer.EnqueuePass(_depthNormalsPass);
}
```

按照上述代码写的话，我们的程序已经可以自由的在场景视图和游戏视图都能渲染效果了 (可以自己写一个显示深度和法线的后处理进行察看)。但当我们在场景视图拖动 GameObject 或者点击 MainCamera 的时候会出现一个 MainCamera 的 Preview 小窗。这个时候渲染又会错误，这里原因也

没太搞清楚，但经过实验是 `ConfigureTarget` 的原因，它设置了渲染对象后貌似会影响到之后的渲染结果，但是像一些 `DrawOpaque` 这种 Pass 都会自动再指定 `_CameraColorAttachmentsA` 这种作为渲染对象，但有一些不会，所以就会影响到后续的渲染结果。因此我们想到就是如果在渲染

`Preview` 面板，我们就跳过当前 Pass，不让它进行渲染。直到这里，你就既可以在场景视图得到 `_CameraDepthNormalsTexture` 也可以在游戏视图得到它了，而且 UI 和 Preview 小窗渲染也不会出 BUG。

至于用法就和 《Unity Shader 入门精要》这本书里一样了，值得注意的是 `_CameraDepthNormalsTexture` 里的数据是压缩过的，我们需要解码函数，这里直接从 `Built-in` 里搬过来了：

```glsl
#ifndef DECODEDEPTHNORMAL_INCLUDE
#define DECODEDEPTHNORMAL_INCLUDE

inline float DecodeFloatRG(float2 enc)
{
    float2 kDecodeDot = float2(1.0, 1 / 255.0);
    return dot(enc, kDecodeDot);
}

inline float3 DecodeViewNormalStereo(float4 enc)
{
    float kScale = 1.7777;
    float3 nn = enc.xyz * float3(2 * kScale, 2 * kScale, 0) + float3(-kScale, -kScale, 1);
    float g = 2.0 / dot(nn.xyz, nn.xyz);
    
    float3 n;
    n.xy = g * nn.xy;
    n.z = g - 1;

    return n;
}

inline void DecodeDepthNormal(float4 enc, out float depth, out float3 normal)
{
    depth = DecodeFloatRG(enc.zw);
    normal = DecodeViewNormalStereo(enc);
}

#endif
```

---

补充一下 `Built-in` 的深度+法线贴图是如何做的：

```glsl
Pass {
    CGPROGRAM
    #pragma vertex vert
    #pragma fragment frag
    #include "UnityCG.cginc"
        
    struct v2f {
        float4 pos : SV_POSITION;
        float4 nz : TEXCOORD0;
        UNITY_VERTEX_OUTPUT_STEREO
    };
        
    v2f vert(appdata_base v) {
        v2f o;
        UNITY_SETUP_INSTANCE_ID(v);
        UNITY_INITIALIZE_VERTEX_OUTPUT_STEREO(o);
        o.pos = UnityObjectToClipPos(v.vertex);
        
        o.nz.xyz = COMPUTE_VIEW_NORMAL;	// 这里计算 view 空间下的 normal
        o.nz.w = COMPUTE_DEPTH_01;	    // 计算被约束到 [0,1] 的线性深度值
        return o;
    }
    
    fixed4 frag(v2f i) : SV_Target {
        // 编码函数, 进行了一个压缩
        return EncodeDepthNormal(i.nz.w, i.nz.xyz);
    }
    ENDCG
}
```

上面两个关键的宏定义如下：

```glsl
// 可以看到这里转化到了 view空间 下的深度值 (Z_view)
/*
	_ProjectionParams 的定义如下：

	// x = 1 or -1 (-1 if projection is flipped)
    // y = near plane
    // z = far plane
    // w = 1/far plane
    float4 _ProjectionParams;

	如果你了解过这里计算深度值的原理就会知道 Z_view / far 其实是做深度归一化
	正常得到的 Z_view 值会是 [Near, Far] 除以 Far, 我们保证最远的地方显示的颜色最亮, 也就是 1
	映射到了 [0,1], 至于取反是因为 view空间中，摄像机正对的z轴为负值。

	同时我们这里的拿到的深度值直接就是线性的了, 因为没有经过透视投影矩阵的压缩
*/
#define COMPUTE_DEPTH_01 -(UnityObjectToViewPos( v.vertex ).z * _ProjectionParams.w)

// 这里比较简单, 就是直接乘以一个 模型-视图矩阵的逆转置矩阵, 变换法线到了 view空间。
// 具体原理参考《Unity Shader入门精要》的 4.7 节
#define COMPUTE_VIEW_NORMAL normalize(mul((float3x3)UNITY_MATRIX_IT_MV, v.normal))
```

至于 `EncodeDepthNormal` 这里就不做原理解释了，但是我贴个源码，感兴趣的可以自己钻研一下：

```glsl
// 将 [0..1) 范围的浮点数编码为每通道 8 位的 RG 格式。请注意，数值 1.0 。
inline float2 EncodeFloatRG( float v )
{
    float2 kEncodeMul = float2(1.0, 255.0);
    float kEncodeBit = 1.0/255.0;
    float2 enc = kEncodeMul * v;
    enc = frac (enc);
    enc.x -= enc.y * kEncodeBit;
    return enc;
}

// 将 view空间法线 编码为 2D [0..1] 范围内的向量。
inline float2 EncodeViewNormalStereo( float3 n )
{
    float kScale = 1.7777;
    float2 enc;
    enc = n.xy / (n.z+1);
    enc /= kScale;
    enc = enc*0.5+0.5;
    return enc;
}

inline float4 EncodeDepthNormal( float depth, float3 normal )
{
    float4 enc;
    enc.xy = EncodeViewNormalStereo (normal);
    enc.zw = EncodeFloatRG (depth);
    return enc;
}
```

最后附赠一下结果，关于 URP14+ 自定义后处理可以参考这篇文章，写的非常好：

[Unity URP14.0 自定义后处理系统 - 知乎](https://zhuanlan.zhihu.com/p/621840900)

![image-20250413215601352](C:\Users\14518\AppData\Roaming\Typora\typora-user-images\image-20250413215601352.png)

![image-20250413215613326](C:\Users\14518\AppData\Roaming\Typora\typora-user-images\image-20250413215613326.png)

![image-20250413215633493](C:\Users\14518\AppData\Roaming\Typora\typora-user-images\image-20250413215633493.png)
