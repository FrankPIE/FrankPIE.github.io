---
layout: post
title: "(译)3D管线之旅：纹理采样器"
short_title: "(译)3D管线之旅4"
keywords: ["GPU", "3D管线", "译"]
description: "A trip through the Graphics Pipeline 2011 的翻译"
category: "Unreliable-Translate"
tags: ["GPU", "3D管线", "图形学", "硬件底层"]
---
{% include JB/setup %}

[原文：A trip through the Graphics Pipeline 2011（需要翻墙）](https://fgiesen.wordpress.com/2011/07/04/a-trip-through-the-graphics-pipeline-2011-part-4/)
--------------------------------------------

## 中文

欢迎大家回到这里，在上一节中我们讨论了顶点着色器，还有一些关于GPU着色器单元的简单知识。虽然通常来说它们只是矢量处理器，但是比起其他矢量架构处理器多了存储资源的能力，这就是纹理采样器（Texture samplers）。纹理采样器是GPU管线中不可缺少的部分，复杂（但也很有趣）到我们需要用单独的一篇文章来讨论它们。我已经等不及了，让我们开始吧。

### 纹理阶段

在我们开始了解真正的纹理操作之前，让我们先对API阶段的纹理操作有个完整的认识。在D3D11中，这主要由三个部分组成：

1. 采样阶段。设置滤波模式，设置寻址模式，设置最高等级各向异性等等都发生在这个阶段。这个阶段功能一般为控制纹理采样如何完成。

2. 基础纹理资源。归根到底就是一个指向内存中原始纹理数据的指针。这个资源指示了纹理是单一纹理还是纹理数组，指示了纹理的多重采样格式（如果有的话），还指示了纹理的物理布局等等。在这个阶段，内存中纹理的值还没有真正确切的被解释，但是内存布局已经确定了。

3. 着色器资源视图（简称SRV）。这是真正决定纹理数据如何被采样器解释的部分。在D3D10以后，资源视图连接基础纹理资源，所以不需要显式被指定。

大多数的时候，你会创建一个带有格式的纹理资源，比如说RGBA，其中每8位数据表示一个基本色，然后创建一个与之匹配的SRV。但是你也可以创建了一个“每8位表示一个基本色的缺省类型”然后匹配多个不同的SRV，底层的基础数据是完全一样的，只不过在创建SRV的时候使用了不同的格式（Format）。比如可以作为UNORM8_SRGB（sRGB空间的无符号8位数据映射在浮点数0～1之间）也可以作为UINT8（无符号八位整数）类型。

一开始看起来创建一个SRV是一个多余的步骤，但是SRV创建时允许API进行运行时检查。这意味着，如果你得到一个有效的SRV，那么就表示你的SRV和资源是兼容的，如果SRV存在，之后的所有操作不再需要进行类型检查，换句话说，这是API高效运行的根本原因。

无论如何，归根结底在硬件层次就是一堆和纹理采样操作相关的状态，采样阶段，选择使用纹理或者格式等等，当然这些状态需要被保持住（在第二章中我介绍了几种不同的方式去管理管线中的状态）。所以再提一次，保持状态有很多不同的方法，从“每次状态改变就刷新管线”到“采样器完全状态无关然后发送整个状态集合跟随每个纹理请求”，当然这之间还有许多别的可选方案。这不需要你来担心，这种事情硬件架构都需要进行严格的开销分析，模拟一些实际情况然后提出使用哪种方法，但是值得重复一下：作为一个PC程序员，不要想当然的以为硬件会采用某种固定的模式来工作。

不要想当然的以为纹理切换的代价是昂贵的，它们可能对于管线来说是状态无关的，所以并没有什么消耗。但是，也别想当然的以为它们是完全没有消耗的，也许，它们每次都会在纹理切换的时候切换多个纹理状态集合。除非你是在游戏主机上做引擎开发（或者对于每个硬件平台进行手动优化），要不然这根本没法给个万能的答案。所以当你开始优化你的程序时，切记对材质进行排序尽可能的避免没必要的状态改变，这肯定会节省你API的工作量，好了，我们就聊到这。不要对硬件会按照固定模式工作抱有幻想，因为在硬件的升级过程中可能会发生改变。

### 剖析一个纹理请求的过程
所以，一次纹理采样请求的过程中我们需要的信息有多少？这取决于纹理的类型和我们使用了哪种采样指令。现在，我们先以2D纹理为例，如果我们处理一个4x各项异性采样的2D纹理，我们需要什么信息？

* 2D纹理坐标---两个浮点数表示，为了坚持在这个系列文章中只使用D3D术语，我称它为UV而不是ST。

* UV在X轴方向上的偏导数：$$ \frac{\partial u} {\partial x}, \frac{\partial v}{\partial x} $$.

* 相似的，我们也需要在Y轴方向上的偏导数：$$ \frac{\partial u}{\partial y}, \frac{\partial v}{\partial y} $$.

所以，这个朴实的2D采样请求（或称为采样梯度变化）需要六个浮点数表示，这也许比你想的要多一些。4个梯度值被用来选择mipmap和各向异性过滤内核的尺寸和形状。你也可以使用纹理采样指令显式指定一个mipmap层级（在HLSL中，这被称为采样等级（SampleLevel）），这样就不需要梯度值了，只需要一个值包含LOD参数，但是这并不能作用于各向异性滤波，这最多只能做到三线性滤波！无论如何，让我们就暂时同意需要6个浮点数吧。这似乎看起来很多，但是我们真的需要在每个纹理请求中发送这么多数据嘛？

答案是：这个做法靠谱。除了像素着色器之外，其他阶段都是使用这样的做法，我们必须这么做（如果我们想要使用各向异性过滤掉的话）。在像素着色器中，事实证明我们不这么做。有个技巧可以允许像素着色器给你梯度指令（可以算出一些值然后向硬件请求“哪个是最近似屏幕空间梯度的值？”），这个做法也可以用于纹理采样器从二维坐标中获得偏导。所以对于一个PS 2D采样指令，假如你愿意在采样器单元中做更多一些的数学计算的话，你真的只需要发送一个UV坐标就OK了。

就是想刺激你们一下：最坏的情况下一次要传递多少个参数呢？在现在的D3D11管线中，最多的是体积纹理（Cubemap）数组。让我们来算一算：

* 3D 纹理坐标 --- u，v，w：3个浮点数。

* 体积纹理（Cubemap）数组索引: 一个int整数（我们当作和一个浮点数开销一样）。

* 在屏幕坐标系x，y方向上的梯度 $$(u，v，w)$$ ：6个浮点数。

对于每个像素采样十个值来说，你就需要存储40个字节的数据。现在，你可能觉得不需要完整的32位数据（这可能对于数组索引和梯度值来说太多了），但是依然还是有大量数据需要被到处传递。

让我们看看我们在讨论的采样需要哪种类型的带宽。就假设是我们最常用的2D纹理（体积纹理（Cubemaps）不会使用太多），最常见的纹理采样请求一般来自于像素着色器，只会有少量纹理采样发生在顶点着色器，而且这也是最常规的采样请求，其次是采样等级（SampleLevel）（这些都是在平常游戏中最常见的使用方式）。这意味着每个像素传送的32位浮点数平均在2个 $$(u ＋ v)$$ 和3个 $$((u ＋ v ＋ w) ／ (u ＋ v ＋ lod))$$ ，就算是2.5个吧，也就是10个字节。

假设一个不大不小的分辨率---1280x720，大概有92万个像素左右。游戏像素着色器平均有多少纹理采样数量？至少3个吧。让我们再适当的夸张一下，所以在3D渲染阶段，我们粗略的提高每个屏幕上的像素采样数量为两倍。然后我们全屏传递一些纹理用于后期处理。这可能又增加了至少每个像素6个采样点，这还是考虑到一些后期处理是使用低分辨率来完成的情况。把这一切都加起来我们大概得到 92 ＊ （3 * 2 ＋ 6）＝ 大概每一帧1100万个纹理采样，如果是30fps，那么就是3亿3000万个采样每秒。每个采样需要10个字节的数据，就是说对于纹理采样请求就有3.3GB／s的数据传输负担。这只是个下限，因为还有一些额外的开销存在（我们马上就会开始讲）。所以我这里可能要 *纠正* 一下关于“一丢丢”这个概念的下边界：）。对于一个DX11显卡上表现良好的现代游戏在高分辨率下运行，拥有比我刚才所列出的更复杂的着色器程序，比起来可能更多的消耗或者更少（延迟渲染（deferred shading）／灯光照明），高帧率，并且更加复杂的后期处理---继续，试着估算一个正常图像质量四分之一分辨率的双边采样（bilateral upsampling）SSAO大概需要多少带宽？（译者：嗯，想想都觉得可怕）

整个纹理带宽其实都不是你能轻易改动的，纹理采样器不是着色器核心的一部分，它们是芯片上的一些独立的单元，然后它自己每秒都在改组数千兆数据。这就是实际的架构，但是有个好的情况是无论如何我们都不需要对体积纹理（Cubemap）使用采样梯度（SampleGrad）：）

###  但是谁说只对单个纹理采样？
答案当然是否定的。我们的纹理请求由着色器单元发出，我们都知道着色器单元是一次处理16到64个像素／顶点／控制点或者一些其它东西的硬件。所以我们的着色器程序不会发送单个的纹理采样请求，而是会一次分发很多。这次，我们假设一次发送16个，理由很简单，之前我选择了32这个数字，但是这个数字不能开方，这在我们讨论2D纹理请求的时候显得有些奇怪。所以，选择一次16个纹理请求。如何创建一个纹理请求呢？首先我们要有一些命令让采样器知道怎么进行工作，添加一些空间用以存放纹理和采样器用到的状态（再一次，看看上面关于状态的部分），然后把它输送到纹理采样器上去。

这会消耗一些时间。

讲真的，纹理采样器有一个非常非常长的流水线（我们马上就会知道为什么这样）。一个纹理采样操作会消耗很长时间然后着色器单元就那么苦苦的等待吗？来，跟我念：吞 吐 量！所以其实事实不是这样的，真实情况是，当发生纹理采样之后，着色器单元会静默的切换到另外一个线程（或者说批）然后做一些别的工作，然后当有结果返回后再切回来。只要有足够多的独立工作让着色器单元去做就可以了！

### 一旦纹理坐标被计算完成...
好吧，这里有一堆计算需要做的:( 在这里，我假设用较简单的双线性采样方法，三线性和各向异性采样需要做更多的工作，见后文。

* 如果是一个采样或者[偏采样（SampleBias）](https://en.wikipedia.org/wiki/Sampling_bias#Types_of_sampling_bias)类型的请求，需要先计算纹理坐标梯度。

* 如果没有显式指定mipmap等级，通过梯度值算出用来采样的Mipmap等级并且如果有指定的LOD偏移的话也需要加入。

* 对于每个计算完的采样位置，通过指定寻址模式（重复模式（warp）／截取（clamp）／镜像（mirro）等）来获得真实的纹理采样位置，然后规范化到［0，1］坐标空间中。

* 如果是一个体积纹理（Cubemap），我们也需要决定哪个面用来进行采样（基于u/v/w的绝对值与符号决定），然后做一个除法使得坐标投影到一个单位立方体中，以处于[－1，1]坐标空间内部。我们同样需要丢弃掉其中一个坐标（这取决于立方体的哪个面）然后将其它的两个坐标按比例缩放到同样的[0，1]标准化坐标空间中。

* 然后，使用这些规则化的坐标转换成像素坐标进行采样，我们需要一些小数部分用来进行双线性差值。

* 最后，根据整数的x/y/z和纹理数组索引，我们就能计算出读取纹理像素点的地址。在这时候，需要一些乘法和加法？

如果你觉得上面的总结听起来并不是很好，这其实已经是我简化过的版本了。上面这些条目甚至没有包括一些有趣的问题比如纹理边界或者体积纹理的边采样和角落采样。相信我，虽然这也许听起来很糟糕，但是如果你的代码要包括这里所有可能发生的事情，你肯定会十分惊讶的，好在硬件已经帮我们完成了。无论如何，我们现在可以顺利的获得内存地址了。并且不论哪儿的内存地址，都只可能会存在于一个缓存或两个相邻的缓存上。

### 纹理缓存
所有人似乎都在使用两层纹理缓存。第二级缓存是一个完全标准包含纹理数据的缓存。第一级缓存不是标准的，因为它有一些附加智能。它也可能小于你期望的大小，每采样器大约只有4-8kb。让我们首先接受这个大小，因为它可能会让大多数人感到惊讶。

事实是这样的：大多数纹理采样是在像素着色器中完成的，并且启用了mip-mapping，而且会特别选择采样的mip级别以使屏幕像素：纹理像素比大致为1：1，这是重点。但是这意味着，除非你碰巧在纹理中一次又一次击中完全相同的位置，否则每个纹理采样操作将平均丢失大约1个纹素---双线性过滤的实际测量值大约为每次请求1.25个丢失（如果你单独跟踪像素就能发现）。即使您更改纹理缓存的大小，该值也是基本一样的，但是一旦纹理缓存足够大以至于可以包含整个纹理（通常在几百KB到几MB之间，该值就会急剧下降，但是这完全不切实际）（译者：缓存非常贵）。

第一，纹理缓存无论如何是一个巨大的成功（因为它将你从每个双线性样本的4个内存访问下降到1.25个）。但是与用于着色器核心的CPU和共享内存不同，从4k的缓存增加到16k所能获得的增益很少。所以我们选择对大的纹理数据采用流式传输数据的处理方式，而不是盲目增大一级缓存。

第二，由于每次采样会有平均1.25次的丢失，纹理采样器流水线需要足够长以维持每个样本被完全读取，而且还要保证不会停滞。让我用不同的方式来表达：纹理采样器管道足够长，不能为存储器读取停止，即使它需要400-800个时钟周期。这会是一个非常长的管道，它真的是一个字面意义上的“管道”，数据从一个流水线寄存器到下一个需要几百个周期，其中没有任何处理，直到采样完成。

所以，较小的一级缓存，较长的管道是我们需要的设计。那么“附加智能”怎么办？好吧，我们有压缩的纹理格式。你在PC上看到的那些，比如S3TC（又名DXTC又名BC1-3），然后是D3D10中引入的只是DXT上的变体的BC4和5，最后是D3D11引入的BC6H和7---这些都是基于块的纹理压缩方法，编码独立的4×4像素块。如果在纹理采样期间对它们进行解码，这意味着你需要能够在每个周期解码多达4个这样的块（比如最坏情况是你的4个双线性采样点刚好落在4个不同的块），并且需要从中读取像素。坦白说，这有点恶心。因此，当4×4的块被带入L1缓存时，它们被解码：在BC3（也称为DXT5）的情况下，从L2缓存中获取一个128位大小的块，然后在纹理缓存中将其解码为16个像素。这样一来，不是每次采样必须解码多达4个块，而是只需要对每个样本解码1.25/(4 * 4)大约0.08个块，至少如果你的纹理去获取其他15个像素的访问模式足够一致，你的解码效果就会更好了:)。哪怕你最终只是使用它的一部分，之后这些像素会离开一级缓存，这仍然是一个巨大的进步。这种技术也不限于DXT块，你可以处理D3D11中缓存填充路径中所需的多于50种不同纹理格式之间的绝大部分差异，这种差异大约是实际像素读取路径的三分之一。例如，UNORM sRGB纹理可以通过纹理缓存中将sRGB像素转换为每个通道为一个16位整数（或每个通道16位浮点数，或甚至是32位浮点数）来处理，滤波然后在线性空间中进行适当地操作。注意这最终会增加L1缓存中纹理元素的占用量，因此可能需要增加L1纹理缓存的大小，不是因为你需要缓存更多的纹理像素，而是因为你缓存的纹理像素胖了。像往常一样，这是一种折中处理。

### 滤波
在这一点上，实际上双线性滤波过程是相当简单直接的。从纹理缓存中抓取4个样本，使用它们的小数部分进行混合，通常使用乘法累加单元来支持这一功能。（实际上会有许多单元一起工作，因为纹理的4个像素通道会同时进行...）。

至于三线性滤波，不过是两个双线性采样和另一个线性插值，只需要在管线中添加更多的乘法累加单元即可。

那么各向异性过滤呢？实际上在较早的阶段管道中是需要做一些额外工作的，大致是最初我们计算用于采样操作的mip层级的时候。我们所要做的是使用梯度值用以确定区域和纹理像素空间中屏幕像素的形状。如果它宽高大致相同，我们只要做一个标准的双线性或者三线性采样即可，但是如果它在一个方向延伸，我们就在该方向上多做几次采样，并将结果混合。这将产生多个采样位置，因此我们通过完整的循环多次双线性或者三线性采样流水线来进行计算，而且实际采样位置和其相对的权重计算是每个硬件供应商密切保护的机密。多年来，硬件提供商们一直在用各式各样的黑科技去解决这个问题，以现在的结果来说，这些算法收敛在一个比较合理的硬件消耗范围内。我不会试图猜测他们在做什么，给你们一个忠告，作为一个图形程序员，你并不需要太关心底层的各向异性过滤算法，只要它不崩溃，不产生明显的锯齿，不会造成程序运行效率下降就行了。

无论如何，除了用于循环所需的采用设置和排序逻辑之外，都不需要大量计算。在这一点上，我们有足够的乘法累积单元来计算各向异性过滤中涉及到的加权和，而在实际的过滤阶段不需要太多额外的硬件单元。:)

### 纹理返回值
现在我们几乎已经在纹理采样管线的最后阶段了。这一切的结果是什么？与纹理请求这种每次请求的数据大小都可能有差别的请求不同，每个纹理采样请求最多请求的是4个值（r，g，b，a）。最常见的情况就是只有着色器会使用这4个值。记住，传输4个浮点数返回值从带宽的角度来说只是洒洒水的事情，在某些情况下，你可能不需要太大的数据量。如果你的着色器采样的是每个像素管道32位浮点数的纹理，你最好返回32位的浮点数，但如果读取的是一个8位UNORM SRGB纹理，32位浮点数就显得有些浪费了，你可以选择使用一些较小的数据格式来解决这个问题。

着色器单元现在有了纹理采样结果，就可以继续你所提交的工作了，这个阶段就结束了。在下一期中，在我讨论完需要完成的工作之后，我们才能真正开始讨论光栅化图元。更新：这里是一个纹理抽样管道的图片，包括一个有趣的错误，我已经修复了！

![photo 1]({{ IMAGE_PATH }}/texture_sample.jpg)

### 像往常一样的结束语
这一次，没有大的免责声明。我在带宽示例中提到的数字是我实验获得的数据，我不能捏造一些游戏的实际数字，但除此之外，我所描述的应该是非常接近你GPU上的真实表现的，虽然我省略了滤波等操作的一些边界情况（主要是因为细节比原理更令人讨厌）。

至于包含解压纹理数据的L1纹理缓存，据我所知，这对于当前硬件来说是准确的。一些老的硬件甚至在L1纹理缓存有一些格式压缩操作，但是由于“不论缓存多大都会产生每次采样平均1.25次丢失”这一规则的存在，这不会带来太多的好处，甚至我觉得不值得如此复杂。这些东西应该被淘汰了。

比较有趣的是嵌入式功率优化的图形芯片，例如PowerVR，我不会在这个系列中涉及到这些芯片，因为我的重点是在PC中的高性能部分，但是如果你有兴趣，我在以前的部分文章中有一些关于它们的讨论。无论如何，PVR芯片有自己的纹理压缩格式，它不是基于块的，并且这些格式与它们的滤波硬件非常紧密地集成，因此我认为他们可以保持在L1缓存中集成纹理压缩（实际上，我甚至不知道他们有二级缓存！）。这是一个有趣的方法，可能在充分利用每个硬件面积区域和降低功耗来说是个不错的主意。虽然我认为“为L1缓存解压缩”这个方法总体提供了更高的吞吐量，但我不会经常提及，因为这里讨论的重点是关于PC GPU的吞吐量:)

