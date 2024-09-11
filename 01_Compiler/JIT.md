# JIT Linker

[TOC]

​	JITLink 是一个用于即时链接（JIT Linking）的库。它是为支持 ORC JIT API 而构建的，最常通过 ORC 的 ObjectLinkingLayer API 访问。JITLink 的开发目标是支持每种对象格式提供的全部功能，包括静态初始化器、异常处理、线程局部变量和语言运行时注册。支持这些功能使 ORC 能够执行依赖于这些功能的源语言生成的代码（例如，C++ 需要对象格式支持静态初始化器以支持静态构造函数，eh-frame 注册用于异常处理，TLV 支持线程局部变量；Swift 和 Objective-C 需要语言运行时注册来支持许多功能）。对于某些对象格式功能的支持完全在 JITLink 内部提供，而对于其他功能则与（原型）ORC 运行时协作提供支持。

JITLink 旨在支持以下功能，其中一些仍在开发中：

1. 将单个可重定位Object跨进程和跨架构链接到目标执行器进程中。
2. 支持所有Object格式功能。
3. 开放的链接器数据结构（LinkGraph）和传递系统。

> ##### *ORC*
>
> ORC (On Request Compilation) 是 LLVM 项目中的一个即时编译（JIT）框架。它提供了一组 API，使用户能够在运行时生成和执行机器代码。ORC 通过支持灵活和可扩展的 JIT 编译管道，为开发人员提供了构建高效和复杂的即时编译系统的工具。
>
> 核心组件：
>
> **ExecutionSession**：管理 JIT 编译的整个生命周期，处理符号查找和模块管理。
>
> **JITDylib**：管理符号的动态库，每个 JITDylib 都包含一组可以在运行时加载和卸载的符号。
>
> **ObjectLinkingLayer**：负责处理对象文件的加载和链接。
>
> **CompileLayer**：负责编译 IR（中间表示）到机器代码。

## 概念一：LinkGraph

​	JITLink 将所有可重定位对象格式映射到一个通用的 LinkGraph 类型，该类型旨在==使链接快速而简单==（也可以手动创建 LinkGraph 实例。参见构造 LinkGraphs）。

​	==可重定位对象格式（例如 COFF、ELF、MachO）==在细节上有所不同，但有一个共同目标：用**注释表示机器级代码和数据，使它们能够在虚拟地址空间中重定位**。为此，它们通常包含文件内部或外部定义的内容名称（符号）、必须作为单元移动的内容块（取决于格式，可以是 sections 或 subsections），以及描述如何根据某些目标符号/section 的最终地址修补内容的注释（重定位）。

​	**在高级别上，LinkGraph 类型将这些概念表示为一个装饰图**。图中的**节点**表示**符号**和**内容，**边表示**重定位**。图的每个元素如下所示：

- **Addressable**：一个可以在执行进程的虚拟地址空间中**分配地址**的图节点。

  - ==绝对和外部符号==使用普通的 Addressable 实例表示。
  - 对象文件内定义的内容使用 Block 子类表示。

- **Block**：一个包含内容（或标记为零填充）的 Addressable 节点，具有父 Section、大小、对齐方式（和对齐偏移量），以及一个 Edge 实例列表。

  Blocks 提供一个容器，用于在目标地址空间中必须保持连续的二进制内容（布局单元）。许多对 LinkGraph 实例的低级操作涉及检查或修改块内容或边。

  - **Content** 以 `llvm::StringRef` 表示，并通过 `getContent` 方法访问。内容仅适用于内容块，不适用于零填充块（使用 `isZeroFill` 进行检查，并在只需要块大小时优先使用 `getSize`，因为它适用于零填充和内容块）**Section** 以 `Section&` 引用表示，并通过 `getSection` 方法访问。Section 类在下面将详细描述。

  	**Size** 以 `size_t` 表示，并通过 `getSize` 方法访问，适用于内容块和零填充块。

  	**Alignment** 以 `uint64_t` 表示，并通过 `getAlignment` 方法访问。它表示块开始位置的最小对齐要求（以字节为单位）。

  	**AlignmentOffset** 以 `uint64_t` 表示，并通过 `getAlignmentOffset` 方法访问。它表示块开始位置的对齐偏移量。这是为了支持那些最小对齐要求来自块内部某个非零偏移数据的块。例如，如果一个块由一个字节（具有字节对齐）后跟一个 `uint64_t`（具有8字节对齐）组成，那么该块将具有8字节对齐和7的对齐偏移量。

  	**Edge 实例列表**。通过 `edges` 方法返回此列表的迭代器范围。Edge 类将在下面详细描述。

- **Symbol** – 从可分配地址的节点（通常是 Block）偏移，具有可选的名称、链接类型、作用域、可调用标志和活跃标志。

  - **Name**：以 `llvm::StringRef` 表示，通过 `getName` 方法访问。如果符号没有名称，则等于 `llvm::StringRef()`。

  	**Linkage**：可以是 Strong 或 Weak，通过 `getLinkage` 方法访问。JITLinkContext 可以使用此标志来确定是否保留或丢弃该符号定义。

  	**Scope**：可以是 Default、Hidden 或 Local，通过 `getScope` 方法访问。JITLinkContext 可以使用此标志来确定谁可以看到该符号。具有默认作用域的符号应全局可见。具有隐藏作用域的符号应仅对同一模拟动态库（例如 ORC JITDylib）或可执行文件中的其他定义可见，而对其他地方不可见。具有本地作用域的符号应仅在当前 LinkGraph 内可见。

  	**Callable**：一个布尔值，如果该符号可以被调用，则设置为 true，并通过 `isCallable` 方法访问。这可以用于自动引入惰性编译的调用存根。

  	**Live**：一个布尔值，可以设置为将该符号标记为根，以用于删除无用代码（dead-stripping）目的（参见通用链接算法）。JITLink 的删除无用代码算法将通过图传播活跃标志到所有可到达的符号，然后删除任何未标记为活跃的符号（和块）。

- **Edge** – 一个包含偏移量（从包含它的 Block 开始隐式计算）、类型（描述重定位类型）、目标和附加值的四元组。

	Edges 表示块和符号之间的重定位，有时也表示其他关系。

	- **Offset**：通过 `getOffset` 方法访问，是从包含该 Edge 的 Block 开始的偏移量。

		**Kind**：通过 `getKind` 方法访问，是一种重定位类型——它描述了根据目标的地址在给定的偏移量上应该对块内容进行哪些更改（如果有的话）。

		**Target**：通过 `getTarget` 方法访问，是一个指向 Symbol 的指针，表示在边缘类型指定的修正计算中与目标相关的地址。

		**Addend**：通过 `getAddend` 方法访问，是一个常量，其解释由边缘的类型决定。

