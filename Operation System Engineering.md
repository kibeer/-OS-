# Operation System Engineering (补档)

6.S081翻译文档： [1.1 课程内容简介 | MIT6.S081 (gitbook.io)](https://mit-public-courses-cn-translatio.gitbook.io/mit6-s081/lec01-introduction-and-examples/1.1-ke-cheng-jian-jie)

xv6 reference: [xv6: a simple, Unix-like teaching operating system (mit.edu)](https://pdos.csail.mit.edu/6.828/2021/xv6/book-riscv-rev2.pdf)

6.S081课程官网： [6.S081 / Fall 2021 (mit.edu)](https://pdos.csail.mit.edu/6.828/2021/tools.html)

南大OS: [02-应用视角的操作系统 (最小 Hello World；追踪系统调用；编译器与编译正确性) [南京大学2024操作系统]_哔哩哔哩_bilibili](https://www.bilibili.com/video/BV1dj421S7yK/?p=2&spm_id_from=pageDriver)

JYY 个人主页： [Yanyan's Wiki (jyywiki.cn)](https://jyywiki.cn/Letter.md)

## 绪论

### 一、应用视角的操作系统

###### 一切都是状态机：状态，初始状态，状态迁移

1. 状态：[StackFrame, StackFrame....] + 全局变量

2. 初始状态：一个StackFrame(main, argc, argv, <u>**PC=0**</u>)；全局变量设为初始值

3. 执行frames[-1].PC

###### 递归的实现：

递归的调用就是在栈顶增加一个StackFrame(..., PC=0)，而返回就是将栈顶帧弹出，所以程序实际上一直在**执行栈顶帧的语句**。非递归向递归转换就是在call自身的时候不断增加栈顶帧，在到达边界时返回就是弹出栈顶帧。

###### 编译器是什么？

状态机之间的翻译器：从一个状态机（C语言）翻译到另一个（汇编）

C 语言编译器在进行代码优化时，遵循的<u>基本准则</u>是在**不改变程序的语义** (即程序的行为和输出结果) 的前提下，提高程序的**执行效率**和/或**减少程序的资源消耗**

###### 编译优化三板斧（不改变语义的例子）

- **函数内联**：将函数调用替换为函数体本身的内容
- **常量传播**：在编译时计算常量表达式的值并替换
- **死代码消除**：删除永远不会被执行到的代码
- 一个程序没有syscall，无法终止(undefined behavior)--> 无法优化的例子

-->系统调用是使程序计算结果**可见**的唯一方法

###### C 代码中的**不可优化部分**

1. **External function calls**: 未知的代码可能包含syscall

2. 编译器提供的 **“不可优化” 标注**: volatile [load|store|inline assembly]

### 二、硬件视角的操作系统

**计算机系统的状态机**：状态（寄存器和内存）可以从初始状态（CPU Reset）通过PC读取指令进行工作。也就是说，只要在电路上固定某些代码，就可以控制计算机。

###### 固件Firmware：

是什么？固定在计算机系统里的代码

功能：运行程序前的计算机系统配置；**加载操作系统**

例子：Legacy BIOS (Basic I/O System)--> UEFI (Unified Extensible Firmware Interface)

###### <u>一点方法</u>

遇到困难能够找到解决方法？一定有别人也遇到了：创造自己的工具

**创造自己的工具**：

- **把复杂的构建过程分解成流程 + 可理解的单点**
- Get your hands dirty
- **总结：理解需要做什么-->构建实现的细节**

例子：很复杂的Makefile，可以用脚本/命令/现成工具做减法，让代码变得可读

### 三、数学视角的操作系统

###### 一些数学思想的启发

<u>暴力枚举</u>带来的启发：将程序的性质写成assertions来避免错误

<u>证明</u>带来的启发：容易**阅读**(self-explain) 和 **验证**(self-evident) 的代码是好代码

编译器的未来？检查代码的逻辑/性质正确性

**如果编译器能够检查代码的语法，逻辑...，AI能够快速生成代码，程序员还能做什么？**

## 并发

### 一、多线程编程

多线程就是单线程之间的切换：C语言线程有单独的栈帧，共享全局变量；汇编线程有“单独”的寄存器，共享内存资源。

例子：POSIX Threads API

###### 多线程的困难：

1. 单线程是确定的，多线程是**不确定的**：共享内存推翻了原子性假设，任何值都可能是其他程序写入/修改的（undefined behavior）。

2. 共享内存让编译器假设的顺序执行程序被推翻，类似于时间的相对性被破坏。

现代处理器也是编译器-->无关的指令可以乱序可以执行。

UNIX哲学的使用：管道小工具实现统计和可读性

### 二、并发控制：互斥

锁lock()：阻止并发的发生，原子化操作。

让当前状态机（线程）独占计算机-->阻止中断，没有别人可以打扰我工作; 但是NON-maskable Interrupts无法被阻止。

###### Peterson 算法

**假设**：load/store 是原子级操作 （以及只有两个线程争抢）

**实现**：AB举旗，公共区域放入对方标志，观察是否为空或者有自己的标志，完成放旗子。

可以解决的**例子**：BARRIOER，**内存屏障`**__sync_synchronize()` = Compiler Barrier** + x86: `mfence` /ARM: `dmb ish`/RISC-V: `fence rw, rw`

--> 但是barrier只能发现问题，无法证明没有问题[ godbolt.org](http://godbolt.org/)

可以通过**硬件实现**，提供lock原子性的暂停。

###### 自旋锁

**原理**： 一个全局status锁，lock()时获得锁，如果无法获得锁则retry（自旋），直到得到锁，工作完成后unlock()，归还锁。

**实现**：原子指令atomic_xchg 或 汇编指令 cmpxchg.

**死锁**：没有线程的lock()可以返回

**<u>如何解决死锁？</u>**

关闭中断：在获得锁之前关闭中断，在返回锁之后打开中断；在获得锁到解锁的过程中中断的状态不能发生变化-->导致死锁。在自旋锁中，保存中断的状态一般是per CPU的，指针lk的成员变量cpu提供当前获得锁的cpu。

**正确性原理**：在写代码之前思考：什么需要被实现？什么是错误的？

自旋锁的**问题**：无法保证Scalability（随着线程增多，performance也应该维持最高水准或仅有一点下降）。

自旋锁的**使用场景**：OSK的并发数据结构（短临界区）：因为临界区几乎不拥堵，迅速结束。但是Kernel有很多并发函数调用，如何解决？

观察：许多OSK object有read-mostly的特点（读写不平衡）。

**<u>解决方案：Read-Copy-Update</u>**：改写=Copy，每个读取对象的CPU都可以复制并看到自己的版本——CPU能在同一时间看到不同的版本。当所有的CPU都发生了一次线程切换时，旧版本对象就会被回收。

###### 应用程序实现互斥：

**问题**：空转线程太多；应用程序无法关闭中断。

**解决**：如何才能让应用程序关闭中断？

应用程序告诉OS需要锁，**将锁的实现放到OS**中，APP使用**syscall**告诉OS获取/解 锁（syscall(SYSCALL_lock), &lk）尝试获得lk，如果失败就切换到其他线程。unlock释放lk，如果有等待锁的线程就唤醒。

**库API：pthread Mutex lock实现**：

pthread_mutex_t lock;
pthread_mutex_init(&lock, NULL);
pthread_mutex_lock(&lock);
pthread_mutex_unlock(&lock);

### 三、并发控制：同步Synchronization

**同步**：控制并发，使得 “两个或两个以上随时间变化的量在变化过程中保持一定的相对关系”。通俗地说，就是让线程们在某一个时刻聚集在一起/做同一件事。

###### 第一个同步实现

线程有先后，先来先等待-->与自旋相似，等待同步条件达成

###### <u>生产者-消费者问题</u>：

Producer 和 Consumer**共享一个缓冲区**

- Producer (生产数据)：如果缓冲区有空位，放入；否则等待
- Consumer (消费数据)：如果缓冲区有数据，取走；否则等待

**解决方案**：

1st v:自旋: mutex_lock-->计算cond-->unlock-->check cond-->if false: retry

2nd v:lock-->check cond-->false:unlock&retry-->true:run&unlock

3rd v:放弃自旋，使用条件变量cv:

```
mutex_lock(&lk); 
while(!cond) {
    cond_wait(&cv, &lk);
} // 等待被唤醒，唤醒后check cond
assert(cond);
mutex_unlock(&lk);
```

```
cond_signal(&cv); // Wake up a (random) 符合条件的 thread 
cond_broadcast(&cv); // Wake up all 符合条件的 threads
```

正确使用**条件变量**： while 循环和 broadcast

- **总是在唤醒后再次检查同步条件**
- **总是唤醒所有潜在可能被唤醒的人**

###### 同步机制的应用：（如何让程序并行运算？/万能的并发控制方法）

1. 只要调度器 (生产者) 分配任务效率够高，算法就能并行

2. 让计算任务转换成**计算图**（有向无环图）：(u,v)∈E 表示 v 要用到前 u 的值。也就是为每个节点设置一个**条件变量**：v 能执行的同步条件：u→v 都已完成；u 完成后，signal 每个 u→v。

3. 使用<u>**线程池**</u>：将线程扔到线程池中，由调度器分配工作：

```
void T_worker() {//线程池中的线程：消费者
    while (1) { consume().run();}
} 
void T_scheduler() {//调度器：生产者
    while (!jobs.empty()) {
        for (auto j : jobs.find_ready()) { produce(j);}
    }
}
```

###### 互斥锁实现同步：

- 创建锁时，立即 “获得” 它 (总是成功)-->其他人想要获得时就会等待，此时 release 就实现了同步。
- Acquire-Release 实现**计算图**：为每一条边 e=(u,v) 分配一个**互斥锁**；初始时，全部处于锁定状态；对于一个节点，它需要获得所有入边的锁才能继续，可以直接计算的节点立即开始计算；计算完成后，释放所有出边对应的锁。
- Release as Synchronization：acq=等待token；rel=发出token。token~资源
- mutex lock是1对1的acq-rel，如果有n个token呢？

###### **信号量：**

信号量就是可以**管理n个token的互斥锁**，没有条件变量的自旋：

```
void P(sem_t *sem) {//取
    // P - prolaag // try + decrease/down/wait/acquire 
    atomic {
        wait_until(sem->count > 0) {
            sem->count--;//semaphores就是信号量
        }
    }
} 
void V(sem_t *sem) { //放
    // V - verhoog // increase/up/post/signal/release 
    atomic {
        sem->count++;
    }
}
```

**信号量的应用**：

1. 实现一次临时的 happens-before: A→B & 管理计数型资源

2. 生产者-消费者问题：producer：P(empty)-->V(fill); consumer：P(fill)-->V(empty)，相当于把token在empty和fill两个口袋间相互传递。

信号量与条件变量的选择：

1. 信号量好用，但是只在特定的使用场景（数值型资源为同步条件），当多个条件叠加，信号量的实现会变得复杂：release-wait 必须实现成 “原子操作”。

2. 条件变量万能，但实现比较繁琐，需要处理好同步条件&锁的实现。

### 四、并发编程应用

HPC：通常计算图容易**静态切分** (机器-线程两级任务分解)，

- 生产者-消费者解决一切： [MPI](https://hpc-tutorials.llnl.gov/mpi/) - “message passing libraries”, [OpenMP](https://www.openmp.org/) - “multi-platform shared-memory parallel programming (C/C++ and Fortran)”

###### <u>进程，线程，协程，GoRoutine</u>

**<u>进程</u>**：

**是什么?** 进程是操作系统中运行的一个**程序实例**，它具有**独立的地址空间和资源**。每个进程都有自己的一组指令、数据和上下文。操作系统通过调度算法为每个进程分配**CPU**时间片来执行。

**特点**：拥有自己的内存空间和系统资源。不同进程之间是隔离的，它们不能直接共享内存，需要通过**进程间通信IPC**来交换数据。

**应用场景**：进程适用于需要**隔离资源**的任务，如多个应用程序同时运行。每个进程拥有独立的内存空间，崩溃一个进程不会影响其他进程。

**<u>线程</u>**：

**是什么?** 线程是进程内的执行单元，一个进程可以包含**多个**线程。线程共享进程的**地址空间和资源**，可以**并发**执行，但是它们拥有**独立的栈空间和指令计数器**。线程之间可以**共享数据**，并且可以更高效地进行通信和同步。

**特点** ：线程是**进程内的执行单元**，与进程**共享同一内存空间**。不同线程可以直接访问同一进程内的共享数据，但也因此需要额外的**同步机制**来避免竞态条件。

**应用场景**：线程适用于需要**并发处理**的任务，如在**图形界面应用**中同时处理用户输入和界面更新。多个线程可以通过共享内存轻松交换信息，但也需要考虑线程安全性问题。

**<u>协程</u>**：

**是什么?** 协程是一种**用户级**的**轻量级线程**，也被称为纤程。它是一种在代码级别进行协作式多任务处理的机制。协程可以在执行过程中**主动让出执行权**给其他协程，从而实现协作式多任务调度和切换。

**特点**：协程是一**种用户态**的轻量级线程，由**程序员控制**其调度和切换。协程可以在**一个线程内部**通过特定的调度方式进行切换，从而实现高效的并发操作。

**应用场景**：协程适用于**高并发、高吞吐量**的任务，如**网络编程、IO密集型操作**等。协程的切换开销较小，允许更多任务同时执行，提高系统的资源利用率。

**<u>Goroutine</u>**：

**是什么？** 概念上是线程，实现是**线程和协程的混合体**。由于协程会在sleep(I/O)时失去并发能力，为了兼顾**多处理器并行和轻量级并发**而产生的。

**实现：**

- 每个 CPU 上有一个 Go Worker，运行协程

- 协程执行 blocking API (sleep, read)
  
  - 偷偷调用 non-blocking 的版本
  - 成功 → 立即继续执行

**Go中的同步与通信**：Channels in Go：类似UNIX的管道

###### AI并发：

Challenge: 既计算密集，又数据密集

解决：模型并行，数据并行，计算划分

**NVIDIA的解决方案**：

SIMD/**Single Instruction, Multiple Data**：Tensor 指令 (Tensor Core)：混合精度 A×B+C；单条指令完成 4×4×4 个乘法运算。

SIMT/**Single Instruction, Multiple Threads**：一个 PC，控制 32 个执行流同时执行（逻辑线程可以更多）；执行流有独立的寄存器：x,y,z 三个寄存器用于标记 “线程号”。

### 五、调试理论与实践

需求 → 设计 → 代码 (**Fault/bug**) → 执行 (**Error**) → 失败 (**Failure**)

###### self-check 的列表：

1. 是怎样的程序 (状态机) 在运行？
2. 我们遇到了怎样的 failure？
3. 我们能从状态机的运行中从易到难得到什么信息？
4. 如何二分检查这些信息和 error 之间的关联？

###### 一些常用的调试工具：

- `ssh`：使用 `-v` 选项检查日志

- `gcc`：使用 `-v` 选项打印各种过程

- `make`：使用 `-nB` 选项查看完整命令历史

- Profiler: `perf` - “采样” 状态机

- Trace: `strace` - 追踪系统调用

- GDB：状态机查看器 [GDB Cheat Sheet (jyywiki.cn)](https://jyywiki.cn/OS/manuals/gdb-cheat-sheet.pdf) /RTFM：[Top (Debugging with GDB) (sourceware.org)

###### 调试理论的推论（如何做）：

需求 → 设计 → 代码 → Fault ： **写好代码**：AI 是否能正确理解/维护你的代码？

Fault → Error：**做好测试**

Error → Failure：**多写断言**assertions

###### 两种并发bug：

**Deadlock死锁**：死锁产生的必要条件：

1. Mutual-exclusion - 一个口袋一个球，得到球才能继续
2. Wait-for - 得到球的人想要更多的球
3. No-preemption - 不能抢别人的持有的球
4. Circular-chain - 形成循环等待球的关系

如何避免死锁？

- 任意时刻系统中的锁都是有限的
- 给所有锁编号 (Lock Ordering)
  - 严格按照从小到大的顺序获得锁

**数据竞争Data Race:** **不同的线程**同时访问**同一内存**，且**至少有一个是写**。

- “内存” 可以是地址空间中的任何内存
  - 可以是全部变量
  - 可以是堆区分配的变量
  - 可以是栈
- “访问” 可以是任何代码
  - 可能发生在你的代码里
  - 可以发生在框架代码里
  - 可能是一行你没有读到过的汇编代码
  - 可能是一条 ret 指令

如何避免数据竞争？

**用锁保护好共享数据**：记得用锁&用对锁

###### 应对并发bugs：

**Sanitizers：**

- [AddressSanitizer](https://clang.llvm.org/docs/AddressSanitizer.html) (asan); [(paper)](https://www.usenix.org/conference/atc12/technical-sessions/presentation/serebryany): 非法内存访问
  - Buffer (heap/stack/global) overflow, use-after-free, use-after-return, double-free, ...;
  - 没有 [KASAN](https://www.kernel.org/doc/html/latest/dev-tools/kasan.html), Linux Kernel 的质量/安全性直接崩盘
- [ThreadSanitizer](https://clang.llvm.org/docs/UndefinedBehaviorSanitizer.html) (tsan): 数据竞争
  - KCSAN: [Concurrency bugs should fear the big bad data-race detector](https://lwn.net/Articles/816850/)
- [MemorySanitizer](https://clang.llvm.org/docs/MemorySanitizer.html) (msan), [UBSanitizer](https://clang.llvm.org/docs/UndefinedBehaviorSanitizer.html) (ubsan), ...

防御性编程：使用简单的实现完成大部分的检查。

## 虚拟化

### 一、OS上的进程

进程是一个状态机

###### fork()：linux中创建进程的方法

```
pid_t fork(void);
```

立刻复制状态机：包括**所有**信息的完整拷贝

- 复制失败返回 pid = -1

- 新创建进程返回 pid = 0

- 执行 fork 的进程返回子进程的进程号(pid)

进程总有 “被创建” 的关系

- 因此总能找到 “父子关系”
- 因此有了进程树 (pstree)

###### execve()：linux中运行可执行程序的方法

```
int execve(const char *filename, char * const argv[], char * const envp[]);
```

将当前进程**重置**成一个可执行文件描述状态机的初始状态

- 执行名为 `filename` 的程序
- 允许对新状态机设置参数 `argv` (v) 和环境变量 `envp` (e)
  - 刚好对应了 `main()` 的参数
- execve 是唯一能够 “执行程序” 的系统调用
- 创建新状态机的方式：fork() + execve()

###### _exit：linux中销毁进程的方法

```
void _exit(int status);
```

- 立即摧毁状态机，允许有一个返回值

- 子进程终止会通知父进程

- 几种写法：
  
  1. exit (0): 会调用atexit
  
  2. _exit(0): 执行 “exit_group” 系统调用终止整个进程 (所有线程
  
  3. syscall(SYS_exit, 0): 执行 “exit” 系统调用终止当前线程

###### 环境变量：“应用程序执行的环境”

- 使用 `env` 命令查看
  - `PATH`: 可执行文件搜索路径-->按照顺序执行文件file
  - `PWD`: 当前路径
  - `HOME`: home 目录
  - `DISPLAY`: 图形输出
  - `PS1`: shell 的提示符
- `export`: 告诉 shell 在创建子进程时设置环境变量

###### 进程的地址空间

`(sudo cat) /proc/[pid]/maps`：查询进程的地址空间

`pmap (-x) pid`：查询pid的进程

- 进程地址空间中的每一段
  
  - 地址 (范围) 和权限 (rwxsp)
  
  - 对应的文件: offset, dev, inode, pathname
  
  - 和 readelf (`-l`) 里的信息互相验证
    
    不陷入内核的系统调用：time (2) 和 gettimeofday (2)

- 地址空间 = 带访问权限的内存段
  
  - 不存在 (不可访问)
  - 不存在 (可读/写/执行)

- 管理 = **增加/删除/修改一段可访问的内存**

###### Memory Map 系统调用：

**在状态机状态上增加/删除/修改一段可访问的内存**

```
//映射
void *mmap(void *addr, size_t length, int prot, int flags, 
            int fd, off_t offset);
int munmap(void *addr, size_t length); 
// 修改映射权限 
int mprotect(void *addr, size_t length, int prot);
```

- MAP_ANONYMOUS: 匿名 (申请) 内存

- fd: 把文件 “搬到” 进程地址空间中 (例子：加载器)

- 瞬间完成内存分配（申请大量内存空间）
  
  - mmap/munmap 为 malloc/free 提供了机制
  - libc 的大 malloc 会直接调用一次 mmap 实现

### 二、系统调用和UNIX Shell和C标准库

###### 操作系统对象：

1. 进程和地址空间（内存pages）

2. 文件：有名字的对象

3. 字节流（终端）或字节序列（普通文件）

UNIX思想：**一切都是文件/对象**-->文件描述符（Windows handle）

**文件描述符**：

指向OS对象的指针（open, close, read/write, lseek指针内赋值/运算, dup赋值）

**管道**：

1. 一个特殊的 “文件” (流)，由读者/写者共享：读口&写口

2. firstobj | secondobj -->UNIX管道

3. IPC的其中一种：interprocess communication

4. 匿名管道：int pipe(int pipefd[2]);
- 返回两个文件描述符
- 进程同时拥有读口和写口

系统调用与环境的封装

###### 输入/输出

**Standard I/O**: [stdio.h](https://www.cplusplus.com/reference/cstdio/)

- `FILE *` 背后其实是一个文件描述符
- 封装了文件描述符上的系统调用 (fseek, fgetpos, ftell, feof, ...)

**The printf() family**

```
printf("x = %d\n", 1); 
fprintf(stdout, "x = %d\n", 1); 
snprintf(buf, sizeof(buf), "x = %d\n", 1);
```

###### 内存分配：操作系统的机制

1. 大段内存，要多少有多少：用 MAP_ANONYMOUS 申请

2. 操作系统**不支持**分配一小段内存：malloc() 和 free()

**实现高效的 malloc/free：**

1. **脱离 workload 做优化就是耍流氓**

2. 通常不考虑 adversarial worst case

3. malloc()：**越小的对象创建/分配越频繁**，小对象分配/回收的 **scalability** 是主要瓶颈。-->设置两套系统：**Fast:** 性能极好、并行度极高、覆盖大部分情况; **Slow:** 不在乎那么快,但把困难的事情做好

4. **malloc**：fast path：浪费一点空间，但**使所有 CPU 都能并行地申请内存**：线程事先瓜分buffer，默认从自己的空间中分配，不够就从全局的池子里借。

5. **小内存**：Segregated List (Slab): 每个 slab 里的每个对象都**一样大**
   
   - 每个线程拥有每个对象大小的 slab
   - fast path → 立即在线程**本地**分配完成; slow path → pgalloc()
   - **分配**：一个内存页被分配成大小是 2k 的分配单元；每个分配单元里塞个指针，就是 free list 了；`(uintptr_t)p & 0xfff` 就可以得到对应的 slab
   - **回收**：直接归还到 slab 中；需要 per-slab 锁 (小心数据竞争)

6. **大内存**：一把大锁保平安

### 三、可执行文件和加载&动态链接和加载

###### 可执行文件：

- 一个操作系统中的**对象** (文件)
- 一个**字节序列** (我们可以把它当字符串编辑)
- 一个描述了状态机初始状态的**数据结构**

###### ELF文件：Executable and Linkable Format

1. 文件三要素：**代码**（字节序列）；**符号**（标记当前位置）；重**定位**（链接时才能确定的数值） -->包含了指针信息

2. ELF**工具集**（**GNU binutils**）：exec (加载)、objdump/readfle/nm (显示)、cc/as (编译)、ld (链接)

3. **生成**可执行文件：
   
   a.**预编译**(源代码.c->源代码.i字符串替换)&**编译**(源代码.i->汇编代码.s生成带标注的指令序列)
   
   b. **汇编**(汇编文件.s->目标文件.o) 文件=sections->三要素
   
   c. **静态链接**(多个目标文件.o->可执行文件a.out)：合并所有的sections，把 sections “平铺” 成字节序列；确定所有符号的位置；解析全部重定位

UNIX脚本track：开头的 **#!解释器 arg** 表示用解释器执行并输入arg参数

4. 加载ELF：将**多段**字节序列复制到地址空间中： **分别**赋予可读/可写/可执行权限；然后跳转到指定的 entry (默认为 _start) 执行

###### **拆解应用程序：**

方案 1: libc.o

- 在加载时完成重定位
  - 加载 = 静态链接
  - 省了磁盘空间，但没省内存
  - 致命缺点：时间 (链接需要解析很多不会用到的符号)

方案 2: libc.so (shared object)

- 编译器生成**位置无关代码**
  - 加载 = mmap
  - 但函数调用时需要额外一次查表
- 好处：多个进程映射**同一个** libc.so，内存中只需要一个副本

###### 动态加载：A Layer of Indirection

1. 编译时，动态链接库调用 = 查表

2. 链接时，收集所有符号，**“生成” 符号信息和相关代码**

3. GOT (Global Offset Table)：对于每个需要动态解析的符号，GOT 中都有一个位置

**动态链接的主要功能**：

实现**代码**的动态链接和加载

怎么决定到底要不要查表？（先编译后链接 的问题）编译器A：全部查表跳转->性能太差；编译器B：全部直接跳转->有时无法跳跃大跨度

-->PLT (Procedure Linkage Table)：如果符号在链接时发现来自动态加载，就在 a.out 里 **“合成” 一段小代码**，全部跳转是唯一选择。



## 内核

### 一、系统调用、中断和上下文切换（线程内）

**syscall**: 跳转到（OS）并获得无限的权力

- 进程的内存被 “拆散”，并且被 Page Table 重组了（分页），需要使用寄存器中的CR3指向物理地址的数据结构，读取虚拟地址中的指令。
- 操作系统内核：配置内存映射

**中断**：强制插入的syscall：立即去做oskernel的代码

1. 中断给了OS最高权限：想开就开；而应用程序没有中断

2. OSkernel会做什么？先封存寄存器，然后执行OS的代码->完成后恢复寄存器，执行sysret

**上下文切换**：设置current context，保存寄存器现场 & 恢复寄存器现场

context：数据结构，表示中断时当前寄存器们的值被保存的内存中的空位

### 二、进程的实现

进程=只能看见虚拟地址空间的线程

###### **分页**：

物理内存分成一个一个的**Pages**，每（一般是**4kb**）页有一个**PPN**（physical page number），进程在获取虚拟地址时，将VPN（virtual page number）**映射**（f函数）到PPN，然后加上后面位上的**offset**就得到了真实的物理地址。

**VPN到PPN的映射：**
内存部分：**Radix Tree**（32位->4byte * 1024=4kb page：**1024叉树**;64位是512叉树），**x级页表**->x * 10 +12->x86_64：PML4/PML5

处理器部分：CR3（PDBR）：指向Radix Tree根节点的指针

- Translation-Lookaside Buffer (TLB): 现代CPU使用一小块关联内存，用来缓存最近访问的虚拟页的PTE，还包含了访问特定内存是否安全的信息(例如，预取指令)

**非常大的内存分配也不会使系统崩溃原因：**

- **Kernel Samepage Merging**：将指向相同内存的指针合并到一块内存

- **Demand paging**：允许程序的**一部分**或全部被加载到物理内存中，而不是一次性将所有内容都加载进来。当程序访问未加载到内存的页面时（Page Fault），操作系统会根据需要通过查找页表将相应的页面加载到内存中，从而满足程序的执行需求。

- **COW fork()** : 子进程和父进程共享数据段、堆和代码段，但内核会将这些共享区域的访问权限设置为只读。如果任何一个进程尝试修改这些共享区域，内核会为该进程创建该区域的一个私有副本

###### **<u>写时复制（Copy-On-Write, COW）</u>**：

**是什么？** 当多个调用者（callers）请求**相同资源**（如内存或磁盘上的数据存储）时，他们会**共享相同的指针**指向同一资源。只有当某个调用者尝试修改资源内容时，系统才会为该调用者创建一份**专用副本（private copy）**。这种策略对其他调用者是透明的，只有在修改资源时才会创建副本，因此在调用者仅进行读取操作时可以共享同一份资源。

**原理**：**写时拷贝技术实际上是运用了一个 “引用计数” 的概念来实现的。在开辟的空间中多维护四个字节来存储引用计数.** 当多开辟一份空间时，引用计数+1，如果释放空间，计数-1，引用计数变为 0 时，真正的释放空间。如果有修改或写的操作，也让原空间的引用计数-1，并且真正开辟新的空间。为1时，最后一份只读副本变成可写的。

### 三、处理器调度

处理器调度：当中断时发生上下文切换：机制

```
Context *on_interrupt(Event ev, Context *ctx) {
    current->context = *ctx; 
    // Can return any valid 
    Context* current = schedule(); 
    return &current->context;
}
```

操作系统具有在中断后选择**任何**进程执行的权利：资源调度：策略

UNIX Niceness：-20 .. 19 的整数，越 nice 越让别人得到 CPU

公平分享 CPU：Round-Robin

Complete Fair Scheduling(CFS)：（Linux的策略）

- 为每个进程记录精确的运行时间

- 中断/异常发生后，切换到运行时间最少的进程执行
  
  - 下次中断/异常后，当前进程的可能就不是最小的了

- 每人执行 1ms，但好人的钟快一些，坏人的钟慢一些：vruntime



## 持久化
