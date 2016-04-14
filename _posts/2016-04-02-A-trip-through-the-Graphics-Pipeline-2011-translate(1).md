---
layout: post
title: "(译)3D管线之旅：软件架构"
keywords: ["GPU", "3D管线", "译"]
description: "A trip through the Graphics Pipeline 2011 的翻译"
category: "Unreliable-Translate"
tags: ["GPU", "3D管线", "图形学", "硬件底层"]
---
{% include JB/setup %}

[原文：A trip through the Graphics Pipeline 2011（需要翻墙）](https://fgiesen.wordpress.com/2011/07/01/a-trip-through-the-graphics-pipeline-2011-part-1/)
----------------------------------
## 中文
我写一些关于图形学的东西已经有一段时间了，我设想在这个地方讨论一些2011年以后图形学相关的硬件和软件的知识。你可以用电脑找到一系列关于图形学的功能性描述，不过一般都不解释“How”或者“Why”。我想尝试填补这块空白，并且尽可能通用地描述这些内容，一些特殊的硬件并不在我的讨论范围之内。我将会在文中大量讨论DX11硬件在windows操作系统上跑D3D9/10/11 API的情况，因为这是我最为熟悉的一种形式——当然这并不是说它的API细节。这第一部分，会讲很多实际在GPU上执行的原生指令。

### 应用程序
应用程序是你编写的，所以一定存在你写的BUG（译者批：没有买卖，就没有伤害）。没错，运行时API和驱动也存在BUG，但是你的BUG并不属于这两者，老老实实的去修BUG吧，骚年。

### 运行时API
使用运行时API实现资源创建/渲染状态设置/绘制（Draw Call）的操作，同时API也提供保持追踪你当前APP设置的状态，验证参数有效性或者一些其他错误，一致性检查，管理用户可视资源（译者注：纹理等等），可能会也可能不会验证shader代码和链接（D3D是会去做检查的，但是OpenGL会将这个功能延迟到驱动层）或许还有一些批处理工作的功能，然后递交这一切到图形驱动——更准确的说，是用户态驱动。

### 用户态驱动（UMD）
这是你的图形程序在CPU端所发生的一切中最为神秘的部分。如果你的app因为调用一些API崩溃了，一般是这里出了问题。在N卡中这个部分集成在“nvd3dum.dll”中，在A卡中则是“atiumd*.dll”。从名字上可以看出，这是用户态代码。它和你的程序（包括运行时API）具有相同的上下文并且运行在同一个地址空间中（译者注：这应该和普通的桌面程序没什么两样），并不具有什么特殊的权限（译者注：估计指的是一些操作系统级别的权限）。它实现了一些更底层API（叫做DDI）以供D3D runtime API调用。这些底层API和调用其的D3D runtime API 功能十分相近，但是具有更为精确的功能，比如内存管理等等。

比如在shader编译发生时，D3D传递一个验证过的shader符号流到UMD中——这指的是代码已经被检查不包含任何语法错误并且符合D3D约束标准（使用正确的类型，使用的纹理单元数量没有超过界限，常量缓存的数量也没超过标准，诸如此类）。这个符号流经过HLSL编译器编译并且经过了一些高层次的优化（各种循环优化，无用代码消除，常量传递，分支预测等等）。这对于驱动来说是个好消息，因为这些耗时间的优化已经在编译期间完成了。然而，一些底层的优化（比如寄存器分配和循环展开）要求驱动自己去完成。长话短说，运行时API就是将shader转变到一个中间状态并且再编译一些内容。shader硬件指令已经和D3D生成的字节码十分接近了，所以其实并不需要在写代码时费劲在shader代码中优化细节（HLSL编译器已经做了许多非常有帮助的优化工作），但是这里依旧有许多底层细节（比如硬件资源限制和调度约束）D3D不知道也不关心，它也无需去关心。

当然，如果你的APP是个非常著名的游戏，NV和AMD的程序员或许会查看你的shader并且针对他们的硬件手动优化它们，以此来防止一些尴尬的事情发生:)。这些shader也是在UMD中被发现并且处理，不用客气（译者注：作者是AMD老司机）。

