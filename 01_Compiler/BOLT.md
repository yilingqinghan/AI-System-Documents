# LLVM-BOLT

[TOC]

## Passes

### *PatchEntries*

- 作用：主要负责修补原始函数入口点，生成并插入跳转代码：打辅助

### *Pettis and Hansen*

- 作用：智能重排代码块来优化程序的执行效率（函数聚类，权重分配，簇合并，簇内排序）
- 架构：通用
- 目的：减少函数调用跳转距离，提高CacheHit率，优化执行路径



### *PLTCall*

- 作用：优化对程序链接表（PLT）的调用。PLT 用于解析动态链接库中函数的地址，但每次调用都涉及额外的查找开销。`PLTCall` 通过将 PLT 调用转换为对全局偏移表（GOT）的直接或间接调用来减少这种开销，优化这一过程
- 结构：通用
- 目的：疑似能够减小二进制大小，优化执行路径，提高性能

### *RegAnalysis*

给下面做辅助：寄存器使用分析，和冲突分析

### *RegReAssign*

- 作用：在二进制优化过程中进行寄存器重新分配
- 架构：通用
- 目的：运行速度+

### *ReorderAlgorithm*

- 作用：改变基本块（即代码片段）的执行顺序来最大化缓存利用率和减少运行时跳转→决定最优的基本块排列顺序
- 架构：通用
- 目的：提高程序执行的速度
- 影响：代码密集度提高，冗余代码识别，函数和循环展开

### *ReorderData*

- 作用：对二进制文件中的数据段进行重新排序，以优化数据访问的效率。通过重新排序，使得热点数据（频繁访问的数据）集中在一起，减少缓存失效，提高内存访问速度，从而提升程序性能。
- 架构：通用
- 目的：提高缓存命中率，减少内存访问延迟，优化程序执行效率

###*ReorderFunctions☆*

- 作用：对二进制程序中的函数进行重新排序和聚类，以优化程序的布局。这种优化可以显著提高指令缓存的命中率和减少分支预测错误，从而提高程序的整体性能。具体的重排序策略包括按照执行次数排序、HFSort算法、HFSort+算法、Pettis-Hansen算法、随机排序以及用户指定的顺序。
- 架构：通用
- 目的：提高指令缓存命中率，减少分支预测错误，减少页面故障和内存访问延迟

### *RetpolineInsertion*

- 作用：缓解分支目标注入攻击（如 Spectre 变种 2）的技术，通过将间接分支（如间接调用和跳转）替换为调用 retpolines。这种方法确保推测执行不能预测性地执行恶意代码

- **BB0**: 这个基本块包含对 BB2 的调用。

	**BB1**: 这个基本块包含一个暂停指令，然后是一个 lfence（如果启用），最后跳转到自身。

	**BB2**: 这个基本块处理实际的重定向。它要么将目标地址移到一个寄存器中，要么从内存中加载，保存到堆栈，然后返回。

- 是二进制代码保护，非性能优化

>Retpoline 是一种特殊类型的跳板，防止推测执行跟随间接分支。Retpoline 本质上迫使 CPU 等待知道正确的目标地址后才继续执行

### *ShrinkWrapping*

- 作用：尽可能推迟和提前在函数中保存和恢复寄存器的操作：将保存寄存器的操作从函数的入口点推迟到第一次使用寄存器之前，并将恢复寄存器的操作从函数的出口点提前到最后一次使用寄存器之后
- 架构：通用（X86）
- 影响：①减少不必要的保存和恢复操作 ②引入额外的跳转和指令 ③对齐

### *SplitFunctions*

- 作用：将一个函数拆分成多个片段（fragments），通常是根据热区（hot）和冷区（cold）来划分。这种拆分可以帮助优化器更好地组织代码，提高代码的缓存命中率，减少不必要的分支跳转，提高执行效率。【需要Perf】

- 架构：通用

- 目的：提高缓存命中率，减少分支跳转，代码大小优化，提升性能

- > 支持：
	>
	> **`SplitProfile2`**：根据分析的热区和冷区信息进行拆分。
	>
	> **`SplitRandom2`**：随机选择拆分点，将函数分为热区和冷区两部分。
	>
	> **`SplitRandomN`**：随机选择多个拆分点，将函数分为多个片段。
	>
	> **`SplitAll`**：将每个基本块单独作为一个片段。

