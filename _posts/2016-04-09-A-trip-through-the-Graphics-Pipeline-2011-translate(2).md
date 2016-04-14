---
layout: post
title: "(译)3D管线之旅：GPU内存架构与命令处理器"
keywords: ["GPU", "3D管线", "译"]
description: "A trip through the Graphics Pipeline 2011 的翻译"
category: "Unreliable-Translate"
tags: ["GPU", "3D管线", "图形学", "硬件底层"]
---
{% include JB/setup %}

[原文：A trip through the Graphics Pipeline 2011（需要翻墙）](https://fgiesen.wordpress.com/2011/07/02/a-trip-through-the-graphics-pipeline-2011-part-2/)
--------------------------------------------
## 中文

### 停一下
在之前的部分我解释了3D渲染指令在被传递到GPU之前的多个阶段，简单说，这些过程远比你想的要复杂的多（译者吐槽：你牛逼，你牛逼）。在文章的最后我提到了命令处理器（command processor），以及为了让它最后能处理命令缓存（command buffer）而做的细致准备。好吧，让我怎么开口呢，我现在不准备直接就讨论关于命令处理器的内容，是的，我欺骗了你。虽然我们确实会在这部分第一次看到关于命令处理器的内容，但是千万要注意一点，命令缓存处于内存——不管是通过PCI-E访问的系统内存还是显存。我希望按照顺序解释这些，所以在说命令处理器之前，我要先和你聊聊关于内存的东西。（译者注：这里的内存是包括系统内存和显存的全部存储空间）。

### 内存子系统（memory subsystem）
GPU有一个特殊的内存子系统，和一般的CPU内存子系统或者其他硬件的内存子系统都有所不同，这主要取决于GPU的工作性质。有别于你所熟悉的内存体系，主要有如下两个不同点：

GPU在带宽（bandwidth）方面超级快，Core i7 2600k的带宽大概是19GB/s，这已经可以称得上是非常快了，真是在出生在了一个好时代。然而，一块GTX480显卡的总带宽却惊人的接近180GB/s，差了N倍。

GPU在缓存(Cache)方面超级慢，一次cache miss相对于一块Nehalem架构（第一代i系列处理器架构）的CPU来说将会消耗大概140个时钟周期左右的时间，如果你对这点存在质疑，这里有一份[技术报告](http://www.anandtech.com/show/2542/5)。而同样的情况对于GTX480来说大概会[消耗400-800个时钟周期](../assets/pdf/Rennich_2011_04_25.pdf)。从测试的结果上来讲，GTX480相对于Intel i7会有平均大概4倍多的内存延迟。除了这些，从主频上来说，i7的主频是2.93GHz，GTX 480的主频只有1.4GHz，这里还存在两倍左右的性能差异，惊人的差异。

等等，我好像发现了一些好玩的事情，怎么不按套路出牌呢！做硬件的又不是白痴，这里存在着一些性能上的权衡？

没错，GPU提升了大量的带宽上限，作为代价，延迟也大量提升了（事实证明，这也大大加大了功耗，不过这并不是我们这篇文章讨论的范围）。一般来讲，GPU被设计成吞吐量超过延迟，所以不需要也不应该同步的等待GPU的计算结果，在结果出现前完全可以异步的做一些其他的工作。（译者注：CPU和GPU是异步工作的，GPU还是大并发的）。

除了一些关于DRAM的内容，以上内容几乎就是你需要了解关于GPU内存的一切了。现在让我们着重的说说DRAM，DRAM不论在物理结构还是逻辑结构上都被设计成了2D网状结构，在行列的交叉处存在一个晶体管和一个电容器，如果你想知道如何利用这些材料自己造DRAM，请去参阅[维基百科](http://en.wikipedia.org/wiki/DRAM#Operation_principle)。总之，这里的重点是——在DRAM中地址被分为行地址和列地址，而且DRAM内部的读写总是以行为单位完成的。这就意味着访问一行数据的开销小于访问同样大小的跨越多行的数据，似乎听起来这只是一个关于DRAM的冷知识，但是注意，当你把这点和我之前所提到的那些内容联系起来看的话，会发现如果只是从内存中读取一些零碎的东西，那么将无法将显卡的带宽能力完全发挥出来，那么你的GPU将优势全无，如果你想要充分发挥显卡的带宽优势，每次最好以行为单位操作数据。

### PCI-E接口（host interface）
从一个图形学程序员的角度来看，实在是对这部分硬件无法产生兴趣。事实上，这点就算对于GPU硬件架构师来说也是一样的。但是问题在于，你得注意不要让它成为你应用的性能瓶颈，所以你应该妥善的使用它，不要让悲剧发生。除此之外，PCI-E使得CPU可以直接访问一些显存和GPU的寄存器，GPU也可以直接读写一部分系统内存。而最让人头疼的是这个传输的延迟要比内存延迟要高得多，因为信号必须被发送到芯片外，在主板上游走大概“一周”时间才能到达目的地（跟在CPU或者GPU内部的速度比较起来看是这样的）。目前16路PCI-E 2.0接口的最大带宽大约在8GB/s（理论上来说），大概是CPU的带宽的一半或者三分之一左右，这是一个可使用的比例。而且PCI-E标准不像早期的AGP标准只有CPU通向GPU的通道，没有反向的通道，PCI-E标准是一种对称的点对点连接，数据可以双向通行。

译者的话：一般来说PCI-E的慢算是老生常谈的一个话题了，而且在程序优化的过程中也是十分需要注意这点的，总之不能频繁的使用让GPU和CPU之间的数据传输，这会极大的限制你程序的性能:)

### 其他一些关于内存的零碎知识
讲真，我们很快就要揭开3D命令的真面目了！我们真的已经非常接近他们了。但是我们还是要先了解一件别的事，因为我们现在有两种内存需要GPU处理——显存(本地的)和被映射的系统内存，访问前者的速度要显著快于后者，如果访问前者是一场“一日游”的话，那么通过PIC-E高速公路访问后者显然要花上“一周”时间。所以问题来了，我们该如何选择？

最简单的解决方案：加一条额外的地址线来告诉你选择哪种方式。这很简单，而且也广泛使用。或者你使用的是一种的统一内存架构（GPU和CPU集成在一起，并且共享内存），就像一些游戏主机。在这个情况下，并没有什么额外的选择，只有内存。如果你是发烧友，那你可以加入MMU（memory management unit），这个组件允许显存与内存地址虚拟化并处在同一个地址空间内，然后你甚至可以利用它完成一些看起来很不可思议的事情，比如频繁的访问显存中一块纹理的某个部分（这访问速度其实还是相当快的）。说些题外话，比如访问一些完全没有被GPU映射到的系统内存——凭空创造东西出来，或者，更一般情况就是通过硬盘读取内容，访问这些内容大概只要花费“50年”的时间就够了——毫不夸张，如果你理解上面我描述的“内存中的一天”是什么概念的话，这确实就是一次从硬盘读取所要消耗的时间。见了鬼的硬盘读写，好吧，我跑题了。

所以，继续回到MMU的话题上来。当你的显存不够用的时候它允许你不通过复制来完成对显存地址空间的碎片整理。这是个好处，这让多个进程可以分享同一个GPU的问题解决起来十分简单。这个运算单元（MMU）被要求使用在GPU上，但是我并不确定各个厂商有没有好好的执行这一标准，虽然这个东西真的很有用（如果谁能确认这件事的话，请告诉我，我会立刻更新我的文章，但是现在我这真的没法肯定）。不论如何，虽然MMU/虚拟内存并不是一个必要的运算单元，但是它确实不是针对某些特殊功能而产生的特殊硬件——我之后会在一些其他地方提到它，所以我先在这里提一下。

还有一个DMA引擎我还没有说，这个东西可以拷贝内存的同时不被我们珍贵的3D硬件/shader核心约束。（译者注：这个组件是完全独立工作的，和CPU，GPU都无关）一般来说，DMA至少可以完成系统内存和显存之间的拷贝（双向的）。其实它还可以执行显存与显存之间的拷贝（比如你需要整理你VRAM中的碎片，相信我，这很有用）。一般来说它不能执行系统内存到系统内存的拷贝——CPU去干这件事会更加方便。

（原作者）更新：我画了个图（看下图）（译者吐槽：这随意的画风！），我画了它用来显示更多的细节——一般GPU都包含好多个内存控制器（memory controllers），每个内存控制器又拥有好多内存存储体。看看前面那个大胖子（Memory Hub），不管是啥，它用来获得带宽。

![GPU内存子系统]({{ IMAGE_PATH }}/gpu_memory.jpg)

现在来看看我们都讨论了些什么，在CPU端准备命令缓存（译者注：由UMD产生）。我们在CPU端准备好了一些命令缓存，通过PCI-E总线，把这些命令缓存的地址传输到GPU端存储在寄存器中。我们必须这么做才能真正获取到数据——如果我们只是通过将命令缓存的地址写入寄存器，那么就会通过PCI-E来完成，如果我们想把命令缓存完整的写入显存中，那么KMD就会设置一个DMA传输器来完成这个动作，所以不论是CPU还是GPU上的shader单元都不需要关心这些传输的过程。所有的流程都解释了，我们现在可以正式的看看一些命令了。

### 终于，命令处理器(command processor)!
终于要正式开始讨论命令处理器了，之前讨论了许许多多东西但始终围绕着的只是一个词：

“缓存”（buffering）。

就像我上面所提的，两个内存传输线路都是高带宽的但也是高延迟的。对于现代的GPU管线来说，弥补高延迟的方式是并行许多独立的线程。但是在这样的情况下，我们的情况却是只有一个单独的命令处理器按顺序处理命令缓存（类似状态转换和渲染命令都需要按照正确的顺序执行）。所以我们做了几乎是最好的处理：设置一个足够大的缓存并且预取足够多的命令来避免各种小尴尬的发生。

命令处理器实际上只是命令处理的前端，它的本质是一个知道如何转换命令（使用一种硬件指定的格式）的状态机，其中可能有一些2D渲染的操作命令——除非有单独的2D命令处理器专门用来处理并且这些命令对3D命令处理器不可见，否则这些命令3D命令处理器会进行处理。但是实际上，就好像老套的VGA芯片还存在于某个地方，在现代GPU中依旧隐藏着专用的2D处理单元用来支持文字模式，4bit/像素的平面模式，平滑卷轴等等功能。我们还是可以很容易的发现他们的存在，不管怎么样，2D处理单元是存在的，我之后恐怕不会再提起这个内容了:)。在解析了命令之后，命令会开始正式的传递一些图元给3D/shader管线了，我会在之后的部分讨论它们。还有一些命令传递到了3D/shader管线，但是并不做任何渲染，有很多理由处理这些命令（比如多种管线配置的命令），我会在更后面的部分讨论这些内容。

有一些命令改变了状态，作为一个程序员，你可能觉得这些命令只是改变了一些变量，虽然本质上来说就是这样的，但是显然情况要更加复杂。GPU是一种大规模并行的处理器，显然你不能简单的通过改变一个全局变量然后还期盼所有事情依旧运行正常，你并不能保证正在执行的一切都不会改变，这会产生bug而且迟早都会暴露出来。我列举了一些目前流行的处理状态的方法，基本上所有的芯片会根据不同的状态采取不同的方法:

* 无论何时你需要改变状态，那么你需要等待引用该状态的所有工作结束（比如刷新部分管线）。历史上，这种方法被图形芯片运用到大部分的状态改变上——这种方式非常简单而且在少量批处理（batch），少量三角形和管线较短的情况下并没有太大的消耗。诚然，当随着这些东西变得越庞大，消耗也同样会变得越大。但是这种方法依旧被使用在状态改变频率较少（一小部分管线的刷新一次并不会对总体造成太大的影响）或者实现其他方式难度过于大的情况下。

* 可以让硬件单元完全去状态化。将你的状态改变命令传递给其相关的状态，然后将其附加到现有的状态并继续往下传递，并且循环。这样的话，状态不会被存储在某个地方，但是始终存在。如果你的管线需要查看状态是能够办到的，因为它们已经被传递进来了（并且它还会在管线的各个阶段继续传递下去，附加到下一个状态中）。如果你的状态恰巧是几个位的数据，这个方式将非常实用。如果它碰巧是全部活跃纹理伴随着采样状态，那就呵呵了。

* 只存储一个状态副本并且在每次状态改变的时候都不得不刷新，这时候需要序列化很多东西。但是当你同时存储两个（或者4个）状态的副本的时候，这个情况就会被显著改善了（译者注：其实就是多缓存机制）。你的状态设置前端可以预先进行一步。也就是说你如果有足够的寄存器（或者说是槽（slot））来存储每种状态的多个版本，而一些管线操作引用着状态0。你就可以安全的改动状态1而不需要停止或者说妨碍到现在正在进行的工作。而且你也不需要将整个状态在管线中传输——只需要传送几个位的数据来指示使用的是状态0还是状态1就可以了。当然，如果状态0和状态1都处于工作状态，你需要改变状态还是需要等待，但是你已经超前的做了一步了。一般这个技术在使用的时候都不会只用2个状态存储器的，显然会更多。

* 对于像采样器（sampler）或者着色器资源视图（Shader Resource View）状态，虽然你可以一次设置大量的状态，但是请不要这么做。你不会想要保留2 * 128个纹理的状态空间的，因为我们同时还要保留一个缓存状态，所以你可能真的会产生这么多空间需求。对于这样的情况，你可以使用一种寄存器重命名方案（register renaming scheme）——保留一个128个物理纹理的描述符池。当然，如果真的有人需要在一个shader中使用128个纹理，状态的改变就会变得十分的慢。但是，一般来说使用纹理数量会小于20个，所以问题还是不大的。

这不是一个完整的列表——但是我主要想说的是，诸如在你的APP中改变一个变量这样一个再简单不过的操作，这背后也许需要大量的硬件支持来防止你的程序变慢。

### 同步
最后，命令家族中最后的一种就是处理CPU到GPU或者GPU到GPU之间的同步。

一般来说，所谓的同步就是“如果事件X发生了，执行Y”。我们先来处理“执行Y”的部分——这里有两种合理的处理方式：
第一种称为推模式（push-model），也就是GPU通知CPU去立刻去做一些事情（“Hey，CPU，我现在正在处理显示器0扫描的垂直回归期，如果你不想画面出现撕裂，现在正是处理的好时候！”）（译者注：这里需要参考[垂直同步](https://en.wikipedia.org/wiki/Screen_tearing)的知识）;另外一种是拉模型（pull-model），就是GPU记录当前发生的一些情况可以让CPU获取（“Hey，GPU老爷，你现在在处理哪个命令捏？” “我瞅瞅，303号命令！”）。拉模型的实现一般需要使用中断，而且应该应用在极少使用和高优先级的事件，因为中断的代价非常昂贵。一旦某个需要CPU主动问询GPU的事件发生了，你需要提供一些CPU可见的寄存器并且提供写入的方式。

假设你的GPU有16个寄存器，现在我们想个方法把 **currentCommandBufferSeqId** 写入寄存器0。方法如下，你给每一个提交到GPU的命令都赋值一个按顺序的ID（这个操作在KMD中执行），在每一条命令开始的时候加入一个同步操作“如果这个命令开始了，把该命令的ID写入寄存器0”。你瞅瞅，这样我们就可以知道GPU在执行哪条命令了！而且我们可以知道GPU是严格的按顺序执行这些命令，也就是说如果ID303的命令被执行，那么可以肯定包括ID302在内的之前所有命令肯定已经执行完毕了，那么也就是说之前的那些部分的命令缓存就可以被KMD回收，释放，修改了。

我们现在也已经有了关于条件X的例子：“如果执行到这”——也许这是最简单的例子，但是无论如何至少我们已经有一个可以用的东西了。列举一些其他真实存在的例子比如“在所有shaders已经完成读取所有来自批处理中的纹理之前”（这个可以让KMD去回收纹理/渲染目标的内存），“当所有渲染目标(render targets)/UAVs(unordered-access views)已经完成渲染时”（这个时候你可以安全把它们当作纹理来使用），“当所有操作都完成时”，等等。

这些操作一般被称作“栅（fences）”，顺便说一句，虽然有许多不同的方法选择值写入状态寄存器，但是就我个人而言，我认为只有一种靠谱的办法是使用时序计数器（sequential counter）（这也许会导致存储了一些其他“无用”的信息）。是的，我就是随口提了一句，别指望我在这个系列里面解释关于这方面的东西，或许我会在别的blog里讨论这个。

现在我们已经说了一半了——现在可以从GPU端报告状态给CPU端，这可以让我们通过驱动靠谱的管理我们的内存（尤其是我们可以得知何时可以安全的回收诸如使用在顶点缓存（vertex buffers），命令缓存（command buffers），纹理（textures）或者其他资源）。但是这并不是全部内容——还有一部分谜题没有解开。当我们纯粹的在GPU端做同步的时候需要什么？举个例子，回到之前提过的关于渲染目标（render target）的那个例子。我们在它没有被渲染完毕之前没法把它当作纹理使用（有一些内容我跳过了，更多的细节留到纹理单元那节再具体讨论）。解决方法是忙等待指令——“直到寄存器M的值为N”。这里的条件可以是等于也可以是小于（根据不同的场景来决定）或者是更加花哨的条件——我这里采用最简单的等于的情况。这样的做法可以允许我们在提交一个批次之前做渲染目标的同步。这也允许我们创建一个GPU刷新操作：“当所有工作都完全后设置寄存器0为++seqId”或者“等待直到寄存器0包含seqId”。一个接着一个完成。GPU和GPU之间的同步问题就是如此解决——当我们介绍关于DX11的通用计算shader时候会提到另外一种更加细致的同步手段，对于正常的渲染而言这是GPU端唯一的同步手段，虽然很简单，但是足够了。

顺便说说，如果你可以有从CPU端写入GPU端寄存器的权限，你也可以利用这个方法完成一件别的事——提交一个局部的命令包含了等待一个特殊的值，然后通过CPU端改变寄存器而不使用GPU。这种做法被利用在D3D11的多线程渲染中提交一个引用了还在CPU端被其他线程锁定了的顶点/索引缓存（也许正在被其他线程修改）的批处理（batch）。这种方式会在真正的渲染被执行前等待，然后CPU端可以修改寄存器的值一旦被引用的缓存被解锁。如果GPU还没有执行到这个等待操作资源就被解锁，那么这个等待就是个空操作，如果真的因为多线程渲染而导致等待，那么GPU就会产生一个类似自旋锁的东西消耗时间等待寄存器被真正设置。非常完美不是吗？不是？实际上你甚至可以不使用CPU端可写寄存器来完成这个点，只要你可以修改你已经提交了的命令缓存就可以了，只要有命令缓存的jump指令。这些细节留给感兴趣的读者自己去了解。

当然，你并不完全必要使用设置寄存器/等待寄存器这样的同步模型。对于GPU之间的同步，你甚至只需要简单的“渲染目标栅（rendertarget barrier）”指令来保证渲染目标被安全使用，再加上一个“刷新所有（flush everything）”命令。
但是我个人更加喜欢设置寄存器来实现同步的方式因为这是一种一石二鸟的办法（可以通知CPU端资源的使用情况，也可以实现GPU端自身的同步）。

（原作者）更新：我在这里给你们画了一幅图（译者注：看下图，依旧还是谜一样的画风，真的不是我本人画的）。我画的有点复杂（译者：我的内心毫无波动，甚至还有点想笑）因为我想在未来讨论细节时更加底层。我稍微解释一下：命令处理器的前端有个先入先出单元，然后是命令解码逻辑单元，执行需要通过和不同的模块比如2D单元，3D前端（正常的3D渲染）或者shader单元（通用计算shaders单元）直接交流来完成，然后这些单元处理同步/等待命令（那些我讨论过的具有公开寄存器的命令），然后一个单元处理命令缓存的跳跃（jumps）/调用（calls）指令（改变目前的指向FIFO中的取值地址）。然后所有的单元分配工作需要反馈完成信息比如纹理不再继续使用了，我们可以回收这块内存。

![命令处理器]({{ IMAGE_PATH }}/command_processor.jpg)

### 结束语
下一章内容我们会真正的走进渲染的工作。只有三章内容讨论GPU，我们要真正开始讨论一些关于顶点数据的知识了（不，还没有到光栅化部分。那个还需要一些时间）。

实际上，在这部分，我已经提及了一部分管线中的内容了。如果我们讨论通用计算shaders，那么下一步就要完整讨论这部分管线内容了。但是我并不会这么做，因为这个主题我将会放到最后来讨论，先讨论普通的渲染管线。

一点免责声明：再次声明，我只给你了一个大致的框架，深入细节是必要的，但是相信我，我有很多东西因为方便没有说（这些内容很好理解）。我觉得我没有放弃任何一点重要的信息。当然，我可能出错，如果你找到一些我写错的地方，请告诉我！

未完待续...

-------------------------------
## English

### Not so fast.
In the previous part I explained the various stages that your 3D rendering commands go through on a PC before they actually get handed off to the GPU; short version: it’s more than you think. I then finished by name-dropping the command processor and how it actually finally does something with the command buffer we meticulously prepared. Well, how can I say this – I lied to you. We’ll indeed be meeting the command processor for the first time in this installment, but remember, all this command buffer stuff goes through memory – either system memory accessed via PCI Express, or local video memory. We’re going through the pipeline in order, so before we get to the command processor, let’s talk memory for a second.

### The memory subsystem
GPUs don’t have your regular memory subsystem – it’s different from what you see in general-purpose CPUs or other hardware, because it’s designed for very different usage patterns. There’s two fundamental ways in which a GPU’s memory subsystem differs from what you see in a regular machine:

The first is that GPU memory subsystems are fast. Seriously fast. A Core i7 2600K will hit maybe 19 GB/s memory bandwidth – on a good day. With tail wind. Downhill. A GeForce GTX 480, on the other hand, has a total memory bandwidth of close to 180 GB/s – nearly an order of magnitude difference! Whoa.

The second is that GPU memory subsystems are slow. Seriously slow. A cache miss to main memory on a Nehalem (first-generation Core i7) takes about 140 cycles if you divide the [memory latency as given by AnandTech](http://www.anandtech.com/show/2542/5) by the clock rate. The GeForce GTX 480 I mentioned previously has a [memory access latency of 400-800 clocks](../assets/pdf/Rennich_2011_04_25.pdf). So let’s just say that, measured in cycles, the GeForce GTX 480 has a bit more than 4x the average memory latency of a Core i7. Except that Core i7 I just mentioned is clocked at 2.93GHz, whereas GTX 480 shader clock is 1.4 GHz – that’s it, another 2x right there. Woops – again, nearly an order of magnitude difference! Wait, something funny is going on here. My common sense is tingling. This must be one of those trade-offs I keep hearing about in the news!

Yep – GPUs get a massive increase in bandwidth, but they pay for it with a massive increase in latency (and, it turns out, a sizable hit in power draw too, but that’s beyond the scope of this article). This is part of a general pattern – GPUs are all about throughput over latency; don’t wait for results that aren’t there yet, do something else instead!

That’s almost all you need to know about GPU memory, except for one general DRAM tidbit that will be important later on: DRAM chips are organized as a 2D grid – both logically and physically. There’s (horizontal) row lines and (vertical) column lines. At each intersection between such lines is a transistor and a capacitor; if at this point you want to know how to actually build memory from these ingredients, [Wikipedia is your friend](http://en.wikipedia.org/wiki/DRAM#Operation_principle). Anyway, the salient point here is that the address of a location in DRAM is split into a row address and a column address, and DRAM reads/writes internally always end up accessing all columns in the given row at the same time. What this means is that it’s much cheaper to access a swath of memory that maps to exactly one DRAM row than it is to access the same amount of memory spread across multiple rows. Right now this may seem like just a random bit of DRAM trivia, but this will become important later on; in other words, pay attention: this will be on the exam. But to tie this up with the figures in the previous paragraphs, just let me note that you can’t reach those peak memory bandwidth figures above by just reading a few bytes all over memory; if you want to saturate memory bandwidth, you better do it one full DRAM row at a time.

### The PCIe host interface
From a graphics programmer standpoint, this piece of hardware isn’t super-interesting. Actually, the same probably goes for a GPU hardware architect too. The thing is, you still start caring about it once it’s so slow that it’s a bottleneck. So what you do is get good people on it to do it properly, to make sure that doesn’t happen. Other than that, well, this gives the CPU read/write access to video memory and a bunch of GPU registers, the GPU read/write access to (a portion of) main memory, and everyone a headache because the latency for all these transactions is even worse than memory latency because the signals have to go out of the chip, into the slot, travel a bit across the mainboard then get to someplace in the CPU about a week later (or that’s how it feels compared to the CPU/GPU speeds anyway). The bandwidth is decent though – up to about 8GB/s (theoretical) peak aggregate bandwidth across the 16-lane PCIe 2.0 connections that most GPUs use right now, so between half and a third of the aggregate CPU memory bandwidth; that’s a usable ratio. And unlike earlier standards like AGP, this is a symmetrical point-to-point link – that bandwidth goes both directions; AGP had a fast channel from the CPU to the GPU, but not the other way round.

### Some final memory bits and pieces
Honestly, we’re very very close to actually seeing 3D commands now! So close you can almost taste them. But there’s one more thing we need to get out of the way first. Because now we have two kinds of memory – (local) video memory and mapped system memory. One is about a day’s worth of travel to the north, the other is a week’s journey to the south along the PCI Express highway. Which road do we pick?

The easiest solution: Just add an extra address line that tells you which way to go. This is simple, works just fine and has been done plenty of times. Or maybe you’re on a unified memory architecture, like some game consoles (but not PCs). In that case, there’s no choice; there’s just the memory, which is where you go, period. If you want something fancier, you add a MMU (memory management unit), which gives you a fully virtualized address space and allows you to pull nice tricks like having frequently accessed parts of a texture in video memory (where they’re fast), some other parts in system memory, and most of it not mapped at all – to be conjured up from thing air, or, more usually, by a magic disk read that will only take about 50 years or so – and by the way, this is not hyperbole; if you stay with the “memory access = 1 day” metaphor, that’s really how long a single HD read takes. A quite fast one at that. Disks suck. But I digress.

So, MMU. It also allows you to defragment your video memory address space without having to actually copy stuff around when you start running out of video memory. Nice thing, that. And it makes it much easier to have multiple processes share the same GPU. It’s definitely allowed to have one, but I’m not actually sure if it’s a requirement or not, even though it’s certainly really nice to have (anyone care to help me out here? I’ll update the article if I get clarification on this, but tbh right now I just can’t be arsed to look it up). Anyway, a MMU/virtual memory is not really something you can just add on the side (not in an architecture with caches and memory consistency concerns anyway), but it really isn’t specific to any particular stage – I have to mention it somewhere, so I just put it here.

There’s also a DMA engine that can copy memory around without having to involve any of our precious 3D hardware/shader cores. Usually, this can at least copy between system memory and video memory (in both directions). It often can also copy from video memory to video memory (and if you have to do any VRAM defragmenting, this is a useful thing to have). It usually can’t do system memory to system memory copies, because this is a GPU, not a memory copying unit – do your system memory copies on the CPU where they don’t have to pass through PCIe in both directions!

Update: I’ve drawn a picture（the photo below） (link since this layout is too narrow to put big diagrams in the text). This also shows some more details – by now your GPU has multiple memory controllers, each of which controls multiple memory banks, with a fat hub in the front. Whatever it takes to get that bandwidth. :)

![photo 1]({{ IMAGE_PATH }}/gpu_memory.jpg)

Okay, checklist. We have a command buffer prepared on the CPU. We have the PCIe host interface, so the CPU can actually tell us about this, and write its address to some register. We have the logic to turn that address into a load that will actually return data – if it’s from system memory it goes through PCIe, if we decide we’d rather have the command buffer in video memory, the KMD can set up a DMA transfer so neither the CPU nor the shader cores on the GPU need to actively worry about it. And then we can get the data from our copy in video memory through the memory subsystem. All paths accounted for, we’re set and finally ready to look at some commands!

### At long last, the command processor!
Our discussion of the command processor starts, as so many things do these days, with a single word:

“Buffering…”

As mentioned above, both of our memory paths leading up to here are high-bandwidth but also high-latency. For most later bits in the GPU pipeline, the method of choice to work around this is to run lots of independent threads. But in this case, we only have a single command processor that needs to chew through our command buffer in order (since this command buffer contains things such as state changes and rendering commands that need to be executed in the right sequence). So we do the next best thing: Add a large enough buffer and prefetch far enough ahead to avoid hiccups.

From that buffer, it goes to the actual command processing front end, which is basically a state machine that knows how to parse commands (with a hardware-specific format). Some commands deal with 2D rendering operations – unless there’s a separate command processor for 2D stuff and the 3D frontend never even sees it. Either way, there’s still dedicated 2D hardware hidden on modern GPUs, just as there’s a VGA chip somewhere on that die that still supports text mode, 4-bit/pixel bit-plane modes, smooth scrolling and all that stuff. Good luck finding any of that on the die without a microscope. Anyway, that stuff exists, but henceforth I shall not mention it again. :) Then there’s commands that actually hand some primitives to the 3D/shader pipe, woo-hoo! I’ll take about them in upcoming parts. There’s also commands that go to the 3D/shader pipe but never render anything, for various reasons (and in various pipeline configurations); these are up even later.

Then there’s commands that change state. As a programmer, you think of them as just changing a variable, and that’s basically what happens. But a GPU is a massively parallel computer, and you can’t just change a global variable in a parallel system and hope that everything works out OK – if you can’t guarantee that everything will work by virtue of some invariant you’re enforcing, there’s a bug and you will hit it eventually. There’s several popular methods, and basically all chips use different methods for different types of state.

 * Whenever you change a state, you require that all pending work that might refer to that state be finished (i.e. basically a partial pipeline flush). Historically, this is how graphics chips handled most state changes – it’s simple and not that costly if you have a low number of batches, few triangles and a short pipeline. Alas, batch and triangle counts have gone up and pipelines have gotten long, so the cost for this type of approach has shot up. It’s still alive and kicking for stuff that’s either changed infrequently (a dozen partial pipeline flushes aren’t that big a deal over the course of a whole frame) or just too expensive/difficult to implement with more specific schemes though.

 * You can make hardware units completely stateless. Just pass the state change command through up to the stage that cares about it; then have that stage append the current state to everything it sends downstream, every cycle. It’s not stored anywhere – but it’s always around, so if some pipeline stage wants to look at a few bits in the state it can, because they’re passed in (and then passed on to the next stage). If your state happens to be just a few bits, this is fairly cheap and practical. If it happens to be the full set of active textures along with texture sampling state, not so much.

 * Sometimes storing just one copy of the state and having to flush every time that stage changes serializes things too much, but things would really be fine if you had two copies (or maybe four?) so your state-setting frontend could get a bit ahead. Say you have enough registers (“slots”) to store two versions of every state, and some active job references slot 0. You can safely modify slot 1 without stopping that job, or otherwise interfering with it at all. Now you don’t need to send the whole state around through the pipeline – only a single bit per command that selects whether to use slot 0 or 1. Of course, if both slot 0 and 1 are busy by the time a state change command is encountered, you still have to wait, but you can get one step ahead. The same technique works with more than two slots.

 * For some things like sampler or texture Shader Resource View state, you could be setting very large numbers of them at the same time, but chances are you aren’t. You don’t want to reserve state space for 2*128 active textures just because you’re keeping track of 2 in-flight state sets so you might need it. For such cases, you can use a kind of register renaming scheme – have a pool of 128 physical texture descriptors. If someone actually needs 128 textures in one shader, then state changes are gonna be slow. (Tough break). But in the more likely case of an app using less than 20 textures, you have quite some headroom to keep multiple versions around.

This is not meant to be a comprehensive list – but the main point is that something that looks as simple as changing a variable in your app (and even in the UMD/KMD and the command buffer for that matter!) might actually need a nontrivial amount of supporting hardware behind it just to prevent it from slowing things down.

### Synchronization
Finally, the last family of commands deals with CPU/GPU and GPU/GPU synchronization.

Generally, all of these have the form “if event X happens, do Y”. I’ll deal with the “do Y” part first – there’s two sensible options for what Y can be here: it can be a push-model notification where the GPU yells at the CPU to do something right now (“Oi! CPU! I’m entering the vertical blanking interval on display 0 right now, so if you want to flip buffers without tearing, this would be the time to do it!”), or it can be a pull-model thing where the GPU just memorizes that something happened and the CPU can later ask about it (“Say, GPU, what was the most recent command buffer fragment you started processing?” – “Let me check… sequence id 303.”). The former is typically implemented using interrupts and only used for infrequent and high-priority events because interrupts are fairly expensive. All you need for the latter is some CPU-visible GPU registers and a way to write values into them from the command buffer once a certain event happens.

Say you have 16 such registers. Then you could assign **currentCommandBufferSeqId** to register 0. You assign a sequence number to every command buffer you submit to the GPU (this is in the KMD), and then at the start of each command buffer, you add a “If you get to this point in the command buffer, write to register 0”. And voila, now we know which command buffer the GPU is currently chewing on! And we know that the command processor finishes commands strictly in sequence, so if the first command in command buffer 303 was executed, that means all command buffers up to and including sequence id 302 are finished and can now be reclaimed by the KMD, freed, modified, or turned into a cheesy amusement park.

We also now have an example of what X could be: “if you get here” – perhaps the simplest example, but already useful. Other examples are “if all shaders have finished all texture reads coming from batches before this point in the command buffer” (this marks safe points to reclaim texture/render target memory), “if rendering to all active render targets/UAVs has completed” (this marks points at which you can actually safely use them as textures), “if all operations up to this point are fully completed”, and so on.

Such operations are usually called “fences”, by the way. There’s different methods of picking the values you write into the status registers, but as far as I am concerned, the only sane way to do it is to use a sequential counter for this (probably stealing some of the bits for other information). Yeah, I’m really just dropping that one piece of random information without any rationale whatsoever here, because I think you should know. I might elaborate on it in a later blog post (though not in this series) :).

So, we got one half of it – we can now report status back from the GPU to the CPU, which allows us to do sane memory management in our drivers (notably, we can now find out when it’s safe to actually reclaim memory used for vertex buffers, command buffers, textures and other resources). But that’s not all of it – there’s a puzzle piece missing. What if we need to synchronize purely on the GPU side, for example? Let’s go back to the render target example. We can’t use that as a texture until the rendering is actually finished (and some other steps have taken place – more details on that once I get to the texturing units). The solution is a “wait”-style instruction: “Wait until register M contains value N”. This can either be a compare for equality, or less-than (note you need to deal with wraparounds here!), or more fancy stuff – I’m just going with equals for simplicity. This allows us to do the render target sync before we submit a batch. It also allows us to build a full GPU flush operation: “Set register 0 to ++seqId if all pending jobs finished” / “Wait until register 0 contains seqId”. Done and done. GPU/GPU synchronization: solved – and until the introduction of DX11 with Compute Shaders that have another type of more fine-grained synchronization, this was usually the only synchronization mechanism you had on the GPU side. For regular rendering, you simply don’t need more.

By the way, if you can write these registers from the CPU side, you can use this the other way too – submit a partial command buffer including a wait for a particular value, and then change the register from the CPU instead of the GPU. This kind of thing can be used to implement D3D11-style multithreaded rendering where you can submit a batch that references vertex/index buffers that are still locked on the CPU side (probably being written to by another thread). You simply stuff the wait just in front of the actual render call, and then the CPU can change the contents of the register once the vertex/index buffers are actually unlocked. If the GPU never got that far in the command buffer, the wait is now a no-op; if it did, it spend some (command processor) time spinning until the data was actually there. Pretty nifty, no? Actually, you can implement this kind of thing even without CPU-writeable status registers if you can modify the command buffer after you submit it, as long as there’s a command buffer “jump” instruction. The details are left to the interested reader :)

