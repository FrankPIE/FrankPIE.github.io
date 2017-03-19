---
layout: post
title: "(译)3D管线之旅：3D管线概述，顶点处理"
short_title: "(译)3D管线之旅3"
keywords: ["GPU", "3D管线", "译"]
description: "A trip through the Graphics Pipeline 2011 的翻译"
category: "Unreliable-Translate"
tags: ["GPU", "3D管线", "图形学", "硬件底层"]
---
{% include JB/setup %}

[原文：A trip through the Graphics Pipeline 2011（需要翻墙）](https://fgiesen.wordpress.com/2011/07/03/a-trip-through-the-graphics-pipeline-2011-part-3/)
--------------------------------------------

## 中文
我们已经讨论了一个DC（draw call）从APP通过多个驱动层最后到达命令处理器的全部过程。现在我们终于将要揭开一些图形处理的神秘面纱了，在这一部分，我们将讨论顶点管线（vertex pipeline）。但是在这之前......

### 先来看看一些常用名词
我们现在已经真正的进入3D管线了，所谓3D管线是由多个执行特殊功能的阶段组合而成的一条流水线。我现在准备先给出我将要讨论的所有阶段的名称——为了确保一致性，大多数名称是按照D3D10/11的官方名称（包括缩写）来描述的。虽然这些阶段我们都会在之后的文章中详细讨论，但是要讨论完还需要挺长一段时间——我是认真的，我已经做好了规划，也就意味着接下去至少我有两周时间会因为这个系列而非常忙碌！不论如何，这里我先给出每个阶段的名称和简单的描述。

* IA — 输入装配（Input Assembler）。读入索引和顶点原始数据。
* VS — 顶点着色器（Vertex shader）。获得输入的原始顶点数据并且处理成为硬件可识别的顶点数据，为下一阶段输出。
* PA — 图元装配（Primitive Assembly）。将顶点装配成为一个个图元（译者注：多为三角形）并且向下一阶段传递。
* HS - 外壳着色器（Hull shader，译者注：OpenGL中称为TESS Control Shader）。接受Patch图元，写入改造后（也可能没有改造）的Patch控制点，作为域着色器（domain shader）的输入，添加一些额外的数据以驱动曲面细分（tessellation）。
* TS — 曲面细分（Tessellator stage）。 创造顶点和细分线（tessellated lines）和三角形连通性。
* DS — 域着色器（Domain shader，译者注：OpenGL中称为TESS Evaluation Shader）。 将获得的控制点，连同从HS阶段的额外数据和TS阶段的细分位置（tessellated positions）再次转换为顶点数据。
* GS — 几何着色器（Geometry shader)。 输入图元，随意的根据邻接信息输出不同的图元。也是主要枢纽。
* SO — 流输出（Stream-out）。将GS的输出数据写入内存。
* RS — 光栅化（Rasterizer）。光栅化图元。
* PS — 像素着色器（Pixel shader，译者注：OpenGL中称为Fragment Shader）。 获得插值处理过的顶点数据，输出像素颜色。也可以写入UAVs（unordered access views）
* OM — 输出合并（Output merger）。获得从PS中得到的像素片段，经过alpha混合（alpha blending）并写入后备帧缓存。
* CS — 通用计算着色器（Compute shader）。这是一条完全独立的流水线。只需要输入常量缓存和线程ID。可以写入缓存和UAVs。

我将列出并且按顺序讨论多个数据流路径（我把IA，PA，RS和OM阶段省略了，因为它们实际上并不会对数据做出什么改变，它们的功能只是重新排列数据，感觉很像粘合剂）

1. VS→PS: D3D9中老式的可编程管线，至今为止，还是渲染中最重要的部分。我将会从头到尾讲一遍。
2. VS→GS→PS: 几何着色（Geometry Shading） (D3D10中的新特性)。
3. VS→HS→TS→DS→PS, VS→HS→TS→DS→GS→PS: 曲面细分 (D3D11中的新特性)。
4. VS→SO, VS→GS→SO, VS→HS→TS→DS→GS→SO: 流输出 (包含或者不包含曲面细分)。
5. CS: 通用计算，D3D11中的新特性。

现在你知道接下来我们会讨论什么了吧。让我们来看看顶点着色器（vertex shaders）！

