---
title: 初探iOS源码调试原理
date: 2021-01-25 15:58:38
tags: iOS

---

# 初探iOS源码调试原理

## 一、问题从开发中常见的调试场景开始

*   打开IDE在某个方法中设置断点，切换到其他源文件后运行程序

*   运行到断点时，程序停止，IDE显示对应文件的源码

*   能够输出的函数堆栈中变量值

这是日常开发中常见的一幕，但调试器是如何做到这一切的？

*   调试器是如何找到对应的源码文件？

*   调试器是如何找到文件中对应的函数来设置断点？

*   调试器是如何在函数内找到并输出变量的值？

## 二、调试器

大部分类 UnixiOS 上的调试器是基于 `ptrace` 系统调用

```
#include <sys/ptrace.h> 
int ptrace(int request, int pid, int addr, int data);

// request: 请求ptrace执行的操作
// pid:     目标进程的id
// addr:    目标进程的地址值
// data:    读取或写入的数据
```

### 常见 ptrace 请求

| 请求              | 作用                                                       |
| ----------------- | ---------------------------------------------------------- |
| PTRACE_TRACEME    | 该进程被其父进程所跟踪，发送给该进程的信号都将通知其父进程 |
| PTRACE_KILL       | 杀掉子进程，使它退出                                       |
| PTRACE_SINGLESTEP | 设置单步执行标志                                           |
| PTRACE_ATTACH     | 附加到指定pid进程                                          |
| PTRACE_GETREGS    | 读取寄存器                                                 |
| PTRACE_PEEKTEXT   | 从内存地址中读取一个字节数据，内存地址由addr给出           |
| PTRACE_POKETEXT   | 往内存地址中写入一个字节数据，内存地址由addr给出           |

``` c
// 获取寄存器状态
struct user_regs_struct regs;
ptrace(PTRACE_GETREGS, child_pid, 0, &regs);

// 获取rip寄存器内容
unsigned instruction = ptrace(PTRACE_PEEKTEXT, child_pid, regs.rip, 0);
procmsg("IP = 0x%08x.  instr = 0x%08x", regs.rip, instruction);

// 示例输出
// IP = 0x38182743.  instr = 0x0fc08944
```

调试器从 `ptrace` 获取到的信息有：

*   regs，部分寄存器信息，如 IP 指令寄存器的地址

*   instruction，当前CPU执行的指令内容

看起来十分有限而且都是十分底层的信息，但我们平时使用的调试器能给我们提供一些更直观和更丰富的信息：

*   当前指令的源码文件、行号

*   根据函数、行号进行断点

*   读取当前栈帧(stack frame|activation record) 下的布局变量信息

因此，调试器必然需要以某种方式将这些基础信息转化为更友好的信息。

**DWARF 就是调试器与源码之间的桥梁。**

## 三、引入DWARF

DWARF（Debugging With Attributed Record Formats）是一种广泛使用的标准化调试数据格式，通常用于源码级别调试，主要服务于调试器。

*   DWARF 第一版发布于 1992 年,主要是为UNIX下的调试器提供必要的调试信息，例如PC地址对应的文件名及行号等信息，以方便源码级调试。到90年代末期，DWARF取代 stabs 成为 Linux上的默认配置。

*   其包含足够的信息以供调试器完成特定的一些功能, 例如显示当前栈帧(Stack Frame)下的局部变量, 尝试修改一些变量, 直接跳到函数末尾等

*   有足够的可扩展性，可为多种语言提供调试信息: 如: Ada, C, C++, Fortran, Java, Objective C, Go, Python, Haskell …

目前 DWARF 已经在类UNIX系统中逐步替换 stabs(symbol table strings)成为主流的调试信息格式。