> 会增加CodeSize：涉及对齐填充、跳转指令、重复代码、额外的数据结构来维护状态

### *StackAllocationAnalysis*

追踪函数执行过程中栈指针（Stack Pointer, SP）的变化

### *StackAvailableExpressions*

追踪程序中堆栈位置的存储指令（store）是否在程序的其他位置可以被重新使用（即检测是否存在可用表达式）

### *StackPointerTracking*

打印Stack信息（只有几行代码）

### *StackReachingUses*

- 作用：追踪堆栈位置上的加载指令，确定存储指令的影响，数据流分析【不包含优化，只分析】

- 架构：通用

- 目的：识别存储和加载之间的关系

- 举例：

	```asm
	BB1:
	    mov eax, [esp + 4]   ; load
	    mov [esp + 8], ebx   ; store
	    jmp BB2
	
	BB2:
	    mov ecx, [esp + 8]   ; load
	    ret
	```

- 分析过程
- **初始化**：`preflight` 函数识别 `BB1` 和 `BB2` 中的加载指令，记录在 `Expressions` 集合中
- **计算活跃表达式**：对于 `BB1`，`computeNext` 函数首先处理 `mov eax, [esp + 4]` 加载指令，将其添加到活跃表达式集合中。然后处理 `mov [esp + 8], ebx` 存储指令，移除所有从 `[esp + 8]` 加载的数据。跳转到 `BB2` 后，处理 `mov ecx, [esp + 8]` 加载指令，将其添加到活跃表达式集合中。

### *StokeInfo*

- 作用：收集和分析二进制函数的信息，并将这些信息输出为 .csv 文件，以供 Stoke（一个用于汇编代码优化的工具）使用

- 架构：通用

- 收集信息：

	```c++
	函数名称
	函数偏移量
	函数大小
	非伪指令数
	基本块数
	循环信息
	寄存器使用情况
	函数得分
	```

	> Stoke：https://github.com/StanfordPL/stoke
	>
	> STOKE 是用于 x86-64 指令集的随机优化器和程序合成器。STOKE 使用随机搜索来探索所有可能的程序转换的极高维空间。虽然任何一种随机转换都不太可能产生理想的代码序列，但重复应用数百万次转换足以产生新颖且非显而易见的代码序列。STOKE 可用于许多不同的场景，例如优化代码以提高性能或大小、从头开始合成实现或以浮点计算的准确性换取性能。作为超级优化器，STOKE 已被证明优于通用和特定领域编译器生成的代码，在某些情况下甚至优于专家手写代码。

### *TailDuplication*

- 作用：通过尾部重复（Tail Duplication）优化来减少分支跳转，提高代码的局部性，从而提升程序的性能【增加CodeSize】

- 架构：通用

- 目的：优化代码的控制流，减少分支跳转次数和跳转距离，从而提高指令缓存的命中率和程序执行的效率

- 条件：它们不是同一个块，尾部块没有后继块，前驱块和尾部块在布局中不相邻，并且它们不包含跳转表

- ```asm
	BB1:
	    ; 一些指令
	    jmp BB3
	
	BB2:
	    ; 一些指令
	    jmp BB3
	
	BB3:
	    ; 尾部指令
	    ; 尾部指令
	    ; 尾部指令
	    ret
	```

- 修改为：

- ```asm
	BB1:
	    ; 一些指令
	    ; 尾部指令
	    ; 尾部指令
	    ; 尾部指令
	    ret
	
	BB2:
	    ; 一些指令
	    ; 尾部指令
	    ; 尾部指令
	    ; 尾部指令
	    ret
	
	; BB3 被移除，因为它的内容已被复制
	```

	

### *ThreeWayBranch*

- 作用；优化包含两个条件分支的基本块，通过重新排列分支指令的顺序和目标，从而提高执行效率。具体来说，它针对具有两个后继基本块的“热”基本块（执行计数较高），并且后继块之一包含单一条件跳转指令的场景进行优化。【需要Perf】
- 架构：通用
- 目的：重新排列分支指令，提高代码执行路径的命中率，减少错误预测和流水线停顿，从而优化性能。这种优化主要应用于热路径中的三分支情况，即一个基本块分支到两个不同的基本块，然后其中一个基本块再分支到两个不同的基本块，共形成三个不同的执行路径。
- 流程：
	- 判断函数是否需要优化
	- 首先检查基本块的执行计数、后继数量以及是否包含跳转表。对于符合条件的基本块，通过分析其两个后继基本块及其内部的跳转指令，计算新的条件码和分支顺序，调整基本块的控制流。

