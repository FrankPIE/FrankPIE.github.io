---
layout: post
title: "使用Win32 API查找系统路径"
keywords: ["C/C++", "OS", "Win32"]
description: "介绍一种查找Windows操作系统基本路径的办法"
category: "Something-About-OS"
tags: ["windows操作系统", "系统路径", “Win32 API”]
---
{% include JB/setup %}

这篇文章算是失踪人口回归，其实我只是太忙了（忙着摸鱼，顺便处理下失恋的伤痛）没时间更新blog罢了，之前的文章翻译也暂时跳票了，因为好像腾讯的某个工作室也在进行翻译，不过等我心血来潮会继续翻译的（这话我自己都不信）。

更新的这篇文章主要是为了介绍今天无意中找到的一种可以在Window下比较简单和统一的查找各种系统路径的方法，我觉得可能大多数人已经知道了，但是这个方法实在是太方便了，大大的解决了困扰我许久的问题，所以在这里还是写出来，以便不知道的童鞋方便查阅。

-----------------------------------------

废话不多说，先上代码

```cpp

//查找系统ProgramData目录
PWSTR *path_str = nullptr

HRESULT hr ＝ SHGetKnownFolderPath(FOLDERID_ProgramData, 0, nullptr, &path_str);

if (SUCCESSED(hr))
{
    wprintf(L"%s", path_str);    //输出字符串
    CoTaskMemFree(path_str);    //释放内存空间
}

```

我这里只列出了主要的代码，如果你直接CV肯定是无法运行的，因为连个函数体和相应的头文件都没有，主要这些都不是重点，自己加上就好了，这段代码肯定是能work的。

简单介绍下这段代码的含义，首先看到 [**SHGetKnownFolderPath**](https://msdn.microsoft.com/en-us/library/bb762188(VS.85).aspx) 这个API肯定是个COM接口，还有一点能确认的是这个API比较新，因为这个并不像早期微软API使用的那种UNICODE和非UNICODE两个版本函数的那种形式，这个API只支持宽字符(也就是说巨硬连兼容都懒得做了)，这是微软近年来的一种普遍做法，所以不知道的童鞋也不必惊讶。

现在来看看这个函数的声明

```cpp
HRESULT SHGetKnownFolderPath(
  _In_     REFKNOWNFOLDERID rfid,
  _In_     DWORD            dwFlags,
  _In_opt_ HANDLE           hToken,
  _Out_    PWSTR            *ppszPath
);
```

介绍一个函数无非是介绍其功能，参数和返回值相应的信息，那么功能我已经说了，是查找各种各样的系统路径。

大家可以看到第一个参数 [**REFKNOWNFOLDERID**](https://msdn.microsoft.com/en-us/library/windows/desktop/dd378457(v=vs.85).aspx),这个参数的本质是一个GUID，你可以理解为是一个枚举变量，这个参数主要就是对应相应的系统路径，具体大家可以按照自己的需求去MSDN上进行查找。

第二个参数是个[Flag](https://msdn.microsoft.com/en-us/library/windows/desktop/dd378447(v=vs.85).aspx)，这个参数可以控制最后返回字符串的内容，详细细节也可以去MSDN上查阅，一般给0就可以了，这里碍于篇幅就不细介绍了。

第三个参数看起来就是个内核对象，那么这个参数主要用于用户权限控制，不同用户下获得的名称当然也是不同的，一般情况下给nullptr就行。

第四个参数就是你的字符串返回值，是个指针的指针，说明这个函数体内部存在内存分配，所以当你程序结束之后一定要注意释放内存，防止内存泄漏，如我上面代码所写。

大概这个方法就介绍到这里，更多的细节请去MSDN上查阅，我就懒得说了，就是酱紫，拜拜。