### 输入装配（IA）阶段
最先做的事情是从索引缓存中读取索引。这里可能会存在数据没有索引化的情况，这种情况下，该阶段会伪造一个ID索引缓存（0 1 2 3 4 ...）来替代这里的索引数据。如果我们已经有了索引缓存，它的内容就可以从内存中被读取到了——不是直接读取，IA一般会设置一个高速数据缓存来缓存顶点或者索引数据的位置从而加速访问。并且这里需要注意索引缓存的读取要先进行边界检查（事实上，在D3D10以后的所有资源都需要这么做）。如果你引用的数据超过了原始的数据边界（比方说，绘制时传递的参数误以为存在6个顶点索引，但是实际上只有5个这种情况），超过边界的部分将返回0。在一些特殊情况下命令会完全无效化，这具备良好定义（译者注：错误的情况下有相应的处理也是具有良好定义的一种形式）。类似的，你也可以在DrawIndexed中传递一个空的buffer进去——这个行为和你引用一个长度为0的索引缓存是一致的，所有的读取都超过边界，所以都返回0。在D3D10以后，大量的错误行为都具备良好的定义而不会造成程序陷入未定义行为的悲剧中。

一旦我们有了顶点数据，其实就意味着我们有了每个顶点和每个实例的数据（在这个阶段，当前的实例ID就是另外一个计数器，非常直接），因为索引其实就可以看成是我们数据布局的声明。只要我们从缓存或者内存中读取然后将其解压成着色器核心需要的浮点数格式输入就完成了。但是实际上，这个读取操作并没有那么直接。因为，硬件其实还有一个已着色顶点数据的缓存，所以当一个顶点被多个三角形所引用（一个完整的闭合三角网格，每个顶点将会被6个三角形所引用！）的时候并不需要每次都需要从cache或者内存中读取数据，在这种情况下，只要通过缓存来分享数据即可。

### 顶点缓存与着色
注意：这部分内容，有一部分是我猜测的。虽然是我依据了那些了解现代GPU的人的公开描述，但是这些信息只告诉我一些相关的内容却并没有告诉我为什么。所以这里存在一些我个人的推断。我简单的猜测了一些细节。也就是说我并没有打算完全的把所有我的臆测都说出来，因为我并不知道靠不靠谱。但是有一点我敢肯定，我在这里描述的东西时可靠而且确实可以有效工作的（在一般的环境下），我只是不能保证这是硬件中真实使用的方法，而且我也会漏下一些特殊的处理技巧：）。

无论如何。长期以来（其中主要涉及包括了3.0 shader model的那一代GPU），顶点着色器和像素着色器是通过不同的硬件单元实现并且有不同的性能权衡。而且顶点缓存结构相当简单：具备一块先入先出内存，可以存储和处理小数目（大概一打到两打那个数目概念）的顶点，拥有足够的内存空间用以应付最极端情况下的顶点属性数量，最后还能够将顶点索引用为标记。非常直接了当的结构。

然后统一的着色器单元出现了。将一种着色器单元用于两种不同的计算处理，设计上来说必然就变的更加复杂了。一方面，需要处理顶点着色，可能在正常使用过程中每帧处理一百万以上顶点数据。而另一方面，处理像素着色，仅仅是为了将1920x1200分辨率的图像填充满整个屏幕空间就需要每帧处理230w像素数据——如果你想渲染跟多的物体，那么会消耗更多。所以猜猜最终是哪个方面进行了妥协。

好吧，我来公布答案：原来那种一次或多或少处理一个顶点的设计被放弃了，作为替换，我们现在有一种更加统一奔放的着色器单元用以处理两种不同的着色处理，这个设计的目的是最大化吞吐量，并且消除延迟，因此每次需要处理更多的顶点，也就是需要批处理（batch）（每一个批处理需要包含多少个顶点？目前来看，这个数字大概为每一批16至64个顶点之间）。

