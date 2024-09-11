# LLVM体系框架

## 前端



## 中端

> 目录结构：
> llvm/lib/IR: IR相关类 LLVM-IR Core 
> llvm/lib/Transforms: IR上的优化Pass

### IR

- **Module  Function  BasicBlock  Instruction(Opcode Operand)  Value  Type**

   - Module

      - 常见的优化基于Module内部
      - Module之间的优化由LTO完成

   - Value: IR最核心概念

      - 可以视为任意数量寄存器
      - 局部变量%，全局变量@开始

   - Type：

      - 所有Value都被赋予静态类型
      - 运算操作数必须类型相同
         - i1、i8、i32、i64…
            - ==不区分有符号无符号数==
               - 因此除法选择div, udiv
               - 因此比较运算icmp : sgt, ugt
         - float、double、half
         - 数组
         - 结构体:{i32,i32,i8}
         - 指针i32*

   - BasicBlock&Instruction: 头文件Instruction.h

      - 指令分类：

         - 终结指令：ret switch br…
         - +-×÷%
         - 逻辑运算：and or shl
         - sm管理：alloca、load、store
         - Cast 操作符：强制转换
         - 其他：icmp、phi、call

      - 所有的Instruction都在Instruction.def中

         - > **SSA与φ**
            >
            > - φ可以处理自引用
            > - φ必须位于基本块开头
            > - φ必须对每个前驱基本块有且仅有一个入口

### Pass

Pass类别：

- FunctionPass
- ModulePass
- RegionPass
- CallGraphSCCPass
- LoopPass
- BasicBlockPass

举例：`opt -mem2reg 1.bc -S -o 1.ll`：消除load,store，引入φ

推荐阅读：死代码消除DCE.cpp：runOnFunction & EliminateDeadCode

> Scalar: ADCE：激进死代码消除

- 利用PassManager定制优化器 ： PM.add(createAMPPass());

### Opt

### 代码解析

基础代码：

> for (BasicBlock &BB: Func) errs() << BB.getName() <<"\n";
> for (Instruction &I: BB) errs() << I << "\n";
> for (inst_iterator | = inst_begin(F), E = inst_end(F); 1!= E; + +1) errs)
> for (User *U: F-> users) ...
> for (Use &U: pi-> operands) ...
> for (BasicBlock *Pred : predecessors(BB)) ...
>
> 插入删除指令/基本块/函数:
>
> - eraseFromParent
> - ReplaceInstWithValue/ReplaceInstWithInst
> - replaceAllUsesWith/replaceUsesOfWith
> - newInst->insertBefore(φ)

**DCE：基础死代码消除分析**