--------------------------------------------

## English

Welcome back. Last part was about vertex shaders, with some coverage of GPU shader units in general. Mostly, they’re just vector processors, but they have access to one resource that doesn’t exist in other vector architectures: Texture samplers. They’re an integral part of the GPU pipeline and are complicated (and interesting!) enough to warrant their own article, so here goes.

### Texture state
Before we start with the actual texturing operations, let’s have a look at the API state that drives texturing. In the D3D11 part, this is composed of 3 distinct parts:

1. The sampler state. Filter mode, addressing mode, max anisotropy, stuff like that. This controls how texture sampling is done in a general way.

2. The underlying texture resource. This boils down to a pointer to the raw texture bits in memory. The resource also determines whether it’s a single texture or a texture array, what multisample format the texture has (if any), and the physical layout of the texture bits – i.e. at the resource level, it’s not yet decided how the values in memory are to be interpreted exactly, but their memory layout is nailed down.

3. The shader resource view (SRV for short). This determines how the texture bits are to be interpreted by the sampler. In D3D10+, the resource view links to the underlying resource, so you never specify the resource explicitly.
Most of the time, you will create a texture resource with a given format, let’s say RGBA, 8 bits per component, and then just create a matching SRV. But you can also create a texture as “8 bits per component, typeless” and then have several different SRVs for the same resource that read the underlying data in different formats, e.g. once as UNORM8_SRGB (unsigned 8-bit value in sRGB space that gets mapped to float 0..1) and once as UINT8 (unsigned 8-bit integer).

