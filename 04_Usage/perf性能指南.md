# Perforever

在功能上，perf很强大，可以对众多的软硬件事件采样，还能采集出跟踪点（trace points）的信息（比如系统调用、TCP/IP事件和文件系统操作。perf的代码和Linux内核代码放在一起，是内核级的工具。perf是在Linux上做剖析分析的首选工具。

```cmd
usage: perf [--version] [--help] [OPTIONS] COMMAND [ARGS]
 
最常用的perf指令:
   annotate        用perf.data文件中找到的build-ids的对象文件创建档案
   archive         Create archive with object files with build-ids found in perf.data file
   bench           对系统调度、内存访问、epoll、Futex等进行压力测试
   buildid-cache   管理build-id缓存
   buildid-list    List the buildids in a perf.data file
   c2c             Shared Data C2C/HITM Analyzer.
   config          读取或设置配置文件中的变量
   data            数据文件相关处理
   diff            读取perf.data文件并显示差分曲线
   evlist          List the event names in a perf.data file
   ftrace          simple wrapper for kernel's ftrace functionality
   inject          Filter to augment the events stream with additional information
   kallsyms        Searches running kernel for symbols
   kmem            Tool to trace/measure kernel memory properties
   kvm             Tool to trace/measure kvm guest os
   list            列出所有象征性的事件类型☆☆☆【忘了就用这个】
   lock            分析锁事件
   mem             分析内存访问
   record          将所有的分析记录写进perf.data【重点】
   report          读取perf.data（由perf记录创建）并显示概况【重点】
   sched           跟踪/测量调度器属性（延迟）的工具
   script          Read perf.data (created by perf record) and display trace output
   stat            Run a command and gather performance counter statistics
   test            Runs sanity tests.
   timechart       Tool to visualize total system behavior during a workload
   top             系统分析工具
   version         display the version of perf binary
   probe           定义新的动态跟踪点
   trace           strace inspired tool
```

## list

使用 `perf list` 【行数达到3800+】仅列举重要的

```shell
  branch-instructions OR branches                    [Hardware event]
  branch-misses                                      [Hardware event]
  bus-cycles                                         [Hardware event]
  cache-misses                                       [Hardware event]
  cache-references                                   [Hardware event]
  cpu-cycles OR cycles                               [Hardware event]
  instructions                                       [Hardware event]
  ref-cycles                                         [Hardware event]
  alignment-faults                                   [Software event]
  bpf-output                                         [Software event]
  cgroup-switches                                    [Software event]
  context-switches OR cs                             [Software event]
  cpu-clock                                          [Software event]
  cpu-migrations OR migrations                       [Software event]
  dummy                                              [Software event]
  emulation-faults                                   [Software event]
  major-faults                                       [Software event]
  minor-faults                                       [Software event]
  page-faults OR faults                              [Software event]
  task-clock                                         [Software event]
  duration_time                                      [Tool event]
  L1-dcache-load-misses                              [Hardware cache event]
  L1-dcache-loads                                    [Hardware cache event]
  L1-dcache-stores                                   [Hardware cache event]
  L1-icache-load-misses                              [Hardware cache event]
  LLC-load-misses                                    [Hardware cache event]
  LLC-loads                                          [Hardware cache event]
  LLC-store-misses                                   [Hardware cache event]
  LLC-stores                                         [Hardware cache event]
  branch-load-misses                                 [Hardware cache event]
  branch-loads                                       [Hardware cache event]
  dTLB-load-misses                                   [Hardware cache event]
  dTLB-loads                                         [Hardware cache event]
  dTLB-store-misses                                  [Hardware cache event]
  dTLB-stores                                        [Hardware cache event]
  iTLB-load-misses                                   [Hardware cache event]
  iTLB-loads                                         [Hardware cache event]
  node-load-misses                                   [Hardware cache event]
  node-loads                                         [Hardware cache event]
  node-store-misses                                  [Hardware cache event]
  node-stores                                        [Hardware cache event]
```

## stat

使用stat来采集程序的运行时间和CPU开销，perf stat所支持的主要参数如下:

