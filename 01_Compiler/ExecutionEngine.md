# LLVM ExecutionEngine

> https://zhuanlan.zhihu.com/p/60936932

[TOC]

## Introduction

### Example

​	JIT（[just-in-time](https://zhida.zhihu.com/search?q=just-in-time&zhida_source=entity&is_preview=1)）即时编译技术是在运行时（runtime）将调用的函数或程序段编译成机器码载入内存，以加快程序的执行。所以，JIT是一种提高程序时间和空间有效性的方法。 程序运行时编译和执行的概念最早出自John McCarthy在1960年发表的论文《Recursive functions of symbolic expressions and their computation by machine》，James Gosling在1993年在关于Java的论文中使用了”JIT”这个术语。JIT可以分为两个阶段：==在运行时生成机器码==和==在运行时执行机器码==。

​	其中，第一个阶段的生成机器码方式与静态编译并无本质不同，**只不过生成的机器码被保存在内存中，而静态编译是在程序运行前将整个程序完全编译为机器码保存在[二进制文件](https://zhida.zhihu.com/search?q=二进制文件&zhida_source=entity&is_preview=1)中**。**运行时 JIT 缓存编译后的机器码，当再次遇到该函数时，则直接从缓存中执行已编译好的机器**。因此，从理论上来说，<u>JIT编译技术的性能会越来越接近静态编译技术</u>。

​	为了模拟JIT的运行原理，如下代码演示了如何在内存中动态生成add函数并执行，该函数的C语言原型如下：

```c++
long add(long num) { return num + 1; }

void *alloc_writable_memory(size_t size) {
  void *ptr =
      mmap(0, size, PROT_READ | PROT_WRITE, MAP_PRIVATE | MAP_ANONYMOUS, -1, 0);
  if (ptr == (void *)-1) {
    perror("mmap");
    return NULL;
  }
  return ptr;
}

void emit_code_into_memory(unsigned char *m) {
  unsigned char code[] = {
      0x48, 0x89, 0xf8,       // mov %rdi, %rax
      0x48, 0x83, 0xc0, 0x01, // add $1, %rax
      0xc3                    // ret
  };
  memcpy(m, code, sizeof(code));
}
// mprotect修改内存区域的保护属性（权限），size为内存区域大小
int make_memory_executable(void *m, size_t size) {
  if (mprotect(m, size, PROT_READ | PROT_EXEC) == -1) {
    perror("mprotect");
    return -1;
  }
  return 0;
}

const size_t SIZE = 1024;
typedef long (*JittedFunc)(long);

// Allocates RWX memory directly.
void emit_to_rw_run_from_rx() {
  void *m = alloc_writable_memory(SIZE);	// 分配可读写的内存
  emit_code_into_memory(m);								// 将机器码写入内存
  make_memory_executable(m, SIZE);				// 修改内存权限为只读和可执行

  JittedFunc func = m;										// 将内存区域作为函数指针
  int result = func(2);										// 执行动态生成的代码
  printf("result = %d\n", result);
}
```

上述代码主要可分为三步：

1. alloc_writable_memory调用mmap**在堆上分配可读/可写/可执行内存块**；
2. emit_code_into_memory将实现add函数的字符串形式机器码拷贝到内存块中。这一步骤可类比为JIT中调用运行时生成机器码；
3. 将内存块转换为指针类型并调用执行。这一步骤可类比为JIT中通过获得函数地址调用函数。

### ExecutionEngine

​	LLVM JIT使用执行引擎来支持LLVM模块的执行。ExecutionEngine类的申明在`<llvm_source>/include/llvm/ExecutionEngine/ExecutionEngine.h`中，执行引擎既可以用JIT也可以用解释器的方式支持执行。==执行引擎负责管理整个客体(guest)程序的执行==，分析需要执行的下一个程序片段。

> 客体程序是指不能被硬件平台原生支持的代码，比如，对于x86平台来说，LLVM IR模块就是客体程序，因为x86平台不能直接执行LLVM IR代码。

​	在LLVM中有三个持续演进的JIT执行引擎实现：llvm::JIT类、llvm::MCJIT类和llvm::ORCJIT类。

- llvm::JIT类在新的LLVM已经不再支持。
- JIT客户端会首先产生一个ExecutionEngine对象。ExecutionEngine对象以IR模块为输入，通过调用ExecutionEngine:: EngineBuilder()初始化。接下来，ExecutionEngine::create()方法生成一个JIT或MCJIT引擎实例。

<img src="/Users/yilingqinghan/Documents/BaiduPan/④笔记/⑦编译/assets/编译器大全.png" alt="编译器大全" style="zoom:10%;" />

### Memory Management

​	JIT引擎的ExecutionManager类==调用LLVM[代码生成](https://zhida.zhihu.com/search?q=代码生成&zhida_source=entity&is_preview=1)器，产生目标平台机器指令的二进制代码保存在内存中，并返回指向编译后函数的指针==。然后通过函数指针指向指令所在内存区域即可执行该函数。在此过程中，内存管理负责执行内存分配、释放、权限处理、库加载空间分配等操作。

​	JIT和MCJIT各自实现派生自RTDyldMemoryManager基类的定制内存管理类。执行引擎客户端也可以定制RTDyldMemoryManager子类，由该子类指定JIT部件在内存中的存放位置。RTDyldMemoryManager定义在`<llvm_source>/ include/llvm/ExecutionEngine/ RTDyldMemoryManager.h`中。

​	RTDyldMemoryManager类声明了如下方法：

- allocateCodeSection()和allocateDataSection()：这两个方法分配内存保存可执行代码和数据，内存管理客户端可以用一个内部节（section）标识符参数追踪分配的节。
- getSymbolAddress()：该方法返回链接的库中可用[符号表](https://zhida.zhihu.com/search?q=符号表&zhida_source=entity&is_preview=1)地址。注意，这个方法不能用于获取JIT编译产生的符号表。调用该方法时必须提供一个std::string实例保存符号名称。
- finalizeMemory()：MCJIT客户端完成对象加载后调用此方法设定内存权限，必须在调用此方法后才能运行生成的代码。

JIT和MCJIT的缺省内存管理子类分别是JITMemoryManager和SectionMemoryManager。

### llvm::JIT

​	支持JIT的LLVM后端要实现二进制代码发射，JIT类通过MachineCodeEmitter子类JITCodeEmitter发射二进制指令，将二进制块字节写入内存。MachineCodeEmitter类用于发射机器码，但与新的MC框架无关，只支持少数几个后端。

MachineCodeEmitter类[成员函数](https://zhida.zhihu.com/search?q=成员函数&zhida_source=entity&is_preview=1)完成以下工作：

- a. allocateSpace()：为当前要发射的函数分配空间。

  b. emitByte()、emitWordLE()、emitWordBE()、emitAlignment()等：将二进制块写入缓存。

  c. 追踪当前缓存地址，即指向下一条要发射指令地址的指针。

  d. ==增加相对缓存中指令地址的重定位==。

​	JIT执行引擎也会用到JITMemoryManager和JITResolver。JITMemoryManager负责管理内存的使用，实现低层内存处理方法。例如，allocateGlobal()方法为全局变量分配内存，startFunctionBody()方法为发射的指令分配内存，并标记为读/写可执行。JITMemoryManager类声明在`<llvm_source>/include/llvm/ ExecutionEngine/JITMemoryManager.h`。

> JITResolver负责记录在哪些位置调用了未编译函数。

​	支持JIT的LLVM后端要实现两个类：<target>CodeEmitter和<target>JITInfo。<target>CodeEmitter包含了一个机器函数（machine function）pass，==将目标机器指令转换为可重定位的机器码==。例如，MipsCodeEmitter会遍历所有函数[基本块](https://zhida.zhihu.com/search?q=基本块&zhida_source=entity&is_preview=1)，为每一条机器指令调用emitInstruction()：

```c++
for (MachineBasicBlock::instr_iterator I = MBB->instr_begin(), E = MBB->instr_end(); I != E;)
	emitInstruction(*I++, *MBB);
}
```

​	为了支持JIT编译，编译器必须提供TargetJITInfo子类（见include/llvm/Target/TargetJITInfo.h），例如MipsJITInfo或X86JITInfo。TargetJITInfo类为所有目标平台编译器都需要实现的公共JIT功能提供了接口，包括为代码生成阶段中的各种活动，如代码发射，实现JIT接口。这些公共JIT功能包括：

- ==支持执行引擎重新编译经过修改的方法==，实现TargetJITInfo::replaceMachineCodeForFunction()方法，并在[原函数](https://zhida.zhihu.com/search?q=原函数&zhida_source=entity&is_preview=1)的对应位置通过补丁方式加入jump指令或调用新函数。这对自修改（self-modifing）代码是必须的。
- TargetJITInfo::relocate()方法在当前发射的函数中的每一个符号引用打补丁以指向正确的存储地址，类似[动态链接](https://zhida.zhihu.com/search?q=动态链接&zhida_source=entity&is_preview=1)的做法。
- TargetJITInfo::emitFunctionStub()方法==发射一个桩函数，桩函数在给定地址调用另一个函数==。<u>==每个目标平台编译器都应提供TargetJITInfo::StubLayout信息==</u>，包括发射的桩函数大小和对齐方式。在发射新的桩函数前，JITEmitter使用桩函数信息为其分配空间。

​	TargetJITInfo类方法的目标不是发射普通指令，但会发射生成桩函数的特殊指令（例如MipsJITInfo::emitFunctionStub()），以及调用新内存位置的特殊指令。

### How to use class JIT

​	JIT是ExecutionEngine子类，声明在<llvm_source>/lib/ExecutionEngine/JIT/JIT.h。JIT类是JIT编译函数的入口。ExecutionEngine::create()以缺省JITMemoryManager为参数调用JIT::createJIT()，然后JIT[构造函数](https://zhida.zhihu.com/search?q=构造函数&zhida_source=entity&is_preview=1)执行以下任务：

- 生成JITEmitter实例；
- 初始化目标信息对象；
- 添加代码生成pass；
- 添加<Target>CodeEmitter pass；

​	当JIT编译某个函数时，引擎中的PassManager对象调用代码生成和JIT指令发射pass。步骤如下：





## Modules



### Interpreter

### MCJIT

### ORC

### RuntimeDyld

### JITLink

## Others