Creating the extra SRV seems like an annoying extra step at first, but the point is that this allows the API runtime to do all type checking at SRV creation time; if you get a valid SRV back, that means the SRV and resource formats are compatible, and no further type checking needs to be done while that SRV exists. In other words, it’s all about API efficiency here.

Anyway, at the hardware level, what this boils down to is just a bag of state associated with a texture sampling operation – sampler state, texture/format to use, etc. – that needs to get kept somewhere (see part 2 for an explanation of various ways to manage state in a pipelined architecture). So again, there’s various methods, from “pipeline flush every time any state changes” to “just go completely stateless in the sampler and send the full set along with every texture request”, with various options inbetween. It’s nothing you need to worry about – this is the kind of thing where HW architects whip up a cost-benefit analysis, simulate a few workloads and then take whichever method comes out ahead – but it’s worth repeating: as PC programmer, don’t assume the HW adheres to any particular model.

Don’t assume that texture switches are expensive – they might be fully pipelined with stateless texture samplers so they’re basically free. But don’t assume they’re completely free either – maybe they are not fully pipelined or there’s a cap on the maximum number of different sets of texture states in the pipeline at any given time. Unless you’re on a console with fixed hardware (or you hand-optimize your engine for every generation of graphics HW you’re targeting), there’s just no way to tell. So when optimizing, do the obvious stuff – sort by material where possible to avoid unnecessary state changes and the like – which certainly saves you some API work at the very least, and then leave it at that. Don’t do anything fancy based on any particular model of what the HW is doing, because it can (and will!) change in the blink of an eye between HW generations.

