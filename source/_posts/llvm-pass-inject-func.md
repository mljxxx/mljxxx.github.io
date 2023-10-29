---
title: 利用llvm pass做简单插桩
date: 2023-10-29 20:02:24
tags:
---
本文介绍利用llvm pass在函数开头插入桩函数调用。
<!--more-->
# 编写pass
我们的目标是在源码编译的函数开头插入如下桩函数的调用：
```
void inject_func(char *func_name);
```
因为需要更改函数，所以我们使用Module Pass，定义如下：
```
struct InjectFuncCall : PassInfoMixin<InjectFuncCall> {
    PreservedAnalyses run(Module &M,ModuleAnalysisManager &) {
    }
};
```
我们需要插入桩函数，首先我们要添加函数声明，函数包括返回值和参数。
```
LLVMContext& context = M.getContext();
//获取返回类型
PointerType *returnType = IntegerType::getInt8PtrTy(context);
//获取参数类型          
PointerType *paramType = IntegerType::getInt8PtrTy(context);
//获取函数类型
FunctionType *funcType = FunctionType::get(returnType,paramType,false);
//获取函数调用类型
FunctionCallee funcCallee = M.getOrInsertFunction("inject_func", funcType);
```
获取函数调用类型后，我们遍历所有函数进行插桩。
```
for(auto &F : M) {
    //判断函数是否为声明或者是我们的桩函数
    if(F.isDeclaration() || F.getName().equals("inject_func")) {
        continue;
    }
    //构建IRBuilder,插入的位置是函数的入口block的第一条可插入指令
    IRBuilder<> builder(&*F.getEntryBlock().getFirstInsertionPt());
    //创建全局常量保存函数名
    auto funcName = builder.CreateGlobalStringPtr(F.getName());
    //创建调用
    builder.CreateCall(funcCallee,funcName);
}
```
# 应用Pass
我们有如下测试用例：
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
使用clang加载我们的pass并且编译出.o文件
```
clang -fpass-plugin=../build/libInjectFuncCall.dylib -c test.cpp -o test.o
```
在另一个文件实现我们的桩函数，我们用的是cpp文件，所以要加extern "C"防止被name mangling
```
//inject_func.cpp
#include <cstdio>
extern "C" {
    void inject_func(char *func_name);
}
void inject_func(char *func_name) {
    printf("%s\n",func_name);
}
```
使用clang编译出.o文件
```
clang -c inject_func.cpp -o inject_func.o
```
使用clang编译，clang会使用ld将两个文件链接起来
```
clang inject_func.o test.o -o main
```
最后我们执行./main，效果如下：
```
main
_Z3addii
3
```
完整代码在[learn_llvm_pass](https://github.com/mljxxx/learn_llvm_pass)