- Section – 一组 Symbol 实例，加上一组 Block 实例，具有名称、一组 ProtectionFlags 和一个序号。

	Sections 使得遍历源对象文件中特定 section 关联的符号或块变得容易。

	- **blocks()** 返回 section 中定义的块集合的迭代器（作为 Block* 指针）。

		**symbols()** 返回 section 中定义的符号集合的迭代器（作为 Symbol* 指针）。

		**Name** 以 `llvm::StringRef` 表示，并通过 `getName` 方法访问。

		**ProtectionFlags** 以 `sys::Memory::ProtectionFlags` 枚举表示，并通过 `getProtectionFlags` 方法访问。这些标志描述了该 section 是否可读、可写、可执行或它们的某种组合。最常见的组合是 RW-（可写数据）、R--（常量数据）和 R-X（代码）。

		**SectionOrdinal** 通过 `getOrdinal` 方法访问，是一个用于相对其他 section 排序的数字。通常在布局内存时用于保持段内节的顺序（具有相同内存保护的一组节）。


> <u>对图论学者而言：</u>
>
> ​	<u>LinkGraph 是二分图，具有一组 Symbol 节点和一组 Addressable 节点。每个 Symbol 节点都有一个（隐含的）边指向其目标 Addressable。每个 Block 具有一组边（可能为空，由 Edge 实例表示）返回到 Symbol 集的元素。为了方便和提高常见算法的性能，符号和块进一步分组到 Sections 中。</u>

### 方法

#### 图元素操作

- **sections** 返回图中所有 sections 的迭代器。
- **findSectionByName** 返回具有给定名称的 section 的指针（作为 Section*），如果不存在则返回 nullptr。
- **blocks** 返回图中所有 blocks 的迭代器（跨所有 sections）。
- **defined_symbols** 返回图中所有已定义符号的迭代器（跨所有 sections）。
- **external_symbols** 返回图中所有外部符号的迭代器。
- **absolute_symbols** 返回图中所有绝对符号的迭代器。
- **createSection** 使用给定的名称和保护标志创建一个 section。
- **createContentBlock** 使用给定的初始内容、父 section、地址、对齐方式和对齐偏移创建一个 block。
- **createZeroFillBlock** 创建一个具有给定大小、父 section、地址、对齐方式和对齐偏移的零填充块。
- **addExternalSymbol** 创建一个具有给定名称、大小和链接性的新的可寻址符号。
- **addAbsoluteSymbol** 创建一个具有给定名称、地址、大小、链接性、范围和存活性的新的可寻址符号。
- **addCommonSymbol** 便捷函数，用于创建一个具有给定名称、范围、section、初始地址、大小、对齐和存活性的零填充块和弱符号。
- **addAnonymousSymbol** 为给定块创建一个新的匿名符号，具有偏移、大小、可调用性和存活性。
- **addDefinedSymbol** 为给定块创建一个新的符号，具有名称、偏移、大小、链接性、范围、可调用性和存活性。
- **makeExternal** 将以前定义的符号转换为外部符号，方法是创建一个新的可寻址对象并将符号指向它。现有块不会被删除，但可以通过调用 removeBlock 手动移除（如果未引用）。所有指向该符号的边仍然有效，但该符号现在必须在此 LinkGraph 之外定义。
- **removeExternalSymbol** 移除一个外部符号及其目标可寻址对象。目标可寻址对象不能被任何其他符号引用。
- **removeAbsoluteSymbol** 移除一个绝对符号及其目标可寻址对象。目标可寻址对象不能被任何其他符号引用。
- **removeDefinedSymbol** 移除一个已定义符号，但不移除其目标块。
- **removeBlock** 移除给定的块。
- **splitBlock** 在给定索引处分割一个块为两个块（在已知块包含可分解记录的情况下非常有用，例如 eh-frame section 中的 CFI 记录）。

#### 图实用操作

- **getName** 返回此图的名称，通常基于输入对象文件的名称。
- **getTargetTriple** 返回执行进程的 `llvm::Triple`。
- **getPointerSize** 返回执行进程中指针的大小（以字节为单位）。
- **getEndianness** 返回执行进程的字节序。
- **allocateString** 从给定的 `llvm::Twine` 中复制数据到链接图的内部分配器。这可以确保在一个 pass 内创建的内容在该 pass 执行完毕后依然有效。

## 实践-X86_64

调试llvm-jitlink.

```shell
% cat print-args.c
#include <stdio.h>

void print_args(int argc, char *argv[]) {
  for (int i = 0; i != argc; ++i)
    printf("arg %i is \"%s\"\n", i, argv[i]);
}

% cat print-args-main.c
void print_args(int argc, char *argv[]);

int main(int argc, char *argv[]) {
  print_args(argc, argv);
  return 0;
}

% clang -c -o print-args.o print-args.c
% clang -c -o print-args-main.o print-args-main.c
% llvm-jitlink print-args.o print-args-main.o -args a b c
arg 0 is "a"
arg 1 is "b"
arg 2 is "c"
```

堆栈:

```c++
llvm::jitlink::JITLinker::linkPhase4	JITLinkGeneric.cpp
llvm::jitlink::JITLinker::linkPhase3	JITLinkGeneric.cpp
llvm::jitlink::JITLinker::linkPhase2	JITLinkGeneric.cpp
llvm::jitlink::JITLinker::linkPhase1	JITLinkGeneric.cpp
llvm::jitlink::JITLinker							JITLinkGeneric.h
llvm::jitlink::link_ELF_x86_64				ELF_X86_64.cpp
llvm::jitlink::link_ELF								ELF.cpp
llvm::jitlink::link										JITLink.cpp
llvm::orc::ObjectLinkingLayer::emit		ObjectLinkingLayer.cpp
llvm::orc::BasicObjectLayerMaterializationUnit::materialize					Layer.cpp
llvm::orc::MaterializationTask::run																	Layer.cpp
llvm::orc::ExecutionSession::runOnCurrentThread											Core.cpp
```

- **阶段一**：启动链接的第一阶段，主要处理链接图的剪枝（Pruning）和优化。
	- *PrePrunePasses*：`ELF_X86_64.cpp`配置
	- *Prune*：在JITLinkGeneric.cpp，架构无关
	- *PostPrunePasses*：`ELF_X86_64.cpp`配置

- **阶段二**：处理分配后的任务，包括解决外部符号引用
	- *PostAllocationPasses*
	- **notifyResolved**：理应无用
	- 获取**JITLinkerBase**::**getExternalSymbolNames**()

- **阶段三**：==**处理外部符号的查找结果，进行符号地址分配，并准备修正代码。**==
  - *PreFixupPasses*
  - 核心：**fixUpBlocks**：重定位
  - *PostFixupPasses*：通用
- **阶段四**：完成链接过程，进行最终的内存提交和客户端通知。

### 官网的阐释

#### JITLink 提供的通用链接算法及其扩展

​	JITLink 提供了一种通用的链接算法，可以通过引入 JITLink Passes 在某些点进行扩展或修改。

​	在每个阶段结束时，链接器将其状态打包成一个持续操作，并调用 JITLinkContext 对象执行（可能有高延迟的）异步操作：分配内存、解析外部符号，最后将链接的内存转移到执行进程。