更有趣的是：一些API状态可能会在编译进shader之后直接结束，举个例子，虽然相对特殊（或者说非常少使用）的特性比如纹理边界（texture borders）可能在纹理过滤器（texture sampler）中并没有实现，但是会在shader中用额外的代码来模拟（或者干脆就不支持）。这意味着有时候相同的shader会有多个生成版本，来应对不同的API的状态组合。（译者注：这里的额外代码大概是UMD生成的。）

这也是为什么你经常在使用一个新的shader或者资源的时候产生延迟的原因，许多创建和编译工作会延迟到真正需要的时候才会被执行(惰性执行，你根本不相信有一些app创建了多少无用的垃圾)。所以图形学程序员都应该知道一个常识——如果你知道一些东西确实会被创建（不止是保留一些无用的内存），你或许需要一次废操作（dummy draw call）来唤醒驱动真实加载。这种做法非常不优雅而且很麻烦，但是自从我1999开始写3D程序开始就只有这种做法，所以只能这么用。

不管如何，让我们继续，UMD也要处理一些历史包袱比如D3D9遗留下来的shader版本和固定管线——是的，所有这些都会被D3D完完整整的传递给UMD。3.0 shader model不算太差（事实上相当不错），但是2.0就有点糟糕，1.x的shader简直就是一坨屎——还记不记得大明湖畔的1.3 pixel shader？ 说到这个，固定管线的顶点处理还自带顶点光照，你说糟糕不糟糕。但是这些特性依旧被D3D所支持，而且内置于任何现代图形驱动中，当然所做的是将这些老版本的shader翻译成新版本的shader（已经这样做有相当长的一段时间了）。

现在我想讨论一下UMD所能做到的事情，比如内存管理。举个例子，UMD处理创建纹理的命令，需要分配一块内存空间。实际上，UMD仅仅是在KMD（内核态）分配好的一块大显存中控制一部分。真正的map和unmap操作（还有管理UMD可见的显存，GPU可见的系统内存）这些都是KMD才具有的功能，UMD做不到。