如果我们用FIFO的思想来设计这个管线流程，那么意味着如果你不想低效的进行着色计算，那么在你等待一次顶点着色装载（shading load）的过程中就会产生16至64个顶点数据的cache miss。FIFO对于这种设计来说是不好的。问题在于如果你一次性对整个批次的顶点进行着色处理，这也就意味着一批顶点的图元装配（PA）只能等待它们顶点着色（VS）完成后才能开始。这时候当你把整批顶点（打个比方就说是32个顶点）加入到FIFO的末端，那么之前的32个顶点就会被移出——但是很可能当前正在进行PA的顶点缓存正在引用这批被移出的顶点。阿哈，这当然是无效的。显然，我们不能引用这些被移出FIFO的顶点缓存，因为不知道什么时候它们就没有了。另外一个问题，我们需要将FIFO设置成多大？如果我们每个批次进行32个顶点的着色处理，至少它得有32个索引标记，但是如果这样，我们就不能引用上一次被处理的32个顶点了（因为已经被移出了），这意味着只有在每次处理的时候队列是空的，才能确保之后行为是有效的。所以怎么办？增加FIFO的大小？比如设置为当前的两倍也就是64个单元大小？这相当大了。注意所有顶点缓存寻找包括比较索引标记的时候是针对所有在FIFO中的标记的－虽然这是完全并行的，但是还是十分的令人讨厌。针对这点我们实现全关联缓存来确保行为有效。还有，我们又是如何处理分配着色装载32个顶点和返回接收结果之间的时间的呢——仅仅只是忙等？这也许会消耗几百个时钟周期，这看起来像是个蠢主意！或许可以让并行进行两个着色装载（shading load）？那么这样一来，至少FIFO就需要两倍也就是64个标记大小的长度，那么之前我们说的提升两倍的意义又没用了，一切都回到了32个顶点时候的问题，问题依旧没有被解决。那么，一个FIFO对应多个着色器核心呢？[阿姆达尔定律](https://zh.wikipedia.org/wiki/%E9%98%BF%E5%A7%86%E8%BE%BE%E5%B0%94%E5%AE%9A%E5%BE%8B)依旧有效－把一个严格的串行流程并行化会造成性能瓶颈。

所以FIFO的思想对于这个环境是不适用的，所以，我们干脆把这个想法完全扔掉。回到我们最基础的问题。我们最基本的需求是什么？获得一批数量合适的顶点用于着色，并且在非必要的情况下不进行着色处理。

好吧，让它简单一点：保留足够的缓存空间给32个顶点（等于一个批次），然后保留类似的高速缓存标记空间给32个项。从一个空的“cache”开始，比如，所有的项都是无效的。对于每个顶点缓存中的图元，在所有索引中进行查找。如果高速缓存命中，那么直接使用，反之，如果未命中，在当前的批次中分配一个位置并把索引标记添加进去。一旦不再具有足够的空间添加一个新的图元了，那么将整个批分发于顶点着色，并且保存索引标记数组（比如我们刚刚进行着色处理的32个顶点的索引），然后开始设置下一个批，又从一个空的cache开始——确保每个批都是完全独立的。

每一个批都会使得一个shader单元工作一段时间（一般来说至少是几百个时钟周期！）。但是这并不是问题，因为我们有好多shader单元——只要为每个批选择不同的单元来执行即可！并行处理。当最终我们得到结果，这时候我们可以用保存下来的标记和原始的索引缓存数据去装配图元然后在流水线上传递下去（这就是“图元装配”所做的工作，这部分内容在之后会涉及）。

顺便一说，当我说到“获得结果”，这是什么意思？它们的终点在哪里？这里主要有两个选项：1.特殊的缓存，2.一些普通的高速缓存与暂存内存。以前一般是使用第一种方式，使用一种固定的组织设计顶点数据（保留着顶点属性的空间等等），但是后来的GPU都选择了后者，这样的话，这不过就是一块内存，这样的设计更加具有弹性，而且你可以将这个内存用于其它着色器阶段，反之，特殊的顶点缓存就没有办法使用在像素着色处理和通用计算流水线。(译者注：在现代的图形管线中都是实用了这种统一内存的形式。)

原作者更新：见下图，描述了顶点着色的流程。

![顶点着色流程]({{ IMAGE_PATH }}/vertex_shade.jpg)

### 着色器单元内部
简单的讲：几乎上可以这么说着色器内部都是你在反汇编HLSL时候看到的那些东西（fxc /dumpbin 是你的好朋友！）。猜猜怎么着，仅仅是一些擅长处理那种代码的的处理器，本质上来说，硬件上完成这些处理的方法就是建立一些东西可以处理接近着色器字节码。不像我之前讨论的东西，这些文档非常详细，如果你感兴趣，只要读一些AMD和NV的会议刊物或者CUDA／Stream的SDK文档即可。

无论如何，这里有个总结：快速的算数计算单元（ALU）构建在浮点加速单元（FMACU）周围，一些硬件至少支持求倒，求反平方根，求以2为底的对数函数，求指数函数。求三角函数，为优化高吞吐量高密度高延迟，使用巨量线程覆盖来降低延迟影响，每个线程拥有少量的寄存器，非常擅长执行顺序语句的代码，但是不擅长处理分支（尤其是当分支代码不连贯的时候）。

几乎所有硬件上都有这样的实现。但是也有些许不同。以前在AMD硬件上普遍直接用4-位宽的SIMD表示HLSL/GLSL以及shader自节码（尽管它们后来不用了），而NVidia不久之前打算将4-通路SIMD转变成标量指令（scalar instruction）。再次提醒，所有的这些在web上都有资料。

真正有兴趣注意一下的是着色器阶段之间的不同。简单来说，其中还真的有几个不一样（译者批：你这也忒简单了～）。举个例子，所有的算数和逻辑指令在所有阶段中都一样。一些结构（比如像素着色器的导数指令和插值属性）却只存在于一些阶段。但是，大多数情况来说是，这些不同取决于不同格式或者种类数据的IO。

有一个特殊的shaders相关的点需要足够的篇幅来讨论，就是纹理采样（和纹理单元）。所以，我把它拿出来独成下一章！下次见！

### 结束语
我再次免责声明关于“顶点缓存与着色”一节：其中有一部分是我的猜测，所以你可以适当的选取你觉得对的部分。

我也不打算讲如何管理暂存内存和高速缓存；缓存大小取决于处理批次的大小和想要输出的顶点属性。缓存大小和管理对于性能是很重要的，但我不在这里详细解释，我也不想解释。虽然很有趣，但是这部分内容与谈论的硬件非常特殊，不用深入了解。

--------------------------------------------

## English

At this point, we’ve sent draw calls down from our app all the way through various driver layers and the command processor; now, finally we’re actually going to do some graphics processing on it! In this part, I’ll look at the vertex pipeline. But before we start…

### Have some Alphabet Soup!
We’re now in the 3D pipeline proper, which in turn consists of several stages, each of which does one particular job. I’m gonna give names to all the stages I’ll talk about – mostly sticking with the “official” D3D10/11 names for consistency – plus the corresponding acronyms. We’ll see all of these eventually on our grand tour, but it’ll take a while (and several more parts) until we see most of them – seriously, I made a small outline of the ground I want to cover, and this series will keep me busy for at least 2 weeks! Anyway, here goes, together with a one-sentence summary of what each stage does.

* IA — Input Assembler. Reads index and vertex data.
* VS — Vertex shader. Gets input vertex data, writes out processed vertex data for the next stage.
* PA — Primitive Assembly. Reads the vertices that make up a primitive and passes them on.
* HS — Hull shader; accepts patch primitives, writes transformed (or not) patch control points, inputs for the domain shader, plus some extra data that drives tessellation.
* TS — Tessellator stage. Creates vertices and connectivity for tessellated lines or triangles.
* DS — Domain shader; takes shaded control points, extra data from HS and tessellated positions from TS and turns them into vertices again.
* GS — Geometry shader; inputs primitives, optionally with adjacency information, then outputs different primitives. Also the primary hub for…
* SO — Stream-out. Writes GS output (i.e. transformed primitives) to a buffer in memory.
* RS — Rasterizer. Rasterizes primitives.
* PS — Pixel shader. Gets interpolated vertex data, outputs pixel colors. Can also write to UAVs (unordered access views).
* OM — Output merger. Gets shaded pixels from PS, does alpha blending and writes them back to the backbuffer.
* CS — Compute shader. In its own pipeline all by itself. Only input is constant buffers+thread ID; can write to buffers and UAVs.

And now that that’s out of the way, here’s a list of the various data paths I’ll be talking about, in order: (I’ll leave out the IA, PA, RS and OM stages in here, since for our purposes they don’t actually do anything to the data, they just rearrange/reorder it – i.e. they’re essentially glue)

1. VS→PS: Ye Olde Programmable Pipeline. In D3D9, this was all you got. Still the most important path for regular rendering by far. I’ll go through this from beginning to end then double back to the fancier paths once I’m done.
2. VS→GS→PS: Geometry Shading (new with D3D10).
3. VS→HS→TS→DS→PS, VS→HS→TS→DS→GS→PS: Tessellation (new in D3D11).
4. VS→SO, VS→GS→SO, VS→HS→TS→DS→GS→SO: Stream-out (with and without tessellation).
5. CS: Compute. New in D3D11.

And now that you know what’s coming up, let’s get started on vertex shaders!

### Input Assembler stage
The very first thing that happens here is loading indices from the index buffer – if it’s an indexed batch. If not, just pretend it was an identity index buffer (0 1 2 3 4 …) and use that as index instead. If there is an index buffer, its contents are read from memory at this point – not directly though, the IA usually has a data cache to exploit locality of index/vertex buffer access. Also note that index buffer reads (in fact, all resource accesses in D3D10+) are bounds checked; if you reference elements outside the original index buffer (for example, issue a DrawIndexed with IndexCount == 6 from a 5-index buffer) all out-of-bounds reads return zero. Which (in this particular case) is completely useless, but well-defined. Similarly, you can issue a DrawIndexed with a NULL index buffer set – this behaves the same way as if you had an index buffer of size zero set, i.e. all reads are out-of-bounds and hence return zero. With D3D10+, you have to work some more to get into the realm of undefined behavior. :)

