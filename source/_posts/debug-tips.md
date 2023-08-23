---
title: 调试小技巧
date: 2023-08-23 15:47:57
tags: llvm
---
我们使用vscode来调试clang，对于较早的clang源码，会在处理完编译参数后启动一个子进程来调用cc1_main函数，vscode无法自动attach到子进程，所以我们需要一个小技巧来解决这个问题。
<!--more-->
所以我们可以在cc1_main函数开始处停止，并在下一行断点，这样就有时间找到子进程的pid,然后去attach对应的子进程了，使用如下代码即可。
```
#include <csignal>
int cc1_main(ArrayRef<const char *> Argv, const char *Argv0, void *MainAddr) {
    raise(SIGSTOP);     //raise an stop signal then attach it.
    ......
}
```
新版本的clang源码默认不会启动子进程了，但是这个技巧也同样适用于一个程序不是由我们自己启动的时候。