但是UMD还是可以做到一些事情，比如[重组纹理（swizzling textures）](https://fgiesen.wordpress.com/2011/01/17/texture-tiling-and-swizzling/)(译者注：我找到中文的翻译再贴上去，不然我自己翻译也行。)（除非GPU可以用硬件实现这个功能，否则一般来说使用的是2D blitting而不是真正的3D流水线），显存和内存之间传输调度等等。最重要的一点是，它可以写入命令缓存（command buffers）（或者叫DMA缓存，我会在文章中交替使用这两个词）一旦KMD分配了这部分缓存并将权限交出去。一个命令缓存当然包含了许多的命令。包括你所有的状态转换，绘制操作都会被UMD转换成硬件可以理解的命令形式并写入命令缓存。就像有许多东西不需要你关注，比如传输纹理或者shaders到显存，UMD的这些行为也是自动完成的。

一般来说，驱动在设计的时候会尽量的把实际操作用UMD来实现。UMD是用户模式的代码，所以运行的时候不存在切换到核心态的开销，而且可以自由地分配内存，在多线程下工作等等——它就是个dll（哪怕它只是被API所调用，而不是直接被你的App使用）。这对于驱动开发也是有好处的——如果UMD崩溃了，顶多是APP跟着挂掉，并不会影响到整个系统。它可以在系统运行中被替换掉（就是个DLL），可以被调试器（Debugger）跟踪到等等。所以不仅仅是高效而且非常方便。

**我说了是只有一个UMD么？唔，我的意思是有很多个。**

如我所说，UMD就是个DLL。即使它托了D3D的福从而可以直接和KMD交互，但是它始终是个DLL，运行在它被调用处理的地址空间中。

但是，如今我们使用的操作系统都是多任务的，当然我们已经用了好久了。

继续谈谈GPU，GPU其实就是个共享的资源。控制显示的驱动只有一个（哪怕使用了显卡交火技术）。我们有许多app想要访问它（需要伪装成只有一个在访问）。这可不是一个简单的工作，过去的做法是同一时间只给一个3D APP访问的权限，当这个APP处于激活状态，其他的程序就不可以访问驱动。即使你尝试让windows操作系统使用GPU渲染窗口，其他程序的访问也不会真正的停止。这也就是我们为什么需要一些可以任意访问GPU的组件而且这些组件可以分配时间片来进行调度（译者注：指UMD呗）。

### 聊聊调度器
这是一个系统组件——要说明的是我这里指的是图形调度器，而不是CPU和IO调度器。这个东西的作用就是如你所想的那样——它来通过时间片来调度不同APP访问3D流水线。一次上下文的切换至少会改变GPU的一些状态（在命令缓存中生成一些额外的命令）可能还会交换一些资源进出显存。当然任何时间只会有一个进程可以提交命令到3D流水线。你会经常听到一些游戏主机程序员的抱怨这些3D API层次太高了，也不可以手动干涉，不必要的性能开销太大了。但是PC上的不管是3D API还是驱动都要比游戏主机上复杂的多，要处理更多的问题——在PC上真的需要去追踪当前所有的状态，随时随地都有可能在底层出现莫名其妙的问题！而且需要解决应用崩溃并且尝试解决性能问题，没有程序员会乐意自己干这些事的，就算是驱动作者也不乐意，最后还是商业化胜利了，人们期望所有应用可以一个接着一个平滑的运行。虽然这个东西并不算完全正确，但是太过吹毛求疵可交不到朋友。

不管如何，下一个话题：核心态（KMD）。

### 核心态驱动（KMD）
这一部分真正的涉及到硬件了。也许会有许多UMD实例在同一时间运行，但是只会有一个KMD，如果KMD挂了，砰，系统蓝屏。不过现在windows操作系统实际上会知道如何干掉崩溃的驱动，并且重新启动它，如果崩溃发生了，可能导致的问题就不是仅仅一部分内核内存被污染了，可能是整个内核都挂了。

KMD一次性解决所有问题，诸如调度多个程序在争抢唯一的GPU内存，响应发出指令分配物理内存的程序，有些需要在启动的时候初始化GPU，设置显示模式（从显示器获得模式信息），管理硬件鼠标光标（hardware mouse cursor ）（只有一个），管理看门狗定时器（HW watchdog timer）使得GPU可以在一定时间内没有得到响应后被重置，中断响应等等。这些都是KMD在做的。

有时候在视频播放器和GPU之间设置了内容保护，已解码的视频对任何非法用户态代码都不可见，这会导致一些糟糕的操作被禁止，比如将一些信息存储到硬盘，这时候KMD也会参与进来。

对于我们来说最重要的是，KMD管理着真正的命令缓存（command buffer）——真正会执行硬件指令消耗硬件资源的命令。UMD产生的命令缓存（command buffer）并不是真正意义上的命令——事实上，他们本质上只是一些GPU可寻址内存上的随机片段。实际上对于他们来说，UMD产生这些命令，提交他们给调度器，然后等待程序进程被启动将这些命令缓存向KMD传递。然后KMD在主命令缓存中写入一条调用该UMD命令的命令，然后根据GPU命令处理器能否从主内存读到这些命令来确定能否执行，如果需要被执行，当然它需要先用DMA传输到显存。主命令缓存一般来说是一块较小的[环形缓冲区（ring buffer）](https://fgiesen.wordpress.com/2010/12/14/ring-buffers-and-queues/)，所能做的就是一些系统或者初始化的指令，然后调用实实在在的3D命令。
但是这始终只是一块内存。显卡知道它的位置——通常在主命令缓存里有一个指出GPU位置的读指示器（read pointer），还有一个写指示器（write pointer）指向KMD已经被写入的最大部分（更精确的说，是已经让GPU得知的那部分）（译者注：大概是队列末端吧）。这些都是存储在被内存映射的硬件寄存器中——KMD会周期性的更新它们（无论是否提交新的任务）。

### 总线
数据当然不能直接写入显卡（除非显卡和CPU集成在一起），需要通过总线——通常是PCI-E，DMA传输等等。通过同一个线路。这个过程不会太长，但是我想在最后再来讨论它。

### 命令处理器
这是GPU前端部分的一个模块，这个部分读取KMD写入的命令。我会在下一个部分讨论它，毕竟这篇文章已经有些长了。

### 一些题外话：OpenGL
OpenGL的软件栈结构和我之前讨论的非常相似，但是它的API和UMD之间几乎没啥差别。不像D3D，GLSL shader根本不会被API重新编辑，全部都由驱动来进行优化。这个不幸是由于有许多不同的3D厂商实现相同特性时产生了不同的bug和特性造成的。不开玩笑了，这意味着所有关于shader的优化必须驱动看见它的时候才能进行，包括开销大的优化。D3D字节码是非常清爽的一种解决方案，只有一个编译器（不需要去关心不同显卡厂商之间的兼容性问题）并且允许一些开销巨大的数据流分析。

### 补充
这篇文章只是个概述，有许多东西我没有提及。比如，不止一个调度器，有许多个实现（驱动可以选择），到目前为止我也没有解释关于CPU和GPU之间的同步问题等等。也许我还忘记了一些重要的东西—如果是这样，请告诉我我会进行补充，不过现在，再见，希望下次能再看到你。


-------------------------------

## English

It’ s been awhile since I posted something here, and I figured I might use this spot to explain some general points about graphics hardware and software as of 2011; you can find functional descriptions of what the graphics stack in your PC does, but usually not the “how” or “why”; I’ll try to fill in the blanks without getting too specific about any particular piece of hardware. I’m going to be mostly talking about DX11-class hardware running D3D9/10/11 on Windows, because that happens to be the (PC) stack I’m most familiar with – not that the API details etc. will matter much past this first part; once we’re actually on the GPU it’s all native commands.

### The application
This is your code. These are also your bugs. Really. Yes, the API runtime and the driver have bugs, but this is not one of them. Now go fix it already.

### The API runtime
You make your resource creation / state setting / draw calls to the API. The API runtime keeps track of the current state your app has set, validates parameters and does other error and consistency checking, manages user-visible resources, may or may not validate shader code and shader linkage (or at least D3D does, in OpenGL this is handled at the driver level) maybe batches work some more, and then hands it all over to the graphics driver – more precisely, the user-mode driver. The user-mode graphics driver (or UMD) This is where most of the “ magic” on the CPU side happens. If your app crashes because of some API call you did, it will usually be in here :). It’s called “nvd3dum.dll” (NVidia) or “atiumd*.dll” (AMD). As the name suggests, this is user-mode code; it’s running in the same context and address space as your app(and the API runtime) and has no elevated privileges whatsoever. It implements a lower-level API (the DDI) that is called by D3D; this API is fairly similar to the one you’re seeing on the surface, but a bit more explicit about things like memory management and such.

This module is where things like shader compilation happen. D3D passes a pre-validated shader token stream to the UMD – i.e. it’s already checked that the code is valid in the sense of being syntactically correct and obeying D3D constraints (using the right types, not using more textures/samplers than available, not exceeding the number of available constant buffers, stuff like that). This is compiled from HLSL code and usually has quite a number of high-level optimizations (various loop optimizations, deadcode elimination, constant propagation, predicating ifs etc.) applied to it – this is good news since it means the driver benefits from all these relatively costly optimizations that have been performed at compile time. However, it also has a bunch of lower-level optimizations (such as register allocation and loop unrolling) applied that drivers would rather do themselves; long story short, this usually just gets immediately turned into a intermediate representation (IR) and then compiled some more; shader hardware is close enough to D3D bytecode that compilation doesn’t need to work wonders to give good results (and the HLSL compiler having done some of the high-yield and high-cost optimizations already definitely helps), but there’s still lots of low-level details (such as HW resource limits and scheduling constraints) that D3D neither knows nor cares about, so this is not a trivial process.

And of course, if your app is a well-known game, programmers at NV/AMD have probably looked at your shaders and wrote hand-optimized replacements for their hardware – though they better produce the same results lest there be a scandal :). These shaders get detected and substituted by the UMD too. You’re welcome.

