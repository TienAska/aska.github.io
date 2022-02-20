# GPU Skinned Fur Instance in Unity

## 前言：

众所周知，目前对于毛发的模拟，最常见的就是Shell Based Fur，简单来说就是顶点沿法线方向外扩，形成堆叠的外壳，然后通过一张Noise作为毛发Pattern，镂空（Clip或Blend）形成毛发形状。基本原理和shading方法互联网上面有许多的介绍了，不在此赘述。本文的重点是介绍一个GPU Instance的支持Skinned Mesh的Unity实现方案。

## 局限：

Shell Based最显著的问题是面数成整数倍膨胀，在桌面端，通常一个比较不错的动物毛皮效果都采用Alpha Clip方式，为了避免断层至少需要100层。在移动端的话这完全是不现实的，实际情况中采用8-16层是较为合理的，但是会出现断层，因此只能采用Alpha Blend方式，同时这也避免了打断Early Z优化。

另一个问题是，如何去绘制Shell。桌面端的方法是预先生成，厚度、生长方向，都可控，把这些信息和层数一起写进VertexColor或者UV，一个Pass绘制，反正性能够容量大。而移动端的思路是多Pass，预先生成对于美术来说总是多一个步骤，即便是可以通过工具自动化，但是生长方向这些额外信息刷起来也麻烦，本来层数就不够多，实际效果中体现并不明显，文件体积也是成倍增长，因此完全没必要的。而且URP又默认不支持多Pass，要不用多个材质球实现多Pass，要不就用RenderFeature扩展管线，总之都是挺麻烦的。

## 方案：

面数高，Pass多，而且又是重复绘制的，这里明显就可以使用GPU Instance来进行优化。获取毛皮模型的Mesh，将层数写入Material Block，[使用DrawMeshInstanced绘制](https://www.notion.so/GPU-Skinned-Fur-Instance-in-Unity-2b6f2320dbcc43bc850218f92d43f6b7)。但是，Unity中skinned mesh renderer并不支持GPU Instance，虽然有些方案可以绕过限制，例如BakeMesh或者[Animation Instacing](https://www.notion.so/GPU-Skinned-Fur-Instance-in-Unity-2b6f2320dbcc43bc850218f92d43f6b7)，但是都有各自存在的问题。

[Unity在2021.2.0中新增了SkinnedMeshRenderer.GetVertexBuffer](https://www.notion.so/GPU-Skinned-Fur-Instance-in-Unity-2b6f2320dbcc43bc850218f92d43f6b7)，完美解决了多年以来的SkinnedMesh的Instance问题。因此，这个流程就是获取VertexBuffer并设置到Material中，使用DrawProcedual绘制，而SV_InstanceID则代表了当前的层数。

## 实现：

本方案使用Unity 2021.2.6f1。

首先，使用GetVertexBuffer()，获取蒙皮后的顶点Buffer，注意在这里获取的仅仅是位置、法线、切线属性，至于UV、Index需要从mesh中获取。

> For a mesh to be compatible with a [SkinnedMeshRenderer](https://docs.unity3d.com/2022.1/Documentation/ScriptReference/SkinnedMeshRenderer.html), it must have multiple vertex streams: one for deformed data (positions, normals, tangents), one for static data (colors and texture coordinates), and one for skinning data (blend weights and blend indices).
>  

将Buffer设置到material中。

![Untitled](https://github.com/TienAska/tienaska.github.io/raw/master/assets/images/GPU-Skinned-Fur-Instance-in-Unity/Untitled.png)

然后，进行绘制，为了方便控制，我将层数写入了Material，这里需要读出来作为Instance的数量。

![Untitled](https://github.com/TienAska/tienaska.github.io/raw/master/assets/images/GPU-Skinned-Fur-Instance-in-Unity/Untitled%201.png)

最后是Shader部分，因为使用的是Procedural绘制，我们只能用SV_VertexID和SV_InstanceID读取Buffer中的顶点。注意这里的Buffer只能是ByteAddressBuffer，因此需要手动去[解析](https://www.notion.so/GPU-Skinned-Fur-Instance-in-Unity-2b6f2320dbcc43bc850218f92d43f6b7)。

> DirectX 11 does not allow [Index](https://docs.unity3d.com/2022.1/Documentation/ScriptReference/GraphicsBuffer.Index.html) or ::Vertex buffers to also be [Structured](https://docs.unity3d.com/2022.1/Documentation/ScriptReference/GraphicsBuffer.Structured.html). For compute shader mesh data access with DirectX 11 compatibility, use [Raw](https://docs.unity3d.com/2022.1/Documentation/ScriptReference/GraphicsBuffer.Raw.html).
>  

![Untitled](https://github.com/TienAska/tienaska.github.io/raw/master/assets/images/GPU-Skinned-Fur-Instance-in-Unity/Untitled%202.png)

注意：因为用的是Procedural绘制，并不需要开启#pragma multi_compile_instancing。因此UNITY_ANY_INSTANCING_ENABLED是未定义的，无需使用UnityInstancing中的宏。

## 效果：

![Untitled](https://github.com/TienAska/tienaska.github.io/raw/master/assets/images/GPU-Skinned-Fur-Instance-in-Unity/Untitled%203.png)

最终通过一个Pass即可实现任意层数的Shell Fur Shading。不需要预先生成网格，可以动态调整层数，支持Skinned Mesh GPU Instance，这些都只需要一个DrawCall。

## 扩展：

因为获取的是蒙皮后的vertex buffer，除了用于做Animation Instance，还可以混合[蒙皮动画](https://www.youtube.com/watch?v=IDRDA-Q8LuY)。

[2021.2.0](https://docs.unity3d.com/2022.1/Documentation/ScriptReference/SkinnedMeshRenderer.html)不仅提供了蒙皮后的顶点缓存，还提供了前一帧的缓存。

![Untitled](https://github.com/TienAska/tienaska.github.io/raw/master/assets/images/GPU-Skinned-Fur-Instance-in-Unity/Untitled%204.png)

基于此可以在CS中模拟出毛皮的物理效果，通过DrawProceduralIndirect，还可以根据距离设置层数，实现自动LOD。

## 引用：

[Bump Noise Cloud - 3D噪点+GPU instancing制作基于模型的体积云](http://walkingfat.com/bump-noise-cloud-3d%e5%99%aa%e7%82%b9gpu-instancing%e5%88%b6%e4%bd%9c%e5%9f%ba%e4%ba%8e%e6%a8%a1%e5%9e%8b%e7%9a%84%e4%bd%93%e7%a7%af%e4%ba%91/)

[Render- Animation Instancing](https://zhuanlan.zhihu.com/p/36896547)

[Unity 2021.2.0 - Unity](https://unity3d.com/unity/whats-new/2021.2.0#:~:text=between%20color%20spaces.-,Graphics,-%3A%20You%20can%20now)

[MinimalCompute/SkinnedMeshBuffer.shader at master · cinight/MinimalCompute](https://github.com/cinight/MinimalCompute/blob/master/Assets/SkinnedMeshBuffer/SkinnedMeshBuffer.shader)