### Anatomy of a texture request
So, how much information do we need to send along with a texture sample request? It depends on the texture type and which kind of sampling instruction we’re using. For now, let’s assume a 2D texture. What information do we need to send if we want to do a 2D texture sample with, say, up to 4x anisotropic sampling?  

* The 2D texture coordinates – 2 floats, and sticking with the D3D terminology in this series, I’m going to call them u/v and not s/t.

* The partial derivatives of u and v along the screen “x” direction: $$ \frac{\partial u} {\partial x}, \frac{\partial v}{\partial x} $$.

* Similarly, we need the partial derivative in the “y” direction too: $$ \frac{\partial u}{\partial y}, \frac{\partial v}{\partial y} $$.

So, that’s 6 floats for a fairly pedestrian 2D sampling request (of the SampleGrad variety) – probably more than you thought. The 4 gradient values are used both for mipmap selection and to choose the size and shape of the anisotropic filtering kernel. You can also use texture sampling instructions that explicitly specify a mipmap level (in HLSL, that would be SampleLevel) – these don’t need the gradients, just a single value containing the LOD parameter, but they also can’t do anisotropic filtering – the best you’ll get is trilinear! Anyway, let’s stay with those 6 floats for a while. That sure seems like a lot. Do we really need to send them along with every texture request?