```c++
//===- DCE.cpp - Code to perform dead code elimination --------------------===//
//
// Part of the LLVM Project, under the Apache License v2.0 with LLVM Exceptions.
// See https://llvm.org/LICENSE.txt for license information.
// SPDX-License-Identifier: Apache-2.0 WITH LLVM-exception
//
//===----------------------------------------------------------------------===//
//
// This file implements dead inst elimination and dead code elimination.
//
// Dead Inst Elimination performs a single pass over the function removing
// instructions that are obviously dead.  Dead Code Elimination is similar, but
// it rechecks instructions that were used by removed instructions to see if
// they are newly dead.
//
//===----------------------------------------------------------------------===//

#include "llvm/Transforms/Scalar/DCE.h"
#include "llvm/ADT/SetVector.h"
#include "llvm/ADT/Statistic.h"
#include "llvm/Analysis/TargetLibraryInfo.h"
#include "llvm/IR/InstIterator.h"
#include "llvm/IR/Instruction.h"
#include "llvm/InitializePasses.h"
#include "llvm/Pass.h"
#include "llvm/Support/DebugCounter.h"
#include "llvm/Transforms/Scalar.h"
#include "llvm/Transforms/Utils/AssumeBundleBuilder.h"
#include "llvm/Transforms/Utils/BasicBlockUtils.h"
#include "llvm/Transforms/Utils/Local.h"
using namespace llvm;

#define DEBUG_TYPE "dce"

STATISTIC(DCEEliminated, "Number of insts removed");        // [统计移除的指令数量]分析数据的话这么定义：☆效果分析,
DEBUG_COUNTER(DCECounter, "dce-transform",                  // 调试计数器
              "Controls which instructions are eliminated");

//===--------------------------------------------------------------------===//
// RedundantDbgInstElimination pass implementation
//

namespace {
struct RedundantDbgInstElimination : public FunctionPass {
  static char ID; // Pass identification, replacement for typeid
  RedundantDbgInstElimination() : FunctionPass(ID) {
    initializeRedundantDbgInstEliminationPass(*PassRegistry::getPassRegistry());
  }
  bool runOnFunction(Function &F) override {                // debug指令消除的入口函数
    if (skipFunction(F))                                    // 如果跳过该FunctionPass,回到bt父级
      return false;
    bool Changed = false; 
    for (auto &BB : F)                                      // 遍历基本块
      Changed |= RemoveRedundantDbgInstrs(&BB);             // 移除冗余调试指令
    return Changed;                                         // 返回是否成功消除死代码(有改动的话)
  }

  void getAnalysisUsage(AnalysisUsage &AU) const override {
    AU.setPreservesCFG();                                   // 声明该Pass保持控制流图不变
  }
};
}

char RedundantDbgInstElimination::ID = 0;                   // pass的ID
INITIALIZE_PASS(RedundantDbgInstElimination, "redundant-dbg-inst-elim",
                "Redundant Dbg Instruction Elimination", false, false)

Pass *llvm::createRedundantDbgInstEliminationPass() {
  return new RedundantDbgInstElimination();
}

PreservedAnalyses
RedundantDbgInstEliminationPass::run(Function &F, FunctionAnalysisManager &AM) {
  bool Changed = false;
  for (auto &BB : F)
    Changed |= RemoveRedundantDbgInstrs(&BB);               // 消除冗余调试指令
  if (!Changed)
    return PreservedAnalyses::all();                        // 如果没有改变，保持所有分析结果
  PreservedAnalyses PA;
  PA.preserveSet<CFGAnalyses>();
  return PA;
}

//===--------------------------------------------------------------------===//
// DeadCodeElimination pass implementation
// 消除死代码的核心函数

static bool DCEInstruction(Instruction *I,
                           SmallSetVector<Instruction *, 16> &WorkList,
                           const TargetLibraryInfo *TLI) {
  if (isInstructionTriviallyDead(I, TLI)) {                 // 指令是否是显然的死代码
    if (!DebugCounter::shouldExecute(DCECounter))
      return false;

    salvageDebugInfo(*I);
    salvageKnowledge(I);

    // 将指令的所有操作数置空，以检查操作数是否因此变为死代码
    for (unsigned i = 0, e = I->getNumOperands(); i != e; ++i) {
      Value *OpV = I->getOperand(i);                       // 获取操作数的值(用Value接收(auto也行))
      I->setOperand(i, nullptr);                           // ⭐️设置操作数

      if (!OpV->use_empty() || I == OpV)                   // 如果操作数仍有使用者 或 等于当前指令(调用自身,递归...)
        continue;

      // If the operand is an instruction that became dead as we nulled out the
      // operand, and if it is 'trivially' dead, delete it in a future loop
      // iteration.                                        // 如果操作数是显然的死代码，将其加入工作列表
      if (Instruction *OpI = dyn_cast<Instruction>(OpV))
        if (isInstructionTriviallyDead(OpI, TLI))
          WorkList.insert(OpI);
    }

    I->eraseFromParent();                                  // 从父节点中删除指令
    ++DCEEliminated;                                       // 删除的指令数量增加
    return true;
  }
  return false;
}

static bool eliminateDeadCode(Function &F, TargetLibraryInfo *TLI) {
  bool MadeChange = false;
  SmallSetVector<Instruction *, 16> WorkList;             // 定义工作列表
  // 遍历原始函数，只将需要重新访问的指令加入工作列表
  // This avoids having to pre-init the worklist with the entire function's worth of instructions.
  for (Instruction &I : llvm::make_early_inc_range(instructions(F))) {
    // 确保当前指令不在之前访问的工作列表中
    if (!WorkList.count(&I))
      MadeChange |= DCEInstruction(&I, WorkList, TLI);    // 处理指令，检查是否需要消除
  }

  while (!WorkList.empty()) {                             // 如果工作列表不为空
    Instruction *I = WorkList.pop_back_val();             // 弹出一个指令
    MadeChange |= DCEInstruction(I, WorkList, TLI);       // 处理指令，检查是否需要消除
  }
  return MadeChange;
}

PreservedAnalyses DCEPass::run(Function &F, FunctionAnalysisManager &AM) {
  if (!eliminateDeadCode(F, &AM.getResult<TargetLibraryAnalysis>(F)))
    return PreservedAnalyses::all();                      // 如果没有改动，保持所有分析结果

  PreservedAnalyses PA;
  PA.preserveSet<CFGAnalyses>();                          // 保持控制流图分析结果
  return PA;
}

namespace {
struct DCELegacyPass : public FunctionPass {              // 定义一个结构体继承自FunctionPass
  static char ID; // Pass identification, replacement for typeid
  DCELegacyPass() : FunctionPass(ID) {                    // 构造函数
    initializeDCELegacyPassPass(*PassRegistry::getPassRegistry());
  }

  bool runOnFunction(Function &F) override {              // 重写runOnFunction函数
    if (skipFunction(F))
      return false;

    TargetLibraryInfo *TLI =                              // 获取目标库信息
        &getAnalysis<TargetLibraryInfoWrapperPass>().getTLI(F);

    return eliminateDeadCode(F, TLI);                     // 消除死代码
  }

  void getAnalysisUsage(AnalysisUsage &AU) const override {
    AU.addRequired<TargetLibraryInfoWrapperPass>();
    AU.setPreservesCFG();
  }
};
}

char DCELegacyPass::ID = 0;                               // 定义Pass的ID
INITIALIZE_PASS(DCELegacyPass, "dce", "Dead Code Elimination", false, false)
                                                          // 创建Pass的函数
FunctionPass *llvm::createDeadCodeEliminationPass() {     // 返回新的Pass实例
  return new DCELegacyPass();
}
```



