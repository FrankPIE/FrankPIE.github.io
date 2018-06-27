---
layout: post
title: (译)3D管线之旅：图元装载，裁剪，投影，视口转换
subtitle: (译)3D管线之旅5
description: > 
    A trip through the Graphics Pipeline 2011 的翻译
tags: 
    - GPU
    - 3D管线 
    - 图形学 
    - 硬件底层
hero: /hero/gpu.jpeg
overlay: green
published: false
---
[原文：A trip through the Graphics Pipeline 2011（需要翻墙）](https://fgiesen.wordpress.com/2011/07/05/a-trip-through-the-graphics-pipeline-2011-part-5/)
--------------------------------------------

## 中文

在上次关于纹理采样器的帖子之后，现在我们回到3D管线的前端。我们已经完成了顶点着色的讨论，所以现在我们可以开始实际渲染相关的东西了，对吧？好吧，暂时还不是。 其实在我们实际开始光栅化图元之前，还有一堆工作需要做。事实上太多了，所以在这篇文章我们不会看到任何光栅化的内容，不得不等到下一篇文章了。

### 图元装载
当我们离开顶点处理流水线时，从着色器单元中得到了一个包含已经着色完成顶点数据的内存块，该内存块中包含的图元是绝对完整的。我们不允许三角形，线或者补块（pathces）之类的图元被拆分。这一点很重要，因为这意味着我们可以独立地处理每个顶点内存块，而不需要多缓存一个。当然，我们可以这么做，但是完全没有必要。

下一步是组合属于单个图元的所有顶点（因此称为“图元装载”）。如果该图元恰好是一个点，则读取一个顶点并传递它。如果是线，则读取两个顶点。如果是三角片，则是三个。类似的还有具有大量控制点的补块（patches）。

总之，我们这里的工作就是收集顶点。我们可以通过读取原始索引缓冲区并存储顶点索引缓存位置映射的副本来完成，或者我们可以存储完整的图元索引数据以及相应的着色顶点数据，这可能需要更多的内存空间用于缓存输出，但是这意味着我们不必再次读取原始索引。任何一种工作方式都可以。

现在我们已经展开构成一个图元的所有顶点。换句话说，我们现在有完整的三角片了，而不只是一堆顶点。那么我们可以光栅化它们了吗？还不是。

### 视口剔除与裁剪
是的，我猜我们最好先做这个，啊哈？这一过程的行为和方法都会像你所期待的那样去完成（即在文档中描述的那种方法）。所以我不会在这里解释多边形裁剪，你可以看看任何一本计算机图形学教科书，虽然大多数描述都混乱的可怕。如果你想得到一个很好的解释，查阅Jim Blinn’s （[这本书](https://www.amazon.com/Jim-Blinns-Corner-Graphics-Pipeline/dp/1558603875)的第13章），虽然你可能想要使用他的［0，w］裁剪空间，但是请避免混乱的使用。

无论如何，裁剪的简短的版本是这样的：你的顶点着色器返回同质裁剪空间上的顶点位置。裁剪空间选择尽可能简单的方程描述视锥体。在D3D的情况下，它们是 $$-w \le x \le w$$，$$-w \le y \le w$$，$$0 \le z \le w$$ 和 $$0 < w$$。注意最后一个方程真正做的是排除同质点（0, 0, 0, 0），这是一个简化的方式。

我们首先需要找出三角形是否部分或完全在这些裁剪平面之外。这可以使用[Cohen-Sutherland](https://en.wikipedia.org/wiki/Cohen%E2%80%93Sutherland_algorithm)输出码，这方法非常有效，计算每个顶点的裁剪输出码（或称为裁剪码）（这可以在顶点着色时间完成，并与位置一起储存）。 然后，对于每个图元，裁剪码的按位与操作将告诉你所有的视锥体平面，基元中的所有顶点都在错误的一边（如果是的话，这意味着图元完全在视锥体之外，并且可以丢弃），剪辑代码的按位或操作产生的裁剪码将告诉你所有的视锥体平面需要裁剪的原始对象。给出一个剪辑码，所需要只需要几个逻辑操作的门（gates）硬件。

此外，着色器还可以生成一组“剔除距离”（如果所有顶点的任何一个剔除距离小于零，则舍弃该三角片）和一组“裁剪距离”（其定义附加的剪裁平面）。这些也被用于图元舍弃和裁剪测试。

实际的裁剪过程，可以采取以下两种形式之一：可以使用多边形裁剪算法（它会增加额外的顶点和三角形），或者我们可以为光栅化添加额外的剪裁平面作为边缘方程（如果这对你来说听起来有点混乱，等到下一部分我解释光栅化的时候，你会明白其含义）。虽然后者更优雅，不需要一个实际的多边形裁剪单元，但我们需要具备处理标准化的32位浮点值转换为有效的顶点坐标的能力。虽然建立一个快速的硬件光栅化可能可以做到这一点，但似乎很难说是消耗最小的方法。所以我假设存在一个实际的裁剪单元，涉及所有裁剪流程（包括生成额外的三角形等）。虽然这是一个累赘，但它其实也很少打扰到正常的裁剪流程，所以这不是一个大事。不知道这是一个比较特殊硬件还是也可能就是使用一个着色器单元来做实际剪辑工作。这取决于这个阶段顶点着色器的硬件负载情况，专用剪裁单元有多大，以及需要多少个这样的单元。我不知道这些问题的答案，但至少从性能方面来说，这并不重要：我们不会经常进行裁剪，因为我们可以使用频带保护裁剪（guard-band clipping）。

### 频带保护裁剪(Guard-band clipping)
这个名字会让人产生误解，它不是使用花哨的算法进行裁剪。恰恰相反，它的做法是直接不做裁剪。

基本思想非常简单：局部在裁剪平面之外的图元并不需要裁剪。GPU上的三角形光栅化实际上通过扫描全屏幕区域（更精确地说，剪切矩形区域），并判断每个像素是否被当前三角形所覆盖？（实际上，虽然有比这个复杂一些但更高效的方式，但是这是最常用的作法）。并且这对于完全在视口范围内的三角形以及对于延伸过去的三角形（例如，处于右和上剪切平面）来说也是一样。只要我们的三角形覆盖测试是可靠的，我们完全不需要单独针对左，右，顶部和底部平面各进行一次裁剪！

该测试通常在一些具有固定精度的整数运算中完成。当你将一个三角形的顶点进一步向外移动，最终将会整数溢出，然后你会得到错误的测试结果。那么对于这个结果，我想我们都应该能同意这时候像素应该没有被三角形覆盖，至少，这种危险行为应该是不合法的！实际上它是违反硬件规范的行为。

这个问题有两个解决方案：第一个是确保你的测试不会生成错误结果，无论你输入的怎样的三角形。如果你能处理好这个情况，那么你不需要针对上述四个平面进行裁剪。这被称为“无限频带保护”，好吧，保护频带实际上就是无限的。方案二是当三角形即将超出光栅器计算不能溢出的安全范围的时候进行裁剪。例如，假设您的光栅化器有足够的数据空间来处理坐标范围为$$-32768 \le X \le 32767$$，$$-32768 \le Y \le 32767$$的三角形（我将一直使用大写的X和Y表示屏幕空间坐标）。虽然看起来仍然在使用视平面进行视口剔除（即“三角形是否在视锥体之外”），但实际上仅是针对被选择的保护带裁剪平面进行裁剪，以便在进行了投影和视口转换之后，坐标还处于安全范围内。我想是时候给一张图了：

![Guard-band clipping illustration]({{ IMAGE_PATH }}/guardband_clip.png)
<center>频带保护裁剪</center>

如图所示，中间的蓝色轮廓白色小矩形表示我们的视口，而其周围的三文鱼色区域是我们的频带保护裁剪平面。虽然图上看起来视口非常小，但我实际上我是选择了一个巨大的视口，以致于你可以看到所有东西！如果我们使用-32768 ~ 32767整数范围作为频带保护裁剪范围，那么视口的大小将是大约5500像素大小，是的，图里的三角形其实是十分巨大的。无论如何，各个三角形分别显示了一些不同情况。黄色三角形是最常见的情况，一个延伸到视口外但没有超过保护频带裁剪面的三角形。这样的情况直接通过，不需要进一步处理。绿色三角形完全在保护带裁剪面内，但在视口区域外，所以它被视口剔除。蓝色三角形延伸到防护带剪辑区域之外，所以需要被剪裁，但是它又完全位于视口区域之外，所以被视口剔除。最后，紫色三角形在视口内部和保护带外部延伸，因此是实际上需要被剪裁的三角形。

正如你看到的，实际上三角形必须被裁剪是非常极端的情况。正如上面说的，很少见，没必要担心。

### 进行正确的裁剪
如果你熟悉算法，至少这的内容不应该令你觉得惊讶，也不应该让你觉得听起来太难。但是细节总是很烦人的。在实践中，三角形裁剪器必须服从一些不显而易见的规则。如果它破坏了这些规则中的任何一个，有时会在共享边缘的相邻三角形之间产生裂缝。这是不允许的。

* 视锥体内的顶点位置必须由裁剪器进行保存，位准确。

* 沿着平面裁剪边AB与裁剪边BA（定向反转）必须产生相同的结果，位精确。（这可以通过使数学上完全对称，或者总是沿着相同方向进行裁剪来确保）。

* 剪切到多个平面的基元必须始终以相同的顺序与平面进行裁剪。（或者一次裁剪所有平面）

* 如果你使用保护带，你必须针对保护带平面进行裁剪，虽然你不能对一些三角形使用保护带，但如果实际上需要裁剪，则会对原始视口平面进行裁剪。不这样就做会导致产生缝隙，如果我没记错的话，在过去实际上有一块图形硬件存在这个bug。

### 那些令人讨厌的远近裁剪面

好吧，虽然我们对上下左右四个平面有一个非常好的解决方案，但是远近平面咋办？特别是近平面很麻烦，因为所有的东西，只是有一点处于视口外，近平面就需要做最多的裁剪。那么我们能怎么做？z保护带平面？但是，这将如何工作，我们实际上不是沿z轴进行光栅化的！事实上，它只是我们内插在三角形的一些值，哔-(人工消音)！

正面的说，虽然它只是我们内插在三角形的一些值。但是事实上，对于z-near测试（$$ Z <0 $$）来说，一旦你插入Z值后真的很容易做，因为它就是个符号位。z-far测试（$$ Z> 1 $$）是一个特别的比较（不是我在这里使用Z而不是z，是因为这些是“屏幕”或投影后的坐标）。但是，我们仍然对每个像素做Z测试，所以这不会是一个较大的额外消耗。用这种方式进行z平面裁剪是一个可靠的方式。但是如果你想支持像NV的‘depth clamp’OpenGL扩展，你需要能够跳过z-near/z-far裁剪。事实上，我认为这个扩展的存在是一个很好的暗示，说明他们正在做这个事，或至少已经使用了一段时间了。

所以我们对规则裁剪平面之一下了个定义：$$ 0 < w $$。我们可以摆脱它吗？答案是肯定的，使用工作在齐次坐标中的光栅化算法，例如，[这个](http://www.cs.unc.edu/~olano/papers/2dh-tri/)。我不知道是否有硬件使用了这个。虽然这是一个优雅的方式，但该算法似乎很难服从（非常严格！）D3D11光栅化规则。但也许有一些很酷的技巧，我不知道。无论如何，这是关于远近平面的剪裁。

### 投影与视口转换

投影只是采取x，y和z坐标，并将它们除以w（除非你使用一个均匀的光栅化器实际上没有投影 - 但我会忽略这种可能性在下面）。这给出了-1和1之间的标准化设备坐标或NDC。然后应用视口变换，将投影的x和y映射到像素坐标（我将称为X和Y），将投影的z映射到范围[ 0,1]（我将称为该值Z），使得在z接近平面Z = 0并且在z远平面Z = 1。

在这一点上，我们还将像素捕捉到子像素网格上的分数坐标。从D3D11开始，硬件必须具有三角坐标的精确的8位子像素精度。这种捕捉将一些非常薄的条纹（否则会导致问题）转变为退化的三角形（根本不需要渲染）。

### 背面和其他三角形剔除
一旦我们有所有顶点的X和Y，我们可以使用边缘向量的叉积来计算带符号的三角形区域。如果面积为负，则三角形逆时针（这里，负区域对应于逆时针，因为我们现在在像素坐标空间中，并且在D3D像素空间y向下不向上增加，因此符号被反转） 。如果面积为正，则顺时针缠绕。如果它是零，它是退化，不覆盖任何像素，所以它可以安全地剔除。在这一点上，我们知道三角形的方向，所以我们可以做背面剔除（如果启用）。

就是这样！我们现在已准备好进行光栅化...。事实上，我们必须先做三角形设置。但这需要一些知识如何进行光栅化，所以我会把它，直到下一部分...看到你！

### 最后的评论
同样，我跳过了一些部分，简化了其他部分，所以这里通常提醒，事情在现实中有点复杂：例如，我假装你只是使用常规均匀剪辑算法。大多数情况下，你可以 - 但是你可以有一些顶点着色器属性被标记为使用屏幕空间线性，而不是透视正确插值。现在，规则均匀剪辑总是进行透视正确插值;在屏幕空间线性属性的情况下，你实际上需要做一些额外的工作，使它不透视正确。 :)

我在某些时候谈论原语，但是大多数时候我只是关注三角形。点和线不是很难，但让我们说实话，他们不是我们在这里。如果你有兴趣，你可以找出细节。 :)

有很多的光栅化算法，有些（像我引用的Olanos 2DH方法）允许你跳过几乎所有的剪裁，但正如我所提到的，D3D11对三角形光栅器有非常严格的要求，所以没有太多的摆动空间HW实现;我不知道这些方法是否可以调整到完全遵循规范（有很多微妙的点，我会覆盖下一次）。所以在这里和下面我假设你不能做超时尚的事情;然后再次，我运行的不那么光滑的方法在光栅器中每个像素的数学略少，所以他们可能会赢得硬件实现反正。当然，我可能会失去魔法小精灵尘埃在角落里解决所有这些问题。这在图形中经常出现。如果你知道一个真棒的解决方案，给我一个喊在评论！

最后，我在这里描述的三角形剔除是最小的;例如，在光栅化时将产生零像素的三角形的类比仅仅零面积的tris大得多，并且如果你能够足够快地（或具有足够的门）找到它，则可以立即放下三角形， t需要经过三角形设置。这是最后一点，你可以在通过三角形设置和至少一些栅格化之前廉价剔除 - 找到其他方式早早拒绝tris在这里慷慨解囊


--------------------------------------------

## English

After the last post about texture samplers, we’re now back in the 3D frontend. We’re done with vertex shading, so now we can start actually rendering stuff, right? Well, not quite. You see, there’s a bunch still left to do before we actually start rasterizing primitives. So much so in fact that we’re not going to see any rasterization in this post – that’ll have to wait until next time.

### Primitive Assembly
When we left the vertex pipeline, we had just gotten a block of shaded vertices back from the shader units, with the implicit promise that this block contains an integral number of primitives – i.e., we don’t allow triangles, lines or patches to be split across multiple blocks. This is important, because it means we can truly treat each block independently and never need to buffer more than one block of shader output – we can, of course, but we don’t have to.

The next step is to assemble all the vertices belonging to a single primitive (hence “primitive assembly”). If that primitive happens to be a point, this just reads exactly one vertex and passes it on. If it’s lines, it reads two vertices. If it’s triangles, three. And so on for patches with larger numbers of control points.

In short, all that happens here is that we gather vertices. We can either do this by reading the original index buffer and keeping a copy of our vertex index->cache position map around (as I described), or we can store the indices for the fully expanded primitives along with the shaded vertex data, which might take a bit more space for the output buffer but means we don’t have to read the indices again here. Either way works fine.

And now we have expanded out all the vertices that make up a primitive. In other words, we now have complete triangles, not just a bunch of vertices. So can we rasterize them already? Not quite.

### Viewport culling and clipping
Oh yeah, that. Yeah, I guess we’d better do that first, huh? This is one part of pipeline that really does exactly what you’d expect, pretty much the way you would expect it too (i.e. the way it’s explained in the docs). So I’m not gonna explain polygon clipping in general here, you can look that up in any computer graphics textbook, although most make a terrible mess of it; if you want a good explanation, use Jim Blinn’s (chapter 13 of [this book](https://www.amazon.com/Jim-Blinns-Corner-Graphics-Pipeline/dp/1558603875)), although you probably want to pass on his alternative [0,w] clip space these days, to avoid confusion if nothing else.

Anyway, clipping. The short version is this: Your vertex shader returns vertex positions on homogeneous clip space. Clip space is chosen to make the equations that describe the view frustum as simple as possible; in the case of D3D, they are $$ -w \le x \le w $$, $$ -w \le y \le w $$, $$ 0 \le z \le w $$, and $$ 0 < w $$; note that all the last equation really does is exclude the homogeneous point (0,0,0,0), which is something of a degenerate case.

We first need to find out if the triangle is partially or even completely outside any of these clip planes. This can be done very efficiently using [Cohen-Sutherland](https://en.wikipedia.org/wiki/Cohen%E2%80%93Sutherland_algorithm)-style out-codes. You compute the clip out-code (or just clip-code) for each vertex (this can be done at vertex shading time and stored along with the positions, for example). Then, for each primitive, the bitwise AND of the clip-codes will tell you all the view-frustum planes that all vertices in the primitive are on the wrong side of (if there’s any, that means the primitive is completely outside the view frustum and can be thrown away), and the bitwise OR of the clip-codes will tell you the planes that you need to clip the primitive against. Given the clipcodes, all this is just a few gates worth of hardware – simple stuff.

Additionally, the shaders can also generate a set of “cull distances” (a triangle will be discarded if any one cull distance for all vertices is less than zero), and a set of “clip distances” (which define additional clipping planes). These get considered for primitive rejection/clip testing too.

The actual clipping process, if invoked, can take one of two forms: we can either use an actual polygon clipping algorithm (which adds extra vertices and triangles), or we can add the clipping planes as extra edge equations to the rasterizer (if that sounds like gibberish to you, wait until the next part where I explain rasterization – it’ll ask make sense eventually). The latter is more elegant and doesn’t require an actual polygon clipper at all, but we need to be able to handle all normalized 32-bit floating point values as valid vertex coordinates; there might be a trick for building a fast HW rasterizer that does this, but it seems tricky to say the least. So I’m assuming there’s an actual clipper, with all that involves (generation of extra triangles etc). This is a nuisance, but it’s also very infrequent (more so than you think, I’ll get to that in a second), so it’s not a big deal. Not sure if that’s special hardware either, or if that path grabs a shader unit to do the actual clipping; depends on whether dispatching a new vertex shading load at this stage is awkward or not, how big a dedicated clipping unit is, and how many of them you need. I don’t know the answer to these questions, but at least from the performance side of things, it doesn’t much matter: we don’t really clip that often. That’s because we can use guard-band clipping.

### Guard-band clipping
The name is something of a misnomer; it’s not a fancy way of doing clipping. In fact, it’s quite the opposite: a straight-forward way of not doing clipping. :)

The underlying idea is very simple: Most primitives that are partially outside the left, right, top and bottom clip planes don’t need to be clipped at all. Triangle rasterization on GPUs works by, in effect, scanning over the full screen area (or more precisely, the scissor rect) and asking for every pixel: “is this pixel covered by the current triangle?” (In reality it’s a bit more complicated and way more efficient than that, but that’s the general idea). And that works just as well for triangles completely within the viewport as it does for triangles that extend past, say, the right and top clipping planes. As long as our triangle coverage test is reliable, we don’t need to clip against the left, right, top and bottom planes at all!

That test is usually done in integer arithmetic with some fixed precision. And eventually, as you move say one triangle vertex further and further out, you’ll get integer overflows and wrong test results. I think we can all agree that the rasterizer producing pixels that aren’t actually inside the triangle is, at the very least, extremely offensive behavior and should be illegal! Which it in fact is – hardware that does this is in violation of the spec.

There’s two solutions for this problem: The first is to make sure that your triangle tests never, ever generate the wrong results, no matter how your input triangle looks. If you manage that, then you don’t ever need to clip against the aforementioned four planes. This is called “infinite guard-band” because, well, the guard-band is effectively infinite. Solution two is to clip triangles eventually, just as they’re about to go outside the safe range where the rasterizer calculations can’t overflow. For example, say that your rasterizer has enough internal bits to deal with integer triangle coordinates that have $$ -32768 \le X \le 32767 $$, $$ -32768 \le Y \le 32767 $$ (note I’m using capital X and Y to denote screen-space positions; I’ll stick with this convention). You still do your viewport cull test (i.e. “is this triangle outside the view frustum”) with the regular view planes, but only actually clip against the guard-band clip planes which are chosen so that after the projection and viewport transforms, the resulting coordinates are in the safe range. I guess it’s time for an image:

![Guard-band clipping illustration]({{IMAGE_PATH}}/guardband_clip.png)
<center>Guard-band clipping</center >

The small white rectangle with blue outline that’s roughly in the middle represents our viewport, while the big salmon-colored area around it is our guard band. It looks like a small viewport in this image, but I actually picked a huge one so you can see anything! With our -32768 .. 32767 guard-band clip range, that viewport would be about 5500 pixels wide – yes, that’s some huge triangles right there :). Anyway, the triangles show off some of the important cases. The yellow triangle is the most common case – a triangle that extends outside the viewport but not the guard band. This just gets passed straight through, no further processing necessary. The green triangle is fully within the guard band, but outside the viewport region, so it would never get here – it’s been rejected above by the viewport cull. The blue triangle extends outside the guard-band clip region and would need to be clipped, but again it’s fully outside the viewport region and gets rejected by the viewport cull. Finally, the purple triangle extends both inside the viewport and outside the guard band, and so actually needs to be clipped.

As you can see, the kinds of triangles you need to actually have to clip against the four side planes are pretty extreme. As said, it’s infrequent – don’t worry about it.

### Aside: Getting clipping right
None of this should be terribly surprising; nor should it sound too difficult, at least if you’re familiar with the algorithms. But the devil’s in the details, always. Here’s some of the non-obvious rules the triangle clipper has to obey in practice. If it ever breaks any of these rules, there’s cases where it will produce cracks between adjacent triangles that share an edge. This isn’t allowed.

* Vertex positions that are inside the view frustum must be preserved, bit-exact, by the clipper.

* Clipping an edge AB against a plane must produce the same results, bit-exact, as clipping the edge BA (orientation reversed) against that plane. (This can be ensured by either making the math completely symmetric, or always clipping an edge in the same direction, say from the outside in).

* Primitives that are clipped against multiple planes must always clip against planes in the same order. (Either that or clip against all planes at once)

* If you use a guard band, you must clip against the guard band planes; you can’t use a guard band for some triangles but then clip against the original viewport planes if you actually need to clip. Again, failing to do this will cause cracks – and if I remember correctly there was actually a piece of graphics hardware in the bad old days that shipped with this bug enshrined in silicon. Oops. :)

### Those pesky near and far planes
Okay, so we have a really nice quick solution for the 4 side planes, but what about near and far? Particularly the near plane is bothersome, since with all the stuff that’s only slightly outside the viewport handled, that’s the plane we do most of our clipping for. So what can we do? A z guard band? But how would that work – we’re not actually rasterizing along the z axis at all! In fact, it’s just some value we interpolate over the triangle, damn!

On the plus side, though, it’s just some value we interpolate over the triangle. And in fact the z-near test ($$ Z < 0 $$) is really easy to do once you interpolate Z – it’s just the sign bit. z-far ($$ Z > 1 $$) is an extra compare though (not I’m using Z not z here, i.e. these are “screen” or post-projection coordinates). But still, we’re doing Z-compares per pixel anyway (Z test!), so it’s not a big extra expense. It depends, but doing z-clip this way is definitely an option. And you need to be able to skip z-near/z-far clipping if you want to support things like NVidias ‘depth clamp’ OpenGL extension; in fact, I would argue the existence of that extension is a pretty good hint that they’re doing this, or at least used to for a while.

So we’re down to one of the regular clip planes: $$ 0 < w $$. Can we get rid of this one too? The answer is yes, with a rasterization algorithm that works in homogeneous coordinates, e.g. [this one](http://www.cs.unc.edu/~olano/papers/2dh-tri/). I’m not sure whether hardware uses that one though. It’s nice an elegant, but it seems like it would be hard to obey the (very strict!) D3D11 rasterization rules to the letter using that algorithm. But maybe there’s some cool tricks that I’m not aware of. Anyway, that’s about it with clipping.

### Projection and viewport transform
Projection just takes the x, y and z coordinates and divides them by w (unless you’re using a homogeneous rasterizer which doesn’t actually project – but I’ll ignore that possibility in the following). This gives us normalized device coordinates, or NDCs, between -1 and 1. We then apply the viewport transform which maps the projected x and y to pixel coordinates (which I’ll call X and Y) and the projected z into the range [0,1] (I’ll call this value Z), such that at the z-near plane Z=0 and at the z-far plane Z=1.

At this point, we also snap pixels to fractional coordinates on the sub-pixel grid. As of D3D11, hardware is required to have exactly 8 bits of subpixel precision for triangle coordinates. This snapping turns some very thin slivers (which would otherwise cause problems) into degenerate triangles (which don’t need to be rendered at all).

### Back-face and other triangle culling
Once we have X and Y for all vertices, we can calculate the signed triangle area using a cross product of the edge vectors. If the area is negative, the triangle is wound counter-clockwise (here, negative areas correspond to counter-clockwise because we’re now in the pixel coordinate space, and in D3D pixel space y increases downwards not upwards, so signs are inverted). If the area is positive, it’s wound clockwise. If it’s zero, it’s degenerate and doesn’t cover any pixels, so it can be safely culled. At this point, we know the triangle orientation so we can do back-face culling (if enabled).

And that’s it! We’re now ready for rasterization… almost. Actually we have to do triangle setup first. But doing that requires some knowledge of how rasterization will be performed, so I’ll put that off until the next part… see you then!

### Final remarks
Again, I skipped some parts and simplified others, so here’s the usual reminder that things are a bit more complicated in reality: For example, I pretended that you just use the regular homogeneous clipping algorithm. Mostly, you do – but you can have some vertex shader attributes flagged as using screen-space linear instead of perspective-correct interpolation. Now, the regular homogeneous clip always does perspective-correct interpolation; in the case of screen-space linear attributes, you actually need to do some extra work to make it not perspective-correct. :)

I talk about primitives some of the time, but mostly I’m just focusing on triangles here. Points and lines aren’t hard, but let’s be honest, they’re not what we’re here for either. You can work out the details if you’re interested. :)

There’s tons of rasterization algorithms out there, some of which (like Olanos 2DH method that I cited) allow you to skip nearly all clipping, but as I mentioned, D3D11 has very strict requirements on the triangle rasterizer so there’s not much wiggle room for HW implementations; I’m not sure if those methods can be tweaked to exactly follow the spec (there’s a lot of subtle points that I’ll cover next time). So here and in the following I’m assuming you can’t do the ultra-sleek thing; then again, the not-quite-so-sleek approaches I’m running with have slightly less math per pixel in the rasterizer, so they might win for HW implementations anyway. And of course I might be missing the magic pixie dust right around the corner that solves all of these problems. That occurs surprisingly often in graphics. If you know an awesome solution, give me a shout in the comments!

Lastly, the triangle culling I’m describing here is the bare minimum; for example, the class of triangles that will generate zero pixels upon rasterization is much larger than just zero-area tris, and if you can find it out quickly enough (or with few enough gates), you can drop the triangle immediately and don’t need to go through triangle setup. This is the last point where you can cull cheaply before going through triangle setup and at least some rasterization – finding other ways to early-reject tris pays off handsomely here.