The answer is: depends. In everything but Pixel Shaders, the answer is yes, we really have to (if we want anisotropic filtering that is). In Pixel Shaders, turns out we don’t; there’s a trick that allows Pixel Shaders to give you gradient instructions (where you can compute some value and then ask the hardware “what is the approximate screen-space gradient of this value?”), and that same trick can be employed by the texture sampler to get all the required partial derivatives just from the coordinates. So for a PS 2D “sample” instruction, you really only need to send the 2 coordinates which imply the rest, provided you’re willing to do some more math in the sampler units.

Just for kicks: What’s the worst-case number of parameters required for a single texture sample? In the current D3D11 pipeline, it’s a SampleGrad on a Cubemap array. Let’s see the tally:

* 3D texture coordinates – u, v, w: 3 floats.

* Cubemap array index: one int (let’s just bill that at the same cost as a float here).

* Gradient of (u,v,w) in the screen x and y directions: 6 floats.

For a total of 10 values per pixel sampled – that’s 40 bytes if you actually store it like that. Now, you might decide that you don’t need full 32 bits for all of this (it’s probably overkill for the array index and gradients), but it’s still a lot of data to be sending around.

In fact, let’s check what kind of bandwidth we’re talking about here. Let’s assume that most of our textures are 2D (with a few cubemaps thrown in), that most of our texture sampling requests come from the Pixel Shader with little to no texture samples in the Vertex Shader, and that the regular Sample-type requests are the most frequent, followed by SampleLevel (all of this is pretty typical for actual rendering you see in games). That means the average number of 32-bit floats values sent per pixel will be somewhere between 2 (u+v) and 3 (u+v+w / u+v+lod), let’s say 2.5, or 10 bytes.

