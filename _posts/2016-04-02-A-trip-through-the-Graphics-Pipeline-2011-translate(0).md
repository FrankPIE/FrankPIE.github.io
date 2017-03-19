---
layout: post
title: "(译)3D管线之旅：索引"
short_title: "(译)3D管线之旅0"
keywords: ["GPU", "3D管线", "译"]
description: "A trip through the Graphics Pipeline 2011 的翻译"
category: "Unreliable-Translate"
tags: ["GPU", "3D管线", "图形学", "硬件底层"]
---
{% include JB/setup %}

[原文：A trip through the Graphics Pipeline 2011（需要翻墙）](https://fgiesen.wordpress.com/2011/07/09/a-trip-through-the-graphics-pipeline-2011-index/)
----------------------------------

## 中文

### 欢迎诸位

这是我目前正在写的关于GPU上实现D3D/OpenGL图形管线的一系列Blog文章的索引页，许多关于这方面的知识已经在图形学程序员间广为流传，并且有许多的论文从不同的角度讨论它们，但是有一点令我颇为恼火的是这些文章不是广泛的概述就是非常细节的讨论个别的部分，提供的知识都不够详细，并且大部分还有一些过时。

写这个系列的目的在于帮助那些打算更好的了解现代3D API（至少是OpenGL 2.0+和D3D9+）和想要理解底层机制的图形学程序员，所以这并不是一份关于图形学管线的新手教程。如果你还没有使用过任何一种3D API，这里大部分东西对于你来说并没有作用。并且我假设你了解当代的硬件设计——你至少需要了解寄存器，[先入先出存储器（FIFOs)](http://electronics.stackexchange.com/questions/97280/trying-to-understand-fifo-in-hardware-context)，caches缓存和管道流水线，并且理解他们是如何工作的。最后，你需要理解一些基础的并行编程机制。GPU是一种大型并行计算单元，没有办法避开这点来讨论其他内容。

一些读者评论说这样描述图形流水线和GPU过于底层，好吧，其实这取决于你站在什么角度来看待这个问题。GPU架构师认为这些知识是高层次的讨论。但是这里所说的高层次讨论并不是当一块新GPU发布时你在网页上看到的那些图表。老实讲，这种报告所涵盖的信息太少了，根本没有真正的去解释其究竟是如何工作的，这只是一种单纯的对新技术的炫耀。好吧，我尝试在这里更接地气一些，也就是说不会有那么多好看的色彩和标准结果，取而代之的是大量的文字，少量色彩单调的图表和一些公式（逃。如果你觉得这些都不成问题，那么敬请期待。

**译者五一更新：我其实翻译的时候一直不太理解FIFO的概念，但是因为翻译到后期这个词的频率出现的越来越多，所以特意去查了资料，仔细一看，果然硬件上的先入先出的定义和软件上的有很大的不同，更新了链接，方便理解。**

-------------------------------

## English

### Welcome

This is the index page for a series of blog posts I’ m currently writing about the D3D/OpenGL graphics pipelines as actually implemented by GPUs. A lot of this is well known among graphics programmers, and there’s tons of papers on various bits and pieces of it, but one bit I’ve been annoyed with is that while there’s both broad overviews and very detailed information on individual components, there’s not much in between, and what little there is is mostly out of date.

This series is intended for graphics programmers that know a modern 3D API (at least OpenGL 2.0+ or D3D9+) well and want to know how it all looks under the hood. It’ snot a description of the graphics pipeline for novices; if you haven’t used a 3D API, most if not all of this will be completely useless to you. I’m also assuming a working understanding of contemporary hardware design – you should at the very least know what registers, FIFOs, caches and pipelines are, and understand how they work. Finally, you need a working understanding of at least basic parallel programming mechanisms. A GPU is a massively parallel computer, there’s no way around it.

Some readers have commented that this is a really low-level description of the graphics pipeline and GPUs; well, it all depends on where you’ re standing. GPU architects would call this a highlevel description of a GPU. Not quite as high-level as the multicolored flowcharts you tend to see on hardware review sites whenever a new GPU generation arrives; but, to be honest, that kind of reporting tends to have a very low information density, even when it’s done well. Ultimately, it’s not meant to explain how anything actually works – it’s just technology porn that’s trying to show off shiny new gizmos. Well, I try to be a bit more substantial here, which unfortunately means less colors and less benchmark results, but instead lots and lots of text, a few mono-colored diagrams and even some (shudder) equations. If that’s okay with you, then here’s the index:

-------------------------------------------------

* [Part 1: Introduction：the Software stack](http://madstrawberry.me/unreliable-translate/A-trip-through-the-Graphics-Pipeline-2011-translate(1).html)

* [Part 2: GPU memory architecture and the Command Processor](http://madstrawberry.me/unreliable-translate/A-trip-through-the-Graphics-Pipeline-2011-translate(2).html)

* [Part 3: 3D pipeline overview, vertex processing](http://madstrawberry.me/unreliable-translate/A-trip-through-the-Graphics-Pipeline-2011-translate(3).html)

* [Part 4: Texture samplers](http://madstrawberry.me/unreliable-translate/A-trip-through-the-Graphics-Pipeline-2011-translate(4).html)

* Part 5: Primitive Assembly, Clip/Cull, Projection, and Viewport transform.

* Part 6: (Triangle) rasterization and setup.

* Part 7: Z/Stencil processing, 3 different ways.

* Part 8: Pixel processing – “fork phase”.

* Part 9: Pixel processing – “join phase”.

* Part 10: Geometry Shaders.

* Part 11: Stream-Out.

* Part 12: Tessellation.

* Part 13: Compute Shaders.

