# lec2：lab0 SPOC思考题

## **提前准备**
（请在上课前完成，option）

- 完成lec2的视频学习
- git pull ucore_os_lab, os_tutorial_lab, os_course_exercises  in github repos。这样可以在本机上完成课堂练习。
- 了解代码段，数据段，执行文件，执行文件格式，堆，栈，控制流，函数调用,函数参数传递，用户态（用户模式），内核态（内核模式）等基本概念。思考一下这些基本概念在不同操作系统（如linux, ucore,etc.)与不同硬件（如 x86, riscv, v9-cpu,etc.)中是如何相互配合来体现的。
- 安装好ucore实验环境，能够编译运行ucore labs中的源码。
- 会使用linux中的shell命令:objdump，nm，file, strace，gdb等，了解这些命令的用途。
- 会编译，运行，使用v9-cpu的dis,xc, xem命令（包括启动参数），阅读v9-cpu中的v9\-computer.md文档，了解汇编指令的类型和含义等，了解v9-cpu的细节。
- 了解基于v9-cpu的执行文件的格式和内容，以及它是如何加载到v9-cpu的内存中的。
- 在piazza上就学习中不理解问题进行提问。

---

## 思考题

- 你理解的对于类似ucore这样需要进程/虚存/文件系统的操作系统，在硬件设计上至少需要有哪些直接的支持？至少应该提供哪些功能的特权指令？
  - 进程切换需要中断，故硬件应支持时钟中断；虚存管理需要实现内存管理单元MMU；文件系统需要持久性存储介质来实现存储。
  - 需要支持进程/虚存/文件系统的特权指令，例如提供中断使能、触发软中断的指令，设置内存寻址模式、设置页表的指令，执行IO读写操作的指令。
- 你理解的x86的实模式和保护模式有什么区别？物理地址、线性地址、逻辑地址的含义分别是什么？
  - x86实模式和保护模式：
    - 实模式：16位寻址空间，可访问的物理内存空间不超过1MB，80386加电启动后处于实模式状态（向下兼容缘故），之后可以切换到保护模式。
    - 保护模式：32位寻址空间4GB，且支持内存分页机制，支持虚拟内存；支持多任务；支持优先级机制
    - 二者区别在于进程内存是否受到保护。实模式不区分系统程序和用户程序，将实际的物理内存看作分段的区域，每个程序中的指针都指向实际的地址，安全性较低。保护模式区分了系统程序和用户程序，物理内存是不可被程序直接访问的，而是OS通过将虚拟地址转化之后访问物理地址，该过程对程序是透明的。
  - 各种地址的含义：
    - 物理地址：内存中实际的地址，是CPU提交到总线上用于访问内存和外设的最终地址。一个系统只有一个物理地址。
    - 线性地址：逻辑地址转变到物理地址的中间地址，是在OS的虚存管理下每个运行的应用程序能访问的地址，即逻辑地址通过段机制处理后形成的地址。
    - 逻辑地址：程序直接使用的地址。
- 你理解的risc-v的特权模式有什么区别？不同 模式在地址访问方面有何特征？
  - RISC-V有四个特权模式：用户模式、监督模式、机器模式和Hypervisor模式。不同模式可使用的指令有所不同。
  - RISC-V有特权指令集，支持如下计算机系统：
    - 仅拥有机器模式的系统
    - 同时拥有机器模式和用户模式的系统
    - 同时拥有所有四个模式的系统
- 理解ucore中list_entry双向链表数据结构及其4个基本操作函数和ucore中一些基于它的代码实现（此题不用填写内容）
- 对于如下的代码段，请说明":"后面的数字是什么含义
```
 /* Gate descriptors for interrupts and traps */
 struct gatedesc {
    unsigned gd_off_15_0 : 16;        // low 16 bits of offset in segment
    unsigned gd_ss : 16;            // segment selector
    unsigned gd_args : 5;            // # args, 0 for interrupt/trap gates
    unsigned gd_rsv1 : 3;            // reserved(should be zero I guess)
    unsigned gd_type : 4;            // type(STS_{TG,IG32,TG32})
    unsigned gd_s : 1;                // must be 0 (system)
    unsigned gd_dpl : 2;            // descriptor(meaning new) privilege level
    unsigned gd_p : 1;                // Present
    unsigned gd_off_31_16 : 16;        // high bits of offset in segment
 };
```