#### 阶段 1

这一阶段在初始配置（包括 Pass 管道设置）完成后立即由链接函数调用。

**运行预剪枝 Passes**：
	这些 Passes 在图被剪枝之前调用。此时，LinkGraph 节点仍然具有其原始的虚拟内存地址（vmaddrs）。在这一序列结束时，将运行一个标记活动节点的 Pass（由 JITLinkContext 提供），以标记初始的一组活动符号。

*显著用例：标记节点为活动状态，访问/复制将被剪枝的图数据（例如，对 JIT 重要但对链接过程不需要的元数据）。*

**剪枝（去除无用代码）LinkGraph**：
	移除所有不在初始活动符号集中的符号和块。这使得 JITLink 可以移除不可达的符号/内容，包括被覆盖的弱符号和冗余的 ODR 定义。

**运行后剪枝 Passes**：
	这些 Passes 在去除无用代码后运行，但在分配内存或节点分配其最终目标虚拟内存地址之前运行。此阶段的 Passes 受益于剪枝，因为已移除无用的函数和数据，但仍然可以向图中添加新内容，因为目标和工作内存尚未分配。

*显著用例：构建全局偏移表（GOT）、过程链接表（PLT）和线程局部变量（TLV）条目。*

**异步分配内存**：
	调用 JITLinkContext 的 JITLinkMemoryManager 为图分配工作和目标内存。在此过程中，JITLinkMemoryManager 将更新图中定义的所有节点的地址到其分配的目标地址。

<u>注意：此步骤仅更新此图中定义的节点的地址。外部符号仍然具有空地址。</u>

#### 阶段 2

**运行后分配 Passes**：
	这些 Passes 在工作和目标内存分配之后运行，但在 JITLinkContext 被通知图中符号的最终地址之前运行。这给这些 Passes 一个机会在任何 JITLink 客户端（尤其是 ORC 对符号解析的查询）尝试访问它们之前，设置与目标地址关联的数据结构。

*显著用例：设置目标地址和 JIT 数据结构之间的映射，例如 `__dso_handle` 和 `JITDylib*` 之间的映射。*

**通知 JITLinkContext 已分配的符号地址**：
	调用 JITLinkContext::notifyResolved 在链接图上，允许客户端对为此图分配的符号地址作出反应。在 ORC 中，这用于通知任何待处理的已解析符号查询，包括来自并发运行的 JITLink 实例的待处理查询，这些实例已达到下一步并等待此图中的符号地址以继续其链接。

**识别外部符号并异步解析其地址**：
	调用 JITLinkContext 来解析图中任何外部符号的目标地址。

#### 阶段 3

**应用外部符号解析结果**：
	这会更新所有外部符号的地址。此时，图中的所有节点都具有其最终目标地址，但节点内容仍然指向对象文件中的原始数据。

**运行预修正 Passes**：
	这些 Passes 在所有节点分配了最终目标地址之后调用，但在节点内容复制到工作内存并修正之前调用。在此阶段运行的 Passes 可以根据地址布局对图和内容进行后期优化。

*显著用例：GOT 和 PLT 的放松，当在分配的内存布局下可以直接访问修正目标时，绕过 GOT 和 PLT 的访问。*

**将块内容复制到工作内存并应用修正**：
	将所有块内容复制到分配的工作内存（遵循目标布局）并应用修正。图块将更新以指向修正后的内容。

**运行后修正 Passes**：
	这些 Passes 在应用修正并更新块以指向修正内容之后调用。后修正 Passes 可以检查块内容，查看将要复制到分配的目标地址的确切字节。

**异步最终确定内存**：
	调用 JITLinkMemoryManager 将工作内存复制到执行进程并应用请求的权限。

#### 阶段 4

**通知上下文图已被发射**：
	调用 JITLinkContext::notifyFinalized 并交接 JITLinkMemoryManager::FinalizedAlloc 对象，用于此图的内存分配。这允许上下文跟踪/保持内存分配并对新发射的定义作出反应。在 ORC 中，这用于更新 ExecutionSession 实例的依赖图，如果它们的所有依赖项也已发射，这些符号（可能还有其他符号）将变为 Ready。

### 核心文件与函数

#### **JITLink.h**

包括*class Edge*的定义。

```c++
class Edge {
public:
  using Kind = uint8_t;

  enum GenericEdgeKind : Kind {
    Invalid,                    // Invalid edge value.
    FirstKeepAlive,             // Keeps target alive. Offset/addend zero.
    KeepAlive = FirstKeepAlive, // Tag first edge kind that preserves liveness.
    FirstRelocation             // First architecture specific relocation.
  };

  using OffsetT = uint32_t;
  using AddendT = int64_t;

  Edge(Kind K, OffsetT Offset, Symbol &Target, AddendT Addend)
      : Target(&Target), Offset(Offset), Addend(Addend), K(K) {}

  OffsetT getOffset() const { return Offset; }
  void setOffset(OffsetT Offset) { this->Offset = Offset; }
  Kind getKind() const { return K; }
  void setKind(Kind K) { this->K = K; }
  bool isRelocation() const { return K >= FirstRelocation; }
  Kind getRelocation() const {
    assert(isRelocation() && "Not a relocation edge");
    return K - FirstRelocation;
  }
  bool isKeepAlive() const { return K >= FirstKeepAlive; }
  Symbol &getTarget() const { return *Target; }
  void setTarget(Symbol &Target) { this->Target = &Target; }
  AddendT getAddend() const { return Addend; }
  void setAddend(AddendT Addend) { this->Addend = Addend; }

private:
  Symbol *Target = nullptr;
  OffsetT Offset = 0;
  AddendT Addend = 0;
  Kind K = 0;
```

### 修改内容

#### ELF.cpp



#### OrcABISupport.h

- 是否需要支持sw的lazy JITing?

## Adding a new target/object backend to LLVM JITLink

https://blog.llvm.org/posts/2023-03-16-adding-new-llvm-jitlink-target-object-backend/

理解：

### Fixup

**Fixup** 是指在链接过程中，修改指令或数据中的地址引用，使其指向正确的目标地址。Fixup 通常涉及调整代码或数据中的相对或绝对地址，以确保它们在最终的内存布局中指向正确的位置。

- **时机**：Fixup 通常发生在所有目标地址都确定之后。在静态链接中，这是在编译完成并确定所有符号地址后进行的。在动态链接或JIT编译中，**Fixup 可能在符号的实际加载和运行时发生**。
- **操作**：Fixup 的操作包括：
  - 修正指令中的相对跳转地址。
  - 调整数据中的指针或偏移量。
  - 更新内存中特定位置的值，使其与实际地址匹配。


​	例如，在JIT编译中，一个函数可能在运行时被动态生成，其**地址在编译时无法确定**。Fixup 会在函数生成并确定地址后，修改所有引用该函数的地方，使其指向正确的地址。

### Resolve

