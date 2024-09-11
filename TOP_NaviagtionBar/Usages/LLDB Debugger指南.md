# LLDB

> 2024年7月15日版本

设计哲学：比GDB更加结构化, 所有指令都以此形式：`<noun> <verb> [-options [option-value]] [argument [argument...]]`

- 参数、选项和选项值均以空格分隔。
- 单引号`'`或双引号`"`（成对）用于保护参数中的空格。
- 转义反斜杠和参数内的双引号应该用反斜杠进行转义`\`。
- 反引号\`将通过表达式解析器运行该值的文本。例如：

```bash
(lldb) memory read -c `len` 0x12345
```

## 基础指令

### 断点

```bash
(lldb) breakpoint set --file foo.c --line 12
(lldb) breakpoint set -f foo.c -l 12
(lldb) breakpoint set --name foo		# 设置函数断点
(lldb) breakpoint set -n foo
(lldb) breakpoint set --name foo --name bar	# 设置一组函数上的断点
(lldb) breakpoint set --method foo	
(lldb) breakpoint set -M foo
(lldb) br s -n "-[SKTGraphicView alignLeftEdges:]"	# 支持缩写
(lldb) breakpoint list	# 显示所有断点
(lldb) breakpoint command add 1.1	# 达到断点时自动执行
Enter your debugger command(s). Type 'DONE' to end.
> bt
> DONE
(lldb) breakpoint modify -c "self == nil" -C bt --auto-continue 1 2 3	# 条件执行断点，当且仅当self=nil时
```

> 别名：用户可以随意自定义 LLDB 的命令集，==而且由于 LLDB 在启动时会读取文件`~/.lldbinit`，因此您可以将所有别名存储在那里，它们通常可供您使用。==您的别名也会记录在命令中，`help`因此您可以提醒自己已设置的内容。
>
> ```bash
> (lldb) command alias bfl breakpoint set -f %1 -l %2
> (lldb) bfl foo.c 12
> (lldb) command unalias b
> (lldb) command alias b breakpoint
> ```

### 监视点

```bash
(lldb) watch set var global
Watchpoint created: Watchpoint 1: addr = 0x100001018 size = 4 state = enabled type = w
   declare @ '/Volumes/data/lldb/svn/ToT/test/functionalities/watchpoint/watchpoint_commands/condition/main.cpp:12'
(lldb) watch modify -c '(global==5)'
(lldb) watch list
Current watchpoints:
Watchpoint 1: addr = 0x100001018 size = 4 state = enabled type = w
   declare @ '/Volumes/data/lldb/svn/ToT/test/functionalities/watchpoint/watchpoint_commands/condition/main.cpp:12'
   condition = '(global==5)'
