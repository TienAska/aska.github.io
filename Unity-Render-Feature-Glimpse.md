Unity Render Feature初探

众所周知，URP开放的Shading是单Pass的。默认的Pass需要写在UniversalForward的LightMode下。在过去的管线中，使用如果要渲染双面半透明的材质，通常是使用两个Pass，先渲染背面然后渲染正面。但是URP中如果想要这个效果的话，该怎么办呢？改管线当然是可行的，而且URP的逻辑也非常简单。但是为了这种特殊的效果，专门去改动管线，丧失了灵活性，并不是很好的解决方案。这里就要提出URP中的非常爽的功能——RenderFeature。

RenderFeature可以灵活的在渲染的各个阶段插入commandbuffer。这个插入点由RenderPassEvent决定，如下图所示：

![render pass events](https://github.com/TienAska/tienaska.github.io/raw/master/assets/images/render-pass-events.png)

可以说是非常的细了。

下面我就利用RenderFeature通过插入一个Pass来实现双面半透明材质。

首先是准备一个Shader，这里我直接复制了Lit.shader。然后在前面添加一个用于渲染背面的Pass。背面和正面的着色都是一样的，因此直接把UniversalForward复制过去，然后把Culling写死为Front。最后自己定义个LightMode，我这里写成BackFace。 
![render states](https://github.com/TienAska/tienaska.github.io/raw/master/assets/images/render-states.png)

不得不说，其实URP源码注释写的挺好的，很容易搞清楚渲染逻辑了。

接下来，创建一个RenderFeature。Project中Create->Rendering->Universal Render Pipeline->Renderer Feature。这里面已经有了模版。我们需要注意的是里面的Pass。Execute里面就是你写commandbuffer的地方。这里我再偷个懒，把DrawObjectPass中的代码复制过来。这个是渲染Universal Forward的Pass。背面和正面的逻辑是一样的，只是使用的Pass不同，因此将ShaderTagId改为&quot;BackFace&quot;。由于背面只适用于半透明的情况，所以RenderQueueRange设置为transparent。插入点RenderPassEvent设置为BeforeRendingTransparents可以保证先于半透明渲染。

![render feature pass](https://github.com/TienAska/tienaska.github.io/raw/master/assets/images/render-feature-pass.png)

这就是全部的逻辑了。

找到你的Forward Renderer，或者新建一个。在里面添加上面写好的RenderFeature。然后找到Render Pipeline Asset（在Project Setting的Graphics里面），在RendererList添加刚刚的Renderer。最后将相机的Renderer选择带有render feature的那个renderer，这样你就可以在Game窗口中看到效果啦。

![forward renderer](https://github.com/TienAska/tienaska.github.io/raw/master/assets/images/forward-renderer.png) ![renderer asset](https://github.com/TienAska/tienaska.github.io/raw/master/assets/images/renderer-asset.png) ![camera settins](https://github.com/TienAska/tienaska.github.io/raw/master/assets/images/camera-settings.png)

下面是效果对比，左边是强行设置渲染两面的半透明，右边是通过上诉方法实现多pass的半透明效果。 ![demo scene](https://github.com/TienAska/tienaska.github.io/raw/master/assets/images/demo-scene.png)

贴图来自乐乐女神的《Unity Shader入门精要》。可以发现左边由于正反面在一个Pass中渲染，顺序没有区分，导致前后关系紊乱。右边分离了正反面的渲染顺序，因此得出了正确的显示效果。

Render Feature的功能其实就是通过插入Pass给TA、图程更多的灵活性来扩展URP。URP虽然有些简陋，但是URP代码的逻辑、注释都挺清晰的。对于初学者来说是个非常的好的学习渲染管线的资源。URP推出这么久感觉并不是很火热。有些人看见和buildin rp的对比，发现缺了很多特性，就觉得它除了轻量一无是处，不愿意使用，其实通过RenderFeature以及改代码等方式，URP绝对能实现出比buildin rp更多、更好的效果。不要等官方来实现feature，Unity现在在3A道路上渐行渐远，我愿称之为有病。

最后展示一下RenderFeature 的另一个妙用。这个作用是在屏幕上显示depth texture。

![bonus](https://github.com/TienAska/tienaska.github.io/raw/master/assets/images/bonus.png) ![bonus shaders](https://github.com/TienAska/tienaska.github.io/raw/master/assets/images/bonus-shaders.png)

这里就是很常见的类似后处理的全屏Blit。Shader更简单就是把相关Buffer输出。

然后给相应的RenderFeature创建renderer并添加到pipeline中。这样就可以在camera中通过切换renderer来显示不同buffer啦。

![view depth](https://github.com/TienAska/tienaska.github.io/raw/master/assets/images/view-depth.png) ![view slice](https://github.com/TienAska/tienaska.github.io/raw/master/assets/images/view-slice.png) ![view worldpos](https://github.com/TienAska/tienaska.github.io/raw/master/assets/images/view-worldpos.png) ![view opaque](https://github.com/TienAska/tienaska.github.io/raw/master/assets/images/view-opaque.png)