## 后端

> 目录结构：
> llvm/lib/LTO: 链接时优化
> llvm/lib/CodeGen: 机器码生成
> llvm/lib/Target



## 工具链

- **`llc`**: LLVM静态编译器，将LLVM IR编译为目标机器代码。
   - `llc -march=x86_64 a.ll`
- **`clang`**：-w 无视警告 -i 预处理
   - 《Clang command line argument reference》
- **`llvm-as`**: LLVM汇编器，将LLVM汇编代码（.ll）转换为LLVM位码（.bc）。
- **`llvm-dis`**: LLVM反汇编器，将LLVM位码（.bc）转换为LLVM汇编代码（.ll）。
- **`llvm-link`**: LLVM链接器，用于将多个LLVM IR模块链接成一个模块。
- **`llvm-cov`**: 代码覆盖率工具，生成覆盖率报告。
- **`llvm-profdata`**: 处理分析数据的工具，用于处理和合并LLVM生成的分析数据文件。
- **`llvm-mc`**: LLVM机器代码工具，用于汇编和反汇编机器代码。
- **`llvm-extract`**: 从LLVM位码文件中提取函数或全局变量。
- **`llvm-ar`**: LLVM版本的ar，用于创建、修改和提取静态库。
- **`llvm-nm`**: 类似于Unix `nm`命令，显示符号表信息。
- **`llvm-objdump`**: 类似于GNU `objdump`命令，反汇编和显示目标文件信息。
- **`llvm-dwarfdump`**: 显示DWARF调试信息。
- **`llvm-size`**: 显示目标文件的节大小。
- **`llvm-strings`**: 提取并显示目标文件中的可打印字符串。
- **`llvm-cxxfilt`**: 类似于`c++filt`，用于符号名称解混淆（demangle）。
- **`llvm-symbolizer`**: 将地址解析为源代码位置。
- **`llvm-config`**: 输出LLVM库和工具的配置信息，用于编译和链接。
- **`llvm-tblgen`**: LLVM表生成器工具，用于生成LLVM的指令、目标描述等表格。
- **`FileCheck`**: 文件检查工具，用于测试LLVM工具和库的输出。
- **`llvm-lit`**: LLVM测试工具，用于运行和管理测试。
- **`bugpoint`**: LLVM自动调试器，自动简化失败的测试用例。
- **`llvm-diff`**: 比较两个LLVM IR模块并显示差异。
- **`lli`**: LLVM的JIT编译器，用于直接执行LLVM位码。
- **`clang-tidy`**: 代码分析和重构工具，用于检测和修复C++代码中的常见错误和风格问题。
- **`clang-format`**: 代码格式化工具，用于统一C、C++、JavaScript等语言的代码风格。
- **`scan-build`**: Clang静态分析工具，用于检测代码中的潜在错误。
- **`scan-view`**: 提供静态分析结果的Web界面。

## 资源

- LLVM类的继承关系: `llvm.org/doxygen`

## 附录

#### 常用抽象数据结构

- ArrayRef
- SmallVector
- SmallSet
- StringSet
- DenseSet
- SparseSet
- ImmutableSet
- StringRef
- Twine