Assume a medium resolution – say, 1280×720, which is about 0.92 million pixels. How many texture samples does your average game pixel shader have? I’d say at least 3. Let’s say we have a modest amount of overdraw, so during the 3D rendering phase, we touch each pixel on the screen roughly twice. And then we finish it off with a few texture-heavy full-screen passes to do post-processing. That probably adds at least another 6 samples per pixel, taking into account that some of that postprocessing will be done at a lower resolution. Add it all up and we have 0.92 * (3*2 + 6) = about 11 million texture samples per frame, which at 30 fps is about 330 million a second. At 10 bytes per request, that’s 3.3 GB/s just for texture request payloads. Lower bound, since there’s some extra overhead involved (we’ll get to that in a second). Note that I’m *cough* erring “a bit” on the low side with all of these numbers :). An actual modern game on a good DX11 card will run in significantly higher resolution, with more complex shaders than I listed, comparable amount of overdraw or even somewhat less (deferred shading/lighting to the rescue!), higher frame rate, and way more complex postprocessing – go ahead, do a quick back-of-the-envelope calculation how much texture request bandwidth a decent-quality SSAO pass in quarter-resolution with bilateral upsampling takes…

Point being, this whole texture bandwidth thing is not something you can just hand-wave away. The texture samplers aren’t part of the shader cores, they’re separate units some distance away on the chip, and shuffling multiple gigabytes per second around isn’t something that just happens by itself. This is an actual architectural issue – and it’s a good thing we don’t use SampleGrad on Cubemap arrays for everything :)

### But who asks for a single texture sample?
The answer is of course: No one. Our texture requests are coming from shader units, which we know process somewhere between 16 and 64 pixels / vertices / control points / … at once. So our shaders won’t be sending individual texture samples, they’ll dispatch a bunch of them at once. This time, I’ll use 16 as the number – simply because the 32 I chose last time is non-square, which just seems weird when talking about 2D texture requests. So, 16 texture requests at once – build that texture request payload, add some command fields at the start so the sampler knows what to do, add some more fields so the sampler knows which texture and sampler state to use (again, see the remarks above on state), and send that off to a texture sampler somewhere.