**Resolve** 是指在链接过程中，查找符号的定义，并将符号名与其实际地址或值关联起来。Resolve 是确定符号引用（如函数调用或变量引用）实际指向的位置的过程。

- **时机**：Resolve 通常在符号表创建后和所有符号定义都可见时进行。在静态链接中，这是在编译完成并所有目标文件合并时。在动态链接或JIT编译中，Resolve 可能在运行时进行，当符号需要被实际使用时。
- **操作**：Resolve 的操作包括：
  - 查找符号表中的符号定义。
  - 将符号名与其定义的地址或值关联起来。
  - 确定所有符号引用的目标地址或值。

例如，当一个模块引用另一个模块中定义的函数时，Resolve 会在链接时找到该函数的定义，并将引用与该函数的实际地址关联起来。

#### 异同

**相同点**：
- 两者都涉及符号的地址处理。
- 都是链接过程中不可或缺的步骤，确保最终生成的可执行文件或模块中的所有符号引用都是正确的。

**不同点**：

- **Fixup** 侧重于修改和调整指令或数据中的地址引用，以确保它们指向正确的目标。这一步通常发生在所有符号地址确定之后。
- **Resolve** 侧重于查找和关联符号定义，使符号名与其实际地址或值关联起来。这一步通常发生在符号地址确定之前或确定过程中。

### JIT Linking介绍

​	与静态链接不同，JIT（即时）链接是在运行时执行的。静态链接器生成存储在磁盘上的可执行文件，而JIT链接器生成可执行文件的内存映像——本质上是准备在内存中执行的字节。JIT链接一个C程序可能感觉很像运行一个shell脚本。然而，在底层，C程序被链接到调用进程的内存中，也通常称为执行器进程。JIT链接器修补执行器进程的内存，以考虑符号在运行时的地址，并执行必要的初始化程序。

​	如果你熟悉动态加载(dynamic loading)，那么JIT链接听起来可能会很熟悉，并且两者有很多共同点，但它们并不相同。JIT链接作用于可重定位对象（而动态加载则针对共享对象/dylib），并同时执行静态链接器和动态加载器的工作。这样做允许JIT链接器剔除冗余符号，这是动态加载无法做到的，这使得JIT链接支持生成大量冗余符号定义的语言（例如C++）的更细粒度的编译。

### 为什么需要JITLink

​	JIT链接主要在预编译语言（例如C、C++、Rust等）的上下文中非常有用。为什么？**在运行时，这些语言没有办法将新的符号定义引入正在运行的进程内存并解析对它们的引用**。虽然动态加载部分解决了这个问题，但它有其缺点（如上所述），并且远远落后于静态链接的体验。

​	通过JIT链接，在运行时，符号引用可以解析为现有符号（来自新JIT生成的代码）或新JIT生成的符号（来自预编译代码）。下面的示例代码展示了这在代码中的表现。

> 动态加载（dynamic loading）是一种在程序运行时动态引入和使用可执行代码的技术，而不是在程序编译或链接时静态包含这些代码。这种技术广泛应用于现代操作系统和编程语言，主要用于加载共享库或动态链接库（如Windows中的DLL，Linux中的.so文件）。
>
> ### 动态加载的基本概念
>
> ​	在传统的静态链接中，所有必需的代码和数据在编译阶段和链接阶段就已经确定并包含在可执行文件中。静态链接器在编译时将所有引用解析为具体的地址，并将所有需要的库打包到最终的可执行文件中。与此相对，动态加载将一些代码和数据的加载和解析延迟到程序运行时。
>
> ​	动态加载通过动态链接器或加载器实现，该加载器在程序运行时负责查找、加载和链接动态库。这样，可以在程序启动时或在程序运行过程中根据需要加载库文件，并在内存中解析符号。
>
> ### 动态加载的实现过程
>
> 1. **加载共享库**：当程序需要使用动态库时，调用操作系统提供的动态加载函数（如`dlopen`，`LoadLibrary`）。动态加载器在指定的路径中查找并加载共享库，将其映射到程序的地址空间。
>
> 2. **解析符号**：加载共享库后，程序需要解析库中的符号（如函数或变量）。动态加载器通过符号表查找并解析这些符号，将它们的地址绑定到程序中的引用位置。常用的符号解析函数包括`dlsym`和`GetProcAddress`。
>
> 3. **执行初始化代码**：一些动态库在加载时可能包含初始化代码段，这些代码在库加载后立即执行。初始化代码通常用于设置库的全局状态或执行必要的初始化步骤。
>
> 4. **使用加载的符号**：一旦动态库加载并符号解析完成，程序就可以使用这些符号，如调用动态库中的函数或访问其中的变量。
>
> ### 动态加载的优点
>
> **内存效率**：动态加载允许多个程序共享相同的库代码，从而减少整体内存消耗。操作系统可以将动态库加载到共享内存区域中，多个进程可以同时访问。
>
> **更新和维护方便**：由于动态库是独立的文件，可以单独更新而不需要重新编译或重新链接整个应用程序。这使得软件维护和更新更加灵活。
>
> **延迟加载**：动态加载允许程序在需要时才加载库文件，而不是在启动时全部加载。这样可以缩短程序的启动时间，并减少内存占用。
>
> **插件机制**：动态加载提供了一种实现插件机制的有效方法，允许程序在运行时加载和使用不同的功能模块，而无需重新编译或重新启动。
>
> ### 动态加载的缺点
>
> **性能开销**：动态加载和符号解析在运行时进行，会带来一定的性能开销。与静态链接相比，动态链接的调用速度可能略慢。
>
> **依赖管理复杂**：动态库的路径、版本和依赖关系需要仔细管理，以避免运行时错误。如果某个动态库丢失或版本不兼容，可能导致程序无法运行。
>
> **安全风险**：动态加载器需要从指定路径加载库文件，<u>如果路径设置不当或加载不受信任的库，可能会引入安全风险，如代码注入攻击</u>。
>
> ### 动态加载的应用
>
> ​	动态加载广泛应用于操作系统内核、应用程序、服务器软件等各种场景。例如，Web服务器可以动态加载和卸载模块以扩展功能，图形界面程序可以动态加载插件以增加新的组件和特性，数据库系统可以动态加载存储过程和函数以支持自定义查询和操作。

​	虽然JIT链接本身对最终用户来说可能并不是非常有用，但它确实是针对预编译语言的某些用例（以及一些即时编译语言的用例）的一种使能工具。

​	以下是一些具体的应用场景，可以看出JIT链接的多样性和实用性：

1. **JIT编译器**：想象一下类似于Java Hotspot VM的JIT编译组件，但适用于静态编译的语言。
2. **调试器表达式求值器**：例如LLDB表达式求值器。
3. **交互式编程环境(REPL)**：如Cling和目前仍处于实验阶段的Clang-REPL。
4. **独立脚本**：例如Swift脚本，其中JIT链接被用来为编译器添加即时模式，通过JIT在原地运行代码，而不是编译它。
5. **可脚本化扩展**：考虑在某个现有应用的上下文中运行即时编译的代码，允许应用通过即时编译的代码而不是预编译的插件来扩展。

