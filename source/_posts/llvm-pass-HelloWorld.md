---
title: llvm pass hello world
date: 2023-10-29 14:35:39
tags:
---
本片介绍使用New PassManager的方式在out of tree编写LLVM Pass
<!--more-->
# CMakeLists
首先我们通过cmake进行构建，需要编写CMakeLists.txt
```
cmake_minimum_required(VERSION 3.13.0)
project(learn_llvm_pass)

#设置cpp std版本
set(CMAKE_CXX_STANDARD 17)
#设置llvm安装路径
set(LLVM_INSTALL_PATH "/opt/homebrew/opt/llvm" CACHE PATH "llvm install path") 
#设置clang编译路径，如果使用Xcode的clang可能会出现符号找不到的错误
set(CMAKE_C_COMPILER "/opt/homebrew/opt/llvm/bin/clang")                       
set(DCMAKE_CXX_COMPILER "/opt/homebrew/opt/llvm/bin/clang++")
#把llvm的cmake路径添加到CMAKE_PREFIX_PATH，让find_package可以找到
list(APPEND CMAKE_PREFIX_PATH "${LLVM_INSTALL_PATH}/lib/cmake/llvm/")
#通过find_package导入llvm        
find_package(LLVM REQUIRED CONFIG)                                             
message(STATUS "${LLVM_INCLUDE_DIRS}")
#添加动态库
add_library(HelloWorld SHARED HelloWorld.cpp)                                  
#添加搜索路径
target_include_directories(HelloWorld PUBLIC "${LLVM_INCLUDE_DIRS}")
#设置符号undefined运行时动态查找         
target_link_libraries(HelloWorld "$<$<PLATFORM_ID:Darwin>:-undefined dynamic_lookup>")  
```
# 编写pass
我们要继承自PassInfoMixin，这个类使用CRTP的方式来实现多态，所以继承类的模版需要传入当前类。
```
struct HelloWorld : PassInfoMixin<HelloWorld>
```
需要实现run方法，run是llvm给我们遍历Module和Function的方法，通过第一个参数来区分遍历的类型,我们在其中打印函数名和参数个数，返回值是PreservedAnalyses，是一个集合表示经过这个pass保留的Analyse，我们没有修改IR的情况，返回PreservedAnalyses::all()即可。同时我们要实现isRequired返回true表示这个pass是必要的。
```
struct HelloWorld : PassInfoMixin<HelloWorld> {
        PreservedAnalyses run(Function &F,FunctionAnalysisManager &) {
            errs() << "Hello from: "<< F.getName() << "\n";
            errs() << "number of arguments: " << F.arg_size() << "\n";
            return PreservedAnalyses::all();
        }
        static bool isRequired() { 
            return true; 
        }
    };
```
之后我们需要注册pass，这是一个固定的格式,通过实现llvmGetPassPluginInfo方法返回PassPluginLibraryInfo，
第一个参数是llvm plugin api版本，通过LLVM_PLUGIN_API_VERSION获取即可，
第二个参数是插件名字，随便取，
第三个是插件版本，随便写，
第四参数是一个闭包，我们可以拿到PassBuilder，通过PassBuilder我们可以在不同阶段注册我们的pass，我们使用registerPipelineParsingCallback,然后在callback中使用FunctionPassManager注册我们的pass。
```
extern "C" LLVM_ATTRIBUTE_WEAK ::llvm::PassPluginLibraryInfo
llvmGetPassPluginInfo() {
    return {
      LLVM_PLUGIN_API_VERSION, "HelloWorld", LLVM_VERSION_STRING,
          [](PassBuilder &PB) {
            PB.registerPipelineParsingCallback(
                [](StringRef Name, FunctionPassManager &FPM,
                    ArrayRef<llvm::PassBuilder::PipelineElement>) -> bool {
                    if(Name.equals("HelloWorld")) {
                        FPM.addPass(HelloWorld());
                        return true;
                    }
                    return false;
            });
          }
    };
}
```
# 加载Pass
上述编写没有问题后，我们使用cmake可以编译出libHelloWorld.dylib，下面我们使用clang生成IR，然后用opt来加载。
测试源文件如下：
```
//test.cpp
#include <cstdio>
using namespace std;

int add(int x,int y) {
    return x+y;
}

int main(int argc,char **args) {
    printf("%d\n",add(1,2));
    return 0;
}
```
使用-emit-llvm -S来生成人可读的IR。
```
clang -emit-llvm -S -o test.ll test.cpp
```
使用opt加载我们的IR,添加--disable-output不输出新的IR。
```
opt --load-pass-plugin ../build/libHelloWorld.dylib --passes=HelloWorld --disable-output test.ll
```
运行后会有如下结果
```
Hello from: _Z3addii
number of arguments: 2
Hello from: main
number of arguments: 2
```
完整代码在[learn_llvm_pass](https://github.com/mljxxx/learn_llvm_pass)