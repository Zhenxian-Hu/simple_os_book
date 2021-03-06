## 创建并执行内核线程

### 实验目标

ucore在lab2完成了内存管理。一个程序如果要加载到内存中运行，通过ucore的内存管理就可以分配合适的空间了。接下来就需要考虑如何使用CPU来“并发”执行多个程序。

操作系统把一个程序加载到内存中运行，这个运行的程序会经历从“出生”到“死亡”的整个“生命”历程。这个运行程序的整个执行过程就是进程。为了记录、描述和管理程序执行的动态变化过程，需要有一个数据结构，这个就是进程控制块。一个进程与一个进程控制块一一对应。为此，ucore就需要建立合适的进程控制块数据结构，并基于进程控制块来完成对进程的管理。

### proj10概述

#### 实现描述

project10是lab3的第一个项目，它基于lab2的最后一个项目proj9.2。主要就是扩展了进程控制块的数据结构，并基于进程控制块实现了对进程的初步管理。并通过创建两个内核线程idleproc和init_main来实现了对CPU的分时使用。

#### 项目组成

    proj10
    ├──  ……
    │   ├── process
    │   │   ├── entry.S
    │   │   ├── proc.c
    │   │   ├── proc.h
    │   │   └── switch.S
    │   ├── schedule
    │   │   ├── sched.c
    │   │   └── sched.h
    │   ├── sync
    │   │   └── sync.h
    │   └── trap
    │       ├── trap.c
    │       ├── ……
    │       └── trapentry.S
    ├── libs
    │   ├── ……
    │   └── unistd.h
    └── ……
    14 directories, 77 files

相对与proj9.2，proj10增加了7个文件，修改了相对重要的3个文件。主要修改和增加的文件如下：

- process/proc.[ch]：实现了进程控制块的定义和基于进程控制块的各种进程管理函数，实现了对进程生命周期管理的绝大部分功能。
- process/entry.S：内核线程的起始入口处和结束处理（通过调用do\_exit完成实际结束工作）。
- process/switch.S：实现了进程上下文（context）切换的函数switch\_to，由于与硬件相关，所以直接采用汇编实现。
- schedule/sched.[ch]：实现了一个先进先出（First In First Out）策略的进程调度。
- trap/trap.c,trapentry.S：
- unistd.h：定义了一系列系统调用号，为后续用户进程访问内核功能做准备，这里暂时用不上。

#### 编译运行

编译并运行proj10的命令如下：

    make
    make qemu
  
则可以得到如下显示界面

    thuos:~/oscourse/ucore/i386/lab3_process/proj10$ make qemu
    (THU.CST) os is loading ...

    Special kernel symbols:
    entry  0xc010002c (phys)
    etext  0xc0113bd7 (phys)
    edata  0xc013fab0 (phys)
    end    0xc0144e74 (phys)
    Kernel executable memory footprint: 276KB
    ……
    check_mm_shm_swap: step2, dup_mmap ok.
    check_mm_shm_swap() succeeded.
    ++ setup timer interrupts
    this initproc, pid = 1, name = "init"
    To U: "Hello world!!".
    To U: "en.., Bye, Bye. :)"
    kernel panic at kern/process/proc.c:317:
    process exit!!.

    Welcome to the kernel debug monitor!!
    Type 'help' for a list of commands.
    K>
    从上图可以看到，proj10创建了一个内核线程init_main，然后启动内核线程initproc运行，此线程调用init_main函数完成了其主要的工作，即输出了一些信息：
    this initproc, pid = 1, name = "init"
    To U: "Hello world!!".
    To U: "en.., Bye, Bye. :)"
    然后就“死亡”了，所占用的资源被回收。由于现在没有其他值得执行的线程，所以ucore就进入到kernel debug monitor了。到底是在哪里判断并进入monitor的呢？我们只需在“K>”提示符后面输入backtrace，就可得到如下输出：
    K> backtrace
    ebp:0xc7eb0ee8 eip:0xc0101c86 args:0x00000000 0x00000000 0xc7eb0f68 0xc0102489 
        kern/debug/kdebug.c:298: print_stackframe+21
    ebp:0xc7eb0ef8 eip:0xc010258b args:0x00000000 0xc7eb0f1c 0x00000000 0x00000000 
        kern/debug/monitor.c:147: mon_backtrace+10
    ebp:0xc7eb0f68 eip:0xc0102489 args:0xc013fac0 0x00000000 0xc011784a 0xc7eb0fdc 
        kern/debug/monitor.c:93: runcmd+134
    ebp:0xc7eb0f98 eip:0xc010250d args:0x00000000 0xc7eb0fdc 0x0000013d 0xc7eb0fcc 
        kern/debug/monitor.c:114: monitor+91
    ebp:0xc7eb0fc8 eip:0xc0102b2f args:0xc0117825 0x0000013d 0xc0117839 0xc0144e44 
        kern/debug/panic.c:30: __panic+106
    ebp:0xc7eb0fe8 eip:0xc0112bc7 args:0x00000000 0xc01178b8 0x00000000 0x00000010 
        kern/process/proc.c:317: do_exit+33
    K>

这里可以清楚的看到，在proc.c的317行（do_exit函数内）调用了panic函数，导致进入了monitor。idleproc 和 initproc 是 ucore 里面两个特殊的内核线程，它们是不允许退出的，所以 do_exit 里面直接调用 panic 了，对于其它普通的进程而言，do_exit 实际上还有很多工作要做，后续章节会进一步讲到。上述执行过程其实包含了对进程整个生命周期的管理。下面我们将从原理和实现两个方面对此进行进一步阐述。