​	尽管上述用例看似不同，但它们实际上是相同的——JIT链接使得代码能以与应用二进制接口（ABI）兼容的方式链接到现有的进程中（这些进程可能已经包含了状态/上下文）。这样的技术使得动态和灵活的编程成为可能，特别是在需要即时更新或修改运行中程序的场景中。

### LLVM JITLink

​	LLVM JITLink 是 LLVM 中的一种即时链接实现，以低级库的形式存在。它支持 LLVM 的 ORC JIT API，这是终端用户通常用于构建运行时链接环境的工具。JITLink 提供了以下基本功能：

1. **利用现有的编译器在运行时生成可重定位的对象。**
2. **在目标执行器进程中分配内存。**
3. **以与应用二进制接口（ABI）兼容的方式将代码链接到目标执行器进程中。**

​	简单来说，一个**在进程 X** 中运行的程序 Y，可以将一个可重定位的对象文件交给 JITLink，JITLink 将会将该对象文件的代码链接到 X 的内存中，并在 X 现有的上下文（全局变量、函数等）下运行它，就好像它是加载到进程 X 中的动态库的一部分一样。这样的机制不仅增加了程序的灵活性，还允许在运行时动态地扩展或修改应用的功能，而无需停机或重新编译整个程序。

> ​	想象有一个正在运行的程序叫做 X。现在，我们有另一个程序 Y，它生成了一个特殊的文件，这个文件包含可以执行的代码，但还没有被安装到任何程序中去。
>
> ​	JITLink 这个工具的作用就是，它可以拿到程序 Y 生成的这个特殊文件，然后把文件里的代码“安装”到正在运行的程序 X 中。一旦代码被安装进去，它就可以在程序 X 的环境中运行，**使用程序 X 的数据和函数等资源，就好像这段代码一开始就是程序 X 的一部分一样**。
>
> ​	这个过程类似于你在电脑上安装一个新的程序插件，一旦安装，插件就能在电脑现有的系统中运行，利用系统的资源。JITLink 允许这种安装和运行过程在程序运行时动态进行，而不需要停止程序 X。这就是所谓的“即时链接”。

### 工程概述

- 任务 - 为 JITLink 添加 i386（目标）/ELF（对象）后端

- **什么是目标？**

	这里的目标指的是硬件架构。i386 是一种 32 位的 x86 架构。

- **什么是对象？**

	这里的对象指的是对象文件格式。ELF 是在 Linux 系统上常用的对象格式。

- **为什么不同的目标/对象组合很重要且需要额外的工作？**

	不同的目标/对象组合很重要，因为每种组合可能使用不同的方法来连接符号引用和符号定义。这些方法通常被称为重定位。

- **最终目标**

	该项目的最终目标是使 LLVM JITLink 能够将一个具有 i386 特定重定位的 32 位 ELF 对象文件链接到一个在 i386 硬件架构上的 32 位进程中。

### 执行流程

#### 了解高级架构

​	LinkGraph 是 LLVM JITLink 中Object File的内部表示。虽然不同的对象格式可能有不同的架构和术语来表示类似的概念，但它们的共同目标是表示可以在虚拟内存中重新定位的机器代码。*LinkGraph* 的目的是为这些概念和不同对象文件格式的细微差别提供一个通用的表示。

一个 ELF 对象包含：

- **Sections（段）** - 必须作为一个单位移动到内存中的任何字节块。
- **Symbols（符号）** - 可以代表数据或可执行指令的命名字节块。符号作为段的子元素出现。
- **Relocations（重定位）** - 一旦解析了重定位目标符号的地址，就修复段内字节的描述。

LinkGraph 能够表示上述所有概念。它首先定义了一些构建块：

- **Addressable（可寻址的）** - 任何可以在执行进程的虚拟地址空间中被分配一个地址的东西。
- **Block（块）** - 是可寻址的，并作为段的一部分出现的字节块。

在这些构建块的基础上，它定义了更高级别的对象格式概念：

- **Symbol（符号）** - 相当于 ELF 格式中的符号。使用从块的基地址（地址）的偏移量和字节大小来表示。
- **Section（段）** - 相当于 ELF 格式中的段。使用一组符号和块来表示。
- **Edge（边缘）** - 相当于 ELF 格式中的重定位。使用包含块起始处的偏移量（指示需要修复的存储位置）、需要用于修复的目标的指针以及指定修补公式的类型来表示。

##### **JITLinkContext**

​	JITLinkContext 代表您正在链接到的目标进程，并且它为 JIT 链接器提供了在进程中查询和采取行动的能力。这包括查找符号和分配内存的能力，以及将链接过程的结果发布到更广泛的环境中。具体来说，JITLinkContext 通知其他人它已经分配给符号的地址以及这些符号何时在内存中可用。

#### 理解JIT链接算法

​	LLVM JITLink 链接算法包括多个阶段，每个阶段都包括对 LinkGraph 的多次遍历和在结束时调用下一个阶段。在每个阶段中，算法根据需要修改 LinkGraph，并最终生成一个可在内存中执行的可重定位对象的映像。

​	最初对我来说不太明显，但一旦理解了，就大大简化了事情的一点是，LinkGraph 本质上就是一个图形！用这种简单的视角重新阅读 LLVM JITLink 的通用 JIT 链接算法的高层描述，使得理解 JIT 链接过程中发生的事情变得更加容易和直观。

​	此外，该算法还为 JITLink 的实现者和用户提供了一些钩子，可以介入链接过程。这些钩子可以用来实现许多事情，包括但不限于链接时优化、测试、验证等。

​	这个过程的多阶段性质意味着它可以逐步构建最终的代码表示，每一步都可能包括优化和调整，使得最终的内存中映像尽可能有效和优化。通过这种方式，**JITLink 不仅仅是执行代码的工具，它还提供了一个灵活的平台，支持高级编程技术，如动态代码生成和即时优化**。

#### 具体成果

​	首先，我设置了一个测试循环，以验证 LLVM JITLink 是否能够将包含有效 i386/ELF 重定位的 32 位 i386 ELF 对象链接到一个 32 位进程中。在构建 LLVM 项目时，默认会在 bin 文件夹中生成并放置现有的 llvm-jitlink 工具，这对我很有帮助。llvm-jitlink 是 JITLink 库的命令行包装器，它接受可重定位对象作为输入，并使用 JITLink 将它们链接到执行进程中。

​	对我来说，最棘手的部分是获得一个 32 位的 llvm-jitlink ELF 可执行文件。默认情况下，Clang 会为主机架构生成可执行文件，因此我需要了解交叉编译（为不同于主机架构的目标进行编译），因为我是在 x86-64 硬件上进行开发。为了在 x86-64 系统上获得一个 32 位的 llvm-jitlink ELF 可执行文件，我需要以下内容：