More fun: Some of the API state may actually end up being compiled into the shader – to give an example, relatively exotic (or at least infrequently used) features such as texture borders are probably not implemented in the texture sampler, but emulated with extra code in the shader (or just not supported at all). This means that there’s sometimes multiple versions of the same shader floating around, for different combinations of API states.

Incidentally, this is also the reason why you’ ll often see a delay the first time you use a new shader or resource; a lot of the creation/compilation work is deferred by the driver and only executed when it’s actually necessary (you wouldn’t believe how much unused crap some apps create!). Graphics programmers know the other side of the story – if you want to make sure something is actually created (as opposed to just having memory reserved), you need to issue a dummy draw call that uses it to “warm it up”. Ugly and annoying, but this has been the case since I first started using 3D hardware in 1999 – meaning, it’s pretty much a fact of life by this point, so get used to it. :)

Anyway, moving on. The UMD also gets to deal with fun stuff like all the D3D9 “legacy” shader versions and the fixed function pipeline – yes, all of that will get faithfully passed through by D3D. The 3.0 shader profile ain’t that bad (it’s quite reasonable in fact), but 2.0 is crufty and the various 1.x shader versions are seriously whack – remember 1.3 pixel shaders? Or, for that matter, the fixed-function vertex pipeline with vertex lighting and such? Yeah, support for all that’s still there in D3D and the guts of every modern graphics driver, though of course they just translate it to newer shader versions by now (and have been doing so for quite some time).