[DWARF发展史](https://www.notion.so/88bc5f6c84f44f18b16bf2e6ed29498f)
[DWARF竞品](https://www.notion.so/59422db137d0499084b5dd4e5c1798b1)

## 四、认识 DWARF

借助一段简单的 C++ 程序来探索 DWARF
```c
// foo.c
int foo(int x, int y) {
  int num = x + y;
  return num * multiplier;
}
int main() {
  int ans = foo(1, 2);
  return 0;
}
```

如果用一段自己的语言来描述这个源码文件：

*   这是一个 c++ 源码文件名字叫 foo.cpp

*   定义了一个 `foo` 函数，接收两个 `int` 类型参数，返回值为 `int`

*   `foo` 函数内定义了一个 `int` 类型的变量叫 `num`

*   ... ...

### 使用 GCC 生成 DWARF 信息

先编译源码，看看在没有DWARF信息情况下调试是什么样的
``` bash
clang -O0 foo.c -o foo
```

生成可执行文件后，使用 `lldb` 调试器进行调试

``` bash
$ lldb foo
(lldb) target create "foo"
Current executable set to '/Users/lindubo/Desktop/DebugDemo/foo' (x86_64).
(lldb) b foo
Breakpoint 1: where = foo`foo, address = 0x0000000100000f60
(lldb) run
Process 34037 launched: '/Users/lindubo/Desktop/DebugDemo/foo' (x86_64)
Process 34037 stopped
* thread #1, queue = 'com.apple.main-thread', stop reason = breakpoint 1.1
    frame #0: 0x0000000100000f60 foo`foo
foo`foo:
->  0x100000f60 <+0>: pushq  %rbp
    0x100000f61 <+1>: movq   %rsp, %rbp
    0x100000f64 <+4>: movl   %edi, -0x4(%rbp)
    0x100000f67 <+7>: movl   %esi, -0x8(%rbp)
Target 0: (foo) stopped.
```

对于 GCC 及 CLang 编译器, 使用参数 `-gdwarf-4` 即可生成 DWARF4 调试信息
``` bash
clang -O0 -gdwarf-4 foo.c -o foo
```

编译后目录下可执行文件，还多了 foo.dSYM

```
$ ll
total 24
-rwxr-xr-x  1 lindubo  ECC\Domain Users   4.5K 10 26 21:00 foo
-rw-r--r--@ 1 lindubo  ECC\Domain Users   109B 10 26 21:00 foo.c
drwxr-xr-x  3 lindubo  ECC\Domain Users    96B 10 26 21:00 foo.dSYM
```

进行调试
```
$ lldb foo
(lldb) target create "foo"
Current executable set to '/Users/lindubo/Desktop/DebugDemo/foo' (x86_64).
(lldb) b foo
Breakpoint 1: where = foo`foo + 10 at foo.c:2:13, address = 0x0000000100000f6a
(lldb) run
Process 93727 launched: '/Users/lindubo/Desktop/DebugDemo/foo' (x86_64)
Process 93727 stopped
* thread #1, queue = 'com.apple.main-thread', stop reason = breakpoint 1.1
    frame #0: 0x0000000100000f6a foo`foo(x=1, y=2) at foo.c:2:13
   1   	int foo(int x, int y) {
-> 2   	  int num = x + y;
   3   	  return num;
   4   	}
   5   	int main() {
   6   	  int ans = foo(1, 2);
   7   	  return 0;
Target 0: (foo) stopped.
```

当没有 DWARF 信息时，断点是停留在汇编层面，可读性非常差，而在有 DWARF 信息时将停留在源码层面。

### 查看 DWARF 信息

使用 file 命令查看 DWARF 的文件描述
```
$ cd foo.dSYM/Contents/Resources/DWARF
$ file foo
foo: Mach-O 64-bit dSYM companion file x86_64
```

可以看到它是一个 `mach-o` 文件，既然是 Mach-O 文件, 可以使用 `machOView` 或者 `size` 命令查看可执行文件包含的 Segment 和 Section：

```
$ size -m -l -x foo
Segment __PAGEZERO: 0x100000000 (vmaddr 0x0 fileoff 0)
Segment __TEXT: 0x1000 (vmaddr 0x100000000 fileoff 0)
	Section __text: 0x4b (addr 0x100000f60 offset 0)
	Section __unwind_info: 0x48 (addr 0x100000fac offset 0)
	total 0x93
Segment __LINKEDIT: 0x1000 (vmaddr 0x100001000 fileoff 4096)
Segment __DWARF: 0x1000 (vmaddr 0x100002000 fileoff 8192)
	Section __debug_line: 0x69 (addr 0x100002000 offset 8192)
	Section __debug_pubnames: 0x23 (addr 0x100002069 offset 8297)
	Section __debug_pubtypes: 0x1a (addr 0x10000208c offset 8332)
	Section __debug_aranges: 0x40 (addr 0x1000020a6 offset 8358)
	Section __debug_info: 0x9e (addr 0x1000020e6 offset 8422)
	Section __debug_abbrev: 0x69 (addr 0x100002184 offset 8580)
	Section __debug_str: 0x71 (addr 0x1000021ed offset 8685)
	Section __apple_names: 0x58 (addr 0x10000225e offset 8798)
	Section __apple_namespac: 0x24 (addr 0x1000022b6 offset 8886)
	Section __apple_types: 0x4f (addr 0x1000022da offset 8922)
	Section __apple_objc: 0x24 (addr 0x100002329 offset 9001)
	total 0x34d
total 0x100003000
```

可以看到有一个名为 **DWARF 的 Segment，下面包含** debug_line、**debug_pubnames、**debug_info … 等Section。调试器所需要的调试信息便存储在这些 Section 中，可以使用 `dwarfdump` 查看：

```
$ dwarfdump foo --debug-info
foo:	file format Mach-O 64-bit x86-64

.debug_info contents:
0x00000000: Compile Unit: length = 0x0000009a version = 0x0004 abbr_offset = 0x0000 addr_size = 0x08 (next unit at 0x0000009e)

0x0000000b: DW_TAG_compile_unit
              DW_AT_producer	("Apple clang version 11.0.3 (clang-1103.0.32.62)")
              DW_AT_language	(DW_LANG_C99)
              DW_AT_name	("foo.c")
              DW_AT_stmt_list	(0x00000000)
              DW_AT_comp_dir	("/Users/lindubo/Desktop/DebugDemo")
              DW_AT_low_pc	(0x0000000100000f60)
              DW_AT_high_pc	(0x0000000100000fab)

0x0000002a:   DW_TAG_subprogram
                DW_AT_low_pc	(0x0000000100000f60)
                DW_AT_high_pc	(0x0000000100000f78)
                DW_AT_frame_base	(DW_OP_reg6 RBP)
                DW_AT_name	("foo")
                DW_AT_decl_file	("/Users/lindubo/Desktop/DebugDemo/foo.c")
                DW_AT_decl_line	(1)
                DW_AT_prototyped	(true)
                DW_AT_type	(0x00000096 "int")
                DW_AT_external	(true)

0x00000043:     DW_TAG_formal_parameter
                  DW_AT_location	(DW_OP_fbreg -4)
                  DW_AT_name	("x")
                  DW_AT_decl_file	("/Users/lindubo/Desktop/DebugDemo/foo.c")
                  DW_AT_decl_line	(1)
                  DW_AT_type	(0x00000096 "int")

0x00000051:     DW_TAG_formal_parameter
                  DW_AT_location	(DW_OP_fbreg -8)
                  DW_AT_name	("y")
                  DW_AT_decl_file	("/Users/lindubo/Desktop/DebugDemo/foo.c")
                  DW_AT_decl_line	(1)
                  DW_AT_type	(0x00000096 "int")

0x0000005f:     DW_TAG_variable
                  DW_AT_location	(DW_OP_fbreg -12)
                  DW_AT_name	("num")
                  DW_AT_decl_file	("/Users/lindubo/Desktop/DebugDemo/foo.c")
                  DW_AT_decl_line	(2)
                  DW_AT_type	(0x00000096 "int")

0x0000006d:     NULL

0x0000006e:   DW_TAG_subprogram
                DW_AT_low_pc	(0x0000000100000f80)
                DW_AT_high_pc	(0x0000000100000fab)
                DW_AT_frame_base	(DW_OP_reg6 RBP)
                DW_AT_name	("main")
                DW_AT_decl_file	("/Users/lindubo/Desktop/DebugDemo/foo.c")
                DW_AT_decl_line	(5)
                DW_AT_type	(0x00000096 "int")
                DW_AT_external	(true)

0x00000087:     DW_TAG_variable
                  DW_AT_location	(DW_OP_fbreg -8)
                  DW_AT_name	("ans")
                  DW_AT_decl_file	("/Users/lindubo/Desktop/DebugDemo/foo.c")
                  DW_AT_decl_line	(6)
                  DW_AT_type	(0x00000096 "int")

0x00000095:     NULL

0x00000096:   DW_TAG_base_type
                DW_AT_name	("int")
                DW_AT_encoding	(DW_ATE_signed)
                DW_AT_byte_size	(0x04)

0x0000009d:   NULL
```

可以看到里面包含了关于源码非常细致的描述，从编译器到源码函数、形参、布局变量、声明位置等等。

DWARF 使用 DIE (The Debugging Information Entry) 的统一形式来描述这些信息，每个 DIE 包括:

```
0x0000002a:   DW_TAG_subprogram
                DW_AT_low_pc	(0x0000000100000f60)
                DW_AT_high_pc	(0x0000000100000f78)
                DW_AT_frame_base	(DW_OP_reg6 RBP)
                DW_AT_name	("foo")
                DW_AT_decl_file	("/Users/lindubo/Desktop/DebugDemo/foo.c")
                DW_AT_decl_line	(1)
                DW_AT_prototyped	(true)
                DW_AT_type	(0x00000096 "int")
                DW_AT_external	(true)
```

*   一个 TAG (DW_TAG_xxx) 表示这是什么类型的信息，如

    *   TAG_subprogram 函数

    *   TAG_formal_parameter 形式参数

    *   TAG_variable 变量

*   多个属性 (DW_AT_xxx) 表示具体的属性信息

    *   AT_low_pc, AT_high_pc 分别代表函数的 起始/结束 PC地址

    *   AT_frame_base 表达函数的栈帧基址(frame base) ，示例中为寄存器 rbp 的值，后面的 DW_OP_fbreg 就是这个栈帧基址

    *   AT_name 描述名称，如文件、函数、变量的名称

    *   AT_decl_file 描述在哪个文件中声明

    *   AT_decl_line 描述在哪一行声明

    *   AT_prototyped 为一个 Bool 值, 为 True时代表这是一个子程序/函数(subroutine)

    *   AT_type 属性描述这个函数返回值的类型是什么, 对于 foo 函数来说, 为int

    *   AT_external 表示这个函数是否为全局可访问

将多个DIE组合起来就能描述整个程序，DIE内也可以进行嵌套，一个文件有多个函数、一个函数有多个参数等，DIE使用空行进行分割，使用缩进的形式来表达嵌套。

*   几个常用的寄存器

    sp/esp/rsp（16bit/32bit/64bit）栈寄存器---指向栈顶

    bp/ebp/rbp 栈基址寄存器---指向栈底

    ip/eip/rip 程序指令寄存器---指向下一条待执行指令

### 验证问题

1.  **如何找到设置断点的函数对应的源码文件？**

    从上面的 debug_info 信息中已经可以清晰的看到，在函数相关的DIE信息中包含了所在的文件、行数，因此调试器可以根据这些信息找到对应的源码和行数。

2.  **如何找到对应的函数设置断点？**

    调试器与 ptrace 的交互都是基于有限的信息（指令、寄存器、内存地址等信息），因此需要将需要设置断点的函数转化为对应的内存地址信息，在对应的地址设置断点。从上面的调试信息中可以看到 `foo` 函数的入口地址为 `0x0000000100000f60` ，我们可以在对应的地址设置断点来测试:

```
(lldb) b 0x0000000100000f60
Breakpoint 3: where = foo`foo at foo.c:1, address = 0x0000000100000f60
(lldb) run
There is a running process, kill it and restart?: [Y/n] y
Process 42581 exited with status = 9 (0x00000009)
Process 42689 launched: '/Users/lindubo/Desktop/DebugDemo/foo' (x86_64)
Process 42689 stopped
* thread #1, queue = 'com.apple.main-thread', stop reason = breakpoint 3.1
    frame #0: 0x0000000100000f60 foo`foo(x=0, y=12325) at foo.c:1
-> 1   	int foo(int x, int y) {
   2   	  int num = x + y;
   3   	  return num;
   4   	}
   5   	int main() {
   6   	  int ans = foo(1, 2);
   7   	  return 0;
Target 0: (foo) stopped.
(lldb)
```

程序确实在 `foo` 函数的入口位置停止。

3.  **如何在函数内找到对应的变量？**

当程序停止在对应位置后，比如 `foo` 函数 `return` 前，用 `p` 命令查看几个变量的值：

```
(lldb) p x
(int) $0 = 1
(lldb) p y
(int) $1 = 2
(lldb) p num
(int) $2 = 3
```

基于前面的调试信息可以看出，x、y、num几个变量的地址分别在 `DW_OP_fbreg` 往前偏移 4、8、12的位置，而 `DW_OP_fbreg` 指的就是函数的栈帧基址即 rbp 寄存器地址：

```
// 读取 rbp 寄存器地址
(lldb) register read rbp
     rbp = 0x00007ffeefbff190
// 计算rbp往前偏移4的地址
(lldb) p/x 0x00007ffeefbff190-4
(long) $4 = 0x00007ffeefbff18c
// 读取4字节内存数据以16进制输出，与形参x值相同为1
(lldb) x/4bx 0x00007ffeefbff18c
0x7ffeefbff18c: 0x01 0x00 0x00 0x00

// 计算读取形参y地址数据
(lldb) p/x 0x00007ffeefbff190-8
(long) $5 = 0x00007ffeefbff188
(lldb) x/4bx 0x00007ffeefbff188
0x7ffeefbff188: 0x02 0x00 0x00 0x00

// 计算读取局部变量num地址及数据
(lldb) p/x 0x00007ffeefbff190-12
(long) $6 = 0x00007ffeefbff184
(lldb) x/4bx 0x00007ffeefbff184
0x7ffeefbff184: 0x03 0x00 0x00 0x00
```

## 参考文章:

[The DWARF Debugging Standard](http://www.dwarfstd.org/)

[DW_TAG 一览表](https://ja.osdn.net/projects/drdeamon64/wiki/TAG%E5%90%8D%28DW_TAG_xxxx%29%E4%B8%80%E8%A6%A7%E3%81%A8%E5%80%A4%E3%80%81%E6%84%8F%E5%91%B3)

[DW_AT 属性一览表](https://ja.osdn.net/projects/drdeamon64/wiki/Attribute%E5%90%8D%28DW_AT_yyyy%29%E4%B8%80%E8%A6%A7%E3%81%A8%E5%80%A4%E3%80%81%E6%84%8F%E5%91%B3)

[lldb常用命令与调试技巧](https://www.jianshu.com/p/ccbb434751a9)

[How debuggers work: Part 3 - Debugging information](https://eli.thegreenplace.net/2011/02/07/how-debuggers-work-part-3-debugging-information)

[How debuggers work: Part 2 - Breakpoints](https://eli.thegreenplace.net/2011/01/27/how-debuggers-work-part-2-breakpoints/)