- **交叉编译器**：一个可以生成 32 位 x86 代码的交叉编译器。Clang 可以在构建配置中指定以下标志来生成 32 位 x86 代码：

    ```
    CMAKE_CXX_FLAGS="-m32" 或 CMAKE_C_FLAGS="-m32" - 指示 Clang 生成 32 位代码而不是默认的 64 位代码。
    LLVM_DEFAULT_TARGET_TRIPLE=X86 - 指示 Clang 默认生成 x86 目标的机器代码。
    ```

- **目标共享库**：32 位 x86 共享库，可能在编译期间进行检查。在我的情况下，安装 libstdc++.i686 和 glibc-devel.i686 就足够了，因为这是我生成包含所有可能的 i386/ELF 重定位的程序所需要的。

生成构建配置的完整命令如下：

```bash
cmake -DCMAKE_CXX_FLAGS="-m32" -DCMAKE_C_FLAGS="-m32" \
-DCMAKE_CXX_COMPILER=<PATH_PREFIX>/bin/clang++ \
-DCMAKE_BUILD_TYPE=Debug \
-DLLVM_TABLEGEN=<LLVM_BUILD_DIR_FOR_HOST_ARCH>/bin/llvm-tblgen \
-DLLVM_DEFAULT_TARGET_TRIPLE=i386-unknown-linux-gnu \
-DLLVM_TARGETS_TO_BUILD=X86 \
-G "Ninja" ../llvm
```

#### 测试循环

测试循环的最后一步是为 LLVM JITLink 设置基础结构，我可以在此基础上开始添加 i386/ELF 重定位。我将这部分基础结构作为我对 LLVM JITLink 的首次提交的一部分。总体而言，我在该提交中实现了两件事：

1. **ELFLinkGraphBuilder_i386**：包含从对象文件解析 i386/ELF 重定位的专门逻辑。
2. **ELFJITLinker_i386**：包含将 i386/ELF 重定位修正到内存中应发出的可执行映像的专门逻辑。

设置测试循环后，我逐步向 LLVM JITLink 添加了对以下 i386/ELF 重定位的支持。

在讨论各个重定位之前，先回顾一下什么是重定位。

#### 重定位的概念

​	编译器生成包含符号引用（除函数本地变量和函数自身外的所有内容）的代码。编译器只是通过程序员使用的名称引用符号，并为链接器在链接过程中完成一组待办事项。

​	在 ELF 对象中，这些待办事项可以在重定位部分找到。它们告诉链接器在解析重定位目标符号的地址后，需要在哪里以及如何修复符号引用。链接器可以解析重定位，因为它可以查看整个编译程序。

#### 各种重定位类型

#### R_386_32

- **描述**：告诉链接器用符号的绝对内存地址替换符号引用。
- **何时使用**：在非位置独立代码（PIC）中引用全局和静态变量。PIC 允许代码加载到内存中的任何地址，而不是固定地址。

```c
// 编译命令 => clang -m32 -c -o obj.o obj.c

// 声明全局变量 x
int x;

int main() {
    // 编译器应在此处生成 R_386_32 重定位。
    x += 1;
    return 0;
}
```

反汇编代码示例：

```assembly
00000000 <main>:
0:   55                      push   %ebp
1:   89 e5                   mov    %esp,%ebp
3:   50                      push   %eax
4:   c7 45 fc 00 00 00 00    movl   $0x0,-0x4(%ebp)

// 编译器想要将 x 的值移入 eax 寄存器，但不知道 x 的地址。
// 因此它为链接器留下一个待办事项，并暂时使用 0 作为 x 的地址。
b:   a1 00 00 00 00          mov    0x0,%eax
                         c: R_386_32     x

10:   83 c0 01                add    $0x1,%eax

// 同样的情况
13:   a3 00 00 00 00          mov    %eax,0x0
                         14: R_386_32    x

18:   31 c0                   xor    %eax,%eax
1a:   83 c4 04                add    $0x4,%esp
1d:   5d                      pop    %ebp
1e:   c3                      ret
```

#### R_386_PC32

- **描述**：告诉链接器使用符号相对于当前程序计数器（PC）的相对偏移量解析符号引用。链接器找到被引用符号相对于 PC 的偏移量，并将其硬编码到相应的汇编指令中。运行时，处理器查看调用指令的编码，并知道指令的操作数表示符号相对于 PC 的偏移量。
- **何时使用**：在 PIC 中调用函数。

```c
// 编译命令 => clang -m32 -ffunction-sections -c -o obj.o obj.c

// 声明全局函数 x
void x {}

int main() {
    // 编译器应在此处生成 R_386_PC32 重定位。
    x();
    return 0;
}
```

反汇编代码示例：

```assembly
00000000 <x>:
0:   55                      push   %ebp
1:   89 e5                   mov    %esp,%ebp
3:   5d                      pop    %ebp
4:   c3                      ret

00000000 <main>:
0:   55                      push   %ebp
1:   89 e5                   mov    %esp,%ebp
3:   83 ec 08                sub    $0x8,%esp
6:   c7 45 fc 00 00 00 00    movl   $0x0,-0x4(%ebp)

// 编译器想要调用函数 x，但不知道其地址。
// 因此它为链接器留下一个待办事项，并暂时使用垃圾字节作为 x 的地址。
d:   e8 fc ff ff ff          call   e <main+0xe>
                         e: R_386_PC32   x

12:   31 c0                   xor    %eax,%eax
14:   83 c4 08                add    $0x8,%esp
17:   5d                      pop    %ebp
18:   c3                      ret
```

#### 动态链接和剩余重定位类型

​	接下来，我们简要介绍动态链接，因为剩余的重定位类型是实现动态链接所需的。

​	在静态链接中，如果您的程序访问了给定库中的单个符号，那么整个库将与您的程序链接，这会导致生成的可执行文件体积增大。而在动态链接中，引用的库在构建时访问，但不会被带入已链接的可执行文件中。相反，这些库中的引用全局变量在加载时（程序加载到内存中运行时）链接，引用函数在调用时链接。

​	动态链接的实现比静态链接更复杂。静态链接下，用户只需要可执行文件，不会遇到缺少库的问题。动态链接下，如果共享库更新，则不需要更新可执行文件，这在分发可执行文件时非常有用。

​	了解 GOT（全局偏移量表）和 PLT（过程链接表）的概念也很重要。

#### R_386_GOTPC

- **描述**：告诉链接器用存储位置与 GLOBAL_OFFSET_TABLE（GOT）符号地址之间的增量替换符号引用。
- **何时使用**：作为 R_386_GOTOFF、R_386_GOT32 和 R_386_PLT32 的使能器，需要使用 GOT 的内存地址。

```c
// 编译命令 => clang -m32 -fPIC -c -o obj.o obj.c

// 声明一个全局静态变量
static int a = 

42;

int main() {
    // 使用 PIC 标志，指示生成位置无关代码。
    return a;
}
```

反汇编代码示例：