Of course, you don’t necessarily need the set register/wait register model; for GPU/GPU synchronization, you can just as simply have a “rendertarget barrier” instruction that makes sure a rendertarget is safe to use, and a “flush everything” command. But I like the set register-style model more because it kills two birds (back-reporting of in-use resources to the CPU, and GPU self-synchronization) with one well-designed stone.

Update: Here, I’ve drawn a diagram（the photo below） for you. It got a bit convoluted so I’m going to lower the amount of detail in the future. The basic idea is this: The command processor has a FIFO in front, then the command decode logic, execution is handled by various blocks that communicate with the 2D unit, 3D front-end (regular 3D rendering) or shader units directly (compute shaders), then there’s a block that deals with sync/wait commands (which has the publicly visible registers I talked about), and one unit that handles command buffer jumps/calls (which changes the current fetch address that goes to the FIFO). And all of the units we dispatch work to need to send us back completion events so we know when e.g. textures aren’t being used anymore and their memory can be reclaimed.

![photo 2]({{ IMAGE_PATH }}/command_processor.jpg)

### Closing remarks
Next step down is the first one doing any actual rendering work. Finally, only 3 parts into my series on GPUs, we actually start looking at some vertex data! (No, no triangles being rasterized yet. That will take some more time).

Actually, at this stage, there’s already a fork in the pipeline; if we’re running compute shaders, the next step would already be … running compute shaders. But we aren’t, because compute shaders are a topic for later parts! Regular rendering pipeline first.

Small disclaimer: Again, I’m giving you the broad strokes here, going into details where it’s necessary (or interesting), but trust me, there’s a lot of stuff that I dropped for convenience (and ease of understanding). That said, I don’t think I left out anything really important. And of course I might’ve gotten some things wrong. If you find any bugs, tell me!

Until the next part…