This will take a while.

No, seriously. Texture samplers have a seriously long pipeline (we’ll soon see why); a texture sampling operation takes way too long for a shader unit to just sit idle for all that time. Again, say it with me: throughput. So what happens is that on a texture sample, a shader unit will just quietly switch to another thread/batch and do some other work, then switch back a while later when the results are there. Works just fine as long as there’s enough independent work for the shader units to do!

### And once the texture coordinates arrive…
Well, there’s a bunch of computations to be done first: (In here and the following, I’m assuming a simple bilinear sample; trilinear and anisotropic take some more work, see below).

* If this is a Sample or SampleBias-type request, calculate texture coordinate gradients first.

* If no explicit mip level was given, calculate the mip level to be sampled from the gradients and add the LOD bias if specified.

* For each resulting sample position, apply the address modes (wrap / clamp / mirror etc.) to get the right position in the texture to sample from, in normalized [0,1] coordinates.

* If this is a cubemap, we also need to determine which cube face to sample from (based on the absolute values and signs of the u/v/w coordinates), and do a division to project the coordinates onto the unit cube so they are in the [-1,1] interval. We also need to drop one of the 3 coordinates (based on the cube face) and scale/bias the other 2 so they’re in the same [0,1] normalized coordinate space we have for regular texture samples.

* Next, take the [0,1] normalized coordinates and convert them into fixed-point pixel coordinates to sample from – we need some fractional bits for the bilinear interpolation.

* Finally, from the integer x/y/z and texture array index, we can now compute the address to read texels from. Hey, at this point, what’s a few more multiplies and adds among friends?

If you think it sounds bad summed up like that, let me take remind you that this is a simplified view. The above summary doesn’t even cover fun issues such as texture borders or sampling cubemap edges/corners. Trust me, it may sound bad now, but if you were to actually write out the code for everything that needs to happen here, you’d be positively horrified. Good thing we have dedicated hardware to do it for us. :) Anyway, we now have a memory address to get data from. And wherever there’s memory addresses, there’s a cache or two nearby.

### Texture cache
Everyone seems to be using a two-level texture cache these days. The second-level cache is a completely bog-standard cache that happens to cache memory containing texture data. The first-level cache is not quite as standard, because it’s got additional smarts. It’s also smaller than you probably expect – on the order of 4-8kb per sampler. Let’s cover the size first, because it tends to come as a surprise to most people.

The thing is this: Most texture sampling is done in Pixel Shaders with mip-mapping enabled, and the mip level for sampling is specifically chosen to make the screen pixel:texel ratio roughly 1:1 – that’s the whole point. But this means that, unless you happen to hit the exact same location in a texture again and again, each texture sampling operation will miss about 1 texel on average – the actual measured value with bilinear filtering is around 1.25 misses/request (if you track pixels individually). This value stays more or less unchanged for a long time even as you change texture cache size, and then drops dramatically as soon as your texture cache is large enough to contain the whole texture (which usually is between a few hundred kilobytes and several megabytes, totally unrealistic sizes for a L1 cache).

Point being, any texture cache whatsoever is a massive win (since it drops you down from about 4 memory accesses per bilinear sample down to 1.25). But unlike with a CPU or shared memory for shader cores, there’s very little gain in going from say 4k of cache to 16k; we’re streaming larger texture data through the cache no matter what.

Second point: Because of the 1.25 misses/sample average, texture sampler pipelines need to be long enough to sustain a full read from memory per sample without stalling. Let me phrase that differently: texture sampler pipes are long enough to not stall for a memory read even though it takes 400-800 cycles. That’s one seriously long pipeline right there – and it really is a pipeline in the literal sense, handing data from one pipeline register to the next for a few hundred cycles without any processing until the memory read is completed.

So, small L1 cache, long pipeline. What about the “additional smarts”? Well, there’s compressed texture formats. The ones you see on PC – S3TC aka DXTC aka BC1-3, then BC4 and 5 which were introduced with D3D10 and are just variations on DXT, and finally BC6H and 7 which were introduced with D3D11 – are all block-based methods that encode blocks of 4×4 pixels individually. If you decode them during texture sampling, that means you need to be able to decode up to 4 such blocks (if your 4 bilinear sample points happen to land in the worst-case configuration of straddling 4 blocks) per cycle and get a single pixel from each. That, frankly, just sucks. So instead, the 4×4 blocks are decoded when it’s brought into the L1 cache: in the case of BC3 (aka DXT5), you fetch one 128-bit block from texture L2, and then decode that into 16 pixels in the texture cache. And suddenly, instead of having to partially decode up to 4 blocks per sample, you now only need to decode 1.25/(4*4) = about 0.08 blocks per sample, at least if your texture access patterns are coherent enough to hit the other 15 pixels you decoded alongside the one you actually asked for :). Even if you only end up using part of it before it goes out of L1 again, that’s still a massive improvement. Nor is this technique limited to DXT blocks; you can handle most of the differences between the >50 different texture formats required by D3D11 in your cache fill path, which is hit about a third as often as the actual pixel read path – nice. For example, things like UNORM sRGB textures can be handled by converting the sRGB pixels into a 16-bit integer/channel (or 16-bit float/channel, or even 32-bit float if you want) in the texture cache. Filtering then operates on that, properly, in linear space. Mind that this does end up increasing the footprint of texels in the L1 cache, so you might want to increase L1 texture size; not because you need to cache more texels, but because the texels you cache are fatter. As usual, it’s a trade-off.