```assembly
00000000 <main>:
0:   55                      push   %ebp
1:   89 e5                   mov    %esp,%ebp
3:   50                      push   %eax
4:   e8 00 00 00 00          call   9 <main+0x9>
9:   58                      pop    %ebx
a:   81 c0 03 00 00 00       add    $0x3,%ebx
                         c: R_386_GOTPC  _GLOBAL_OFFSET_TABLE_
```

#### R_386_GOTOFF

- **描述**：告诉链接器用符号地址与 GOT 基地址之间的偏移量解析符号引用。
- **何时使用**：共享库和可执行文件以位置无关方式访问内部符号。

```c
// 编译命令 => clang -m32 -fPIC -c -o obj.o obj.c

// 声明一个全局静态变量
static int a = 42;

int main() {
   return a;
}
```

反汇编代码示例：

```assembly
00000000 <main>:
0:   55                      push   %ebp
1:   89 e5                   mov    %esp,%ebp
3:   50                      push   %eax
4:   e8 00 00 00 00          call   9 <main+0x9>
9:   58                      pop    %eax
a:   81 c0 03 00 00 00       add    $0x3,%ebx
                         c: R_386_GOTPC  _GLOBAL_OFFSET_TABLE_
10:   c7 45 fc 00 00 00 00    movl   $0x0,-0x4(%ebp)
17:   8b 80 00 00 00 00       mov    0x0(%ebx),%eax
                         19: R_386_GOTOFF        a
1d:   83 c4 04                add    $0x4,%esp
20:   5d                      pop    %ebp
21:   c3                      ret
```

#### R_386_GOT32

- **描述**：告诉链接器用 GOT 基地址与符号在 GOT 中的条目之间的偏移量解析符号引用（本质上计算 GOT 的索引）。
- **何时使用**：共享库和可执行文件以位置无关方式访问外部数据符号。

```c
// 编译命令 => clang -m32 -fPIC -c -o obj.o obj.c

// 声明一个外部定义的变量
extern int a;

int main() {
    return a;
}
```

反汇编代码示例：

```assembly
00000000 <main>:
0:   55                      push   %ebp
1:   89 e5                   mov    %esp,%ebp
3:   50                      push   %eax
4:   e8 00 00 00 00          call   9 <main+0x9>
9:   59                      pop    %ecx
a:   81 c1 03 00 00 00       add    $0x3,%ebx
                         c: R_386_GOTPC  _GLOBAL_OFFSET_TABLE_
10:   c7 45 fc 00 00 00 00    movl   $0x0,-0x4(%ebp)
17:   8b 81 00 00 00 00       mov    0x0(%ebx),%eax
                         19: R_386_GOT32        a
1d:   8b 00                   mov    (%eax),%eax
1f:   83 c4 04                add    $0x4,%esp
22:   5d                      pop    %ebp
23:   c3                      ret
```

#### R_386_PLT32

- **描述**：告诉链接器用符号的 PLT 条目解析符号引用。
- **何时使用**：共享库和可执行文件以位置无关方式访问外部函数符号。

```c
// 编译命令 => clang -m32 -fPIC -c -o obj.o obj.c

// 声明一个外部定义的函数
extern int foo(void);

int main(void) {
    return foo();
}
```

反汇编代码示例：

```assembly
00000000 <main>:
0:   55                      push   %ebp
1:   89 e5                   mov    %esp,%ebp
3:   53                      push   %ebx
4:   50                      push   %eax
5:   e8 00 00 00 00          call   a <main+0xa>
a:   5b                      pop    %ebx
b:   81 c3 03 00 00 00       add    $0x3,%ebx
                         d: R_386_GOTPC  _GLOBAL_OFFSET_TABLE_
11:   c7 45 f8 00 00 00 00    movl   $0x0,-0x8(%ebp)
18:   e8 fc ff ff ff          call   19 <main+0x19>
                         19: R_386_PLT32    foo
1d:   83 c4 04                add    $0x4,%esp
20:   5b                      pop    %ebx
21:   5d                      pop    %ebp
22:   c3                      ret
```

以上是一些关键的重定位类型和它们在 i386/ELF 上的使用示例。这些重定位类型使得动态链接成为可能，并提高了程序的灵活性和可维护性。

### 测试

​	虽然我之前谈到过设置“测试循环”，但在这里我想简要讨论一下回归测试的主题——不是“为什么”和“是什么”，而是“怎么做”。LLVM 项目中已经有一些优秀的测试工具，但我发现相关的文档有些滞后。具体来说，我想重点介绍一些在为 LLVM JITLink 的目标对象后端编写回归测试时可能会用到的工具。

​	在继续之前，我想提一下这个 LLVM 的高级测试指南。该指南应该能让你了解在何处/如何创建测试文件，如何让你的测试文件被测试运行器（LLVM 集成测试工具 - lit）发现，以及如何使用测试运行器运行测试。

​	话虽如此，让我们通过下面的示例测试文件，讨论一些你可能会用到的工具，以编写一个针对 LLVM JITLink 目标对象后端的回归测试。

```assembly
// 回归测试文件是汇编文件（“.s”扩展名）。

// 文件必须以所谓的“RUN”行开头。
// 每个“RUN”行告诉 lit 如何运行测试文件。
// RUN 行看起来和感觉就像是在运行 shell 命令。

// 每个回归测试可能会以以下两个 RUN 行开头，
// 虽然具体的 RUN 命令可能需要根据测试用例的需要进行修改。

# RUN: llvm-mc -triple=i386-unknown-linux-gnu -position-independent -filetype=obj -o %t.o %s

// 注意 llvm-jitlink 是如何使用“-noexec”选项运行的。
// 该选项告诉 llvm-jitlink 不要运行加载到内存中的代码。
// 这是重要的，因为 JITLink 可能正在为与运行回归测试的 LLVM 构建/发布流水线不同的架构链接和加载代码。
# RUN: llvm-jitlink -noexec %t.o

// llvm-jitlink 还要求每个文件都有一个“main”函数。
// 你的测试代码可以放在这里，但不是必须的。
.text
.globl  main
.p2align        4, 0x90
.type   main,@function
main:
    ret
.size   main, .-main
```

​	我们在这些目标对象后端回归测试中主要想确定的是，加载到内存中的代码中的重定位是否被正确修正。也就是说，我们必须确切地检查某些内存位置的某些字节是否如预期那样。让我们来看一些更复杂的测试用例，这些用例将展示我们可能需要执行的不同类型的检查以及如何执行这些检查。

