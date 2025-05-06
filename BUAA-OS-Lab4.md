# Lab4

## 指导书学习笔记

### 一、概念回顾

#### 1、用户态和内核态（也称用户模式和内核模式）：

它们是CPU 运行的两种状态。根据Lab3的说明，在MOS 操作系统实验使用的仿真 4Kc CPU 中，该状态由 `CP0 SR` 寄存器中 `UM`(User Mode) 位和 `EXL`(Exception Level) 位的值标志。

#### 2、用户空间和内核空间：

它们是虚拟内存（进程的地址空间）中的两部分区域。根据Lab2 的说明，MOS 中的用户空间包括 `kuseg`，而内核空间主要包括 `kseg0` 和 `kseg1`。每个进程的用户空间通常通过页表映射到不同的物理页，而内核空间则直接映射到固定的物理页**1**以及外部硬件设备。CPU 在内核态下可以访问任何内存区域，对物理内存等硬件设备有完整的控制权，而在用户态下则只能访问用户空间。

#### 3、（用户）进程和内核：

进程是资源分配与调度的基本单位，拥有独立的地址空间，而内核负责管理系统资源和调度进程，使进程能够并发运行。与前两对概念不同，进程和内核并不是对立的存在，可以认为内核是存在于所有进程地址空间中的一段代码。












---

## Exercise笔记

### 4.1

* 题目

```
填写user/lib/syscall_wrap.S 中的msyscall 函数，使得用户部分的系统
调用机制可以正常工作。
```

* 我的代码

```
#include <asm/asm.h>

LEAF(msyscall)
	// Just use 'syscall' instruction and return.
	syscall
	jr	ra
END(msyscall)

```

* 心得

在指导书中已经有提示：

这是一个叶函数，并且只需要执行自陷指令 `syscall`并正常返回即可（代码中也有注释提示），所以直接补充两条指令。

### 4.2

* 题目

```c
kern/syscall_all.c 中的提示，完成do_syscall 函数，使得内核部
分的系统调用机制可以正常工作。
```

* 我的代码

```c
void do_syscall(struct Trapframe *tf) {
	int (*func)(u_int, u_int, u_int, u_int, u_int);
	int sysno = tf->regs[4];
	if (sysno < 0 || sysno >= MAX_SYSNO) {
		tf->regs[2] = -E_NO_SYS;
		return;
	}

	/* Step 1: Add the EPC in 'tf' by a word (size of an instruction). */
	tf->cp0_epc+=4;

	/* Step 2: Use 'sysno' to get 'func' from 'syscall_table'. */
	func = syscall_table[sysno];

	/* Step 3: First 3 args are stored in $a1, $a2, $a3. */
	u_int arg1 = tf->regs[5];
	u_int arg2 = tf->regs[6];
	u_int arg3 = tf->regs[7];

	/* Step 4: Last 2 args are stored in stack at [$sp + 16 bytes], [$sp + 20 bytes]. */
	u_int arg4, arg5;
	arg4 = *(u_int *)(tf->regs[29]+16);
	arg5 = *(u_int *)(tf->regs[29]+20);

	/* Step 5: Invoke 'func' with retrieved arguments and store its return value to $v0 in 'tf'.
	 */
	tf->regs[2] = func(arg1, arg2, arg3, arg4, arg5);
}
```

* 心得

这里基本是根据提示写代码，首先按要求分别三次赋值，之后调用 `func`，主要的难点是搞清楚各个寄存器号，以及指针和数值之间的转化。

### 4.3












---