```text
-a, --all-cpus        system-wide collection from all CPUs
-A, --no-aggr         disable CPU count aggregation
-B, --big-num         print large numbers with thousands' separators
-C, --cpu <cpu>       list of cpus to monitor in system-wide
-D, --delay <n>       ms to wait before starting measurement after program start (-1: start with events disabled)
-d, --detailed        detailed run - start a lot of events
-e, --event <event>   event selector. use 'perf list' to list available events
-G, --cgroup <name>   monitor event in cgroup name only
-g, --group           put the counters into a counter group
-I, --interval-print <n>
                      print counts at regular interval in ms (overhead is possible for values <= 100ms)
-i, --no-inherit      child tasks do not inherit counters
-M, --metrics <metric/metric group list>
                      monitor specified metrics or metric groups (separated by ,)
-n, --null            null run - dont start any counters
-o, --output <file>   output file name
-p, --pid <pid>       stat events on existing process id
-r, --repeat <n>      repeat command and print average + stddev (max: 100, forever: 0)
-S, --sync            call sync() before starting a run
-t, --tid <tid>       stat events on existing thread id
-T, --transaction     hardware transaction statistics
-v, --verbose         be more verbose (show counter open errors, etc)
```

举例：`perf stat -p [Pid]` 使用top可以查看ProcessId

得到诸如以下的结果：

```text
Performance counter stats for process id '997'（临时的进程id）:
 
          22624.22 msec task-clock                #    0.188 CPUs utilized
              1225      context-switches          #    0.054 K/sec
                 1      cpu-migrations            #    0.000 K/sec
                 0      page-faults               #    0.000 K/sec
       39516466339      cycles                    #    1.747 GHz
       23012315521      instructions              #    0.58  insn per cycle
        3381064757      branches                  #  149.444 M/sec
         256850857      branch-misses             #    7.60% of all branches
 
     120.484878500 seconds time elapsed
```

## record

剖析采样可以帮助我们采集到程序运行的特征，而且剖析精度非常高，可以定位到具体的代码行和指令块。

```text
-a, --all-cpus        system-wide collection from all CPUs
-b, --branch-any      sample any taken branches
-B, --no-buildid      do not collect buildids in perf.data
-c, --count <n>       event period to sample
-C, --cpu <cpu>       list of cpus to monitor
-d, --data            Record the sample addresses
-D, --delay <n>       ms to wait before starting measurement after program start (-1: start with events disabled)
-e, --event <event>   event selector. use 'perf list' to list available events
-F, --freq <freq or 'max'>
                      profile at this frequency
-g                    enables call-graph recording
-G, --cgroup <name>   monitor event in cgroup name only
-I, --intr-regs[=<any register>]
                      sample selected machine registers on interrupt, use '-I?' to list register names
-i, --no-inherit      child tasks do not inherit counters
-j, --branch-filter <branch filter mask>
                      branch stack filter modes
-k, --clockid <clockid>
                      clockid to use for events, see clock_gettime()
-m, --mmap-pages <pages[,pages]>
                      number of mmap data pages and AUX area tracing mmap pages
-N, --no-buildid-cache
                      do not update the buildid cache
-n, --no-samples      don't sample
-o, --output <file>   output file name
-P, --period          Record the sample period
-p, --pid <pid>       record events on existing process id
-q, --quiet           don't print any message
-R, --raw-samples     collect raw sample records from all opened counters
-r, --realtime <n>    collect data with this RT SCHED_FIFO priority
-S, --snapshot[=<opts>]
                      AUX area tracing Snapshot Mode
-s, --stat            per thread counts
-t, --tid <tid>       record events on existing thread id
-T, --timestamp       Record the sample timestamps
-u, --uid <user>      user to profile
-v, --verbose         be more verbose (show counter open errors, etc)
```

> 例如：通过“-F 999”选项，我把采样频率设置为999Hz，每秒采样999次。

```text
perf record -F 999 -p 997
然后perf会将记录的数据存储在perf.data中
```

## report

- 采集完数据，我们就可以通过perf report命令寻找采样中的性能瓶颈。

```haskell
Samples: 21K of event 'cycles', Event count (approx.): 38100133435
Overhead  Command  Shared Object      Symbol
  99.99%  test     test               [.] print                                                               
   0.00%  test     [kernel.kallsyms]  [k] update_sd_lb_stats.constprop.0                                       
   0.00%  test     [kernel.kallsyms]  [k] _raw_spin_unlock_irq                                                 
   0.00%  test     [kernel.kallsyms]  [k] shift_arg_pages                                                     
   0.00%  perf     [kernel.kallsyms]  [k] perf_event_exec
```