### Filtering
And at this point, the actual bilinear filtering process is fairly straightforward. Grab 4 samples from the texture cache, use the fractional positions to blend between them. That’s a few more of our usual standby, the multiply-accumulate unit. (Actually a lot more – we’re doing this for 4 channels at the same time…)

Trilinear filtering? Two bilinear samples and another linear interpolation. Just add some more multiply-accumulates to the pile.

Anisotropic filtering? Now that actually takes some extra work earlier in the pipe, roughly at the point where we originally computed the mip-level to sample from. What we do is look at the gradients to determine not just the area but also the shape of a screen pixel in texel space; if it’s roughly as wide as it is high, we just do a regular bilinear/trilinear sample, but if it’s elongated in one direction, we do several samples across that line and blend the results together. This generates several sample positions, so we end up looping through the full bilinear/trilinear pipeline several times, and the actual way the samples are placed and their relative weights are computed is a closely guarded secret for each hardware vendor; they’ve been hacking at this problem for years, and by now both converged on something pretty damn good at reasonable hardware cost. I’m not gonna speculate what it is they’re doing; truth be told, as a graphics programmer, you just don’t need to care about the underlying anisotropic filtering algorithm as long as it’s not broken and produces either terrible artifacts or terrible slowdowns.

Anyway, aside from the setup and the sequencing logic to loop over the required samples, this does not add a significant amount of computation to the pipe. At this point we have enough multiply-accumulate units to compute the weighted sum involved in anisotropic filtering without a lot of extra hardware in the actual filtering stage. :)

### Texture returns
And now we’re almost at the end of the texture sampler pipe. What’s the result of all this? Up to 4 values (r, g, b, a) per texture sample requested. Unlike texture requests where there’s significant variation in the size of requests, here the most common case by far is just the shader consuming all 4 values. Mind you, sending 4 floats back is nothing to sneeze at from a bandwidth point of view, and again you might want to shave bits in some case. If your shader is sampling a 32-bit float/channel texture, you’d better return 32-bit floats, but if it’s reading a 8-bit UNORM SRGB texture, 32 bit returns are just overkill, and you can save bandwidth by using a smaller format on the return path.

And that’s it – the shader unit now has its texture sampling results back and can resume working on the batch you submitted – which concludes this part. See you again in the next installment, when I talk about the work that needs to be done before we can actually start rasterizing primitives. Update: And here’s a picture of the texture sampling pipeline, including an amusing mistake that I’ve fixed in post like a pro!

![photo 1]({{ IMAGE_PATH }}/texture_sample.jpg)

### The usual post-script
This time, no big disclaimers. The numbers I mentioned in the bandwidth example are honestly just made up on the spot since I couldn’t be arsed to look up some actual figures for current games :), but other than that, what I describe here should be pretty close to what’s on your GPU right now, even though I hand-waved past some of the corner cases in filtering and such (mainly because the details are more nauseating than they are enlightening).

As for texture L1 cache containing decompressed texture data, to the best of my knowledge this is accurate for current hardware. Some older HW would keep some formats compressed even in L1 texture cache, but because of the “1.25 misses/sample for a large range of cache sizes” rule, that’s not a big win and probably not worth the complexity. I think that stuff’s all gone now.

An interesting bit are embedded/power-optimized graphics chips, e.g. PowerVR; I’ll not go into these kinds of chips much in this series since my focus here is on the high-performance parts you see in PCs, but I have some notes about them in the comments for previous parts if you’re interested. Anyway, the PVR chips have their own texture compression format that’s not block-based and very tightly integrated with their filtering hardware, so I would assume that they do keep their textures compressed even in L1 texture cache (actually, I don’t know if they even have a second cache level!). It’s an interesting method and probably at a fairly sweet spot in terms of useful work done per area and energy consumed. But I think the “depack to L1 cache” method gives higher throughput overall, and as I can’t mention often enough, it’s all about throughput on high-end PC GPUs :)