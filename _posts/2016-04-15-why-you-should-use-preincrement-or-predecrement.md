---
layout: post
title: "为什么更倾向使用前置++/--"
keywords: ["c++"]
description: "c++为什么更倾向于使用前置递增/递减运算符"
category: "Cpp-Healthy-Tips"
tags: ["C/C++"]
---

{% include JB/setup %}

首先我要声明，我不是来跟你讨论诸如 “++i++++这样的语句执行会有什么样的结果？” 这种鬼畜问题的，我又不是谭院士（谭院士就是毁人不倦的一代典范！）。

为什么更倾向于使用前置的递增/递减运算符，这个问题其实答案非常简单，因为后置运算多了一次拷贝工作（实际上可能更多），随便一本语法书都可以巴拉巴拉跟你讲一大堆，尤其谭院士会把你不想知道的也告诉你。

用C++的操作符重载来写的话大概会是这样：

```cpp
class Foo 
{
public:
	//preincrement
	void operator++()
	{
		foo_ += 1;
	}

	//postincrement
	Foo operator++(int)
	{
		Foo f(*this);

		foo_ += 1;

		return f;
	}

private:
	int foo_;
}
```

以上代码我教科书般的重载了Class Foo的前置++和后置++运算符（--运算符也是同样的做法），从代码上答案已经显而易见了，不懂代码的人从代码量上就能看出问题所在——大规模使用后置运算符简直就是性能杀手。但是其实我觉得我基本不能依靠这么一两次的拷贝或者构造的开销就轻易的说服你改变代码使用习惯，所以我将给你讲述为什么C++中更加倾向于使用前置而非后置。

为什么在别的语言中并没有建议我们这么做？其实在其他主流的静态编译语言中，这种问题根本不存在——因为C压根不能重载，C#/Java对重载操作符也是极其限制。你使用这种操作符的权限大概也就被局限在内置类型中，讨论内置类型复制导致的性能开销简直就是吃饱了撑的，请在这一点上绝对相信你的编译器。

那么我们有没有机会撞到类似的坑呢？答案是绝对的，举个最简单的例子，我们要使用STL的迭代器（对于STL的迭代器我想大部分人还停留在很浅薄的认识上，我几乎觉得是让STL化腐朽为神奇的关键所在，我计划在以后单独讨论它），这个你几乎每天都在用，会有很多人告诉我，迭代器不就是个指针吗？如果有人这么告诉我这个答案，我会笑笑和他说，实在是图样图森破。

来看看迭代器的本质吧，迭代器模式在四人帮的《设计模式》中有描述——不暴露对象的本质。所以我要告诉你，迭代器就是迭代器，他可以是指针也可以是代理容器甚至可以是个对象，它可以是你想要的任何东西，只要他不暴露对象的本质，并且提供他该有的功能，那么它就是一个迭代器——vector< bool >的迭代器实现就是个代理类型，活生生的例子，当然如果我不告诉你，你可能完全感觉不出来vector< bool >的实现是一个模板全特化。在这样的情况下，如果要循环遍历一个大数目的容器，你还敢放心大胆的使用后置++来递增你的迭代器吗？我想你不会倔强到这个程度吧？更加不要遑论你还时刻处于类存在重载的危险当中。

最后我想说的是，我并不是跟你讨论一个内置类型或者指针的++，--造成的性能损失，不要把时间花在这种层面的问题上，大声告诉我，你们不愿意做谭院士！我们讨论的是更大的问题，你如果不注意这点的话很可能就在你们系统的代码里面搞出一个性能大新闻，所以，现在，立刻马上，改变你的编码习惯！