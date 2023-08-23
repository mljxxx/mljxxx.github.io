---
title: 下载和构建llvm
date: 2023-08-23 14:34:46
tags: llvm
---
&ensp;想要学习llvm，我们首先要下载和编译llvm，下面介绍下如何在MacOS上通过ninja编译llvm。
<!--more-->
# 安装依赖
&ensp;首先我们要安装编译器，我们使用clang来编译llvm，使用homebrew安装llvm，llvm中包含了clang。
```
brew install cmake llvm
```
之后要安装构建工具cmake和ninja，同样使用homebrew即可安装。
```
brew install cmake ninja
```
# 下载仓库
llvm在github上有两个仓库分别为[llvm/llvm-project](https://github.com/llvm/llvm-project)和[apple/llvm-project](https://github.com/apple/llvm-project)，这里我选择是的apple/llvm-project仓库，分支选择的是[stable/20230725](https://github.com/apple/llvm-project/tree/stable/20230725),仓库和分支的差异可以查看[AppleBranchingScheme](https://github.com/apple/llvm-project/blob/stable/20230725/apple-docs/AppleBranchingScheme.md)。
```
git clone git@github.com:apple/llvm-project.git
git checkout stable/20230725
```
# 编译和安装
我们使用ninja来编译，首先我们要通过cmake生成ninja的所需要的构建文件，在仓库根目录下新建build文件夹，并切换到文件夹目录下，运行如下命令即可。
因为后续想要调试llvm，所以通过CMAKE_BUILD_TYPE来指定构建Debug产物，同时还想学习clang，所以通过LLVM_ENABLE_PROJECTS指定编译llvm同时也编译clang。
还有一些其他选项可以选择，比如通过CMAKE_INSTALL_PREFIX来指定llvm安装的路径,通过LLVM_CREATE_XCODE_TOOLCHAIN=ON可以编译出Xcode所需的toolchain。
```
mkdir build
cd build
cmake -G Ninja -DCMAKE_BUILD_TYPE=Debug -DLLVM_ENABLE_PROJECTS="clang" ../llvm
```
生成完ninja所需要的构建文件后使用ninja命令即可开始编译，可以通过-j指定编译线程数。
```
ninja -j10
```
编译完成后，产物就都在build目录下了，可以选择安装到之前指定的目录下。
```
ninja install //安装llvm
ninja install-xcode-toolchain //安装xcode toolchain
```
# 遇到的问题和解决方式
- 重新生成ninja构建文件时后一直循环re-run cmake，使用ninja -t commands build.ninja就会输出运行对应的命令，然后将build.ninja中对应行注释即可。
- 提示pkg-config search path错误，通过CMAKE_PREFIX_PATH指定MacOS SDK的路径即可，参考https://github.com/apple/swift/pull/32436
```
-DCMAKE_PREFIX_PATH=${xcrun --sdk macosx --show-sdk-path}
```
- Xcode使用toolchain来编译遇到的问题。
  - 有些pod编译报module相关的错误，可以尝试设置CLANG_ENABLE_MODULES=NO和GCC_PRECOMPILE_PREFIX_HEADER=NO。
  - 提示swift库符号加载错误，在Xcode Build Settings中的LIBRARY_SEARCH_PATHS添加下面路径。
    ```
    $(SDKROOT)/usr/lib/swift
    /Applications/Xcode.app/Contents/Developer/Toolchains/XcodeDefault.xctoolchain/usr/lib/swift/$(PLATFORM_NAME)
    /Applications/Xcode.app/Contents/Developer/Toolchains/XcodeDefault.xctoolchain/usr/lib/swift-5.0/$(PLATFORM_NAME)
    ```
  - 提示'TARGET_OS_*' is not defined，将llvm-project/clang/lib/Driver/ToolChains/Darwin.cpp中包含CC1Args.push_back("-Werror=undef-prefix")的行注释。