Then there’ s things like memory management. The UMD will get things like texture creation commands and need to provide space for them. Actually, the UMD just suballocates some larger memory blocks it gets from the KMD (kernel-mode driver); actually mapping and unmapping pages (and managing which part of video memory the UMD can see, and conversely which parts of system memory the GPU may access) is a kernel-mode privilege and can’t be done by the UMD.

But the UMD can do things like [swizzling textures](https://fgiesen.wordpress.com/2011/01/17/texture-tiling-and-swizzling/) (unless the GPU can do this in hardware, usually using 2D blitting units not the real 3D pipeline) and schedule transfers between system memory and (mapped) video memory and the like. Most importantly, it can also write command buffers (or “DMA buffers” – I’ll be using these two names interchangeably) once the KMD has allocated them and handed them over. A command buffer contains, well, commands :). All your state-changing and drawing operations will be converted by the UMD into commands that the hardware understands. As will a lot of things you don’t trigger manually – such as uploading textures and shaders to video memory.

In general, drivers will try to put as much of the actual processing into the UMD as possible; the UMD is user-mode code, so anything that runs in it doesn’t need any costly kernel-mode transitions, it can freely allocate memory, farm work out to multiple threads, and so on – it’s just a regular DLL (even though it’s loaded by the API, not directly by your app). This has advantages for driver development too – if the UMD crashes, the app crashes with it, but not the whole system; it can just be replaced while the system is running (it’s just a DLL!); it can be debugged with a regular debugger; and so on. So it’s not only efficient, it’s also convenient.

But there’ s a big elephant in the room that I haven’ t mentioned yet.

**Did I say “ user-mode driver” ? I meant “ user-mode drivers” .**

As said, the UMD is just a DLL. Okay, one that happens to have the blessing of D3D and a direct pipe to the KMD, but it’s still a regular DLL, and in runs in the address space of its calling process.

But we’ re using multi-tasking OSes nowadays. In fact, we have been for some time.

This “ GPU” thing I keep talking about? That’ s a shared resource. There’ s only one that drives your main display (even if you use SLI/Crossfire). Yet we have multiple apps that try to access it (and pretend they’re the only ones doing it). This doesn’t just work automatically; back in The Olden Days, the solution was to only give 3D to one app at a time, and while that app was active, all others wouldn’t have access. But that doesn’t really cut it if you’re trying to have your windowing system use the GPU for rendering. Which is why you need some component that arbitrates access to the GPU and allocates time-slices and such.

### Enter the scheduler.
This is a system component – note the “the” is somewhat misleading; I’m talking about the graphics scheduler here, not the CPU or IO schedulers. This does exactly what you think it does – it arbitrates access to the 3D pipeline by time-slicing it between different apps that want to use it. A context switch incurs, at the very least, some state switching on the GPU (which generates extra commands for the command buffer) and possibly also swapping some resources in and out of video memory. And of course only one process gets to actually submit commands to the 3D pipe at any given time.

You’ ll often find console programmers complaining about the fairly high-level, hands-off nature of PC 3D APIs, and the performance cost this incurs. But the thing is that 3D APIs/drivers on PC really have a more complex problem to solve than console games – they really do need to keep track of the full current state for example, since someone may pull the metaphorical rug from under them at any moment! They also work around broken apps and try to fix performance problems behind their backs; this is a rather annoying practice that no-one’s happy with, certainly including the driver authors themselves, but the fact is that the business perspective wins here; people expect stuff that runs to continue running (and doing so smoothly). You just won’t win any friends by yelling “BUT IT’S WRONG!” at the app and then sulking and going through an ultra-slow path.

Anyway, on with the pipeline. Next stop: Kernel mode!

### The kernel-mode driver (KMD)
This is the part that actually deals with the hardware. There may be multiple UMD instances running at any one time, but there’s only ever one KMD, and if that crashes, then boom you’re dead – used to be “blue screen” dead, but by now Windows actually knows how to kill a crashed driver and reload it (progress!). As long as it happens to be just a crash and not some kernel memory corruption at least – if that happens, all bets are off.

The KMD deals with all the things that are just there once. There’ s only one GPU memory, even though there’s multiple apps fighting over it. Someone needs to call the shots and actually allocate (and map) physical memory. Similarly, someone must initialize the GPU at startup, set display modes (and get mode information from displays), manage the hardware mouse cursor (yes, there’s HW handling for this, and yes, you really only get one! :), program the HW watchdog timer so the GPU gets reset if it stays unresponsive for a certain time, respond to interrupts, and so on. This is what the KMD does. There’ s also this whole content protection/DRM bit about setting up a protected/DRM’ed path between a video player and the GPU so no the actual precious decoded video pixels aren’t visible to any dirty user-mode code that might do awful forbidden things like dump them to disk (…whatever). The KMD has some involvement in that too.

Most importantly for us, the KMD manages the actual command buffer. You know, the one that the hardware actually consumes. The command buffers that the UMD produces aren’t the real deal – as a matter of fact, they’re just random slices of GPU-addressable memory. What actually happens with them is that the UMD finishes them, submits them to the scheduler, which then waits until that process is up and then passes the UMD command buffer on to the KMD. The KMD then writes a call to command buffer into the main command buffer, and depending on whether the GPU command processor can read from main memory or not, it may also need to DMA it to video memory first. The main command buffer is usually a (quite small) [ring buffer](https://fgiesen.wordpress.com/2010/12/14/ring-buffers-and-queues/) – the only thing that ever gets written there is system/initialization commands and calls to the “real”, meaty 3D command buffers.

But this is still just a buffer in memory right now. Its position is known to the graphics card – there’s usually a read pointer, which is where the GPU is in the main command buffer, and a write pointer, which is how far the KMD has written the buffer yet (or more precisely, how far it has told the GPU it has written yet). These are hardware registers, and they are memory-mapped – the KMD updates them periodically (usually whenever it submits a new chunk of work)…

### The bus
…but of course that write doesn’ t go directly to the graphics card (at least unless it’s integrated on the CPU die!), since it needs to go through the bus first – usually PCI Express these days. DMA transfers etc. take the same route. This doesn’t take very long, but it’s yet another stage in our journey. Until finally…

### The command processor!
This is the frontend of the GPU – the part that actually reads the commands the KMD writes. I’ll continue from here in the next installment, since this post is long enough already :) 

### Small aside: OpenGL OpenGL
is fairly similar to what I just described, except there’ s not as sharp a distinction between the API and UMD layer. And unlike D3D, the (GLSL) shader compilation is not handled by the API at all, it’s all done by the driver. An unfortunate side effect is that there are as many GLSL frontends as there are 3D hardware vendors, all of them basically implementing the same spec, but with their own bugs and idiosyncrasies. Not fun. And it also means that the drivers have to do all the optimizations themselves whenever they get to see the shaders – including expensive optimizations. The D3D bytecode format is really a cleaner solution for this problem – there’s only one compiler (so no slightly incompatible dialects between different vendors!) and it allows for some costlier data-flow analysis than you would normally do.

### Omissions and simplifcations
This is just an overview; there’ s tons of subtleties that I glossed over. For example, there’s not just one scheduler, there’s multiple implementations (the driver can choose); there’s the whole issue of how synchronization between CPU and GPU is handled that I didn’t explain at all so far. And so on. And I might have forgotten something important – if so, please tell me and I’ll fix it! But now, bye and hopefully see you next time.