Once we have the index, we have all we need to read both per-vertex and per-instance data (the current instance ID is just another counter, fairly straightforward, at this stage anyway) from the input vertex streams. This is fairly straightforward – we have a declaration of the data layout; just read it from the cache/memory and unpack it into the float format that our shader cores want for input. However, this read isn’t done immediately; the hardware is running a cache of shaded vertices, so that if one vertex is referenced by multiple triangles (and in a fully regular closed triangle mesh, each vertex will be referenced by about 6 tris!) it doesn’t need to be shaded every time – we just reference the shaded data that’s already there!

### Vertex Caching and Shading
Note: The contents of this section are, in part, guesswork. They’re based on public comments made by people “in the know” about current GPUs, but that only gives me the “what”, not the “why”, so there’s some extrapolation here. Also, I’m simply guessing some of the details here. That said, I’m not talking completely out of my ass here – I’m confident that what I’m describing here is both reasonable and works (in the general sense), I just can’t guarantee that it’s actually that way in real HW or that I didn’t miss any tricky details. :)

Anyway. For a long time (up to and including the shader model 3.0 generation of GPUs), vertex and pixel shaders were implemented with different units that had different performance trade-offs, and vertex caches were a fairly simple affair: usually just a FIFO for a small number (think one or two dozen) of vertices, with enough space for a worst-case number of output attributes, using the vertex index as a tag. As said, fairly straightforward stuff.