(lldb) c
Process 15562 resuming
(lldb) about to write to 'global'...
Process 15562 stopped and was programmatically restarted.
Process 15562 stopped and was programmatically restarted.
Process 15562 stopped and was programmatically restarted.
Process 15562 stopped and was programmatically restarted.
Process 15562 stopped
* thread #1: tid = 0x1c03, 0x0000000100000ef5 a.out`modify + 21 at main.cpp:16, stop reason = watchpoint 1
   frame #0: 0x0000000100000ef5 a.out`modify + 21 at main.cpp:16
   13
   14        static void modify(int32_t &var) {
   15            ++var;
-> 16        }
   17
   18        int main(int argc, char** argv) {
   19            int local = 0;
(lldb) bt
* thread #1: tid = 0x1c03, 0x0000000100000ef5 a.out`modify + 21 at main.cpp:16, stop reason = watchpoint 1
   frame #0: 0x0000000100000ef5 a.out`modify + 21 at main.cpp:16
   frame #1: 0x0000000100000eac a.out`main + 108 at main.cpp:25
   frame #2: 0x00007fff8ac9c7e1 libdyld.dylib`start + 1
(lldb) frame var global
(int32_t) global = 5
(lldb) watch list -v
Current watchpoints:
Watchpoint 1: addr = 0x100001018 size = 4 state = enabled type = w
   declare @ '/Volumes/data/lldb/svn/ToT/test/functionalities/watchpoint/watchpoint_commands/condition/main.cpp:12'
   condition = '(global==5)'
   hit_count = 5     ignore_count = 0
(lldb)
```

### 进程process

```bash
(lldb) process launch
(lldb) run
(lldb) r
(lldb) process attach --pid 123	# 启动或附加到某个进程(将调试器（例如LLDB）连接到已经在运行的进程)
(lldb) process attach --name Sketch	# 通过进程名称附加
(lldb) process attach --name Sketch --waitfor
```

### 程序控制thread

```bash
(lldb) thread continue	# 即c
Resuming thread 0x2c03 in process 46915
Resuming process 46915
(lldb) thread step-in    # 相当于GDB的 "step" 或 "s"
(lldb) thread step-over  # 相当于GDB的 "next" 或 "n"
(lldb) thread step-out   # 相当于GDB的 "finish" 或 "f"
(lldb) thread step-inst       # 相当于GDB的 "stepi" / "si"
(lldb) thread step-over-inst  # 相当于GDB的 "nexti" / "ni"
(lldb) thread until 100	# 相当于GDB的"until"
```

### 堆栈和变量(GDB的bt和线程切换)

```bash
# 当程序停止后，LLDB会选择一个当前线程，通常是因为某个原因停止的那个线程，以及该线程中的一个当前帧（在停止时总是最底层的帧）。许多检查状态的命令都在当前线程/帧上操作。
(lldb) thread list	# 要检查进程的当前状态，可以从线程开始;星号（*）表示线程1是当前线程。要获取该线程的回溯信息，可以使用
Process 46915 state is Stopped
* thread #1: tid = 0x2c03, 0x00007fff85cac76a, where = libSystem.B.dylib`__getdirentries64 + 10, stop reason = signal = SIGSTOP, queue = com.apple.main-thread
  thread #2: tid = 0x2e03, 0x00007fff85cbb08a, where = libSystem.B.dylib`kevent + 10, queue = com.apple.libdispatch-manager
  thread #3: tid = 0x2f03, 0x00007fff85cbbeaa, where = libSystem.B.dylib`__workq_kernreturn + 10
(lldb) thread backtrace	# 堆栈信息 可以使用"bt"
thread #1: tid = 0x2c03, stop reason = breakpoint 1.1, queue = com.apple.main-thread
  frame #0: 0x0000000100010d5b, where = Sketch`-[SKTGraphicView alignLeftEdges:] + 33 at /Projects/Sketch/SKTGraphicView.m:1405
  frame #1: 0x00007fff8602d152, where = AppKit`-[NSApplication sendAction:to:from:] + 95
  frame #2: 0x00007fff860516be, where = AppKit`-[NSMenuItem _corePerformAction] + 365
  frame #3: 0x00007fff86051428, where = AppKit`-[NSCarbonMenuImpl performActionWithHighlightingForItemAtIndex:] + 121
  frame #4: 0x00007fff860370c1, where = AppKit`-[NSMenu performKeyEquivalent:] + 272
  frame #5: 0x00007fff86035e69, where = AppKit`-[NSApplication _handleKeyEquivalent:] + 559
  frame #6: 0x00007fff85f06aa1, where = AppKit`-[NSApplication sendEvent:] + 3630
  frame #7: 0x00007fff85e9d922, where = AppKit`-[NSApplication run] + 474
  frame #8: 0x00007fff85e965f8, where = AppKit`NSApplicationMain + 364
  frame #9: 0x0000000100015ae3, where = Sketch`main + 33 at /Projects/Sketch/SKTMain.m:11
  frame #10: 0x0000000100000f20, where = Sketch`start + 52
(lldb) thread backtrace all	# 查看所有线程的回溯

(lldb) thread select 2	# 选择当前线程
######### 检查堆栈帧状态

(lldb) frame variable	# 打印所有变量
self = (SKTGraphicView *) 0x0000000100208b40
_cmd = (struct objc_selector *) 0x000000010001bae1
sender = (id) 0x00000001001264e0
selection = (NSArray *) 0x00000001001264e0
i = (NSUInteger) 0x00000001001264e0
c = (NSUInteger) 0x00000001001253b0

(lldb) frame variable self	# 打印指定变量
(SKTGraphicView *) self = 0x0000000100208b40

(lldb) frame variable *self	# 支持&、*、->、[]等
(SKTGraphicView *) self = 0x0000000100208b40
(NSView) NSView = {
  (NSResponder) NSResponder = {
    ...

(lldb) frame variable &self	
(SKTGraphicView **) &self = 0x0000000100304ab

(lldb) frame variable argv[0]
(char const *) argv[0] = 0x00007fff5fbffaf8 "/Projects/Sketch/build/Debug/Sketch.app/Contents/MacOS/Sketch"


```

### 寄存器和内存(GDB的i r, x/wx等)

```bash
(lldb) memory read 0x1000 0x1010	# 读取地址0x1000开始的16个字节
(lldb) memory read 0x1000 --size 16	#	同上
(lldb) memory write 0x1000 0x42	# 向地址0x1000写入值0x42
(lldb) memory find 0x1000 --size 256 --pattern 0x42	# 在地址0x1000开始的256字节内查找模式0x42
```

```bash
(lldb) register read	# 显示所有寄存器
(lldb) register read x0	# 读取特定寄存器
(lldb) register write x0 0x42	# 将寄存器x0的值设置为0x42
(lldb) register info x0	# 显示寄存器x0的详细信息
```

### 指令级别调试(GDB的x/xi)

```bash
(lldb) disassemble --name main	# 反汇编main
(lldb) disassemble --frame	# 反汇编当前所在函数
(lldb) disassemble --start-address <start_address> --end-address <end_address>	# 反汇编特定地址
(lldb) disassemble --start-address 0x1000 --count 16	# 反汇编从地址0x1000开始的16个字节
(lldb) memory read --type instruction --count <num_instructions> <start_address>	# 直接将内存内容显示为指令
(lldb) memory read --type instruction --count 4 0x1000	# 从地址0x1000开始读取4条指令
```

### 代码窗口(GDB的tui,list)

```bash
(lldb) gui
(lldb) source list -n <function_name>	# 查看源代码
```

### 脚本

lldb支持python，使用`script`即可访问

## GDB,LLDB对照表

| GDB                                                          | LLDB                                                         | 解释                                                         |
| :----------------------------------------------------------- | :----------------------------------------------------------- | :----------------------------------------------------------- |
| r                                                            | r                                                            | 运行                                                         |
| r <args>                                                     | r <args>                                                     | 带参数运行                                                   |
| gdb --args a.out 1 2 3                                       | ==lldb -- a.out 1 2 3==                                      | 命令行                                                       |
| set args 1 2 3                                               | settings set target.run-args 1 2 3                           | 设置参数                                                     |
| set env DEBUG 1                                              | env DEBUG=1                                                  | 前设置进程的环境变量                                         |
| show args                                                    | settings show target.run-args                                | 显示参数                                                     |
| attach 123                                                   | attach -p 123                                                | 附加到进程 ID 为 123 的进程【一般为服务器】                  |
| attach a.out                                                 | pro at -n a.out                                              | 附加到名为a.out的进程                                        |
| attach -waitfor a.out                                        | pro at -n a.out -w                                           | 等待名为`a.out`启动并附加的进程                              |
| target remote eorgadd:8000                                   | gdb -remote eorgadd:8000                                     | `eorgadd`连接到在系统、端口[#](https://lldb.llvm.org/use/map.html#attach-to-a-remote-gdb-protocol-server-running-on-system-eorgadd-port-8000)上运行的远程 gdb 协议服务器`8000` |
| target remote localhost:8000                                 | gdb -remote 8000                                             | 连接到本地系统上运行的远程 gdb 协议服务器，端口`8000`        |
| s                                                            | ==s==                                                        | ==单步进入==                                                 |
| n                                                            | ==n==                                                        | ==单步不进入==                                               |
| si                                                           | ==si==                                                       | ==指令级单步进入==                                           |
| ni                                                           | ==ni==                                                       | ==指令级单步不进入==                                         |
| finish                                                       | ==finish==                                                   | ==退出当前函数==                                             |
| return <RETURN EXPRESSION>                                   | thread return <RETURN EXPRESSION>                            | 立即从当前选定的帧返回，并带有可选的返回值                   |
| until 12                                                     | thread ==until 12==                                          | 运行直到到达第 12 行或控制权离开当前函数                     |
| frame                                                        | f                                                            | 显示当前框架和源代码行                                       |
| break main                                                   | ==b main==                                                   | ==设置所有名为main的断点==                                   |
| break test.c:12                                              | b test.c:12                                                  | 设置行号断点                                                 |
| 无法实现                                                     | breakpoint set --method main                                 | 在所有名称为main的C++代码上设置断点                          |
| rbreak regular-expression                                    | br s -r regular-expression                                   | 用正则表达式设置断点                                         |
| break foo if strcmp(y,"hello") == 0                          | ==br s -n foo -c '(int)strcmp(y,"hello") == 0'==             | 条件断点                                                     |
| info break                                                   | ==br l==                                                     | ==列出所有断点==                                             |
| del 1                                                        | br del 1                                                     | 删除断点                                                     |
| disable 1                                                    | br dis 1                                                     | 禁用断点                                                     |
| enable 1                                                     | br en 1                                                      | 启用断点                                                     |
| watch global_var                                             | ==wa s v global_var==                                        | ==当变量被写入时设置一个观察点==                             |
| watch -location g_char_ptr                                   | wa s e -- my_ptr                                             | 当内存位置被写入时设置一个观察点                             |
| 无法实现                                                     | watchpoint modify -c '(global==5)'                           | 设置观察点的条件                                             |
| info break                                                   | watch l                                                      | 列出所有断点                                                 |
| info args ,  info locals                                     | ==fr v==                                                     | ==显示当前框架的参数和局部变量==                             |
| info locals                                                  | fr v -a                                                      | 显示当前帧的局部变量                                         |
| p bar                                                        | p bar                                                        | 显示局部变量的内容`bar`                                      |
| p/x bar                                                      | fr v -f x bar                                                | 以十六进制格式显示的局部变量的内容                           |
| p baz                                                        | ta v baz                                                     | 显示全局变量的内容`baz`                                      |
| 无法实现                                                     | ta v                                                         | 显示当前源文件中定义的全局/静态变量                          |
| display argc                                                 | (lldb) target stop-hook add --one-liner "frame variable argc argv" <br />(lldb) ta st a -o "fr v argc argv" <br />(lldb) display argc <br /> | 每次停止时打印值                                             |
| p *ptr@10                                                    | parray 10 ptr                                                | 打印内存中的整数数组，假设我们有一个指针，如[#](https://lldb.llvm.org/use/map.html#print-an-array-of-integers-in-memory-assuming-we-have-a-pointer-like-int-ptr)`int *ptr` |
| print (int) printf ("Print nine: %d.", 4 + 5)                | expr (int) printf ("Print nine: %d.", 4 + 5)                 | 在当前框架中评估广义表达式                                   |
| set $foo = 5                                                 | expr unsigned int $foo = 5                                   | 创建一个变量                                                 |
| info threads                                                 | thread list                                                  | 列出程序中的线程                                             |
| thread 1                                                     | t 1                                                          | 选择线程`1`作为后续命令的默认线程                            |
| bt                                                           | bt                                                           | 显示当前线程的堆栈回溯                                       |
| bt 5                                                         | bt 5                                                         | 回溯当前线程的前五帧                                         |
| frame 12                                                     | f 12                                                         | 根据索引为当前线程选择不同的堆栈帧                           |
| 无法实现                                                     | frame info                                                   | 列出当前线程中当前选定框架的信息                             |
| up                                                           | up                                                           | 选择调用当前堆栈帧的堆栈帧                                   |
| down                                                         | down                                                         | 选择当前堆栈帧调用的堆栈帧                                   |
| up 2, down 3                                                 | fr s -r2, fr s -r-3                                          | 使用相对便宜量量选择不同的堆栈框架                           |
| info registers                                               | ==register read==                                            | ==显示当前线程的通用寄存器==                                 |
| p $rax = 123                                                 | register write rax 123                                       | 将新的十进制值写入`123`当前线程寄存器`rax`                   |
| jump *$pc+8                                                  | register write pc `$pc+8`                                    | 跳过当前程序计数器（指令指针）前面的 8 个字节                |
| 无法实现                                                     | register read/d                                              | 显示当前线程的通用寄存器，格式为有符号十进制数               |
| info all-registers                                           | re r -a                                                      | 显示当前线程的所有寄存器集中的所有寄存器                     |
| info all-registers rax rsp rbp                               | register read rax rsp rbp                                    | 显示当前线程中名为`rax`、`rsp`和的寄存器的值                 |
| p/t $rax                                                     | p/t $rax                                                     | 以二进制格式显示当前线程中rax寄存器的值                      |
| x/4xw 0xbffff3c0                                             | x/4xw 0xbffff3c0                                             | 从地址读取内存`0xbffff3c0`并显示 4 个十六进制`uint32_t`值    |
| x argv[0]                                                    | memory read `argv[0]`                                        | 从表达式`argv[0]`开始读取内存                                |
| (gdb) set logging on <br />(gdb) set logging file /tmp/mem.txt<br /> (gdb) x/512bx 0xbffff3c0<br /> (gdb) set logging off | (lldb) memory read --outfile /tmp/mem.txt --count 512 0xbffff3c0 <br />(lldb) me r -o/tmp/mem.txt -c512 0xbffff3c0 <br />(lldb) x/512bx -o/tmp/mem.txt 0xbffff3c0 | 从地址读取`512`内存字节`0xbffff3c0`并将结果以文本形式保存到本地文件 |
| dump memory /tmp/mem.bin 0x1000 0x2000                       | me r -o /tmp/mem.bin -b 0x1000 0x2000                        | 将开始0x1000于并结束于0x2000的二进制内存数据保存到文件中     |
| disassemble                                                  | di -                                                         | 反汇编当前函数                                               |
| disassemble main                                             | di -n main                                                   | 反汇编所有名为 main 的函数                                   |
| disassemble 0x1eb8 0x1ec3                                    | di -s 0x1eb8 -e 0x1ec3                                       | 反汇编一个地址                                               |
| x/20i 0x1eb8                                                 | di -s 0x1eb8 -c 20                                           | 从给定地址反汇编指令                                         |
| 无法实现                                                     | di -f -m                                                     | 显示当前帧的当前函数的混合源代码和反汇编代码                 |
| 无法实现                                                     | di -f -b                                                     | 反汇编当前帧的当前函数并显示操作码字节                       |
| 无法实现                                                     | di -l                                                        | 反汇编当前帧的当前源代码行                                   |
| info shared                                                  | image list                                                   | 列出主要的可执行文件和所有依赖的共享库                       |
| info symbol 0x1ec4                                           | im loo -a 0x1ec4                                             | 查找可执行文件或任何共享库中的原始地址的信息                 |
| info function <FUNC_REGEX>                                   | 找调试符号:image lookup -r -n <FUNC_REGEX><br />找非调试符号:image lookup -r -s <FUNC_REGEX> | 在二进制文件中中查找与正则表达式匹配的函数                   |
| info line 0x1ec4                                             | image lookup -v --address 0x1ec4                             | 查找完整的源代码行信息                                       |
| ptype Point                                                  | im loo -t Point                                              | 根据Point名称查找类型的信息                                  |
| maintenance info sections                                    | image dump sections                                          | 转储主可执行文件和任何共享库中的所有段                       |
| 无法实现                                                     | image dump sections a.out                                    | 转储a.out模块中的所有段                                      |
| 无法实现                                                     | image dump symtab                                            | 从主可执行文件和任何共享库中转储所有符号                     |
| 无法实现                                                     | image dump symtab a.out liba.so                              | 转储所有符号`a.out`和`liba.so`                               |
| apropos keyword                                              | apropos keyword                                              | 搜索关键字的命令帮助                                         |

## 栈帧和线程格式

![image-20240722200418017](/Users/yilingqinghan/Documents/BaiduPan/④笔记/④使用指南/assets/image-20240722200418017-1649864.png)

堆栈回溯框架有类似的信息行：

```
(lldb) thread backtrace
* thread #1, queue = 'com.apple.main-thread', stop reason = breakpoint 1.1
    frame #0: 0x0000000100000e85 a.out`main + 4 at test.c:19
    frame #1: 0x0000000100000e40 a.out`start + 52
```

可以用如下手段设置：

```
(lldb) settings set thread-stop-format STRING
(lldb) settings set frame-format STRING
(lldb) settings set thread-format STRING
```

支持打印的变量：

| **变量名**                                      | **描述**                                                     |
| :---------------------------------------------- | :----------------------------------------------------------- |
| `file.basename`                                 | 当前帧的当前编译单元文件基名。                               |
| `file.fullpath`                                 | 当前帧的当前编译单元文件完整路径。                           |
| `language`                                      | 当前框架的当前编译单元语言。                                 |
| `frame.index`                                   | 帧索引（0、1、2、3……）                                       |
| `frame.no-debug`                                | 如果框架没有调试信息，则计算结果为 true。                    |
| `frame.pc`                                      | 程序计数器的通用帧寄存器。                                   |
| `frame.sp`                                      | 堆栈指针的通用框架寄存器。                                   |
| `frame.fp`                                      | 帧指针的通用帧寄存器。                                       |
| `frame.flags`                                   | 标志寄存器的通用帧寄存器。                                   |
| `frame.reg.NAME`                                | 通过名称访问任何平台特定的寄存器（替换`NAME`为所需寄存器的名称）。 |
| `function.name`                                 | 当前函数或符号的名称。                                       |
| `function.name-with-args`                       | 带有参数和值的当前函数的名称或符号名称。                     |
| `function.name-without-args`                    | 当前函数的名称，不包含参数和值（用于在 中包含函数名称`disassembly-format`） |
| `function.mangled-name`                         | 当前函数或符号的混乱名称。                                   |
| `function.pc-offset`                            | 当前函数或符号内的程序计数器偏移量                           |
| `function.addr-offset`                          | 当前函数的偏移量（以字节为单位），格式为“ + dddd”            |
| `function.concrete-only-addr-offset-no-padding` | 类似于，`function.addr-offset`但输出中没有空格（例如“+dddd”），并且偏移量是根据最近的具体函数计算的 - 不包括内联函数 |
| `function.changed`                              | 当格式化的行与上一行的符号上下文不同时，将计算为 true（可用于`disassembly-format`在新函数的开头单独打印新函数名称）。此变量不考虑内联函数 |
| `function.initial-function`                     | 如果这是第一个函数的开始，而不是函数的改变，则计算结果为真（可用于`disassembly-format`打印被反汇编的第一个函数的函数名称） |
| `line.file.basename`                            | 当前帧中当前行条目的文件的行表条目基名。                     |
| `line.file.fullpath`                            | 当前帧中当前行条目的文件的行表条目完整路径。                 |
| `line.number`                                   | 当前帧中当前行条目的行表条目行号。                           |
| `line.start-addr`                               | 当前帧中当前行条目的行表条目起始地址。                       |
| `line.end-addr`                                 | 当前帧中当前行条目的行表条目结束地址。                       |
| `module.file.basename`                          | 当前模块（共享库或可执行文件）的基本名称                     |
| `module.file.fullpath`                          | 当前模块（共享库或可执行文件）的基本名称                     |
| `process.file.basename`                         | 该进程的文件的基本名称                                       |
| `process.file.fullpath`                         | 该进程的文件的全名                                           |
| `process.id`                                    | 下级运行系统的本机进程 ID。                                  |
| `process.name`                                  | 运行时进程的名称                                             |
| `thread.id`                                     | 当前线程的线程标识符                                         |
| `thread.index`                                  | 基于线程索引 ID 的唯一索引，保证线程来去时的唯一性。         |
| `thread.name`                                   | 如果目标操作系统支持命名线程，则为线程的名称                 |
| `thread.queue`                                  | 如果目标操作系统支持调度队列，则为线程的队列名称             |
| `thread.stop-reason`                            | 线程停止的文本原因。如果线程有可识别的框架，则显示其可识别的停止原因。否则，获取停止信息描述。 |
| `thread.stop-reason-raw`                        | 线程停止的文本原因。始终返回停止信息描述。                   |
| `thread.return-value`                           | 最新一步操作的返回值（目前仅限于步出操作）。                 |
| `thread.completed-expression`                   | 刚刚完成中断的表达式评估的线程的表达式结果。                 |
| `target.arch`                                   | 当前目标的架构                                               |
| `script.target:python_func`                     | 使用 Python 函数生成一段文本输出                             |
| `script.process:python_func`                    | 使用 Python 函数生成一段文本输出                             |
| `script.thread:python_func`                     | 使用 Python 函数生成一段文本输出                             |
| `script.frame:python_func`                      | 使用 Python 函数生成一段文本输出                             |
| `current-pc-arrow`                              | 如果当前 pc 值匹配，则打印`->`或`` ``（用于`disassembly-format`） |
| `addr-file-or-load`                             | 将地址格式化为加载地址，或者如果进程尚未启动，则格式化为加载地址（用于`disassembly-format`） |

**很多时候，提示中的信息可能不可用，如果信息无效，您不会希望将其打印出来。为了解决这个问题，您可以将必须解析的所有内容封装到范围中。范围以 { 开头并以 结尾 } 。例如，为了仅在当前帧的信息可用时显示当前帧行表条目基名和行号：**

```bash
"{ at {$line.file.basename}:${line.number}}"
```

> 举例：
>
> ```bash
> "frame #${frame.index}: ${frame.pc}{ ${module.file.basename}`${function.name}{${function.pc-offset}}}{ at ${line.file.basename}:${line.number}}\n"
> # 对应
> frame #0: 0x0000000100000e85 a.out`main + 4 at test.c:19
> 
> ```
>
> - 始终打印帧索引和帧 PC :，`frame #${frame.index}: ${frame.pc}`
> - 如果当前帧有有效模块，则仅打印模块并随后打勾：，`{ ${module.file.basename}`}`
> - 打印带有可选偏移量的函数名称：`{${function.name}{${function.pc-offset}}}`，
> - 打印行信息（如果可用）：，`{ at ${line.file.basename}:${line.number}}`
> - 然后以换行符结束：`\n`。

### DIY

修改自己的格式字符串时，最好从框架和线程格式字符串的默认值开始。可以使用以下命令访问这些值：settings show

```bash
(lldb) settings show thread-format
thread-format (format-string) = "thread #${thread.index}: tid = ${thread.id%tid}{, ${frame.pc}}{ ${module.file.basename}{`${function.name-with-args}{${frame.no-debug}${function.pc-offset}}}}{ at ${line.file.basename}:${line.number}}{, name = '${thread.name}'}{, queue = '${thread.queue}'}{, activity = '${thread.info.activity.name}'}{, ${thread.info.trace_messages} messages}{, stop reason = ${thread.stop-reason}}{\nReturn value: ${thread.return-value}}{\nCompleted expression: ${thread.completed-expression}}\n"
(lldb) settings show frame-format
frame-format (format-string) = "frame #${frame.index}:{ ${frame.no-debug}${frame.pc}}{ ${module.file.basename}{`${function.name-with-args}{${frame.no-debug}${function.pc-offset}}}}{ at ${line.file.basename}:${line.number}}{${function.is-optimized} [opt]}\n"
```

在制作线程格式时，您需要用范围 ({ frame-content }) 包围来自堆栈帧的任何信息，因为线程格式并不总是希望显示帧信息。在显示线程的回溯时，我们不需要在线程信息中重复帧零的信息：

```bash
(lldb) thread backtrace
thread #1: tid = 0x2e03, stop reason = breakpoint 1.1 2.1
  frame #0: 0x0000000100000e85 a.out`main + 4 at test.c:19
  frame #1: 0x0000000100000e40 a.out`start + 52
```

与框架相关的变量有：

- `${file.*}`
- `${frame.*}`
- `${function.*}`
- `${line.*}`
- `${module.*}`

查看线程的默认格式，并强调框架信息：

```bash
thread #${thread.index}: tid = ${thread.id}{, ${frame.pc}}{ ${module.file.basename}`${function.name}{${function.pc-offset}}}{, stop reason = ${thread.stop-reason}}{, name = ${thread.name}}{, queue = ${thread.queue}}\n
```

我们可以看到，所有的帧信息都包含在范围中，这样当我们只想显示线程信息的上下文中显示线程信息时，我们就可以这样做。

​	对于线程和帧格式，您都可以使用\${script.target:python_func}、\${script.process:python_func} 和 \${script.thread:python_func}（当然，对于帧格式，还可以使用 ${script.frame:python_func}）在所有情况下，python_func 的签名预计为：

```
def python_func(object,unused):
  ...
  return string
```

其中对象是与您正在使用的关键字相关联的 SB 类的一个实例。

例如假设你的函数如下所示：

```bash
def thread_printer_func (thread,unused):
  return "Thread %s has %d frames\n" % (thread.name, thread.num_frames)
```

你可以这样设置：

```bash
(lldb) settings set thread-format "${script.thread:thread_printer_func}"
```

你会看到如下输出：

```
* Thread main has 21 frames
```

### 变量格式：重点

LLDB 有一个数据格式化程序子系统，允许用户为其变量定义自定义显示选项。通常，当您输入或运行某些表达式时，LLDB 会自动选择根据类型显示结果的方式，如下例所示：`frame variable`

```bash
(lldb) frame variable
(uint8_t) x = 'a'
(intptr_t) y = 124752287
```

很多情况下，可以使用LLDB type关联其他格式的数据，例如：

```bash
(lldb) frame variable
(uint8_t) x = chr='a' dec=65 hex=0x41
(intptr_t) y = 0x76f919f
```

此外，某些数据结构可能以用户难以阅读的方式对其数据进行编码【⚠️深有体会！】，在这种情况下，可以使用数据格式化程序以人类可读的方式显示数据。例如，如果没有格式化程序，则打印包含`std::deque<int>`元素的a 的结果将类似于：`{2, 3, 4, 5, 6}`

```bash
(lldb) frame variable a_deque
(std::deque<Foo, std::allocator<int> >) $0 = {
   std::_Deque_base<Foo, std::allocator<int> > = {
      _M_impl = {
         _M_map = 0x000000000062ceb0
         _M_map_size = 8
         _M_start = {
            _M_cur = 0x000000000062cf00
            _M_first = 0x000000000062cf00
            _M_last = 0x000000000062d2f4
            _M_node = 0x000000000062cec8
         }
         _M_finish = {
            _M_cur = 0x000000000062d300
            _M_first = 0x000000000062d300
            _M_last = 0x000000000062d6f4
            _M_node = 0x000000000062ced0
         }
      }
   }
}
```

这很难让人理解。而适当的格式化程序能够产生以下输出：

```
(lldb) frame variable a_deque
(std::deque<Foo, std::allocator<int> >) $0 = size=5 {
   [0] = 2
   [1] = 3
   [2] = 4
   [3] = 5
   [4] = 6
}
```

有几个与数据可视化相关的功能：格式、摘要、过滤器、合成子项。

为了反映这一点，type 命令有五个子命令：

```
type format
type summary
type filter
type synthetic
type category
```

每个命令（除）都有四个可用的子命令：`type category`

- `add`：将新的打印选项与一种或多种类型关联
- `delete`：删除现有关联
- `list`：提供所有关联的列表
- `clear`：删除所有关联

### 类型格式

类型格式使您能够快速覆盖显示原始类型（通常的基本 C/C++/ObjC 类型：int、float、char 等）的默认格式。

如果由于某种原因您希望程序中的所有 int 变量都以十六进制打印出来，则可以向 int 类型添加格式。通过输入：

```bash
(lldb) type format add --format hex int
```

> 例：
>
> ```
> typedef int A;
> typedef A B;
> typedef B C;
> typedef C D;
> ```
>
> 并且您希望将所有 A 显示为十六进制，将所有 C 显示为字节数组，并保留其他类型的默认值（尽管它看起来很牵强，但这个例子在大型软件系统中并不不切实际）。如果你只是输入：
>
> ```
> (lldb) type format add -f hex A
> (lldb) type format add -f uint8_t[] C
> ```
>
> B 类型的值将显示为十六进制，而 D 类型的值将显示为字节数组，如下所示：
>
> ```
> (lldb) frame variable -T
> (A) a = 0x00000001
> (B) b = 0x00000002
> (C) c = {0x03 0x00 0x00 0x00}
> (D) d = {0x04 0x00 0x00 0x00}
> ```
>
> 这是因为默认情况下 LLDB 通过 typedef 链级联格式。为了避免这种情况，您可以使用选项 -C no 来阻止级联，从而制作实现目标所需的两个命令：
>
> ```
> (lldb) type format add -C no -f hex A
> (lldb) type format add -C no -f uint8_t[] C
> ```
>
> 提供所需的输出：
>
> ```bash
> (lldb) frame variable -T
> (A) a = 0x00000001
> (B) b = 2
> (C) c = {0x03 0x00 0x00 0x00}
> (D) d = 4
> ```
>
> 请注意，匹配类型时，诸如 const 和 volatile 之类的限定符将被删除，例如：
>
> ```
> (lldb) frame var x y z
> (int) x = 1
> (const int) y = 2
> (volatile int) z = 4
> (lldb) type format add -f hex int
> (lldb) frame var x y z
> (int) x = 0x00000001
> (const int) y = 0x00000002
> (volatile int) z = 0x00000004
> ```
>
> 您需要查看的两个附加选项是 –skip-pointers (-p) 和 –skip-references (-r)。这两个选项分别阻止 LLDB 将类型 T 的格式应用于类型 T* 和 T& 的值。
>
> ```
> (lldb) type format add -f float32[] int
> (lldb) frame variable pointer *pointer -T
> (int *) pointer = {1.46991e-39 1.4013e-45}
> (int) *pointer = {1.53302e-42}
> (lldb) type format add -f float32[] int -p
> (lldb) frame variable pointer *pointer -T
> (int *) pointer = 0x0000000100100180
> (int) *pointer = {1.53302e-42}
> ```
>
> 虽然它们可以应用于指针和引用，但格式不会尝试在应用格式之前取消引用指针并提取值，这意味着您实际上是在格式化存储在指针中的地址，而不是指针值。因此，您可能希望在定义格式时使用 -p 选项。
>
> 如果您需要删除自定义格式，只需键入 type format delete，后跟适用该格式的类型的名称。即使您在同一个命令上为多种类型定义了相同的格式，type format delete 也只会删除作为参数传递的类型名称的格式。
>
> 要删除所有格式，请使用。要查看所有定义的格式，请使用类型格式列表。`type format clear`
>
> 但是，如果您需要做的只是以自定义格式显示一个变量，同时保持其他相同类型的变量不变，则只需输入：
>
> ```bash
> (lldb) frame variable counter -f hex
> ```

### API接口

| **格式名称**                                                 | **缩写** | **描述**                                                     |
| :----------------------------------------------------------- | :------- | :----------------------------------------------------------- |
| `default`                                                    |          | 使用默认的 LLDB 算法来选择格式                               |
| `boolean`                                                    | B        | 将其显示为真/假布尔值，使用惯常规则：0 为假，其他均为真      |
| `binary`                                                     | b        | 将其显示为位序列                                             |
| `bytes`                                                      | y        | 依次显示字节                                                 |
| `bytes with ASCII`                                           | Y        | 显示字节，但也尝试将它们显示为 ASCII 字符                    |
| `character`                                                  | c        | 将字节显示为 ASCII 字符                                      |
| `printable character`                                        | C        | 将字节显示为可打印的 ASCII 字符                              |
| `complex float`                                              | F        | 将该值解释为复数浮点数的实部和虚部                           |
| `c-string`                                                   | s        | 将其显示为以 0 结尾的 C 字符串                               |
| `decimal`                                                    | d        | 将其显示为有符号整数（这不执行强制类型转换，它只是将字节显示为有符号的整数） |
| `enumeration`                                                | E        | 将其显示为枚举，如果可用则打印值的名称，否则打印整数值       |
| `hex`                                                        | x        | 以十六进制表示法显示（这不执行强制转换，它只是将字节显示为十六进制） |
| `float`                                                      | f        | 将其显示为浮点数（这不执行强制转换，它只是将字节解释为 IEEE754 浮点值） |
| `octal`                                                      | o        | 以八进制表示                                                 |
| `OSType`                                                     | O        | 将其显示为 MacOS OSType                                      |
| `unicode16`                                                  | U        | 将其显示为 UTF-16 字符                                       |
| `unicode32`                                                  |          | 将其显示为 UTF-32 字符                                       |
| `unsigned decimal`                                           | u        | 将其显示为无符号整数（这不执行强制转换，它只是将字节显示为无符号整数） |
| `pointer`                                                    | p        | 将其显示为本机指针（除非这真的是一个指针，否则生成的地址可能无效） |
| `char[]`                                                     |          | 将其显示为字符数组                                           |
| `int8_t[], uint8_t[]` `int16_t[], uint16_t[]` `int32_t[], uint32_t[]` `int64_t[], uint64_t[]` `uint128_t[]` |          | 将其显示为相应整数类型的数组                                 |
| `float32[], float64[]`                                       |          | 将其显示为相应数组浮点类型                                   |
| `complex integer`                                            | l        | 将此值解释为复数整数的实部和虚部                             |
| `character array`                                            | a        | 将其显示为字符数组                                           |
| `address`                                                    | A        | 将其显示为地址目标（符号/文件/行 + 偏移量），也可能是该地址指向的字符串 |
| `hex float`                                                  |          | 将其显示为十六进制浮点数                                     |
| `instruction`                                                | i        | 将其显示为反汇编的操作码                                     |
| `void`                                                       | v        | 不显示任何内容                                               |

## 类型摘要

### 类型摘要

类型格式通过显示变量值的不同显示方式来工作。然而，它们仅适用于基本类型。当你想以自定义格式显示类或结构体时，无法使用格式。

另一种功能，类型摘要，通过从类、结构体等（聚合类型）中提取信息并以用户定义的格式排列信息来工作，如以下示例所示：

添加摘要前…

```sh
(lldb) frame variable -T one
(i_am_cool) one = {
   (int) x = 3
   (float) y = 3.14159
   (char) z = 'E'
}
```

添加摘要后…

```sh
(lldb) frame variable one
(i_am_cool) one = int = 3, float = 3.14159, char = 69
```

有两种使用类型摘要的方法：第一种是将摘要字符串绑定到类型；第二种是编写一个返回摘要字符串的Python脚本。两种选项都通过`type summary add`命令启用。

示例中显示输出的命令是：

```sh
(lldb) type summary add --summary-string "int = ${var.x}, float = ${var.y}, char = ${var.z%u}" i_am_cool
```

最初，我们将重点放在摘要字符串上，然后描述Python绑定机制。

### 摘要字符串

摘要字符串使用一种简单的控制语言编写，如上面的代码片段所示。摘要字符串包含一系列令牌，这些令牌由LLDB处理以生成摘要。

摘要字符串可以包含普通文本、控制字符和访问当前对象及整体程序状态信息的特殊变量。

普通文本是任何不包含 `{`、`}`、`$` 或 `\` 字符的字符序列，这些字符是语法控制字符。

特殊变量位于`${`前缀之间，以`}`后缀结尾。变量可以是简单名称，或者可以引用自身具有子项的复杂对象。换句话说，变量看起来像`${object}`或`${object.child.otherchild}`。变量还可以用其他符号前缀或后缀来改变其值的处理方式。一个例子是`${*var.int_pointer[0-3]}`。

基本上，语法与描述帧和线程格式的语法相同，加上特定于摘要字符串的附加符号。主要的符号是`${var}`，用于引用创建摘要的变量。

最简单的操作是通过输入其表达路径来获取类或结构体的成员变量。在前面的示例中，字段`float y`的表达路径是`.y`。因此，要让摘要字符串显示`y`，你可以输入`${var.y}`。

如果你有如下代码：

```c
struct A {
   int x;
   int y;
};
struct B {
   A x;
   A y;
   int *z;
};
```

类型为`B`的对象中`x`成员的`y`成员的表达路径是`.x.y`，你可以输入`${var.x.y}`在类型`B`的摘要字符串中显示它。

默认情况下，为类型`T`定义的摘要也适用于`T*`和`T&`类型（如果需要，可以禁用此行为）。因此，表达路径不区分`.`和`->`，上述表达路径`.x.y`同样适用于显示`B*`，甚至如果`B`的实际定义是：

```c
struct B {
   A *x;
   A y;
   int *z;
};
```

这不同于`frame variable`命令的行为，后者相反，会强制区分。上述选择的理由是取消这种区分使你可以为类型`T`编写一次摘要字符串，并用于`T`和`T*`实例。由于摘要字符串主要用于提取嵌套成员的信息，因此对象的指针与对象本身一样好用。

如果需要访问`B::z`指向的整数值，不能简单地说`${var.z}`，因为该符号引用的是指针`z`。为了解引用并获取指向的值，应说`${*var.z}`。`${*var}`告诉LLDB获取表达路径指向的对象，然后解引用它。在此示例中，它等效于C/C++语法中的`*(bObject.z)`。由于`.`和`->`运算符都可以使用，无需在表达路径中间进行解引用（例如，读取`*(B::x)`中的`A::x`时，不需要输入`${*(var.x).x}`）。为了实现该效果，你可以简单地写`${var.x->x}`，甚至`${var.x.x}`。`*`运算符仅绑定到整个表达路径的结果，而不是逐段绑定，且无法使用括号更改该行为。

当然，摘要字符串可以包含多个`${var}`说明符，并且可以同时使用`${var}`和`${*var}`说明符。

### 格式化摘要元素

表达路径可以包含格式代码。类似于前面讨论的类型格式，你也可以自定义在摘要字符串中显示变量的方式，而不管它们的类型应用了什么格式。要做到这一点，你可以在表达路径中使用`%format`，如`${var.x->x%u}`，这将以无符号整数的形式显示`x`的值。

此外，还可以通过使用以冒号 `:` 标记开头的LLVM格式字符串实现自定义输出。例如，比较`${var.byte%x}`和`${var.byte:x-}`。前者使用LLDB内置的十六进制格式化（`x`），无条件地插入`0x`前缀，并且还会将值零填充以匹配类型的大小。后者使用`llvm::formatv`格式化（`:x-`），只会打印十六进制值，不带`0x`前缀，也不填充。这种原始控制在将多个部分组合成一个更大的整体时非常有用。

你还可以使用一些其他特殊格式标记，这些标记本身在格式中不可用，但在此上下文中有特殊含义：

| 符号 | 描述                                                         |
| ---- | ------------------------------------------------------------ |
| %S   | 使用此对象的摘要（聚合类型的默认值）                         |
| %V   | 使用此对象的值（非聚合类型的默认值）                         |
| %@   | 使用语言运行时的特定描述（对于C++无效，对于Objective-C调用`NSPrintForDebugger` API） |
| %L   | 使用此对象的位置（内存地址、寄存器名称等）                   |
| %#   | 使用此对象的子项数量                                         |
| %T   | 使用此对象的数据类型名称                                     |
| %N   | 打印变量的基本名称                                           |
| %>   | 打印此项的表达路径                                           |

自LLDB 3.7.0起，你还可以指定`${script.var:pythonFuncName}`。

预期你使用的函数名指定的函数其签名与Python摘要函数相同。函数返回的字符串将原样放置在输出中。

不能将元素访问或格式符号与这种语法结合使用。例如，以下内容无效，会导致摘要评估失败：

```sh
${script.var.element[0]:myFunctionName%@}
```

### 内联元素

`type summary add`命令的`--inline-children`选项（`-c`）告诉LLDB不要查找摘要字符串，而是只打印对象的所有子项列表在一行上。

例如，给定一个类型对：

```sh
(lldb) frame variable --show-types a_pair
(pair) a_pair = {
   (int) first = 1;
   (int) second = 2;
}
```

如果输入以下命令：

```sh
(lldb) type summary add --inline-children pair
```

输出变为：

```sh
(lldb) frame variable a_pair
(pair) a_pair = (first=1, second=2)
```

当然，你也可以通过输入以下命令获得相同的效果：

```sh
(lldb) type summary add pair --summary-string "(first=${var.first}, second=${var.second})"
```

虽然最终结果相同，但使用`--inline-children`通常可以节省时间。如果不需要查看变量的名称，而只需查看其值，可以将`--omit-names`（`-O`，大写字母o）选项与`--inline-children`结合使用以获得：

```sh
(lldb) frame variable a_pair
(pair) a_pair = (1, 2)
```

这当然等同于输入：

```sh
(lldb) type summary add pair --summary-string "(${var.first}, ${var.second})"
```

### 位域和数组语法

有时，基本类型的值实际上表示打包在一起的多个不同值，在位域中。

在经典视图中，没有办法查看它们。十六进制显示可以有所帮助，但如果位实际跨越半字节边界，帮助有限。

二进制视图可以毫不含糊地显示所有内容，但对于现实生活中的场景来说往往太详细和难以阅读。

为了解决这个问题，LLDB支持在摘要字符串中原生的位域格式。如果表达路径指向所谓的标量类型（通常是int、float、char、double、short、long、long long、double、long double及其无符号变体），你可以要求LLDB只抓取值中的一些位并以任何你喜欢的格式显示它们。如果只需要一个位，可以使用`[n]`，就像索引数组一样。要提取多个位，可以使用类似切片的语法：`[n-m]`，例如：

```sh
(lldb) frame variable float_point
(float) float_point = -3.14159
(lldb) type summary add --summary-string "Sign: ${var[31]%B} Exponent: ${var[30-23]%x} Mantissa: ${var[0-22]%u}" float
(lldb) frame variable float_point
(float) float_point = -3.14159 Sign: true Exponent: 0x00000080 Mantissa: 4788184
```

在此示例中，LLDB通过从浮点对象中提取位域来显示浮点变量的内部表示。

输入范围时，极值`n`和`m`总是包含在内，并且索引的顺序无关紧要。

LLDB还允许使用类似的语法在摘要字符串中显示数组成员。例如，你可能希望使用比默认更紧凑的表示法显示给定类型的所有数组，然后只深入到对调试任务有趣的个别数组成员。你可以告诉LLDB以特殊方式格式化数组，可能与数组成员的数据类型格式化方式无关。例如：

```sh
(lldb) frame variable sarray
(Simple [3]) sarray = {
   [0] = {
      x = 1
      y = 2
      z = '\x03'
   }
   [1] = {
      x = 4
      y = 5
      z = '\x06'
   }
   [2] = {
      x = 7
      y = 8
      z = '\t'
   }
}

(lldb) type summary add --summary-string "${var[].x}" "Simple [3]"

(lldb) frame variable sarray
(Simple [3]) sarray = [1,4,7]
```

`[]`符号表示：如果`var`是一个数组并且我知道它的大小，请将此摘要字符串应用于数组的每个元素。这里，我们要求LLDB显示数组每个元素的`.x`，实际上也是如此。如果发现其中一些整数异常，可以进一步详细检查该项，而不会受到数组格式的影响：

```sh
(lldb) frame variable sarray[1]
(Simple) sarray[1] = {
   x = 4
   y = 5
   z = '\x06'
}
```

你还可以使用相同的语法要求LLDB仅打印数组范围的子集以提取位域：

```sh
(lldb) type summary add --summary-string "${var[1-2].x}" "Simple [3]"

(lldb) frame variable sarray
(Simple [3]) sarray = [4,7]
```

如果你处理的是一个你知道是数组的指针，可以使用这种语法显示指针数组中包含的元素，而不仅仅是指针值。然而，由于指针没有大小概念，空方括号`[]`操作符无效，必须明确提供上下界。

一般来说，LLDB需要方括号操作符`[]`来正确处理数组和指针，对于指针还需要一个范围。然而，定义了一些特殊情况以使生活更轻松：

可以使用`%s`格式打印0终止字符串（C字符串），省略方括号，如：

```sh
(lldb) type summary add --summary-string "${var%s}" "char *"
```

这种语法适用于`char*`和`char[]`，因为LLDB可以依靠最终的0终止符来知道字符串何时结束。

LLDB为`char*`和`char[]`定义了默认的摘要字符串，这种特殊情况在调试器启动时自动定义如下：

```sh
(lldb) type summary add --summary-string "${var%s}" "char *"
(lldb) type summary add --summary-string "${var%s}" -x "char \[[0-9]+]"
```

任何数组格式（如`int8_t[]`、`float32{}`等），以及`y`、`Y`和`a`格式都可以用于打印非聚合类型的数组，即使省略方括号：

```sh
(lldb) type summary add --summary-string "${var%int32_t[]}" "int [10]"
```

然而，这一特性不适用于指针，因为LLDB无法检测指向数据的结尾。这也不适用于其他格式（如布尔），必须指定方括号操作符才能获得预期输出。

### Python 脚本

大多数时候，摘要字符串在总结变量内容方面已经足够好。然而，当你需要做的不仅仅是提取一些值并重新排列它们以显示时，摘要字符串就不再是一个有效的工具。这是因为摘要字符串缺乏对变量值进行任何类型计算的能力。

为了解决这个问题，你可以绑定一些Python脚本代码作为你的数据类型的摘要，这个脚本可以像摘要字符串那样提取子变量，并对提取的值进行有效计算。下面是一个小示例，假设我们有一个矩形类（Rectangle）：

```cpp
class Rectangle
{
private:
   int height;
   int width;
public:
   Rectangle() : height(3), width(5) {}
   Rectangle(int H) : height(H), width(H*2-1) {}
   Rectangle(int H, int W) : height(H), width(W) {}
   int GetHeight() { return height; }
   int GetWidth() { return width; }
};
```

摘要字符串在减少默认查看模式使用的屏幕空间方面非常有效，但如果我们想显示矩形对象的面积和周长，它们就无效了。

为了实现这一点，我们可以简单地将一个小的Python脚本附加到Rectangle类，如下例所示：

```sh
(lldb) type summary add -P Rectangle
Enter your Python command(s). Type 'DONE' to end.
def function (valobj, internal_dict, options):
   height_val = valobj.GetChildMemberWithName('height')
   width_val = valobj.GetChildMemberWithName('width')
   height = height_val.GetValueAsUnsigned(0)
   width = width_val.GetValueAsUnsigned(0)
   area = height * width
   perimeter = 2 * (height + width)
   return 'Area: ' + str(area) + ', Perimeter: ' + str(perimeter)
DONE
```

结果输出如下：

```sh
(lldb) frame variable
(Rectangle) r1 = Area: 20, Perimeter: 18
(Rectangle) r2 = Area: 72, Perimeter: 36
(Rectangle) r3 = Area: 16, Perimeter: 16
```

为了编写有效的摘要脚本，你需要了解LLDB的公共API，这是Python代码访问LLDB对象模型的方式。有关API的更多详细信息，你应该查看LLDB API参考文档。

### Python 脚本简介

你的脚本封装在一个函数中，该函数传递两个参数：`valobj`和`internal_dict`。

- `internal_dict`是LLDB使用的内部支持参数，不应修改它。
- `valobj`是封装实际变量的对象，其类型为`SBValue`。对`SBValue`可以进行的许多操作中，最基本的是通过调用`GetChildMemberWithName()`并传递子变量名称来检索其包含的子对象（本质上是它包装的对象的字段）。

如果变量有值，可以请求它并使用`GetValue()`作为字符串返回，或者使用`GetValueAsSigned()`或`GetValueAsUnsigned()`作为有符号/无符号数字返回。还可以通过调用`GetData()`检索一个`SBData`对象，然后从`SBData`中读取对象的内容。

如果需要深入多个层次的层级结构，可以使用方法`GetValueForExpressionPath()`，传递一个表达路径，就像你可以对摘要字符串使用的那些路径（不同之一是解引用指针不是通过在路径前加*前缀，而是通过对返回的`SBValue`调用`Dereference()`方法）。如果需要访问数组切片，不能通过这种方法调用实现，必须使用`GetChildAtIndex()`逐个查询数组项。此外，处理自定义格式也是你需要自己处理的。

### 定义Python摘要

以下是一些定义Python摘要的方式：

1. **使用`--python-script`选项**：

   ```sh
   (lldb) type summary add --python-script "height = valobj.GetChildMemberWithName('height').GetValueAsUnsigned(0); width = valobj.GetChildMemberWithName('width').GetValueAsUnsigned(0); return 'Area: %d' % (height*width)" Rectangle
   ```

2. **使用`--python-function (-F)`选项**：

   ```sh
   (lldb) type summary add -F my_module.my_function Rectangle
   ```

   你需要确保函数已经在交互解释器中定义，或者通过`command script import`命令从文件中加载。LLDB无法找到你传递的函数时会发出警告，但仍会注册绑定。

### 正则表达式类型名

要将自定义摘要字符串与数组类型关联，必须将数组大小作为类型名的一部分。这在使用不同大小的数组时会变得冗长。

如果使用`-x`选项，类型名将被视为正则表达式而不是类型名。例如：

```sh
(lldb) type summary add --summary-string "${var[].x}" -x "Simple \[[0-9]+\]"
(lldb) frame variable
(Simple [3]) sarray = [1,4,7]
(Simple [2]) sother = [3,6]
```

上面的示例适用于`Simple [3]`以及任何其他`Simple`对象的数组。

### 命名摘要

对于给定类型，可能有不同有意义的摘要表示。然而，目前每次只能将一个摘要与类型关联。如果需要临时覆盖变量的关联摘要字符串，可以使用命名摘要。

命名摘要通过在创建时将名称附加到摘要。然后，当需要将摘要附加到变量时，`frame variable`命令支持一个`--summary`选项，告诉LLDB使用给定的命名摘要而不是默认摘要。

```sh
(lldb) type summary add --summary-string "x=${var.integer}" --name NamedSummary
(lldb) frame variable one
(i_am_cool) one = int = 3, float = 3.14159, char = 69
(lldb) frame variable one --summary NamedSummary
(i_am_cool) one = x=3
```

定义命名摘要时，将其绑定到一个或多个类型是可选的。即使将命名摘要绑定到一个类型，后来更改该类型的摘要字符串，命名摘要也不会更改。你可以使用`type summary delete`命令删除命名摘要，就像摘要名称是应用摘要的数据类型一样。

使用`--summary`选项附加到变量的摘要，其语义与使用`-f`选项附加的自定义格式相同：它保持附加状态，直到你附加新的摘要，或者让程序重新运行。

### 合成子元素

摘要在能够通过表达路径导航时效果很好。为了让LLDB做到这一点，必须提供适当的调试信息。

有些类型是“黑箱”的，即没有提供其内部结构的信息。这种情况下，表达路径无法正确工作。

在其他情况下，尽管内部结构可以用于表达路径，但它们并不提供用户友好的对象值表示。例如，考虑由GNU C++库实现的STL向量：

```sh
(lldb) frame variable numbers -T
(std::vector<int>) numbers = {
   (std::_Vector_base<int, std::allocator<int> >) std::_Vector_base<int, std::allocator<int> > = {
      (std::_Vector_base<int, std::allocator<int> >::_Vector_impl) _M_impl = {
            (int *) _M_start = 0x00000001001008a0
            (int *) _M_finish = 0x00000001001008a8
            (int *) _M_end_of_storage = 0x00000001001008a8
      }
   }
}
```

在这里，你可以看到类型的实现，并且可以为该实现编写摘要，但这不会帮助你推断向量中实际存储的项目。

你可能希望看到的是这样的输出：

```sh
(lldb) frame variable numbers -T
(std::vector<int>) numbers = {
   (int) [0] = 1
   (int) [1] = 12
   (int) [2] = 123
   (int) [3] = 1234
}
```

合成子元素是一种实现该结果的方法。

该功能基于为变量提供一组新的子元素，替换默认通过调试信息提供的子元素。在该示例中，我们可以使用合成子元素将向量项作为`std::vector`对象的子元素提供。

### 创建合成子元素

要创建合成子元素，你需要提供一个符合给定接口的Python类（用接口一词是因为Python没有明确的接口概念，这意味着Python类必须实现一组给定的方法）：

```python
class SyntheticChildrenProvider:
   def __init__(self, valobj, internal_dict):
      # 初始化Python对象，使用valobj作为提供合成子元素的变量
      self.valobj = valobj
   def num_children(self, max_children):
      # 返回对象应该拥有的子元素数量
      return 0
   def get_child_index(self, name):
      # 返回给定名称的合成子元素的索引
      return -1
   def get_child_at_index(self, index):
      # 返回表示给定索引处子元素的新LLDB SBValue对象
      return None
   def update(self):
      # 更新此Python对象的内部状态，当LLDB中的变量状态变化时调用
      return False
   def has_children(self):
      # 如果对象可能有子元素，返回True；如果可以保证没有子元素，返回False
      return False
   def get_value(self):
      # 返回一个SBValue作为合成值的值
      return None
```

需要注意的是，由Python格式化器引发的异常会被LLDB静默捕获，应由格式化器本身适当处理。更具体地说，如果发生异常，LLDB可能会假定给定对象没有子元素，或者跳过打印某些子元素，因为它们是逐个打印的。

### 示例：提供合成子元素

例如，假设我们有一个`Foo`类，其合成子元素提供者类`Foo_Provider`可用，在文件`~/Foo_Tools.py`中的Python模块中。以下交互将`Foo_Provider`设置为LLDB中的合成子元素提供者：

```sh
(lldb) command script import ~/Foo_Tools.py
(lldb) type synthetic add Foo --python-class Foo_Tools.Foo_Provider
(lldb) frame variable a_foo
(Foo) a_foo = {
   x = 1
   y = "Hello world"
}
```

LLDB为libstdcpp和libcxx提供的核心子集STL类以及若干Foundation类提供合成子元素。

### 扩展摘要字符串

合成子元素扩展了摘要字符串，通过启用一个新的特殊变量`${svar}`。

此符号告诉LLDB将表达路径指向合成子元素，而不是实际的子元素。例如：

```sh
(lldb) type summary add --expand -x "std::vector<" --summary-string "${svar%#} items"
(lldb) frame variable numbers
(std::vector<int>) numbers = 4 items {
   (int) [0] = 1
   (int) [1] = 12
   (int) [2] = 123
   (int) [3] = 1234
}
```

LLDB在调用摘要字符串提供者之前调用合成子元素提供者，这允许后者访问实际的可显示子元素。这适用于内联摘要字符串和基于Python的摘要提供者。

需要注意的是，当通过编程访问具有合成子元素提供者的变量的子元素或子元素计数时，LLDB隐藏了实际的原始子元素。例如，假设我们有一个`std::vector`，它具有标记其数据开始的实际内存属性`__begin`。在执行合成子元素提供者之后，`std::vector`变量将不再显示`__begin`作为子元素，即使通过SB API也是如此。相反，它将具有由提供者计算的子元素。如果需要实际的原始子元素，调用`value.GetNonSyntheticValue()`即可获得值的原始版本。

### 动态类型发现

在进行Objective-C开发时，你可能会注意到一些变量的类型为`id`（例如，从`NSArray`中提取的项目）。默认情况下，LLDB不会显示对象的实际类型。实际上，它可以像运行时在调用选择器时那样动态发现Objective-C变量的类型。要显示该发现结果，需要对`frame variable`或`expression`使用特殊选项`--dynamic-type`。

`--dynamic-type`可以有三个值：

- `no-dynamic-values`：默认，防止动态类型发现
- `no-run-target`：启用动态类型发现，只要不需要在目标上运行代码
- `run-target`：启用代码执行以进行动态类型发现

如果指定`no-run-target`或`run-target`，LLDB将检测变量的动态类型并显示适当的格式化器。例如：

```sh
(lldb) expr @"Hello"
(NSString *) $0 = 0x00000001048000b0 @"Hello"
(lldb) expr -d no-run @"Hello"
(__NSCFString *) $1 = 0x00000001048000b0 @"Hello"
```

因为LLDB使用一种检测算法，不需要在目标进程上调用任何函数，所以`no-run-target`已经足够使其工作。

### 类别

类别是一种分组相关格式化器的方法。例如，LLDB本身将libstdc++类型的格式化器分组在一个名为`gnu-libstdc++`的类别中。基本上，类别充当容器，用于存储同一库或操作系统版本的格式化器。

默认情况下，LLDB中创建了几个类别：

- `default`：这是每个格式化器最终进入的类别，除非指定了另一个类别
- `objc`：不特定依赖于macOS的基本和常见Objective-C类型的格式化器
- `gnu-libstdc++`：libstdcpp实现的`std::string`、`std::vector`、`std::list`和`std::map`的格式化器
- `libcxx`：libcxx实现的`std::string`、`std::vector`、`std::list`和`std::map`的格式化器
- `system`：需要格式化器的基本类型
- `AppKit`：Cocoa类
- `CoreFoundation`：CF类
- `CoreGraphics`：CG类
- `CoreServices`：CS类
- `VectorTypes`：几个向量类型的紧凑显示

如果你想为格式化器使用自定义类别，所有`type … add`命令都提供一个`--category (-w)`选项，用于指定类别名称以添加格式化器。要删除格式化器，你必须指定正确的类别。

类别可以处于两种状态之一：启用和禁用。类别初始为禁用状态，可以使用`type category enable`命令启用。要禁用已启用的类别，使用命令`type category disable`。

### 查找格式化器

给定一个变量，查找格式化器（包括格式）涉及一组相当复杂的规则。具体来说，LLDB开始在每个启用的类别中查找，按照启用的顺序（最新启用的在前）。在每个类别中，LLDB执行以下操作：

1. 如果有变量类型的格式化器，使用它
2. 如果这是一个指针对象，并且有指向类型的格式化器且不跳过指针，使用它
3. 如果这是一个引用对象，并且有引用类型的格式化器且不跳过引用，使用

它
4. 如果这是一个Objective-C类并且启用了动态类型，查找对象的动态类型的格式化器。如果禁用了动态类型，或者查找失败，查找对象声明类型的格式化器
5. 如果对象的类型是typedef，遍历typedef层次结构（如果编译器没有发出足够的信息，LLDB可能无法做到这一点）。如果层次结构的任何级别有一个有效的格式化器可以级联，使用它。
6. 如果所有尝试都失败了，重复上述搜索，查找正则表达式而不是精确匹配

如果这些尝试中的任何一个返回了要使用的有效格式化器，使用该格式化器，并终止搜索（不继续查找其他类别）。如果在当前类别中没有找到任何内容，按照相同算法扫描下一个启用的类别。如果没有更多启用的类别，搜索失败。

需要注意的是，先前的LLDB版本定义了级联不仅包括遍历typedef链，还包括继承链。由于显著降低性能，此功能已被移除。你需要为继承链中的每个类型设置格式化器，以应用该格式化器。

## LLDB 的符号化

LLDB被分为包含调试器核心的共享库和实现调试和命令解释器的驱动程序。LLDB可以用于符号化崩溃日志，并且通常能提供比其他符号化程序更多的信息：

- 内联函数
- 地址范围内的变量及其位置

最简单的符号化形式是加载一个可执行文件：

```sh
(lldb) target create --no-dependents --arch x86_64 /tmp/a.out
```

我们在使用`target create`命令时使用了`--no-dependents`标志，这样我们就不会从当前系统加载所有依赖的共享库。当我们进行符号化时，通常是符号化在另一个系统上运行的二进制文件，即使主可执行文件可能引用`/usr/lib`中的共享库，我们通常也不希望加载当前计算机上的版本。

使用`image list`命令将显示与当前目标关联的所有共享库列表。如预期的那样，我们当前只有一个二进制文件：

```sh
(lldb) image list
[  0] 73431214-6B76-3489-9557-5075F03E36B4 0x0000000100000000 /tmp/a.out
      /tmp/a.out.dSYM/Contents/Resources/DWARF/a.out
```

现在我们可以查找一个地址：

```sh
(lldb) image lookup --address 0x100000aa3
      Address: a.out[0x0000000100000aa3] (a.out.__TEXT.__text + 131)
      Summary: a.out`main + 67 at main.c:13
```

由于我们没有为二进制文件中的各个部分指定滑动地址或任何加载地址，这里使用的地址是文件地址。文件地址指的是每个对象文件定义的虚拟地址。

如果我们在`target create`时没有使用`--no-dependents`选项，我们会加载所有依赖的共享库：

```sh
(lldb) image list
[  0] 73431214-6B76-3489-9557-5075F03E36B4 0x0000000100000000 /tmp/a.out
      /tmp/a.out.dSYM/Contents/Resources/DWARF/a.out
[  1] 8CBCF9B9-EBB7-365E-A3FF-2F3850763C6B 0x0000000000000000 /usr/lib/system/libsystem_c.dylib
[  2] 62AA0B84-188A-348B-8F9E-3E2DB08DB93C 0x0000000000000000 /usr/lib/system/libsystem_dnssd.dylib
[  3] C0535565-35D1-31A7-A744-63D9F10F12A4 0x0000000000000000 /usr/lib/system/libsystem_kernel.dylib
...
```

现在如果我们使用文件地址进行查找，可能会有多个匹配项，因为大多数共享库的虚拟地址空间从零开始：

```sh
(lldb) image lookup -a 0x1000
      Address: a.out[0x0000000000001000] (a.out.__PAGEZERO + 4096)
      Address: libsystem_c.dylib[0x0000000000001000] (libsystem_c.dylib.__TEXT.__text + 928)
      Summary: libsystem_c.dylib`mcount + 9
      Address: libsystem_dnssd.dylib[0x0000000000001000] (libsystem_dnssd.dylib.__TEXT.__text + 456)
      Summary: libsystem_dnssd.dylib`ConvertHeaderBytes + 38
      Address: libsystem_kernel.dylib[0x0000000000001000] (libsystem_kernel.dylib.__TEXT.__text + 1116)
      Summary: libsystem_kernel.dylib`clock_get_time + 102
...
```

为了避免获取多个文件地址匹配项，可以指定共享库的名称以限制搜索范围：

```sh
(lldb) image lookup -a 0x1000 a.out
      Address: a.out[0x0000000000001000] (a.out.__PAGEZERO + 4096)
```

### 定义部分的加载地址

在符号化崩溃日志时，如果总是需要将崩溃日志地址调整为文件地址，这会很麻烦。为了避免进行任何转换，可以为目标中的模块设置部分的加载地址。一旦设置了任何部分的加载地址，查找将切换为使用加载地址。可以通过相同的数量滑动可执行文件中的所有部分，或为单个部分设置加载地址。`target modules load --slide`命令允许我们为所有部分设置加载地址。

下面是一个将`a.out`中的所有部分通过添加0x123000进行滑动的示例：

```sh
(lldb) target create --no-dependents --arch x86_64 /tmp/a.out
(lldb) target modules load --file a.out --slide 0x123000
```

通常通过名称指定每个部分的实际加载位置更容易。macOS上的崩溃日志具有指定每个二进制文件__TEXT段地址的Binary Images部分。指定滑动需要首先找到__TEXT段的原始（文件）地址，并减去两个值。如果指定__TEXT段的地址，可以使用`target modules load`命令加载部分地址：

```sh
(lldb) target create --no-dependents --arch x86_64 /tmp/a.out
(lldb) target modules load --file a.out __TEXT 0x100123000
```

我们指定__TEXT部分加载在0x100123000。现在我们已经定义了部分在目标中的加载位置，任何查找现在将使用加载地址，这样我们就不必对崩溃日志中的地址进行任何数学计算，可以直接使用原始地址：

```sh
(lldb) image lookup --address 0x100123aa3
      Address: a.out[0x0000000100000aa3] (a.out.__TEXT.__text + 131)
      Summary: a.out`main + 67 at main.c:13
```

### 加载多个可执行文件

在符号化崩溃日志时，通常涉及多个可执行文件。当这种情况发生时，可以为主可执行文件或一个共享库创建目标，然后使用`target modules add`命令向目标添加更多模块。

假设我们有一个包含以下图像的Darwin崩溃日志：

```sh
Binary Images:
   0x100000000 -    0x100000ff7 <A866975B-CA1E-3649-98D0-6C5FAA444ECF> /tmp/a.out
0x7fff83f32000 - 0x7fff83ffefe7 <8CBCF9B9-EBB7-365E-A3FF-2F3850763C6B> /usr/lib/system/libsystem_c.dylib
0x7fff883db000 - 0x7fff883e3ff7 <62AA0B84-188A-348B-8F9E-3E2DB08DB93C> /usr/lib/system/libsystem_dnssd.dylib
0x7fff8c0dc000 - 0x7fff8c0f7ff7 <C0535565-35D1-31A7-A744-63D9F10F12A4> /usr/lib/system/libsystem_kernel.dylib
```

首先我们使用主可执行文件创建目标，然后添加任何额外的共享库：

```sh
(lldb) target create --no-dependents --arch x86_64 /tmp/a.out
(lldb) target modules add /usr/lib/system/libsystem_c.dylib
(lldb) target modules add /usr/lib/system/libsystem_dnssd.dylib
(lldb) target modules add /usr/lib/system/libsystem_kernel.dylib
```

如果你有独立文件中的调试符号，例如macOS上的dSYM文件，可以使用`--symfile`选项指定它们的路径来创建目标和添加模块：

```sh
(lldb) target create --no-dependents --arch x86_64 /tmp/a.out --symfile /tmp/a.out.dSYM
(lldb) target modules add /usr/lib/system/libsystem_c.dylib --symfile /build/server/a/libsystem_c.dylib.dSYM
(lldb) target modules add /usr/lib/system/libsystem_dnssd.dylib --symfile /build/server/b/libsystem_dnssd.dylib.dSYM
(lldb) target modules add /usr/lib/system/libsystem_kernel.dylib --symfile /build/server/c/libsystem_kernel.dylib.dSYM
```

然后使用Binary Images部分中的第一个地址为每个图像设置__TEXT部分的加载地址：

```sh
(lldb) target modules load --file a.out 0x100000000
(lldb) target modules load --file libsystem_c.dylib 0x7fff83f32000
(lldb) target modules load --file libsystem

_dnssd.dylib 0x7fff883db000
(lldb) target modules load --file libsystem_kernel.dylib 0x7fff8c0dc000
```

现在任何尚未符号化的堆栈回溯都可以使用原始回溯地址进行符号化。

给定以下原始回溯：

```sh
Thread 0 Crashed:: Dispatch queue: com.apple.main-thread
0   libsystem_kernel.dylib           0x00007fff8a1e6d46 __kill + 10
1   libsystem_c.dylib                0x00007fff84597df0 abort + 177
2   libsystem_c.dylib                0x00007fff84598e2a __assert_rtn + 146
3   a.out                            0x0000000100000f46 main + 70
4   libdyld.dylib                    0x00007fff8c4197e1 start + 1
```

我们现在可以符号化加载地址：

```sh
(lldb) image lookup -a 0x00007fff8a1e6d46
(lldb) image lookup -a 0x00007fff84597df0
(lldb) image lookup -a 0x00007fff84598e2a
(lldb) image lookup -a 0x0000000100000f46
```

### 获取变量信息

如果在`image lookup --address`命令中添加`--verbose`标志，可以获取详细信息，通常包括一些局部变量的位置：

```sh
(lldb) image lookup --address 0x100123aa3 --verbose
      Address: a.out[0x0000000100000aa3] (a.out.__TEXT.__text + 110)
      Summary: a.out`main + 50 at main.c:13
      Module: file = "/tmp/a.out", arch = "x86_64"
CompileUnit: id = {0x00000000}, file = "/tmp/main.c", language = "ISO C:1999"
   Function: id = {0x0000004f}, name = "main", range = [0x0000000100000bc0-0x0000000100000dc9)
   FuncType: id = {0x0000004f}, decl = main.c:9, compiler_type = "int (int, const char **, const char **, const char **)"
     Blocks: id = {0x0000004f}, range = [0x100000bc0-0x100000dc9)
             id = {0x000000ae}, range = [0x100000bf2-0x100000dc4)
   LineEntry: [0x0000000100000bf2-0x0000000100000bfa): /tmp/main.c:13:23
     Symbol: id = {0x00000004}, range = [0x0000000100000bc0-0x0000000100000dc9), name="main"
   Variable: id = {0x000000bf}, name = "path", type= "char [1024]", location = DW_OP_fbreg(-1072), decl = main.c:28
   Variable: id = {0x00000072}, name = "argc", type= "int", location = r13, decl = main.c:8
   Variable: id = {0x00000081}, name = "argv", type= "const char **", location = r12, decl = main.c:8
   Variable: id = {0x00000090}, name = "envp", type= "const char **", location = r15, decl = main.c:8
   Variable: id = {0x0000009f}, name = "aapl", type= "const char **", location = rbx, decl = main.c:8
```

有趣的部分是列出的变量。这些变量是指定地址范围内的参数和局部变量。这些变量条目有位置，在上面用粗体显示。崩溃日志通常包含每个堆栈的第一帧的寄存器信息，能够重建一个或多个局部变量通常可以帮助你从崩溃日志中获取更多信息。需要注意的是，这仅对第一帧有用，并且仅当崩溃日志有线程的寄存器信息时才有用。

### 使用Python API进行符号化

上述所有命令都可以通过Python脚本桥完成。下面的代码将重新创建目标，并添加我们在Darwin崩溃日志示例中添加的三个共享库：

```python
triple = "x86_64-apple-macosx"
platform_name = None
add_dependents = False
target = lldb.debugger.CreateTarget("/tmp/a.out", triple, platform_name, add_dependents, lldb.SBError())
if target:
      # 获取可执行模块
      module = target.GetModuleAtIndex(0)
      target.SetSectionLoadAddress(module.FindSection("__TEXT"), 0x100000000)
      module = target.AddModule ("/usr/lib/system/libsystem_c.dylib", triple, None, "/build/server/a/libsystem_c.dylib.dSYM")
      target.SetSectionLoadAddress(module.FindSection("__TEXT"), 0x7fff83f32000)
      module = target.AddModule ("/usr/lib/system/libsystem_dnssd.dylib", triple, None, "/build/server/b/libsystem_dnssd.dylib.dSYM")
      target.SetSectionLoadAddress(module.FindSection("__TEXT"), 0x7fff883db000)
      module = target.AddModule ("/usr/lib/system/libsystem_kernel.dylib", triple, None, "/build/server/c/libsystem_kernel.dylib.dSYM")
      target.SetSectionLoadAddress(module.FindSection("__TEXT"), 0x7fff8c0dc000)

      load_addr = 0x00007fff8a1e6d46
      # so_addr是段偏移地址，或lldb.SBAddress对象
      so_addr = target.ResolveLoadAddress (load_addr)
      # 获取段偏移地址的符号上下文，包括模块、编译单元、函数、块、行条目和符号
      sym_ctx = so_addr.GetSymbolContext (lldb.eSymbolContextEverything)
      print(sym_ctx)
```

### 使用内置Python模块进行符号化

LLDB在`lldb`包中包含一个名为`lldb.utils.symbolication`的模块。此模块包含许多符号化函数，通过允许你创建表示符号化类对象的对象来简化符号化过程，例如：

- `lldb.utils.symbolication.Address`
- `lldb.utils.symbolication.Section`
- `lldb.utils.symbolication.Image`
- `lldb.utils.symbolication.Symbolicator`

#### lldb.utils.symbolication.Address

此类表示将被符号化的地址。它将缓存已查找的任何信息：模块、编译单元、函数、块、行条目、符号。它通过将`lldb.SBSymbolContext`作为成员变量来实现这一点。

#### lldb.utils.symbolication.Section

此类表示可能加载到`lldb.utils.symbolication.Image`中的部分。它具有允许你从崩溃日志文件中提取的文本中设置部分的帮助函数。

#### lldb.utils.symbolication.Image

此类表示可能加载到我们用于符号化的目标中的模块。此类包含可执行路径、可选符号文件路径、三元组和需要加载的部分列表。如果需要，许多这些对象不会加载到目标中，除非符号化需要它们。你通常有一个崩溃日志，其中加载了100到200个不同的共享库，但你的崩溃日志堆栈回溯只使用其中的几个共享库。只有包含堆栈回溯地址的图像需要加载到目标中以进行符号化。

此类的子类将想要重写`locate_module_and_debug_symbols`方法：

```python
class CustomImage(lldb.utils.symbolication.Image):
   def locate_module_and_debug_symbols (self):
      # 根据崩溃日志中的信息定位模块和符号文件
```

重写此函数允许客户端找到正确的可执行模块和符号文件，因为它们可能驻留在构建服务器上。

#### lldb.utils.symbolication.Symbolicator

此类通过仅加载需要加载的`lldb.utils.symbolication.Image`实例来协调符号化过程，以符号化提供的地址。

#### lldb.macosx.crashlog

`lldb.macosx.crashlog`是一个在macOS构建中分发的包，它继承了上述类。此模块解析Darwin崩溃日志中的信息，并创建表示图像、部分和堆栈回溯的符号化对象。然后，它使用`lldb.utils.symbolication`中的函数符号化崩溃日志。

此模块在LLDB命令解释器中安装了一个新命令`crashlog`，这样你可以使用它来解析和符号化macOS崩溃日志：

```sh
(lldb) command script import lldb.macosx.crashlog
"crashlog" and "save_crashlog" command installed, use the "--help

" option for detailed help
(lldb) crashlog /tmp/crash.log
...
```

安装的命令内置帮助，显示在符号化时可以使用的选项：

```sh
(lldb) crashlog --help
Usage: crashlog [options]  [FILE ...]
```

符号化一个或多个 Darwin 崩溃日志文件，以提供源文件和行信息、内联堆栈帧以及崩溃线程第一个帧的反汇编位置。如果将此脚本导入LLDB命令解释器，解释器中将添加一个`crashlog`命令，以便在LLDB命令行中使用。解析并符号化崩溃日志后，将创建一个目标，其中加载了崩溃日志文件中找到的所有共享库的加载地址。这允许你像在崩溃日志描述的位置停止程序一样探索程序，并且可以使用崩溃日志中找到的地址执行函数的反汇编和查找。

### 选项

- `-h, --help`: 显示此帮助信息并退出
- `-v, --verbose`: 显示详细的调试信息
- `-g, --debug`: 显示详细的调试日志
- `-a, --load-all`: 加载所有可执行图像，而不仅仅是崩溃堆栈帧中找到的图像
- `--images`: 显示图像列表
- `--debug-delay=NSEC`: 为调试器暂停NSEC秒
- `-c, --crashed-only`: 仅符号化崩溃的线程
- `-d DISASSEMBLE_DEPTH, --disasm-depth=DISASSEMBLE_DEPTH`: 设置应该反汇编的堆栈帧深度（默认为1）
- `-D, --disasm-all`: 启用对所有线程帧的反汇编（不仅仅是崩溃的线程）
- `-B DISASSEMBLE_BEFORE, --disasm-before=DISASSEMBLE_BEFORE`: 指定反汇编前的指令数
- `-A DISASSEMBLE_AFTER, --disasm-after=DISASSEMBLE_AFTER`: 指定反汇编后的指令数
- `-C NLINES, --source-context=NLINES`: 显示源上下文的NLINES行（默认=4）
- `--source-frames=NFRAMES`: 显示NFRAMES的源代码（默认=4）
- `--source-all`: 显示所有线程的源代码，而不仅仅是崩溃的线程
- `-i, --interactive`: 解析所有崩溃日志并进入交互模式

### 使用方法

将此脚本导入LLDB命令解释器后，可以通过以下命令符号化崩溃日志：

```sh
(lldb) command script import lldb.macosx.crashlog
"crashlog" and "save_crashlog" command installed, use the "--help" option for detailed help
(lldb) crashlog /tmp/crash.log
```

此命令将在解释器中安装`crashlog`和`save_crashlog`命令，用于解析和符号化macOS崩溃日志。

### 示例

以下是一些示例选项和它们的作用：

- `crashlog -v /tmp/crash.log`: 详细符号化崩溃日志
- `crashlog --source-context=10 /tmp/crash.log`: 显示崩溃上下文的10行源代码
- `crashlog --disasm-depth=3 /tmp/crash.log`: 反汇编崩溃堆栈的前三个帧
- `crashlog --interactive /tmp/crash.log`: 解析所有崩溃日志并进入交互模式

这些命令允许你以不同的详细程度和上下文信息符号化崩溃日志，以便更好地调试和分析崩溃原因。

## macOS上的符号管理

在macOS上，调试符号通常存放在独立的捆绑文件中，称为dSYM文件。这些捆绑文件包含DWARF调试信息和与构建及调试信息相关的其他资源。

`DebugSymbols.framework`框架在给定UUID时有助于定位dSYM文件。它可以使用多种方法定位符号：

- Spotlight
- 显式搜索路径
- 隐式搜索路径
- 文件映射UUID路径
- 运行一个或多个Shell脚本

`DebugSymbols.framework`还具有全局默认值，可以修改这些默认值，使所有调试工具（lldb、gdb、sample、CoreSymbolication.framework）都能轻松找到重要的调试符号。`DebugSymbols.framework`默认值的域是`com.apple.DebugSymbols`，可以使用`defaults` shell命令读取、写入或修改这些默认值：

```sh
% defaults read com.apple.DebugSymbols
% defaults write com.apple.DebugSymbols KEY ...
% defaults delete com.apple.DebugSymbols KEY
```

以下是一些可以用来增强符号定位的默认键值对设置：

#### DBGFileMappedPaths

此默认值可以指定为单个字符串或字符串数组。每个字符串表示一个包含指向dSYM文件的文件映射UUID值的目录。有关更多详细信息，请参阅下文的“文件映射UUID目录”部分。每当`DebugSymbols.framework`被要求查找dSYM文件时，它将首先在任何文件映射UUID目录中查找匹配项。

```sh
% defaults write com.apple.DebugSymbols DBGFileMappedPaths -string /path/to/uuidmap1
% defaults write com.apple.DebugSymbols DBGFileMappedPaths -array /path/to/uuidmap1 /path/to/uuidmap2
```

#### DBGShellCommands

此默认值可以指定为单个字符串或字符串数组。指定一个shell脚本，该脚本将运行以查找dSYM。shell脚本将在给定一个UUID值作为shell命令参数时运行，并期望返回一个属性列表。请参阅下文定义的属性列表格式。

```sh
% defaults write com.apple.DebugSymbols DBGShellCommands -string /path/to/script1
% defaults write com.apple.DebugSymbols DBGShellCommands -array /path/to/script1 /path/to/script2
```

#### DBGSpotlightPaths

指定要限制Spotlight搜索的目录，作为字符串或字符串数组。当其他默认值被提供给`com.apple.DebugSymbols`时，除非此默认值设置为空数组，否则将禁用Spotlight搜索：

```sh
# Specify an empty array to keep Spotlight searches enabled in all locations
% defaults write com.apple.DebugSymbols DBGSpotlightPaths -array

# Specify an array of paths to limit spotlight searches to certain directories
% defaults write com.apple.DebugSymbols DBGSpotlightPaths -array /path/dir1 /path/dir2
```

### Shell脚本属性列表格式

用`DBGShellCommands`默认键指定的shell脚本将按指定的顺序运行，直到找到匹配项。shell脚本将以单个UUID字符串值（如“23516BE4-29BE-350C-91C9-F36E7999F0F1”）被调用。shell脚本必须通过将属性列表写入STDOUT来响应。返回的属性列表必须包含UUID字符串值作为根键值，每个UUID都有一个字典。这些字典可以包含以下一个或多个键：

- **DBGArchitecture**：文本架构或目标三元组，如“x86_64”、“i386”或“x86_64-apple-macosx”。
- **DBGBuildSourcePath**：构建dSYM文件时使用的路径前缀。调试信息将包含具有此前缀的路径。
- **DBGSourcePath**：构建完成后源代码所在的路径前缀。在构建项目时，构建机器通常会在临时目录中托管源代码，然后将源代码移动到另一个位置进行归档。如果调试信息中的路径与源代码当前托管的位置不匹配，则指定此路径以及`DBGBuildSourcePath`将帮助开发工具在调试或符号化时始终显示源代码。
- **DBGDSYMPath**：dSYM捆绑包内的dSYM mach-o文件的路径。
- **DBGSymbolRichExecutable**：符号丰富的可执行文件的路径。二进制文件通常在构建并打包到发行版中后被剥离。如果构建系统保存了未剥离的可执行文件，可以提供此可执行文件的路径。
- **DBGError**：如果无法为提供的UUID定位二进制文件，可以返回一个用户可读的错误。

以下是一个包含两个架构的二进制文件的示例shell脚本输出：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
    <key>23516BE4-29BE-350C-91C9-F36E7999F0F1</key>
    <dict>
        <key>DBGArchitecture</key>
        <string>i386</string>
        <key>DBGBuildSourcePath</key>
        <string>/path/to/build/sources</string>
        <key>DBGSourcePath</key>
        <string>/path/to/actual/sources</string>
        <key>DBGDSYMPath</key>
        <string>/path/to/foo.dSYM/Contents/Resources/DWARF/foo</string>
        <key>DBGSymbolRichExecutable</key>
        <string>/path/to/unstripped/executable</string>
    </dict>
    <key>A40597AA-5529-3337-8C09-D8A014EB1578</key>
    <dict>
        <key>DBGArchitecture</key>
        <string>x86_64</string>
        <key>DBGBuildSourcePath</key>
        <string>/path/to/build/sources</string>
        <key>DBGSourcePath</key>
        <string>/path/to/actual/sources</string>
        <key>DBGDSYMPath</key>
        <string>/path/to/foo.dSYM/Contents/Resources/DWARF/foo</string>
        <key>DBGSymbolRichExecutable</key>
        <string>/path/to/unstripped/executable</string>
    </dict>
</dict>
</plist>
```

当被请求查找dSYM文件时，shell脚本没有超时，因此请确保不要制作高延迟或需要长时间下载的shell脚本，除非这是你真正想要的。这可能会在LLDB和GDB的调试会话中、使用CoreSymbolication或Report Crash进行符号化时减慢速度，并且不会向用户显示任何可见的反馈。你可以快速返回一个包含单个`DBGError`键的plist，表示已达到超时。你可能还希望执行新进程以进行下载，以便在shell脚本退出后你的下载仍然可以继续，以便后续的调试会话可以使用缓存的文件。还需要跟踪当前的下载进程，以防你收到相同UUID的多个请求，从而避免同时下载相同的文件。还需要验证下载是否成功，然后才将文件放入缓存中。

### 在dSYM捆绑包中嵌入UUID属性列表

由于dSYM文件是捆绑包，你也可以将UUID信息plist文件放在dSYM捆绑包的`Contents/Resources`目录中。创建dSYM捆绑包中的UUID plist文件的主要原因之一是，它将帮助LLDB和其他开发工具显示源代码。LLDB目前知道如何检查这些plist文件，因此可以自动重新映射调试信息中的源位置信息。

例如，我们可以将上面返回的两个UUID值分离出来，并将其保存到dSYM捆绑包中：

```sh
% ls /path/to/foo.dSYM/Contents/Resources
23516BE4-29BE-350C-91C9-F36E7999F0F1.plist
A40597AA-5529-3337-8C09-D8A014EB1578.plist

% cat /path/to/foo.dSYM/Contents/Resources/23516BE4-29BE-350C-91C9-F36E7999F0F1.plist
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
   <key>DBGArchitecture</key>
   <string>i386</string>
   <key>DBGBuildSourcePath</key>
   <string>/path/to/build/sources</string>
   <key>DBGSourcePath</key>
   <string>/path/to/actual/sources</string>
   <key>DBGDSYMPath</key>
   <string>/path/to/foo.dSYM/Contents/Resources/DWARF/foo</string>
   <key>DBGSymbolRichExecutable</key>


   <string>/path/to/unstripped/executable</string>
   <key>DBGVersion</key>
   <string>3</string>
   <key>DBGSourcePathRemapping</key>
   <dict>
       <key>/path/to/build/time/src/location1</key>
       <string>/path/to/debug/time/src/location</string>
       <key>/path/to/build/time/src/location2</key>
       <string>/path/to/debug/time/src/location</string>
   </dict>
   <key>DBGSymbolRichExecutable</key>
   <string>/path/to/unstripped/executable</string>
</dict>
</plist>
```

注意输出非常接近shell脚本输出所需的内容，因此通过将两个plist组合成一个plist并将UUID用作字符串键，内容为plist的内容，可以非常容易地创建shell脚本的结果。

LLDB将从dSYM捆绑包中的每个UUID plist文件中读取以下条目：`DBGSymbolRichExecutable`、`DBGBuildSourcePath`和`DBGSourcePath`，以及`DBGSourcePathRemapping`（如果`DBGVersion`为3或更高）。`DBGBuildSourcePath`和`DBGSourcePath`用于重新映射单个文件路径。例如，文件可能在构建时位于`/BuildDir/SheetApp/SheetApp-37`，但在调试时位于`/SourceDir/SheetApp/SheetApp-37`，这两个路径可以在这些键中列出。如果有多个源路径重映射，可以使用`DBGSourcePathRemapping`字典，其中可以包含任意数量的条目。如果plist中同时存在`DBGSourcePathRemapping`和`DBGBuildSourcePath/DBGSourcePath`，将首先使用`DBGSourcePathRemapping`条目进行路径重映射。这可能允许在`DBGSourcePathRemapping`字典中进行更具体的重映射，并在`DBGBuildSourcePath/DBGSourcePath`对中进行不太具体的重映射作为最后手段。

### 文件映射UUID目录

文件映射目录可用于高效地查找本地或远程的dSYM文件。UUID通过将前20个十六进制数字分成4个字符的块来分解，并在每个块内部创建一个目录，最后在最深的目录中创建一个符号链接，其名称为最后12个十六进制数字。符号链接的值是dSYM捆绑包中包含DWARF的mach-o文件的完整路径。每当`DebugSymbols.framework`被要求查找dSYM文件时，它将首先在任何文件映射UUID目录中查找快速匹配项，如果默认设置适当。

例如，如果我们从上面的示例UUID plist信息中创建一个文件映射UUID目录缓存在`~/Library/SymbolCache/dsyms/uuids`中，我们可以轻松看到布局方式：

```sh
% find ~/Library/SymbolCache/dsyms/uuids -type l
~/Library/SymbolCache/dsyms/uuids/2351/6BE4/29BE/350C/91C9/F36E7999F0F1
~/Library/SymbolCache/dsyms/uuids/A405/97AA/5529/3337/8C09/D8A014EB1578
```

文件映射目录中的最后条目是指向dSYM捆绑包中实际dsym mach文件的符号链接：

```sh
% ls -lAF ~/Library/SymbolCache/dsyms/uuids/2351/6BE4/29BE/350C/91C9/F36E7999F0F1
~/Library/SymbolCache/dsyms/uuids/2351/6BE4/29BE/350C/91C9/F36E7999F0F1@ -> ../../../../../../dsyms/foo.dSYM/Contents/Resources/DWARF/foo
```

然后，你可以告诉DebugSymbols检查此UUID文件映射缓存：

```sh
% defaults write com.apple.DebugSymbols DBGFileMappedPaths ~/Library/SymbolCache/dsyms/uuids
```

### 定位dSYM的Shell脚本提示

dSYM查找shell脚本的一种可能实现是让脚本下载并缓存文件到已知位置。然后为找到的每个UUID值创建一个UUID映射，以便下次查询dSYM文件时可以使用缓存版本。因此，shell脚本用于初始下载和缓存文件，后续访问将使用缓存并避免每次都调用shell脚本。

然后`DebugSymbols.framework`的默认值包括启用你的shell脚本，启用文件映射路径设置，以便快速找到已下载的dSYM文件，而无需每次运行shell脚本，同时保持Spotlight启用，以便仍然找到其他普通的dSYM文件：

```sh
% defaults write com.apple.DebugSymbols DBGShellCommands /path/to/shellscript
% defaults write com.apple.DebugSymbols DBGFileMappedPaths ~/Library/SymbolCache/dsyms/uuids
% defaults write com.apple.DebugSymbols DBGSpotlightPaths -array
```

希望这能帮助解释`DebugSymbols.framework`如何帮助任何公司实现智能符号查找和缓存，且开销最小。

## 远程调试

远程调试是指在一个系统上调试运行在另一个系统上的进程。我们将运行调试器的系统称为本地系统，而运行被调试进程的系统称为远程系统。

为了启用远程调试，LLDB使用客户端-服务器架构。客户端部分运行在本地系统上，而服务器部分运行在远程系统上。客户端和服务器使用gdb-remote协议进行通信，通常通过TCP/IP传输。有关该协议的更多信息可以在LLDB源代码库的`docs/lldb-gdb-remote.txt`文件中找到。除了gdb-remote stub外，LLDB的服务器部分还包括一个平台二进制文件，负责执行高级调试操作，例如从远程系统复制文件，并可以在远程系统上执行任意shell命令。

为了减少代码复杂性并改进远程调试体验，Linux和macOS上的LLDB即使在本地调试进程时也使用远程调试stub。这是通过在本地生成一个远程stub进程并通过回环接口与之通信来实现的。对于本地调试，这个过程对用户是透明的，因为不需要文件传输，平台二进制文件也不会被使用。

### 远程调试的准备

虽然实际调试过程（单步执行、获取回溯、评估表达式）与本地调试相同，但远程调试需要更多的准备工作，因为需要在远程系统上手动启动所需的二进制文件。此外，如果远程系统运行不同的操作系统或架构，则需要单独编译服务器组件。

#### 远程系统

在Linux和Android上，所有必要的远程功能都包含在`lldb-server`二进制文件中。该二进制文件结合了平台和gdb-remote stub的功能。单个二进制文件简化了部署并减少了代码大小，因为这两种功能共享了大量代码。`lldb-server`二进制文件也是静态链接到LLDB的其他部分的（与默认情况下动态链接到`liblldb.so`的LLDB不同），因此它不依赖于LLDB的其他部分。在macOS和iOS上，远程gdb功能由`debugserver`二进制文件实现，你需要将其与`lldb-server`一起部署。

上述二进制文件需要存在于远程系统上以启用远程调试。你可以在远程系统上直接编译它们，也可以从本地机器复制它们。如果在本地编译而远程架构与本地架构不同，则需要交叉编译正确版本的二进制文件。有关交叉编译LLDB的更多信息，请参阅构建页面。

一旦二进制文件就位，你只需要在平台模式下运行`lldb-server`并指定它应该监听的端口。例如，以下命令：

```sh
remote% lldb-server platform --listen "*:1234" --server
```

将启动LLDB平台并等待来自任何地址的端口1234的传入连接。指定一个地址而不是`*`将只允许来自该地址的连接。将`--server`参数添加到命令行将为每个传入连接派生一个新进程，从而允许多个并行调试会话。

#### 本地系统

在本地系统上，你需要让LLDB知道你打算进行远程调试。这是通过`platform`命令及其子命令实现的。第一步，你需要为你的远程系统选择正确的平台插件。可以通过`platform list`命令获得可用插件的列表：

```sh
local% lldb
(lldb) platform list
```

输出的可用平台插件列表应包含许多用于调试不同类型系统的插件。远程插件以“remote-”为前缀。例如，要调试远程Linux应用程序：

```sh
(lldb) platform select remote-linux
```

选择平台插件后，你应该会收到一个确认选择的平台提示，并说明你未连接。这是因为远程插件需要连接到其远程平台对应部分才能运行。这可以通过`platform connect`命令实现。该命令需要多个参数（如需了解更多信息，请使用`help`命令），但通常你只需指定要连接的地址，例如：

```sh
(lldb) platform connect connect://remote:1234
  Platform: remote-linux
    Triple: x86_64-gnu-linux
  Hostname: remote
 Connected: yes
WorkingDir: /tmp
```

请注意，平台的工作目录是`/tmp`。默认情况下，启动进程时可执行文件将上传到此目录。

在这之后，你应该能够正常调试。你可以使用`process attach`附加到现有的远程进程，或者使用`target create`和`process launch`启动一个新进程。平台插件将透明地处理上传或下载可执行文件以进行调试。如果你的应用程序需要其他文件，可以使用`platform`命令（例如`get-file`、`put-file`、`mkdir`等）传输它们。还可以使用`platform shell`命令进一步准备环境。

在使用“remote-android”平台时，客户端LLDB会转发两个端口，一个用于连接到平台，另一个用于连接到gdbserver。客户端端口可以通过环境变量`ANDROID_PLATFORM_LOCAL_PORT`和`ANDROID_PLATFORM_LOCAL_GDB_PORT`分别进行配置。

### 在远程机器上启动本地构建的进程

#### 在平台工作目录中安装并运行

要在远程系统的平台工作目录中启动本地构建的进程：

```sh
(lldb) file a.out
(lldb) run
```

这将使LLDB创建一个使用你交叉构建的“a.out”可执行文件的目标。`run`命令将使LLDB将“a.out”上传到平台的当前工作目录，前提是文件已更改。平台连接允许我们传输文件，但也允许我们获取另一端文件的MD5校验和，并且只有在文件更改时才上传文件。LLDB将自动启动一个以gdbremote模式运行的`lldb-server`，以允许你调试该可执行文件，连接到它并启动调试会话。

#### 更改平台工作目录

你可以在连接到平台时更改平台工作目录：

```sh
(lldb) platform settings -w /usr/local/bin
```

你可以使用`platform status`命令验证它是否成功：

```sh
(lldb) platform status
  Platform: remote-linux
    Triple: x86_64-gnu-linux
  Hostname: remote
 Connected: yes
WorkingDir: /usr/local/bin
```

如果我们再次运行程序，它将被安装到`/usr/local/bin`中。

#### 通过指定远程安装路径进行安装和运行

如果你希望“a.out”可执行文件安装到“/bin/a.out”而不是平台的当前工作目录，我们可以使用Python设置平台文件规范：

```sh
(lldb) file a.out
(lldb) script lldb.target.module['a.out'].SetPlatformFileSpec("/bin/a.out")
(lldb) run
```

现在，当你运行程序时，程序将被上传到“/bin/a.out”而不是平台当前工作目录。默认情况下，只有主可执行文件会在启动应用程序时上传到远程系统。如果你有需要上传的共享库，可以将本地构建的共享库添加到当前目标，并设置其平台文件规范：

```sh
(lldb) file a.out
(lldb) target module add /local/build/libfoo.so
(lldb) target module add /local/build/libbar.so
(lldb) script lldb.target.module['libfoo.so'].SetPlatformFileSpec("/usr/lib/libfoo.so")
(lldb) script lldb.target.module['libbar.so'].SetPlatformFileSpec("/usr/local/lib/libbar.so")
(lldb) run
```

### 附加到远程进程

如果你想附加到远程进程，可以首先列出远程系统上的进程：

```sh
(lldb) platform process list
223 matching processes were found on "remote-linux"
PID    PARENT USER       TRIPLE                   NAME
====== ====== ========== ======================== ============================
68639  90652             x86_64-apple-macosx      lldb
...
```

然后，附加到远程进程的操作与指定远程进程ID一样简单：

```sh
(lldb) attach 68639
```

这将附加到远程进程ID为68639的进程，并允许你开始调试该进程。

## 在 QEMU 中测试 LLDB

#### QEMU 系统模式仿真

在没有实际硬件的情况下，可以使用 QEMU 在仿真环境中测试 LLDB。以下描述了如何设置 QEMU 仿真环境以测试 LLDB 的说明。

`llvm-project/lldb/scripts/lldb-test-qemu` 下的脚本可以快速帮助使用 QEMU 设置虚拟的 LLDB 测试环境。目前，这些脚本支持 Arm 或 AArch64，但可以轻松添加对其他架构的支持。

- `setup.sh` 用于从源代码构建 Linux 内核镜像和 QEMU 系统仿真可执行文件。
- `rootfs.sh` 用于生成用于 QEMU 系统模式仿真的 Ubuntu 根文件系统镜像。
- `run-qemu.sh` 利用 QEMU 启动带有根文件系统镜像的 Linux 内核镜像。

一旦启动内核后，我们就可以在仿真环境中运行 `lldb-server`。以下说明和 QEMU 帮助脚本是在 Ubuntu Bionic/Focal x86_64 主机上测试的。请根据你的主机发行版/架构进行相应更新。

注意：本页面的说明和 QEMU 帮助脚本在 Ubuntu Bionic/Focal (x86_64) 主机上进行了验证。此外，脚本需要 sudo/root 权限来安装依赖项和设置 QEMU 主机/客户机网络。

下面是一些常见 LLDB QEMU 测试帮助脚本的使用示例：

### 生成用于 QEMU 系统仿真的 Ubuntu 根文件系统镜像

示例：生成 1 GB 大小的 Ubuntu Bionic (armhf) 根文件系统镜像

```sh
$ bash rootfs.sh --arch armhf --distro bionic --size 1G
```

示例：生成 2 GB 大小的 Ubuntu Focal (arm64) 根文件系统镜像

```sh
$ bash rootfs.sh --arch arm64 --distro focal --size 2G
```

`rootfs.sh` 已经过测试，可生成 Ubuntu Bionic 和 Focal 镜像，但也可以用来生成其他 Debian Linux 发行版的根文件系统镜像。

`rootfs.sh` 默认将生成的镜像的用户名设置为主机计算机上的当前用户名。

### 使用 `setup.sh` 从源代码构建 QEMU 或交叉编译 Linux 内核

示例：构建 QEMU 二进制文件和 Arm/AArch64 Linux 内核镜像

```sh
$ bash setup.sh --qemu --kernel arm
$ bash setup.sh --qemu --kernel arm64
```

示例：仅构建 Linux 内核镜像

```sh
$ bash setup.sh --kernel arm
$ bash setup.sh --kernel arm64
```

示例：构建 `qemu-system-arm` 和 `qemu-system-aarch64` 二进制文件

```sh
$ bash setup.sh --qemu
```

示例：从工作目录中删除 `qemu.git`、`linux.git` 和 `linux.build`

```sh
$ bash setup.sh --clean
```

### 使用 `run-qemu.sh` 运行 QEMU Arm 或 AArch64 系统仿真

`run-qemu.sh` 具有以下依赖项：

1. 按照 [QEMU 文档](https://wiki.qemu.org/Documentation/Networking/NAT) 设置 QEMU 的桥接网络。
2. 确保 `/etc/qemu-ifup` 脚本可用且具有可执行权限。
3. QEMU 二进制文件必须使用 `setup.sh` 从源代码构建，或通过命令行参数 `--qemu` 提供。
4. Linux 内核镜像必须使用 `setup.sh` 从源代码构建，或通过命令行参数 `--kernel` 提供。
5. 如果使用 `setup.sh` 构建了 Linux 内核和 QEMU 二进制文件，则 `linux.build` 和 `qemu.git` 文件夹必须存在于当前目录中。
6. `--sve` 选项将启用 AArch64 SVE 模式。
7. `--sme` 选项将启用 AArch64 SME 模式（SME 需要 SVE，因此也将启用 SVE）。
8. `--mte` 选项将启用 AArch64 MTE（内存标签）模式（可以单独使用或与 `--sve` 一起使用）。

示例：使用 `run-qemu.sh` 运行 QEMU Arm 或 AArch64 系统仿真

```sh
$ sudo bash run-qemu.sh --arch arm --rootfs <rootfs 镜像路径>
$ sudo bash run-qemu.sh --arch arm64 --rootfs <rootfs 镜像路径>
```

示例：使用命令行提供的内核镜像和 QEMU 二进制文件运行 QEMU

```sh
$ sudo bash run-qemu.sh --arch arm64 --rootfs <rootfs 镜像路径> \
--kernel <Linux 内核镜像路径> --qemu <QEMU 二进制文件路径>
```

### 在 QEMU 系统仿真环境中运行 `lldb-server` 的步骤

#### 使用桥接网络

1. 确保桥接网络在主机和 QEMU 虚拟机之间启用。
2. 找出分配给仿真环境中 `eth0` 的 IP 地址。
3. 设置主机和仿真环境之间的 SSH 访问。
4. 登录仿真环境并安装依赖项：

```sh
$ sudo apt install python-dev libedit-dev libncurses5-dev libexpat1-dev
```

5. 交叉编译 LLDB 服务器以用于 AArch64 Linux：请访问 [LLDB 编译说明](https://lldb.llvm.org/resources/build.html) 了解如何交叉编译 LLDB 服务器。
6. 将 LLDB 服务器可执行文件传输到仿真环境：

```sh
$ scp lldb-server username@ip-address-of-emulation-environment:/home/username
```

7. 在 QEMU 虚拟机中运行 `lldb-server`。
8. 尝试连接到运行在 QEMU 虚拟机中的 `lldb-server`，指定 ip:port。

#### 无桥接网络

如果没有桥接网络，你需要从虚拟机到主机转发各个端口（请参考 QEMU 手册中的具体选项）。

- 至少一个端口用于连接到初始的 `lldb-server`。
- 如果你想在平台模式下使用 `lldb-server` 并让它为你启动一个 gdbserver 实例，则需要另一个端口。
- 如果你想对 `lldb-server` 平台运行测试，则需要更多端口。

如果你在进行后两种操作之一，还应限制 `lldb-server` 尝试使用的端口，否则它会随机选择一个几乎肯定没有转发的端口。下面是一个示例：

```sh
$ lldb-server platform --server --listen 0.0.0.0:54321 \
  --min-gdbserver-port 49140 --max-gdbserver-port 49150
```

这样配置的结果是：

- `lldb-server` 平台模式在端口 54321 上监听外部连接。
- 当要求启动一个新的 gdbserver 模式实例时，它将使用 49140 到 49150 范围内的端口。

你的虚拟机配置应该转发端口 54321 和 49140 到 49150 以使其工作。

## 使用 Intel Processor Trace 进行跟踪

Intel PT 是现代 Intel CPU 提供的一项技术，允许高效地跟踪进程执行的所有指令。LLDB 可以收集这些跟踪并使用其符号化堆栈进行转储。详细信息可以参考[这里](https://easyperf.net/blog/2019/08/23/Intel-Processor-Trace)。

#### 先决条件

确认你的 CPU 支持 Intel PT（请参阅[此处](https://www.intel.com/content/www/us/en/support/articles/000056730/processors.html)）并且操作系统是 Linux。

检查系统中是否存在以下文件：

```sh
$ cat /sys/bus/event_source/devices/intel_pt/type
```

输出应为一个数字。如果没有，尝试升级内核。

#### 构建指令

1. 克隆并构建低级 Intel PT 解码库 LibIPT 库：

```sh
$ git clone git@github.com:intel/libipt.git
$ mkdir libipt-build
$ cmake -S libipt -B libipt-build
$ cd libipt-build
$ make
```

这将在 `<libipt-build>/lib` 和 `<libipt-build>/libipt/include` 目录中生成一些文件。

2. 配置并构建支持 Intel PT 的 LLDB：

```sh
$ cmake \
    -DLLDB_BUILD_INTEL_PT=ON \
    -DLIBIPT_INCLUDE_PATH="<libipt-build>/libipt/include" \
    -DLIBIPT_LIBRARY_PATH="<libipt-build>/lib" \
    ... other common configuration parameters
$ cd <lldb-build> && ninja lldb lldb-server # 如果使用 Ninja
```

#### 如何使用

调试进程时，可以开启 Intel PT 跟踪，这将“记录”进程将执行的所有指令。开启后，可以继续调试，并在任何断点处检查指令列表。例如：

```sh
lldb <target>
> b main
> run
> process trace start # 开始跟踪所有线程，包括未来的线程
# 继续调试直到遇到断点

> thread trace dump instructions
# 这应该会输出类似的内容

thread #2: tid = 2861133, total instructions = 5305673
  libc.so.6`__GI___libc_read + 45 at read.c:25:1
    [4962255] 0x00007fffeb64c63d    subq   $0x10, %rsp
    [4962256] 0x00007fffeb64c641    movq   %rdi, -0x18(%rbp)
  libc.so.6`__GI___libc_read + 53 [inlined] __libc_read at read.c:26:10
    [4962257] 0x00007fffeb64c645    callq  0x7fffeb66b640            ; __libc_enable_asynccancel
  libc.so.6`__libc_enable_asynccancel
    [4962258] 0x00007fffeb66b640    movl   %fs:0x308, %eax
  libc.so.6`__libc_enable_asynccancel + 8
    [4962259] 0x00007fffeb66b648    movl   %eax, %r11d

# 可以继续按回车键查看更多指令
```

括号中的数字是指令索引，默认情况下会选择当前线程。

#### 配置跟踪大小

CPU 以压缩格式将指令列表存储在环形缓冲区中，该缓冲区保留最新信息。默认情况下，LLDB 为每个线程使用 4KB 的缓冲区，但你可以通过运行以下命令更改它。大小必须是 2 的幂且至少为 4KB。

```sh
thread trace start all -s <size_in_bytes>
```

参考：1MB 的跟踪缓冲区可以轻松存储约 500 万条指令。

#### 打印更多指令

如果想一次转储更多指令，可以运行：

```sh
thread trace dump instructions -c <count>
```

#### 打印其他线程的指令

默认情况下，转储指令时会选择当前线程，但可以通过以下方式选择其他线程：

```sh
thread trace dump instructions <#thread index>
# 例如
thread trace dump instructions 8
```

#### 崩溃分析

如果调试和跟踪的进程崩溃了，可以执行以下命令检查它是如何崩溃的，无需特别操作：

```sh
thread trace dump instructions
```

例如：

```sh
 * thread #1, name = 'a.out', stop reason = signal SIGFPE: integer divide by zero
     frame #0: 0x00000000004009f1 a.out`main at main.cpp:8:14
   6       int x;
   7       cin >> x;
-> 8       cout << 12 / x << endl;
   9       return 0;
   10  }
 (lldb) thread trace dump instructions -c 5
 thread #1: tid = 604302, total instructions = 8388
   libstdc++.so.6`std::istream::operator>>(int&) + 181
     [8383] 0x00007ffff7b41665    popq   %rbp
     [8384] 0x00007ffff7b41666    retq
   a.out`main + 66 at main.cpp:8:14
     [8385] 0x00000000004009e8    movl   -0x4(%rbp), %ecx
     [8386] 0x00000000004009eb    movl   $0xc, %eax
     [8387] 0x00000000004009f0    cltd
```

注意：当前，跟踪中不包括失败的指令，但将来可能会包含以便于阅读。

#### 离线跟踪分析

可以使用自定义 Intel PT 收集器记录跟踪，并使用 LLDB 解码和符号化跟踪。为此，`trace load` 命令非常有用。使用 `trace load` 需要首先创建一个包含跟踪会话定义的 JSON 文件。例如：

```json
{
  "type": "intel-pt",
  "cpuInfo": {
    "vendor": "GenuineIntel",
    "family": 6,
    "model": 79,
    "stepping": 1
  },
  "processes": [
    {
      "pid": 815455,
      "triple": "x86_64-*-linux",
      "threads": [
        {
          "tid": 815455,
          "iptTrace": "trace.file" # 从 AUX 缓冲区中提取的原始线程特定跟踪
        }
      ],
      "modules": [ # 这些是所有共享库 + 主可执行文件
        {
          "file": "a.out", # 如果与 systemPath 相同，则为可选
          "systemPath": "a.out",
          "loadAddress": 4194304,
        },
        {
          "file": "libfoo.so",
          "systemPath": "/usr/lib/libfoo.so",
          "loadAddress": "0x00007ffff7bd9000",
        },
        {
          "systemPath": "libbar.so",
          "loadAddress": "0x00007ffff79d7000",
        }
      ]
    }
  ]
}
```

你可以通过输入以下命令查看完整的模式：

```sh
trace schema intel-pt
```

JSON 文件主要包含跟踪进程的所有共享库及其内存加载地址。如果在获取跟踪的同一计算机上进行分析，使用 `systemPath` 字段就足够了。如果在不同的机器上进行分析，需要将这些文件复制过来，并且 `file` 字段应指向相对于 JSON 文件的文件位置。准备好 JSON 文件和模块文件后，可以简单运行：

```sh
lldb
> trace load /path/to/json
> thread trace dump instructions <optional thread index>
```

然后就像在实时会话中的情况一样。

#### 参考资料

- 此功能的原始 RFC 文档。
- 关于 Meta 使用 Intel Processor Trace 的一些详细信息可以在[此博客文章](https://easyperf.net/blog/2019/08/23/Intel-Processor-Trace)中找到。

## 按需符号
在 LLDB 中，可以为生成超出正常调试会话所需的调试信息的项目启用按需符号功能。一些构建系统为所有二进制文件启用调试信息，可能会产生数十GB的调试信息。这种情况下，调试会话加载时间可能会显著增加，并且在调试信息未索引时会减慢开发者的生产效率。特别是当所有二进制文件都有完整的调试信息时，表达式评估可能会变得缓慢，因为每个模块都会查询非常常见的类型，或者全局名称查找由于拼写错误而失败。

#### 什么时候考虑启用此功能？
如果你的构建系统为许多二进制文件生成了调试信息，而调试时只需关注其中的少数几个二进制文件，就可以考虑启用此功能。一些构建系统将调试信息作为项目范围的开关启用，而控制构建方式的构建系统文件难以修改，只为链接的少部分文件生成调试信息。如果调试会话启动时间由于过多的调试信息而变慢，此功能可能会在日常使用中帮助你提高生产效率。

#### 如何启用按需符号？
通过 LLDB 设置启用此功能：

```sh
(lldb) settings set symbols.load-on-demand true
```

用户也可以将此命令放入 `~/.lldbinit` 文件中，以便每次都启用。

#### 此功能如何工作？
此功能通过选择性地为用户关注的模块启用调试信息工作。它设计为在启用时无需用户设置其他任何设置，并将尝试自动确定何时启用模块的调试信息访问。具有调试信息的所有模块开始时都将其调试信息关闭，以进行昂贵的名称和类型查找。调试信息的行表始终保持启用，以允许用户通过文件和行可靠地设置断点。随着用户调试其目标，一些简单的操作可以导致模块启用其调试信息（称为水合）：设置文件和行断点、任何映射到模块的堆栈帧的PC、按函数名称设置断点、按名称查找全局变量。

大多数用户通过文件和行设置断点，这是用户通知调试器他们希望关注此模块的一种简便方法。按文件和行设置断点是按需符号使用时的一种主要方法，因为这是在代码的有趣区域停止程序的最常用方式。

一旦用户命中断点或由于其他原因（如崩溃、断言或信号）停止程序，调试器将计算一个或多个线程的堆栈帧。任何堆栈帧的PC值包含在某个模块的部分内，这些模块的调试信息将被启用。这使我们能够为用户停止的代码区域启用调试信息，并允许仅启用重要子集模块的调试信息。

按需符号加载尝试避免在调试信息中索引名称，并进行了一些权衡以允许函数和全局变量的名称匹配，而无需始终索引所有调试信息。按函数名称设置断点可以工作，但我们尝试通过依赖模块的符号表而避免使用调试信息。调试信息名称索引是我们尝试通过按需符号加载避免的最昂贵操作之一，因此这是此功能的主要权衡之一。当按函数名称设置断点时，如果符号表包含匹配项，将为该模块启用调试信息，并使用调试信息完成查询。这意味着按内联函数名称设置断点可能会失败，因为内联函数不存在于符号表中。当使用按需符号加载时，建议不要剥离本地符号的符号表，因为这将允许用户可靠地设置所有具体函数的断点。剥离的符号表删除了符号表中的本地符号，这意味着静态函数和未导出的函数将不会出现在符号表中。这可能导致按函数名称设置断点失败，而以前不会失败。

全局变量查找依赖与按函数名称设置断点相同的方法：如果尚未为模块启用调试信息，我们使用符号表查找匹配项。如果在符号表中找到全局变量查找的匹配项，我们将启用调试信息并使用调试信息完成查询。建议不要剥离符号表，因为静态变量和其他未导出的全局变量将不会出现在符号表中，可能导致匹配未找到。

#### 其他可能失败的情况
按需符号加载功能尝试限制在调试信息中进行昂贵的名称查找。因此，一些按名称查找可能会失败，当未启用此功能时它们不会失败：
- 按函数名称为内联函数设置断点
- 表达式解析器请求按名称查找类型时的类型查找
- 按名称查找被剥离的全局变量

按函数名称设置断点可能会因内联函数而失败，因为此信息仅包含在调试信息中。内联函数不会创建符号，除非在同一模块中有一个具体副本。因此，当启用此功能时，请求时可能不会在所有内联函数处停止。按文件和行设置断点仍然是有效使用按需符号加载的好方法，仍然可以在内联函数调用处停止。

表达式解析器在用户输入表达式时经常尝试按名称查找类型。这是表达式评估中最昂贵的部分之一，因为用户可以在表达式中输入类似“iterator”的内容，这可能导致所有模块中所有 STL 类型的匹配结果。这种全局类型查找查询如果启用调试信息，可能会导致找到成千上万的结果。当前大多数调试信息的创建方式是将类型信息内联到每个模块中。通常，每个模块将包含任何在代码中使用的类型的完整类型定义。这意味着当你在停止时输入表达式时，你拥有所有变量、参数和当前堆栈帧中的全局变量的调试信息，我们应该能够仅使用已启用调试信息的模块找到重要类型。

表达式解析器还可以要求显示全局变量，并且可以按名称查找它们。要使此功能在启用按需符号加载时可靠工作，只需不剥离符号表，表达式解析器应该能够找到你的变量。如果符号表被剥离，大多数已导出的全局变量仍会在符号表中，但静态变量和未导出的全局变量则不会。

#### 如何排除此功能对调试造成阻碍的问题？
添加了可以启用的日志记录，以帮助通知我们的工程师在启用此功能时某些内容未生成结果。可以在调试会话期间启用此日志记录，并将其发送给 LLDB 工程师以帮助排除这些情况。要启用日志记录，请输入以下命令：

```sh
(lldb) log enable -f /tmp/ondemand.txt lldb on-demand
```

启用日志记录时，我们可以完全看到每个查询，如果未启用此功能，本应生成结果，并允许我们排除故障。在表达式前启用此日志记录，按名称设置断点或执行类型查找，可以帮助我们看到导致失败的模式，并帮助我们改进此功能。

## 在 AArch64 Linux 上使用 LLDB
本页详细介绍了使用 LLDB 调试某些 AArch64 扩展的细节。如果未提及某些内容，可能是它的行为符合预期。

这不是 ptrace 和 Linux 内核文档的替代品。这涵盖了 LLDB 如何选择使用这些内容以及如何影响用户体验。

#### 可扩展矢量扩展 (SVE)
请参阅 [此处](https://developer.arm.com/documentation/102411/0100/) 了解此扩展，并参阅 [此处](https://www.kernel.org/doc/html/latest/arm64/sve.html) 了解 Linux 内核如何处理它。

在 LLDB 中，您将看到以下新寄存器：

- z0-z31 矢量寄存器，每个寄存器的大小等于矢量长度。
- p0-p15 断言寄存器，每个寄存器包含每字节1位，与矢量长度成比例。每个寄存器的大小为矢量长度 / 8。
- ffr 第一个故障寄存器，与断言寄存器大小相同。
- vg 矢量长度，以“粒度”为单位。每个粒度为8字节。

示例：

```sh
Scalable Vector Extension Registers:
      vg = 0x0000000000000002
      z0 = {0x01 0x01 0x01 0x01 0x01 0x01 0x01 0x01 <...> }
    <...>
      p0 = {0xff 0xff}
    <...>
     ffr = {0xff 0xff}
```

上例中的矢量长度为16字节。在 LLDB 中，您将始终看到“vg”寄存器，其值为2（8*2=16）。在内核代码或应用程序中，您可能会看到“vq”，这是以四字字（16字节）为单位的矢量长度。“vl”是以字节为单位的矢量长度。

#### 更改矢量长度
可以在调试会话中写入 vg 寄存器。写入当前矢量长度不会改变任何内容。如果增加矢量长度，寄存器可能会被重置为0。如果减少矢量长度，LLDB 将截断 Z 寄存器，但其他寄存器将被重置为0。

应避免假设在更改矢量长度后 SVE 状态与之前相同。无论是在调试目标内部还是由 LLDB 更改。如果需要更改矢量长度，请在函数首次使用 SVE 之前进行更改。

#### Z 寄存器呈现
LLDB 不会预测 SVE Z 寄存器的使用方式。由于 LLDB 不知道将来指令如何解释寄存器，因此它不会更改寄存器的可视化，默认显示字节大小的矢量。

如果您知道将使用的格式，可以使用格式选项：

```sh
(lldb) register read z0 -f uint32_t[]
    z0 = {0x01010101 0x01010101 0x01010101 0x01010101}
```

#### FPSIMD 和 SVE 模式
在调试目标首次使用 SVE 之前，处于内核术语中的 SIMD 模式。仅使用 FPU。在此状态下，LLDB 仍将显示 SVE 寄存器，但值只是 FPU 值零扩展到矢量长度。

首次访问 SVE 时，进程进入 SVE 模式，此时 Z 值为实际的 Z 寄存器值。

通过 LLDB 写入 SVE 寄存器也会触发此操作。请注意，没有办法在 LLDB 内撤销此更改。然而，调试目标本身可以执行某些操作返回到 SIMD 模式。

#### 表达式评估
如果评估一个表达式，所有 SVE 状态将在表达式评估前保存，并在评估后恢复，包括寄存器值和矢量长度。

### 可扩展矩阵扩展 (SME)
请参阅 [此处](https://developer.arm.com/documentation/ddi0602/latest/) 了解此扩展，并参阅 [此处](https://www.kernel.org/doc/html/latest/arm64/sme.html) 了解 Linux 内核如何处理它。

SME 为 SVE 添加了“流模式”，该模式具有自己的矢量长度，称为“流矢量长度”。

在 LLDB 中，您将看到以下新寄存器：

- tpidr2：一个额外的每线程指针，为 SME ABI 保留。它不是可扩展的，仅为指针大小（即 64 位）。
- z0-z31 流 SVE 寄存器。这些寄存器与非流寄存器同名，因此您只能在 LLDB 中看到活动集。无法读取或写入非活动模式的寄存器。它们的大小等于流矢量长度。
- za：阵列存储寄存器。即“可扩展矩阵扩展”的“矩阵”部分。这是一个正方形，由长度等于流矢量长度（svl）的行组成。总大小为 svl * svl。
- svcr：流矢量控制寄存器。这实际上是一个伪寄存器，但它匹配架构定义的 SVCR 的内容。该寄存器应用于检查流模式和/或 za 是否处于活动状态。此寄存器为只读。
- svg：流矢量长度，以粒度为单位。此值与非流模式的矢量长度无关，可以独立变化。此寄存器为只读。

注意：

在非流模式下，vg 寄存器显示非流矢量长度，svg 寄存器显示流矢量长度。在流模式下，vg 和 svg 显示流模式矢量长度。因此，在流模式下，无法在 LLDB 内读取非流矢量长度。这是 LLDB 实现的限制，而非架构的限制，架构会独立存储两种长度。

示例：

```sh
Scalable Vector Extension Registers:
      vg = 0x0000000000000002
      z0 = {0x01 0x01 0x01 0x01 0x01 0x01 0x01 0x01 0x01 0x01 0x01 <...> }
      <...>
      p0 = {0xff 0xff}
      <...>
     ffr = {0xff 0xff}

<...>

Thread Local Storage Registers:
     tpidr = 0x0000fffff7ff4320
    tpidr2 = 0x1122334455667788

Scalable Matrix Extension Registers:
       svg = 0x0000000000000002
      svcr = 0x0000000000000003
        za = {0x00 <...> 0x00}
```

在上例中，流矢量长度为16字节，我们处于流模式。请注意，svcr 的位0和1被设置，表示我们处于流模式且 ZA 处于活动状态。vg 和 svg 报告相同的值，因为 vg 显示的是流模式矢量长度。

#### 更改流矢量长度
为简化 LLDB，svg 为只读。这意味着只有在调试目标处于流模式时，才能使用 LLDB 更改流矢量长度。

与非流 SVE 一样，这样做会使 SVE 寄存器的内容变得未定义。这也会禁用 ZA，这遵循 Linux 内核的处理方式。

#### 非活动 ZA 寄存器的可见性
LLDB 不处理可以在运行时出现和消失的寄存器（SVE 更改大小但不会消失）。因此，当 za 未启用时，LLDB 将返回一个 0 块。该块将匹配 za 的预期大小：

```sh
(lldb) register read za svg svcr
    za = {0x00 0x00 0x00 0x00 0x00 0x00 0x00 0x00 0x00 <...> }
   svg = 0x0000000000000002
  svcr = 0x0000000000000001
```

请注意，svcr 位2未设置，表示 za 未激活。

如果从 LLDB 写入 za，则 za 将变为激活状态。没有办法在 LLDB 内撤销此更改。至于更改矢量长度，调试目标仍可以执行某些操作再次禁用 za。

要知道 za 是否处于激活状态，请参阅 svcr 寄存器的位2，即 SVCR.ZA。

#### ZA 寄存器呈现
与 SVE 一样，LLDB 不知道调试目标将如何使用 za，因此不知道如何最好地显示它。随时随地，任何给定指令都可能以多种方式和大小解释其内容。

因此，LLDB 默认显示 za 作为一个大向量的独立字节。您可以使用

格式选项覆盖此显示（参见上面的 SVE 示例）。

#### 表达式评估
在表达式评估后，模式（流或非流）、流矢量长度和 ZA 状态将恢复。除了所有为 SVE 保存的内容。

### 可扩展矩阵扩展 2 (SME2)
可扩展矩阵扩展 2 在与 SME 相同的架构规范中记录，并由与 SME 相同的内核文档页面涵盖。

SME2 添加了1个新寄存器，zt0。此寄存器是固定大小的 512 位寄存器，用于 SME2 中添加的新指令。它在 LLDB 中显示在现有的 SME 寄存器集中。

zt0 可以是活动的或非活动的，与 za 一样。相同的 SVCR.ZA 位控制此状态。非活动的 zt0 显示为 0，与 za 一样。不过，在 zt0 的情况下，LLDB 不需要伪造值。ptrace 已经为非活动的 zt0 返回了一个 0 块。

与 za 一样，从 LLDB 写入非活动的 zt0 将启用它和 za。调试目标也可以这样做。如果写入的是 za，zt0 将变为激活状态但值全为 0。

由于 svcr 是只读的，目前无法从 LLDB 内部停用这些寄存器（尽管运行中的进程仍可以这样做）。

要检查 zt0 是否激活，请参考 SVCR.ZA，而不是 zt0 的值。

#### ZT0 寄存器呈现
与 za 一样，zt0 的含义取决于使用它的指令，因此 LLDB 不尝试猜测这一点，默认显示为字节向量。

#### 表达式评估
zt0 的值及其是否激活将在表达式评估前保存，并在评估后恢复。

## 故障排除

#### 文件和行断点未被命中

首先，您必须确保源文件是用调试信息编译的。通常，这意味着在编译源文件时传递 `-g` 参数给编译器。

在设置实现源文件（例如 `.c`, `.cpp`, `.cxx`, `.m`, `.mm` 等）中的断点时，LLDB 默认只搜索文件名匹配的编译单元。如果您的代码做了一些复杂的事情，比如使用 `#include` 包含源文件：

```sh
$ cat foo.c
#include "bar.c"
#include "baz.c"
...
```

这将导致“bar.c”中的断点被内联到“foo.c”的编译单元中。如果您的代码这样做，或者您的构建系统以某种方式组合多个文件，以至于一个实现文件中的断点将被编译到另一个实现文件中，您需要告诉 LLDB 始终搜索内联断点位置。为此，可以在 `~/.lldbinit` 文件中添加以下行：

```sh
$ echo "settings set target.inline-breakpoint-strategy always" >> ~/.lldbinit
```

这告诉 LLDB 始终在所有编译单元中搜索断点位置，即使实现文件不匹配。由于内联函数通常在头文件中定义，并且经常导致多个断点具有匹配的头文件路径的源代码行信息，因此在头文件中设置断点总是会搜索所有编译单元。

如果您使用源文件的完整路径设置文件和行断点，例如在 macOS 上使用 Xcode 在其 GUI 中点击源视图的边栏设置断点，这个路径必须与调试信息中的完整路径匹配。如果路径不匹配，可能是由于传递了一个解析后的源文件路径，该路径与调试信息中的未解析路径不匹配，这可能会导致断点无法解析。尝试仅使用文件基名设置断点。

如果您使用的是 IDE 并在文件系统中移动了项目并重新构建，有时执行清理然后构建可以解决问题。这将修复某些 .o 文件在移动后没有重新构建的问题，因为构建文件夹中的 .o 文件可能仍包含旧的源位置的陈旧调试信息。

#### 如何检查是否有调试符号？

检查模块是否有任何编译单元（源文件）是检查模块中是否有调试信息的好方法：

```sh
(lldb) file /tmp/a.out
(lldb) image list
[  0] 71E5A649-8FEF-3887-9CED-D3EF8FC2FD6E 0x0000000100000000 /tmp/a.out
      /tmp/a.out.dSYM/Contents/Resources/DWARF/a.out
[  1] 6900F2BA-DB48-3B78-B668-58FC0CF6BCB8 0x00007fff5fc00000 /usr/lib/dyld
....
(lldb) script lldb.target.module['/tmp/a.out'].GetNumCompileUnits()
1
(lldb) script lldb.target.module['/usr/lib/dyld'].GetNumCompileUnits()
0
```

在上面，我们可以看到“/tmp/a.out”确实有一个编译单元，而“/usr/lib/dyld”没有。

我们还可以使用 Python 列出模块的所有编译单元的完整路径：

```sh
(lldb) script
Python Interactive Interpreter. To exit, type 'quit()', 'exit()' or Ctrl-D.
>>> m = lldb.target.module['a.out']
>>> for i in range(m.GetNumCompileUnits()):
...   cu = m.GetCompileUnitAtIndex(i).file.fullpath
/tmp/main.c
/tmp/foo.c
/tmp/bar.c
>>>
```

这可以帮助显示源文件的实际完整路径。有时 IDE 会通过完整路径设置断点，而路径与调试信息中的完整路径不匹配，这可能会导致 LLDB 无法解析断点。您可以使用 `breakpoint list` 命令和 `--verbose` 选项查看 IDE 设置的任何源文件和行断点的完整路径：

```sh
(lldb) breakpoint list --verbose
```

以上方法和步骤可以帮助您解决 LLDB 中断点无法命中的问题，并确保调试符号和路径信息的一致性。

## Links

This page contains links to external resources on how to use LLDB. Being listed on this page is not an endorsement.

### Blog Posts

### [Dancing in the Debugger — A Waltz with LLDB (2014)](https://www.objc.io/issues/19-debugging/lldb-debugging/)

A high level overview of LLDB with a focus on debugging Objective-C code.

### Videos

### [LLDB: Beyond “po” (2019)](https://developer.apple.com/videos/play/wwdc2019/429/)

LLDB is a powerful tool for exploring and debugging your app at runtime. Discover the various ways to display values in your app, how to format custom data types, and how to extend LLDB using your own Python 3 scripts.

### [Advanced Debugging with Xcode and LLDB (2018)](https://developer.apple.com/videos/play/wwdc2018/412/)

Discover advanced techniques, and tips and tricks for enhancing your Xcode debugging workflows. Learn how to take advantage of LLDB and custom breakpoints for more powerful debugging. Get the most out of Xcode’s view debugging tools to solve UI issues in your app more efficiently.

### [Debugging with LLDB (2012)](https://developer.apple.com/videos/play/wwdc2012/415/)

LLDB is the next-generation debugger for macOS and iOS. Get an introduction to using LLDB via the console interface and within Xcode’s graphical debugger. The team that created LLDB will demonstrate the latest features and improvements, helping you track down bugs more efficiently than ever before.

### [Migrating from GDB to LLDB (2011)](https://developer.apple.com/videos/play/wwdc2011/321/)

LLDB is the next-generation debugger for macOS and iOS. Discover why you’ll want to start using LLDB in your own development, get expert tips from the team that created LLDB, and see how it will help you track down bugs more efficiently than ever before.

### Books

### [Advanced Apple Debugging & Reverse Engineering (2018)](https://www.raywenderlich.com/books/advanced-apple-debugging-reverse-engineering/)

A book about using LLDB on Apple platforms.

### Extensions

### [facebook/chisel](https://github.com/facebook/chisel)

Chisel is a collection of LLDB commands to assist in the debugging of iOS apps.

### [DerekSelander/LLDB](https://github.com/DerekSelander/LLDB)

A collection of LLDB aliases/regexes and Python scripts.