```assembly
// llvm-jitlink 允许你指定 jitlink-check 表达式。
// jitlink-check 表达式是对工作内存的检查。

// jitlink-check 表达式可以与 `decode_operand` 函数一起使用。
// `decode_operand` 解码给定标签处的指令，然后访问你指定的操作数编号。
//
// 在下面的表达式中，decode_operand 解码标签 `foo` 处的操作数，访问其第 0 个操作数 `external_data` 并检查其值是否等于 `0xDEADBEEF` 所代表的字节。
//
// 注意 - 操作数编号并不总是与你所见的有一一对应的关系，虽然在这种情况下 `external_data` 确实是指令的第 0 个操作数，但对于另一条指令，其操作数编号可能会有所不同。
# jitlink-check: decode_operand(foo, 0) = 0xDEADBEEF
     .globl foo 
    .p2align 4, 0x90
    .type foo,@function
foo:
    movl external_data, %eax
.size foo, .-foo


// jitlink-check 表达式的右侧不一定必须是字节字面值。它可以是标签和其他函数的表达式。
//
// 在下面的 jitlink-check 表达式中，右侧正在计算标签 `foo` 的地址与标签 `bar` 的指令执行时程序计数器的地址之间的差异。
# jitlink-check: decode_operand(bar, 0) = foo - next_pc(bar)
     .globl bar
    .p2align 4
    .type bar,@function
bar:
    calll foo
.size bar, .-bar

// 也可以在右侧使用 `got_addr` 函数来访问符号的 GOT 条目的地址。
//
// 在下面的 jitlink-check 表达式中，右侧正在计算符号 `named_data` 的 GOT 条目地址与 GOT 符号本身之间的偏移量。
# jitlink-check: decode_operand(test_got, 4) = got_addr(test_file_name.o, named_data) - _GLOBAL_OFFSET_TABLE_
     .globl test_got
     .p2align 4, 0x90
     .type test_got,@function
test_got:
    leal named_data@GOT, %eax
.size test_got, .-test_got

// jitlink-check 表达式的左侧也可以通过“转换”符号、标签或函数为机器寄存器大小的指针来手动构造。
//
// 在下面的 jitlink-check 表达式中，左侧通过将 `named_data` 的 GOT 条目地址转换为 32 位指针来构造。然后解引用构造的指针并与 `named_data` 标签进行比较。
# jitlink-check: *{4}(got_addr(test_file_name.o, named_data)) = named_data
    .globl test_got
    .p2align 4, 0x90
    .type test_got,@function
test_got:
    leal named_data@GOT, %eax
.size test_got, .-test_got
```

​	注意 - 上述呈现的 jitlink-check 表达式的种类并不是可用的详尽列表，而只是对我使用 jitlink-check 表达式的一些方式的总结。

### 资源

1. [Chris Kanich’s videos](https://www.youtube.com/playlist?list=PLhy9gU5W1fvUND_5mdpbNVHC1WCIaABbP) for the systems programming course at University of Illinois, Chicago
2. Lang Hames' videos ([1](https://www.youtube.com/watch?v=hILdR8XRvdQ&t=2577s), [2](https://www.youtube.com/watch?v=MOQG5vkh9J8), and [3](https://www.youtube.com/watch?v=i-inxFudrgI&t=2243s)) on LLVM ORC APIs and JITLink. These videos were extremely valuable in understanding JITLink’s raison d’être and the context in which it is used.
3. Linkers and Loaders by John R. Levine
4. [Oracle’s Linker and Libraries guide](https://docs.oracle.com/cd/E23824_01/html/819-0690/toc.html)
5. [LLVM JITLink documentation](https://llvm.org/docs/JITLink.html)
6. [LLVM testing infrastructure guide](https://llvm.org/docs/TestingGuide.html)
7. Articles by Eli Bendersky on [position-independent code](https://eli.thegreenplace.net/2011/11/03/position-independent-code-pic-in-shared-libraries/#id14) and [load time relocation of shared libraries](https://eli.thegreenplace.net/2011/08/25/load-time-relocation-of-shared-libraries).

### 注释

1. **AOT，静态编译语言**

   AOT（Ahead-Of-Time，提前编译）语言，例如C、C++、Rust，与解释型语言（如Java）不同，没有一个可以在运行时引入新符号并为其执行符号解析的运行时。比如，Java有Java虚拟机（JVM），其加载和链接行为可以定制以实现上述任务。

2. **JIT链接**

   JIT链接主要在预编译语言的链接中有用（这确实是它的灵感来源），但不仅限于此。在LLVM JITLink中，通过JITLinkContext，你可以与其他（非静态编译的）代码链接，因此对于任何想在运行时与C/C++互操作的人来说都是有用的。理论上，你也可以用它启动一个纯粹的JIT语言（我认为Julia就是这么做的）。优点是可以与现有语言、编译器、工具互操作，缺点是相比直接管理自身链接的自定义JIT，它比较笨重。

3. **Clang-REPL**

   Clang-REPL是将Cling（一个独立工具）移入LLVM基础设施中的努力。

4. **JITLinkContext**

   事实上，给定合适的JITLinkContext，JITLink甚至可以将对象链接到不同的进程中。LLDB利用这一功能（通过LLVM较早的MCJIT API）在调试器中JIT链接表达式，但在被调试的进程中运行它们，这些进程可能在不同的机器上。↩︎

### 什么是GOT和PLT？

​	GLOBAL_OFFSET_TABLE (GOT) 和 PROCEDURE_LINKAGE_TABLE (PLT)，GOT 和 PLT 是链接过程中使用的两个表，使动态链接得以实现。动态链接代码需要是位置无关的 ，这意味着它应该能够加载到内存中的任何地址并继续工作——它引用的所有符号以及它包含的所有引用符号都必须是可解析的。**动态链接共享库可以使用PC相对寻址来满足内部符号的这一位置无关性要求**，因为共享库的代码在内存中是连在一起的。然而，这些库也可能引用外部符号或包含被其他共享库或主可执行文件引用的符号。由于共享库在内存中的加载地址不是固定的，它们需要另一层抽象来解析外部符号，无论是数据还是函数。GOT 和 PLT 就是这些抽象。

#### GLOBAL_OFFSET_TABLE (GOT)
- **作用**：GOT 用于存储全局变量和函数的地址，支持位置无关代码 (PIC)。
- **工作原理**：当一个程序访问一个外部变量时，GOT提供了该变量在内存中的实际地址。编译时，访问外部变量的指令会引用GOT中的一个条目，该条目在加载时由动态链接器填写。
  
#### PROCEDURE_LINKAGE_TABLE (PLT)
- **作用**：PLT 用于存储函数入口点的地址，支持动态函数调用。
- **工作原理**：当一个程序调用一个外部函数时，PLT提供了该函数的实际入口地址。初次调用外部函数时，PLT中的条目指向一个跳转指令，跳转到动态链接器的解析代码。解析器找到实际的函数地址并更新PLT条目，使后续调用直接跳转到目标函数。

#### 动态链接的过程
1. **程序加载时**：动态链接器加载共享库，并填充GOT和PLT中的初始条目。
2. **符号解析**：首次访问外部符号时，通过GOT或PLT触发动态链接器进行符号解析，将实际地址填充到相应的表项中。
3. **后续访问**：后续访问直接通过已填充的GOT或PLT条目，快速跳转到目标地址，无需再次解析。

这种机制使得共享库可以在不同的内存地址加载，同时确保代码能够正确引用和调用外部符号，从而实现位置无关性和动态链接的灵活性。