答：表示该域在struct中所占据的位数。根据C++语法，冒号后面的数字限定了该类型所占用的bit数。

- 对于如下的代码段，

```
#define SETGATE(gate, istrap, sel, off, dpl) {            \
    (gate).gd_off_15_0 = (uint32_t)(off) & 0xffff;        \
    (gate).gd_ss = (sel);                                \
    (gate).gd_args = 0;                                    \
    (gate).gd_rsv1 = 0;                                    \
    (gate).gd_type = (istrap) ? STS_TG32 : STS_IG32;    \
    (gate).gd_s = 0;                                    \
    (gate).gd_dpl = (dpl);                                \
    (gate).gd_p = 1;                                    \
    (gate).gd_off_31_16 = (uint32_t)(off) >> 16;        \
}
```
如果在其他代码段中有如下语句，
```
unsigned intr;
intr=8;
SETGATE(intr, 1,2,3,0);
```
请问执行上述指令后， intr的值是多少？

答：结果为131075，即十六进制下的0x20003。可讲上述内容编写为如下代码：

```C
#include <stdio.h>

typedef unsigned uint32_t;

#define STS_IG32 0xE
#define STS_TG32 0xF
#define SETGATE(gate, istrap, sel, off, dpl)         \
{                                                    \
	(gate).gd_off_15_0 = (uint32_t)(off)&0xffff;     \
	(gate).gd_ss = (sel);                            \
	(gate).gd_args = 0;                              \
	(gate).gd_rsv1 = 0;                              \
	(gate).gd_type = (istrap) ? STS_TG32 : STS_IG32; \
	(gate).gd_s = 0;                                 \
	(gate).gd_dpl = (dpl);                           \
	(gate).gd_p = 1;                                 \
	(gate).gd_off_31_16 = (uint32_t)(off) >> 16;     \
}

struct gatedesc {
	unsigned gd_off_15_0 : 16;   // low 16 bits of offset in segment
	unsigned gd_ss : 16;         // segment selector
	unsigned gd_args : 5;        // # args, 0 for interrupt/trap gates
	unsigned gd_rsv1 : 3;        // reserved(should be zero I guess)
	unsigned gd_type : 4;        // type(STS_{TG,IG32,TG32})
	unsigned gd_s : 1;           // must be 0 (system)
	unsigned gd_dpl : 2;         // descriptor(meaning new) privilege level
	unsigned gd_p : 1;           // Present
	unsigned gd_off_31_16 : 16;  // high bits of offset in segment
};

int main(int argc, char const* argv[]) {
	unsigned intr = 8;
	struct gatedesc intr_elem = *(struct gatedesc *) &intr;

	SETGATE(intr_elem, 1, 2, 3, 0);

	intr = *(unsigned *) &intr_elem;
	printf("%d\n", intr);
	printf("0x%x\n", intr);
	return 0;
}
```



### 课堂实践练习

#### 练习一

1. 请在ucore中找一段你认为难度适当的AT&T格式X86汇编代码，尝试解释其含义。

2. (option)请在rcore中找一段你认为难度适当的RV汇编代码，尝试解释其含义。

#### 练习二

宏定义和引用在内核代码中很常用。请枚举ucore或rcore中宏定义的用途，并举例描述其含义。

#### reference
 - [Intel格式和AT&T格式汇编区别](http://www.cnblogs.com/hdk1993/p/4820353.html)
 - [x86汇编指令集  ](http://hiyyp1234.blog.163.com/blog/static/67786373200981811422948/)
 - [PC Assembly Language, Paul A. Carter, November 2003.](https://pdos.csail.mit.edu/6.828/2016/readings/pcasm-book.pdf)
 - [*Intel 80386 Programmer's Reference Manual*, 1987](https://pdos.csail.mit.edu/6.828/2016/readings/i386/toc.htm)
 - [IA-32 Intel Architecture Software Developer's Manuals](http://www.intel.com/content/www/us/en/processors/architectures-software-developer-manuals.html)
