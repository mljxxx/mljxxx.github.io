---
title: 加法变减法
date: 2023-10-29 15:50:31
tags:
---
上一篇介绍了llvm pass的编写，本篇我们实现一个将加法变成减法的pass。
<!--more-->
# 编写pass
我们通过Function的迭代器遍历BasicBlock，再通过BasicBlock的迭代器遍历Instruction，判断Instruction是BinaryOperator，类型为Add来找到add指令。
之后我们通过原来的add指令创建Sub指令，参数通过getOperand获取，创建位置在add指令之前。
需要拷贝下setHasNoSignedWrap和setHasNoUnsignedWrap，之后调用replaceAllUsesWith替换add指令。
替换完成后将add指令从该BasicBlock移出，并更新迭代器。
```
PreservedAnalyses run(Function &F,FunctionAnalysisManager &) {
    for(auto &BB : F) {
        for(BasicBlock::iterator iterator = BB.begin();iterator!=BB.end();) {
            Instruction& ins = *iterator;
            errs() << ins.getOpcodeName() << "\n";
            if (ins.isBinaryOp() && ins.getOpcode() == llvm::Instruction::Add) {
                BinaryOperator *op = BinaryOperator::Create(llvm::Instruction::Sub, ins.getOperand(0), ins.getOperand(1),Twine(),&ins);
                op->setHasNoSignedWrap(ins.hasNoUnsignedWrap());
                op->setHasNoUnsignedWrap(ins.hasNoUnsignedWrap());
                ins.replaceAllUsesWith(op);
                iterator = ins.eraseFromParent();
                continue;
            }
            iterator++;
        }
    }
    return PreservedAnalyses::none();
}
```
# 加载pass
使用上一篇的测试用例，生成如下IR:
```
; Function Attrs: noinline nounwind optnone ssp uwtable(sync)
define i32 @_Z3addii(i32 noundef %0, i32 noundef %1) #0 {
  %3 = alloca i32, align 4
  %4 = alloca i32, align 4
  store i32 %0, i32* %3, align 4
  store i32 %1, i32* %4, align 4
  %5 = load i32, i32* %3, align 4
  %6 = load i32, i32* %4, align 4
  %7 = add nsw i32 %5, %6
  ret i32 %7
}
```
我们使用opt加载我们的pass
```
opt --load-pass-plugin ../build/libReplaceAdd.dylib -passes="ReplaceAdd" -S -o test_new.ll test.ll
```
之后我们得到新的IR如下
```
; Function Attrs: noinline nounwind optnone ssp uwtable(sync)
define i32 @_Z3addii(i32 noundef %0, i32 noundef %1) #0 {
  %3 = alloca i32, align 4
  %4 = alloca i32, align 4
  store i32 %0, ptr %3, align 4
  store i32 %1, ptr %4, align 4
  %5 = load i32, ptr %3, align 4
  %6 = load i32, ptr %4, align 4
  %7 = sub i32 %5, %6
  ret i32 %7
}
```
我们可用clang将IR转成可执行文件进行执行,输出结果为减法。
```
clang test_new.ll -o test
./test
```
# 使用clang加载pass
我们需要更改下pass的注册时机，使用registerOptimizerEarlyEPCallback来注册,并且ModulePassManager需要Module的pass,所以我们使用createModuleToFunctionPassAdaptor把Function包装成ModulePass。
```
extern "C" LLVM_ATTRIBUTE_WEAK ::llvm::PassPluginLibraryInfo
llvmGetPassPluginInfo() {
    return {
      LLVM_PLUGIN_API_VERSION, "ReplaceAdd", LLVM_VERSION_STRING,
          [](PassBuilder &PB) {
            PB.registerOptimizerEarlyEPCallback([](ModulePassManager &MPM, OptimizationLevel level) {
                MPM.addPass(createModuleToFunctionPassAdaptor(ReplaceAdd()));
            });
          }
    };
}
```
我们使用-fpass-plugin来加载我们的pass,可以直接生成新的IR。
```
clang -fpass-plugin=../build/libHelloWorld.dylib -S -emit-llvm -o test_new.ll test.cpp
```
为了控制是否开启pass我们通过cl::opt来获取参数，代码如下：
```
static llvm::cl::opt<bool> enable_add_to_sub("enable-add-to-sub", llvm::cl::desc("pass variable"), llvm::cl::value_desc("pass variable"));
PreservedAnalyses run(Function &F,FunctionAnalysisManager &) {
    if(enable_add_to_sub) {
        #.......
        return PreservedAnalyses::none();
    } else {
        return PreservedAnalyses::all();
    }
}
```
使用mllvm来传递参数，但是需要用-Xclang -load -Xclang .dylib在加载一次才能让cl::opt生效。
```
clang -Xclang -load -Xclang ../build/libReplaceAdd.dylib -fpass-plugin=../build/libReplaceAdd.dylib test.cpp -mllvm -enable-add-to-sub -emit-llvm -S -o test_new.ll
```
完整代码在[learn_llvm_pass](https://github.com/mljxxx/learn_llvm_pass)