And then unified shaders happened. If you unify two types of shaders that used to be different, the design is necessarily going to be a compromise. So on the one hand, you have vertex shaders, which (at that time) touched maybe up to 1 million vertices a frame in normal use. On the other hand you had pixel shaders, which at 1920×1200 need to touch at least 2.3 million pixels a frame just to fill the whole screen once – and a lot more if you want to render anything interesting. So guess which of the two units ended up pulling the short straw?

Okay, so here’s the deal: instead of the vertex shader units of old that shaded more or less one vertex at a time, you now have a huge beast of a unified shader unit that’s designed for maximum throughput, not latency, and hence wants large batches of work (How large? Right now, the magic number seems to be between 16 and 64 vertices shaded in one batch).

So you need between 16-64 vertex cache misses until you can dispatch one vertex shading load, if you don’t want to shade inefficiently. But the whole FIFO thing doesn’t really play ball with this idea of batching up vertex cache misses and shading them in one go. The problem is this: if you shade a whole batch of vertices at once, that means you can only actually start assembling triangles once all those vertices have finished shading. At which point you’ve just added a whole batch (let’s just say 32 here and in the following) of vertices to the end of the FIFO, which means 32 old vertices now fell out – but each of these 32 vertices might’ve been a vertex cache hit for one of the triangles in the current batch we’re trying to assemble! Uh oh, that doesn’t work. Clearly, we can’t actually count the 32 oldest verts in the FIFO as vertex cache hits, because by the time we want to reference them they’ll be gone! Also, how big do we want to make this FIFO? If we’re shading 32 verts in a batch, it needs to be at least 32 entries large, but since we can’t use the 32 oldest entries (since we’ll be shifting them out), that means we’ll effectively start with an empty FIFO on every batch. So, make it bigger, say 64 entries? That’s pretty big. And note that every vertex cache lookup involves comparing the tag (vertex index) against all tags in the FIFO – this is fully parallel, but it also a power hog; we’re effectively implementing a fully associative cache here. Also, what do we do between dispatching a shading load of 32 vertices and receiving results – just wait? This shading will take a few hundred cycles, waiting seems like a stupid idea! Maybe have two shading loads in flight, in parallel? But now our FIFO needs to be at least 64 entries long, and we can’t count the last 64 entries as vertex cache hits, since they’ll be shifted out by the time we receive results. Also, one FIFO vs. lots of shader cores? [Amdahl’s law](https://en.wikipedia.org/wiki/Amdahl%27s_law) still holds – putting one strictly serial component in a pipeline that’s otherwise completely parallel is a surefire way to make it the bottleneck.

This whole FIFO thing really doesn’t adapt well to this environment, so, well, just throw it out. Back to the drawing board. What do we actually want to do? Get a decently-sized batch of vertices to shade, and not shade vertices (much) more often than necessary.

So, well, keep it simple: Reserve enough buffer space for 32 vertices (=1 batch), and similarly cache tag space for 32 entries. Start with an empty “cache”, i.e. all entries invalid. For every primitive in the index buffer, do a lookup on all the indices; if it’s a hit in the cache, fine. If it’s a miss, allocate a slot in the current batch and add the new index to the cache tag array. Once we don’t have enough space left to add a new primitive anymore, dispatch the whole batch for vertex shading, save the cache tag array (i.e. the 32 indices of the vertices we just shaded), and start setting up the next batch, again from an empty cache – ensuring that the batches are completely independent.

Each batch will keep a shader unit busy for some while (probably at least a few hundred cycles!). But that’s no problem, because we got plenty of them – just pick a different unit to execute each batch! Presto parallelism. We’ll eventually get the results back. At which point we can use the saved cache tags and the original index buffer data to assemble primitives to be sent down the pipeline (this is what “primitive assembly” does, which I’ll cover in the later part).

By the way, when I say “get the results back”, what does that mean? Where do they end up? There’s two major choices: 1. specialized buffers or 2. some general cache/scratchpad memory. It used to be 1), with a fixed organization designed around vertex data (with space for 16 float4 vectors of attributes per vertex and so on), but lately GPUs seem to be moving towards 2), i.e. “just memory”. It’s more flexible, and has the distinct advantage that you can use this memory for other shader stages, whereas things like specialized vertex caches are fairly useless for the pixel shading or compute pipeline, to give just one example.