### *ValidateInternalCalls*

- 作用：验证内部调用，修复控制流图，追踪和调整栈指针
- 架构：X86
	- 如果找到内部调用指令，则将当前基本块分割成两个部分，并创建一个新的基本块来处理后续指令。更新控制流图，将新的基本块加入处理队列，并将当前基本块的所有后继基本块转移到新创建的基本块上。
	- 通过CFG分析栈指针，确保正确



### *ValidateMemRefs*

- 作用：验证和修复二进制代码中的内存引用，特别是跳转表（Jump Table）引用。

- 架构：X86

	- **验证内存引用**：检查二进制函数中的内存引用，确保这些引用是合法的。
	- **修复不合法的内存引用**：特别是针对跳转表（Jump Table）引用，如果发现不合法的引用（例如引用了另一个函数的跳转表），则将这些引用替换为正常的只读数据引用。
	- **提高代码正确性和稳定性**：通过修复不合法的内存引用，防止潜在的运行时错误，确保程序的正确性和稳定性。

- 处理跳转表：（B去修改A是不对的）

	```asm
	.section .rodata
	.align 4
	.LJumptableFuncA:
	    .long .Lcase1FuncA
	    .long .Lcase2FuncA
	    .long .Lcase3FuncA
	
	.text
	.global FuncA
	FuncA:
	    movl $1, %eax                 # Load case index (1)
	    movl .LJumptableFuncA(,%eax,4), %ebx  # Load address from jump table
	    jmp *%ebx                     # Jump to the case label
	
	.Lcase1FuncA:
	    # Case 1 code for FuncA
	    ret
	
	.Lcase2FuncA:
	    # Case 2 code for FuncA
	    ret
	
	.Lcase3FuncA:
	    # Case 3 code for FuncA
	    ret
	
	.global FuncB
	FuncB:
	    movl $2, %eax                 # Load case index (2)
	    movl .LJumptableFuncA(,%eax,4), %ebx  # Load address from jump table of FuncA
	    jmp *%ebx                     # Jump to the case label
	```

- 修改为：

	```asm
	.global FuncB
	FuncB:
	    movl $2, %eax                 # Load case index (2)
	    movl .LDataSegment(,%eax,4), %ebx  # Load address from data segment
	    jmp *%ebx                     # Jump to the case label
	
	.section .rodata
	.align 4
	.LDataSegment:
	    .long .Lcase1FuncB
	    .long .Lcase2FuncB
	    .long .Lcase3FuncB
	
	.Lcase1FuncB:
	    # Case 1 code for FuncB
	    ret
	
	.Lcase2FuncB:
	    # Case 2 code for FuncB
	    ret
	
	.Lcase3FuncB:
	    # Case 3 code for FuncB
	    ret
	```

	

### *VeneerElimination*

- 作用：移除链接器插入的“Veneer”代码，并将调用这些“Veneer”的地方重定向到它们的实际目标
- 架构：Aarch64
- 遍历所有二进制函数，识别出其中的Veneer函数，将其标记为伪函数，并记录每个Veneer函数的目标符号。接下来，算法处理可能存在的嵌套Veneer跳转，确保每个Veneer最终指向正确的目标。之后，算法再次遍历所有函数，检查每个调用指令，如果该指令的目标是一个Veneer，则将其目标更新为实际的目标函数。最后，算法输出移除了多少个Veneer以及更新了多少个调用指令的信息。这个过程有效地减少了运行时的指令数，从而提高了程序的执行效率。
- 目标：**消除Veneer**，更新所有调用这些Veneer的指令，使其直接跳转到最终的目标地址，而不再通过Veneer中转，从而减少运行时的指令数，提高执行效率。

>```asm
>veneer:
>	LDR PC, [PC, #offset] ; 从PC相对的一个偏移位置加载目标地址
>	.word target_address  ; 实际目标地址
>```
>
>**函数调用**：当函数的调用者和被调用者距离较远时，需要使用Veneer。
>
>**分支跳转**：类似地，当分支跳转的目标地址超出指令能表示的范围时，也需要使用Veneer。
>
>
