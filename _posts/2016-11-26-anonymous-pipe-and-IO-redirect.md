---
layout: post
title: "匿名管道通信与IO重定向"
keywords: ["C/C++", "OS", "Win32"]
description: "介绍如何通过匿名管道实现子进程输出重定向"
category: "Something-About-OS"
tags: ["windows操作系统", "系统路径", "Win32"]
---
{% include JB/setup %}

### 写在前面的废话
最近我真是高产似母猪，这篇文章主要介绍一下windows操作系统上进程间通信的一种方法——[管道](https://msdn.microsoft.com/en-us/library/windows/desktop/aa365780(v=vs.85).aspx)。在windows上管道分为[匿名管道](https://msdn.microsoft.com/en-us/library/windows/desktop/aa365139(v=vs.85).aspx)和[命名管道](https://msdn.microsoft.com/en-us/library/windows/desktop/aa365590(v=vs.85).aspx)，这里主要介绍的是匿名管道以及如何使用匿名管道实现子进程的IO重定向。（命名管道我之后会单开一篇来讲，相对于匿名管道来说，它实在是太复杂了）。

撇开应用场景谈技术不是玩具就是耍流氓，作为码农届梁朝伟的我当然干不出这种事。所以大家试想一下这种场景，你手头有个代码规模巨大的CUI程序，有一天，你的产品经理颠儿颠儿的跑来跟你语重心长的说：“黑框框太丑了，我想要个GUI程序，我不管，你想办法。”这时候，除了忍住抽死他的冲动，还是要想想怎么搞定这事，如果要改程序源码，那你今年的KPI也就彻底交待了，如果有一种办法可以让“黑框框”中的输出可以在不改动代码的情况下直接输出到GUI程序中的话，那这个世界该有多么美好啊！

------------------------------------------------

### 正文在这里
还是老规矩，废话不多说，先上代码

```cpp
//父进程代码ParentProcess
void ReadFromChildProcess(void * contexts)
{
    HANDLE pipe_read_handle = static_cast<HANDLE>(context);

    BOOL  read_successed = FALSE;
    DWORD readed_size = 0;

    char buffer[64] = { 0 };

    do 
    {
        read_successed = ReadFile(pipe_read_handle, buffer, 32, &readed_size, nullptr);

        printf("%s", buffer);

        ZeroMemory(buffer, 64);
    } while(read_successed && readed_size != 0);

    CloseHandle(pipe_read_handle);
}

int _tmain(int argc, _TCHAR* argv[])
{
    HANDLE pipe_read_end_handle = nullptr;
    HANDLE pipe_write_end_handle = nullptr;

    SECURITY_ATTRIBUTES pipe_attribute;

    pipe_attribute.nLength = sizeof(SECURITY_ATTRIBUTES);
    pipe_attribute.bInheritHandle = TRUE;
    pipe_attribute.lpSecurityDescriptor = nullptr;

    ::CreatePipe(&pipe_read_end_handle,
         &pipe_write_end_handle,
         &pipe_attribute,
         0);

    TCHAR cmd[] = TEXT("ChildProcess");

    PROCESS_INFORMATION process_information;
    ZeroMemory(&process_information, sizeof(PROCESS_INFORMATION));

    STARTUPINFO startup_info;
    ZeroMemory(&startup_info, sizeof(STARTUPINFO));

    startup_info.cb = sizeof(STARTUPINFO);
    startup_info.dwFlags = STARTF_USESTDHANDLES | STARTF_USESHOWWINDOW;
    startup_info.hStdError = pipe_write_end_handle;
    startup_info.hStdOutput = pipe_write_end_handle;
    startup_info.wShowWindow = SW_HIDE;

    auto successed 
        = CreateProcess(nullptr,
                     cmd,
                     nullptr, 
                     nullptr, 
                     TRUE, 
                     0, 
                     nullptr, 
                     nullptr, 
                     &startup_info, 
                     &process_information);

    if (successed) 
    {
        CloseHandle(process_information.hProcess);
        CloseHandle(process_information.hThread);
        CloseHandle(pipe_write_end_handle);

        HANDLE read_thread = reinterpret_cast<HANDLE>(_beginthread(ReadFromChildProcess, 0, pipe_read_end_handle));

        WaitForSingleObject(read_thread, INFINITE);
    }

    return 0;
}
```

```cpp
//子进程代码ChildProcess
int _tmain(int argc, _TCHAR* argv[])
{
    setvbuf(stdout, nullptr, _IONBF, 0);
    setvbuf(stderr, nullptr, _IONBF, 0);

    uint32_t index = 0;

    while(index < 20)
    {
       printf("Hello World! %d\n", index++);

       Sleep(1000);
    }

    return 0;
}
```

**NOTE: 这段代码没有加头文件，而且因为是手打所以可能会有笔误，但是代码逻辑我在VS2013上肯定调试通顺了，懂原理就行，不确保CV之后能用。**

------------------------------

这两段程序代码分别是父进程和子进程代码，我这里为了方便并没有创建一个GUI程序作为父进程。虽然两个都是CUI程序，但是我确确实实实现了匿名管道和输出重定向。其中有太多的细节，比如进程创建，内核句柄的继承行为，还有一些API的使用细节啊，我比较懒，不想说得太细，这些细节资料也比较多，大家可以通过阅读《windows核心编程》和查阅MSDN来获取这部分知识，我就不再打那么多字了，如果你还不了解这些东西，请先去阅读它们。这里主要讲解代码的思路，和其中的一些坑（我还有个小坑现在还没解决，但是已经有了个大致思路了，当然也希望知道的老司机不吝赐教）。

虽然子进程的代码不多，但是前两行代码就涉及到我之前说到的没有解决的坑，所以我还是先讲父进程的代码。父进程代码的思路其实很简单，通过[CreatePipe](https://msdn.microsoft.com/en-us/library/windows/desktop/aa365152(v=vs.85).aspx)函数创建管道的两个HANDLE对象，并设置他们让它们可以被子进程所继承。然后在创建子进程时，通过设置STARTUPINFO的StdHandle来重定向子进程的标准输入输出对象，所以我讲到这里，聪明的你应该已经猜到了，没错，标准输入输出对象的底层实现是内核对象，准确来说，是FILE对象，所以通过这个设置我们就可以将子进程的stdout对象替换掉了，printf的操作就会往管道里写入了，而不是再输出在控制台上了，这就是重定向的本质。

然后，我们需要创建一个线程来读取管线中的内容，需要线程的原因是因为[ReadFile](https://msdn.microsoft.com/en-us/library/windows/desktop/aa365467(v=vs.85).aspx)这个操作是同步操作，如果管道中一直没有内容，那么就会造成阻塞，所以必然是需要线程。
然后大家看代码可以看见一个比较奇怪的现象，我分配了64个byte的内存，但是我只用了一半，这其实是个小坑，我在测试这段代码的时候发现，如果我读取全部size的内容就会出现乱码，原因我不知道，但是这个办法可以有效的hack掉这个问题。

最后，我就要说我没解决掉的坑了，如果子进程存在一段代码会执行很长一段时间，然后才输出一些内容（这点我用了Sleep来模拟），标准输入输出的缓存会导致管道一直不会被写入，因为内容都被buffer住了。我原本考虑的是在父进程中直接设置子进程的buffer为no buffer，但我发现并不能做到，所以你就看到了子进程中开头的那两行代码。这在一些你自己的代码中不存在问题，只要在main函数内加两句代码就行了，改动量不大，但是，对于一个黑盒程序，可能马上就GG了。

大概就说到这里，对于最后那个坑，我这里有个思路，就是直接设置管道对象的buffer为no buffer，然后让子进程继承，然而，思路归思路，我并没有找到解决方法。就好气！！