Update: And here’s a picture(see below) of the vertex shading dataflow as described so far.

![photo 1]({{ IMAGE_PATH }}/vertex_shade.jpg)

### Shader Unit internals
Short versions: It’s pretty much what you’d expect from looking at disassembled HLSL compiler output (fxc /dumpbin is your friend!). Guess what, it’s just processors that are really good at running that kind of code, and the way that kind of thing is done in hardware is building something that eats something fairly close to shader bytecode, in spirit anyway. And unlike the stuff that I’ve been talking about so far, it’s fairly well documented too – if you’re interested, just check out conference presentations from AMD and NVidia or read the documentation for the CUDA/Stream SDKs.

Anyway, here’s the executive summary: fast ALU mostly built around a FMAC (Floating Multiply-ACcumulate) unit, some HW support for (at least) reciprocal, reciprocal square root, log2, exp2, sin, cos, optimized for high throughput and high density not low latency, running a high number of threads to cover said latency, fairly small number of registers per thread (since you’re running so many of them!), very good at executing straight-line code, bad at branches (especially if they’re not coherent).

All that is common to pretty much all implementations. There’s some differences, too; AMD hardware used to stick directly with the 4-wide SIMD implied by the HLSL/GLSL and shader bytecode (even though they seem to be moving away from that lately), while NVidia decided to rather turn the 4-way SIMD into scalar instructions a while back. Again though, all that’s on the Web already!

What’s interesting to note though is the differences between the various shader stages. The short version is that really are rather few of them; for example, all the arithmetic and logic instructions are exactly the same across all stages. Some constructs (like derivative instructions and interpolated attributes in pixel shaders) only exist in some stages; but mostly, the differences are just what kind (and format) of data are passed in and out.

There’s one special bit related to shaders though that’s a big enough subject to deserve a part on its own. That bit is texture sampling (and texture units). Which, it turns out, will be our topic next time! See you then.

### Closing remarks
Again, I repeat my disclaimer from the “Vertex Caching and Shading” section: Part of that is conjecture on my part, so take it with a grain of salt. Or maybe a pound. I don’t know.

I’m also not going into any detail on how scratch/cache memory is managed; the buffer sizes depend (primarily) on the size of batches you process and the number of vertex output attributes you expect. Buffer sizing and management is really important for performance, but I can’t meaningfully explain it here, nor do I want to; while interesting, this stuff is very specific to whatever hardware you’re talking about, and not really very insightful.