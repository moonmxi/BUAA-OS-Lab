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
	tf->cp0_epc += 4;

	/* Step 2: Use 'sysno' to get 'func' from 'syscall_table'. */
	func = syscall_table[sysno];

	/* Step 3: First 3 args are stored at $a1, $a2, $a3. */
	u_int arg1 = tf->regs[5];
	u_int arg2 = tf->regs[6];
	u_int arg3 = tf->regs[7];

	/* Step 4: Last 2 args are stored in stack at [$sp + 16 bytes], [$sp + 20 bytes] */
	u_int arg4, arg5;
	arg4 = *(u_int *)(tf->regs[29]+16);
	arg5 = *(u_int *)(tf->regs[29]+20);

	/* Step 5: Invoke 'func' with retrieved arguments and store its return value to $v0 in 'tf'.
	 */
	tf->regs[2] = func(arg1, arg2, arg3, arg4, arg5);
}
```

这里基本是根据提示写代码，首先按要求分别三次赋值，之后调用 `func`，主要的难点是搞清楚各个寄存器号，以及指针和数值之间的转化。

### 4.3

* 题目

```c
 实现kern/env.c 中的envid2env 函数。
实现通过一个进程的id 获取该进程控制块的功能。提示：可以利用include/env.h 中
的宏函数ENVX() ，用于获取目标Env 块在env 数组中的下标。
```

* 我的代码

```c
int envid2env(u_int envid, struct Env **penv, int checkperm) {
	struct Env *e;

	/* Step 1: Assign value to 'e' using 'envid'. */
	if (envid == 0) {
		*penv = curenv;
		return 0;
	} else {
		e = envs + ENVX(envid);
	}

	if (e->env_status == ENV_FREE || e->env_id != envid) {
		return -E_BAD_ENV;
	}

	/* Step 2: Check when 'checkperm' is non-zero. */
	if (checkperm && e->env_id != curenv->env_id  &&e->env_parent_id != curenv->env_id) {
		return -E_BAD_ENV;
	}

	/* Step 3: Assign 'e' to '*penv'. */
	*penv = e;
	return 0;
}
```

这道题要认真看提示要求——

```c
Convert an existing 'envid' to an 'struct Env *'.
 *   If 'envid' is 0, set '*penv = curenv', otherwise set '*penv = &envs[ENVX(envid)]'.
 *   In addition, if 'checkperm' is non-zero, the requested env must be either 'curenv' or its
 *   immediate child.
```

明确写出了各个步骤的代码怎么写。

### 4.4

* 题目

```c
实现 kern/syscall_all.c 中的 int sys_mem_alloc(u_int envid,
u_int va, u_int perm) 函数。
```

* 我的代码

```c
int sys_mem_alloc(u_int envid, u_int va, u_int perm) {
	struct Env *env;
	struct Page *pp;

	/* Step 1: Check if 'va' is a legal user virtual address using 'is_illegal_va'. */
	if(is_illegal_va(va)){
		return -E_INVAL;
	}

	/* Step 2: Convert the envid to its corresponding 'struct Env *' using 'envid2env'. */
	try(envid2env(envid, &env, 1));

	/* Step 3: Allocate a physical page using 'page_alloc'. */
	try(page_alloc(&pp));

	/* Step 4: Map the allocated page at 'va' with permission 'perm' using 'page_insert'. */
	return page_insert(env->env_pgdir, env->env_asid, pp, va, perm);
}
```

同样是看提示写代码——

```c
/* Overview:
 *   Allocate a physical page and map 'va' to it with 'perm' in the address space of 'envid'.
 *   If 'va' is already mapped, that original page is sliently unmapped.
 *   'envid2env' should be used with 'checkperm' set, like in most syscalls, to ensure the target is
 * either the caller or its child.
 *
 * Post-Condition:
 *   Return 0 on success.
 *   Return -E_BAD_ENV: 'checkperm' of 'envid2env' fails for 'envid'.
 *   Return -E_INVAL:   'va' is illegal (should be checked using 'is_illegal_va').
 *   Return the original error: underlying calls fail (you can use 'try' macro).
 *
 * Hint:
 *   You may want to use the following functions:
 *   'envid2env', 'page_alloc', 'page_insert', 'try' (macro)
 */
```

### 4.5

* 题目

```c
实现 kern/syscall_all.c 中的 int sys_mem_map(u_int srcid,
u_int srcva, u_int dstid, u_int dstva, u_int perm) 函数。
```

* 我的代码

```c
int sys_mem_map(u_int srcid, u_int srcva, u_int dstid, u_int dstva, u_int perm) {
	struct Env *srcenv;
	struct Env *dstenv;
	struct Page *pp;

	/* Step 1: Check if 'srcva' and 'dstva' are legal user virtual addresses using
	 * 'is_illegal_va'. */
	if(is_illegal_va(srcva) || is_illegal_va(dstva)){
		return -E_INVAL;
	}

	/* Step 2: Convert the 'srcid' to its corresponding 'struct Env *' using 'envid2env'. */
	try(envid2env(srcid, &src, 1));

	/* Step 3: Convert the 'dstid' to its corresponding 'struct Env *' using 'envid2env'. */
	try(envid2env(dstid, &dst, 1));

	/* Step 4: Find the physical page mapped at 'srcva' in the address space of 'srcid'. */
	/* Return -E_INVAL if 'srcva' is not mapped. */
	if ((pp = page_lookup(srcenv->env_pgdir, srcva, NULL)) == NULL) {
		return -E_INVAL;
	}

	/* Step 5: Map the physical page at 'dstva' in the address space of 'dstid'. */
	return page_insert(dstenv->env_pgdir, dstenv->env_asid, pp, dstva, perm);
}
```

和上一道题几乎一样，按步骤根据注释填函数即可。

### 4.6

* 题目

```c
实现 kern/syscall_all.c 中的 int sys_mem_unmap(u_int envid, u_int
va) 函数。
```

* 我的代码

```c
int sys_mem_unmap(u_int envid, u_int va) {
	struct Env *e;

	/* Step 1: Check if 'va' is a legal user virtual address using 'is_illegal_va'. */
	if(is_illegal_va(srcva) || is_illegal_va(dstva)){
		return -E_INVAL;
	}

	/* Step 2: Convert the envid to its corresponding 'struct Env *' using 'envid2env'. */
	try(envid2env(envid, &e, 1));

	/* Step 3: Unmap the physical page at 'va' in the address space of 'envid'. */
	page_remove(e->env_pgdir, e->env_asid, va);
	return 0;
}
```

和前两道题一样，照葫芦画瓢。

### 4.7

* 题目

```c
实现 kern/syscall_all.c 中的 void sys_yield(void) 函数。
```

* 我的代码

```c
void __attribute__((noreturn)) sys_yield(void) {
	// Hint: Just use 'schedule' with 'yield' set.
	schedule(1);
}
```

这个函数的功能是实现用户进程对CPU 的放弃，从而调度其他的进程。题目提示利用 `schedule`函数实现，在这个函数中判断条件是 `if (yield || count <= 0 || e == NULL || e->env_status != ENV_RUNNABLE)`，所以传入的 `yield`应该置1。

### 4.8

* 题目

```c
实现 kern/syscall_all.c 中的 int sys_ipc_recv(u_int dstva) 函数和
int sys_ipc_try_send(u_int envid, u_int value, u_int srcva, u_int perm) 函数。
请注意在修改进程控制块的状态后，应同步维护调度队列。
```

* 我的代码

```c
int sys_ipc_recv(u_int dstva) {
	/* Step 1: Check if 'dstva' is either zero or a legal address. */
	if (dstva != 0 && is_illegal_va(dstva)) {
		return -E_INVAL;
	}

	/* Step 2: Set 'curenv->env_ipc_recving' to 1. */
	curenv->env_ipc_recving = 1;

	/* Step 3: Set the value of 'curenv->env_ipc_dstva'. */
	curenv->env_ipc_dstva = dstva;

	/* Step 4: Set the status of 'curenv' to 'ENV_NOT_RUNNABLE' and remove it from
	 * 'env_sched_list'. */
	curenv->env_status = ENV_NOT_RUNNABLE;
	TAILQ_REMOVE(&env_sched_list, curenv, env_sched_link);

	/* Step 5: Give up the CPU and block until a message is received. */
	((struct Trapframe *)KSTACKTOP - 1)->regs[2] = 0;
	schedule(1);
}

int sys_ipc_try_send(u_int envid, u_int value, u_int srcva, u_int perm) {
	struct Env *e;
	struct Page *p;

	/* Step 1: Check if 'srcva' is either zero or a legal address. */
	if (srcva != 0 && is_illegal_va(srcva)) {
		return -E_INVAL;
	}

	/* Step 2: Convert 'envid' to 'struct Env *e'. */
	/* This is the only syscall where the 'envid2env' should be used with 'checkperm' UNSET,
	 * because the target env is not restricted to 'curenv''s children. */
	try(envid2env(envid, &e, 0));

	/* Step 3: Check if the target is waiting for a message. */
	if (!e->env_ipc_recving) {
		return -E_IPC_NOT_RECV;
	}

	/* Step 4: Set the target's ipc fields. */
	e->env_ipc_value = value;
	e->env_ipc_from = curenv->env_id;
	e->env_ipc_perm = PTE_V | perm;
	e->env_ipc_recving = 0;

	/* Step 5: Set the target's status to 'ENV_RUNNABLE' again and insert it to the tail of
	 * 'env_sched_list'. */
	e->env_status = ENV_RUNNABLE;
	TAILQ_INSERT_TAIL(&env_sched_list, e, env_sched_link);

	/* Step 6: If 'srcva' is not zero, map the page at 'srcva' in 'curenv' to 'e->env_ipc_dstva'
	 * in 'e'. */
	/* Return -E_INVAL if 'srcva' is not zero and not mapped in 'curenv'. */
	if (srcva != 0) {
		p = page_lookup(curenv->env_pgdir, srcva, NULL);
		if (p == NULL) {
			return -E_INVAL;
		}
		try(page_insert(e->env_pgdir, e->env_asid, p, e->env_ipc_dstva, perm));
	}
	return 0;
}
```

感觉前面几个题做完了，这道题依旧是照葫芦画瓢，这几道题的难点主要是要回顾之前写过的函数和宏！

### 4.9

* 题目

```c
请根据上述步骤以及代码中的注释提示，填写 kern/syscall_all.c 中的
sys_exofork 函数。
```

* 我的代码

```c
int sys_exofork(void) {
	struct Env *e;

	/* Step 1: Allocate a new env using 'env_alloc'. */
	try(env_alloc(&e, curenv->env_id));

	/* Step 2: Copy the current Trapframe below 'KSTACKTOP' to the new env's 'env_tf'. */
	e->env_tf=*((struct Trapframe *)KSTACKTOP - 1);

	/* Step 3: Set the new env's 'env_tf.regs[2]' to 0 to indicate the return value in child. */
	e->env_tf.regs[2]=0;

	/* Step 4: Set up the new env's 'env_status' and 'env_pri'.  */
	e->env_status=ENV_NOT_RUNNABLE;
	e->env_pri=curenv->env_pri;

	return e->env_id;
}
```

感觉Lab4的注释提示给的太慷慨了，这道题主要的难点是对 `env_tf`的设置，涉及到以前的内容，要回顾一下。

### 4.10

* 题目

```c
结合代码注释以及上述提示，填写user/lib/fork.c 中的duppage 函数。
```

* 我的代码

```c
static void duppage(u_int envid, u_int vpn) {
	int r;
	u_int addr;
	u_int perm;

	/* Step 1: Get the permission of the page. */
	addr=vpn*PAGE_SIZE;
	perm = vpt[vpn] & 0xfff;

	/* Step 2: If the page is writable, and not shared with children, and not marked as COW yet,
	 * then map it as copy-on-write, both in the parent (0) and the child (envid). */
	int flag = 0;
	if ((perm & PTE_D) && !(perm & PTE_LIBRARY)) {
		perm = (perm & ~ PTE_D) | PTE_COW;
		flag = 1;
	}
	syscall_mem_map(0, addr, envid, addr, perm);

	if (flag) {
		syscall_mem_map(0, addr, 0, addr, perm);
	}
}
```

这道题相对来说，需要自己写的东西比较多，难度比较大。

首先是根据提示找到虚拟地址，也就是虚拟页号✖️页大小。之后根据提示，使用 `vpt`来找到对应页的权限位，这里 `#define vpt ((const volatile Pte *)UVPT)`，也就是把用户页表的起始地址设置为页表项指针类型，那么这个时候vpn就是在这个页表里面的下标，这样就能找到对应页表项，取其权限位。

之后根据题目要求，在

```
(perm & PTE_D) && !(perm & PTE_LIBRARY)
```

的情况下设置权限位为写时复制，如果这个执行了这步操作，就需要给父进程和子进程都复制一次，否则只需要复制子进程。

### 4.11

* 题目

```c
根据上述提示以及代码注释，完成kern/tlbex.c 中的do_tlb_mod 函数，
设置好保存的现场中EPC 寄存器的值。
```

* 我的代码

```c
void do_tlb_mod(struct Trapframe *tf) {
	struct Trapframe tmp_tf = *tf;

	if (tf->regs[29] < USTACKTOP || tf->regs[29] >= UXSTACKTOP) {
		tf->regs[29] = UXSTACKTOP;
	}
	tf->regs[29] -= sizeof(struct Trapframe);
	*(struct Trapframe *)tf->regs[29] = tmp_tf;
	Pte *pte;
	page_lookup(cur_pgdir, tf->cp0_badvaddr, &pte);
	if (curenv->env_user_tlb_mod_entry) {
		tf->regs[4] = tf->regs[29];
		tf->regs[29] -= sizeof(tf->regs[4]);
		// Hint: Set 'cp0_epc' in the context 'tf' to 'curenv->env_user_tlb_mod_entry'.
		/* Exercise 4.11: Your code here. */
		tf->cp0_epc = curenv->env_user_tlb_mod_entry;

	} else {
		panic("TLB Mod but no user handler registered");
	}
}
```

这道题填写起来不难，按要求写出来这一句赋值即可，但是比较难理解——

首先将 `sp`寄存器设置到 `UXSTACKTOP`的位置，之后在用户异常栈为 `trapframe`分配空间。随后从当前进程控制块取出用户空间下的 `TLB`异常处理函数，并设置到 `EPC`中，同时设定 `a0`，自减 `sp`，留出一个参数的空间。

### 4.12

* 题目

```c
完成kern/syscall_all.c 中的sys_set_tlb_mod_entry 函数。
```

* 我的代码

```c
int sys_set_tlb_mod_entry(u_int envid, u_int func) {
	struct Env *env;

	/* Step 1: Convert the envid to its corresponding 'struct Env *' using 'envid2env'. */
	try(envid2env(envid, &env, 1));

	/* Step 2: Set its 'env_user_tlb_mod_entry' to 'func'. */
	env->env_user_tlb_mod_entry=func;

	return 0;
}
```

这道题就比较简单了，按照提示来写就好，填空的语句和前面的题相似。

### 4.13

* 题目

```c
填写user/lib/fork.c 中的cow_entry 函数。
```

* 我的代码

```c
static void __attribute__((noreturn)) cow_entry(struct Trapframe *tf) {
	u_int va = tf->cp0_badvaddr;
	u_int perm;

	/* Step 1: Find the 'perm' in which the faulting address 'va' is mapped. */
	perm=vpt[VPN(va)]&0xfff;
	if(!(perm&PTE_COW)) {
		user_panic("perm doesn't have PTE_COW");
	}

	/* Step 2: Remove 'PTE_COW' from the 'perm', and add 'PTE_D' to it. */
	perm=(perm & ~PTE_COW) | PTE_D;

	/* Step 3: Allocate a new page at 'UCOW'. */
	syscall_mem_alloc(0, (void *)UCOW, perm);

	/* Step 4: Copy the content of the faulting page at 'va' to 'UCOW'. */
	/* Hint: 'va' may not be aligned to a page! */
	memcpy((void *)UCOW, (void *)ROUNDDOWN(va, PAGE_SIZE), PAGE_SIZE);

	// Step 5: Map the page at 'UCOW' to 'va' with the new 'perm'.
	syscall_mem_map(0, (void *)UCOW, 0, (void *)va, perm);

	// Step 6: Unmap the page at 'UCOW'.
	syscall_mem_unmap(0, (void *)UCOW);

	// Step 7: Return to the faulting routine.
	int r = syscall_set_trapframe(0, tf);
	user_panic("syscall_set_trapframe returned %d", r);
}
```

这道题也是一步一步按照提示来写代码，需要注意，我们在用户态下，需要使用 `syscall_*`函数进行操作。

### 4.14

* 题目

```c
填写kern/syscall_all.c 中的sys_set_env_status 函数。
```

* 我的代码

```c
int sys_set_env_status(u_int envid, u_int status) {
	struct Env *env;

	/* Step 1: Check if 'status' is valid. */
	if (status != ENV_RUNNABLE && status != ENV_NOT_RUNNABLE) {
		return -E_INVAL;
	}

	/* Step 2: Convert the envid to its corresponding 'struct Env *' using 'envid2env'. */
	try(envid2env(envid, &env, 1));

	/* Step 3: Update 'env_sched_list' if the 'env_status' of 'env' is being changed. */
	if (env->env_status != ENV_NOT_RUNNABLE && status == ENV_NOT_RUNNABLE) {
		TAILQ_REMOVE(&env_sched_list, env, env_sched_link);
	}
	else if (env->env_status != ENV_RUNNABLE && status == ENV_RUNNABLE) {
		TAILQ_INSERT_TAIL(&env_sched_list, env, env_sched_link);
	}

	/* Step 4: Set the 'env_status' of 'env'. */
	env->env_status = status;

	/* Step 5: Use 'schedule' with 'yield' set if ths 'env' is 'curenv'. */
	if (env == curenv) {
		schedule(1);
	}
	return 0;
}
```

这道题的提示给的更加直白，直接照葫芦画瓢。

---

## Thinking

### 4.1

* 内核通过使用 `SAVE_ALL`宏来保存现场，具体来说，该宏首先将现场的栈指针保存在了k0(为内核保留的寄存器)中，接着，将栈指针寄存器指向了内核栈顶部并为 `Trap_frame`结构体分配了栈空间，最后将CPU所有的现场信息全部保存在内核栈上的 `TF`结构体中。
* **不可以** ，就拿$a0来说，在 `handle_exception`函数中，有语句 `move a0 sp`，这句语句的意义是将指向内核栈上的 `TF`结构体的指针作为传递给 `do_syscall`这个handler函数的参数，也就是说，现场已经遭到破坏，我们想要访问调用 `msyscall`时传入的参数，需要通过内核栈上的TF结构体来实现。
* 根据我们前面所述，调用 `msyscall`时的所有现场信息已经保存到内核栈上的 `TF`结构体中了，对于$a0-$a3四个参数我们可以直接通过访问 `TF`结构体中对应的寄存器信息取得,然而对于保存在函数栈帧中的参数，则需要通过 `TF`结构体中保存的现场栈指针来访问.
* 内核处理系统调用的过程中对 `Trapframe`做了两点改变:

```
tf->cp0_epc += 4;
tf->regs[2] = return_val;
```

修改epc是为了在返回用户态后程序从 `syscall`指令的下一条开始运行，不进行该操作会重复运行 `syscall`进入内核态；而修改 `TF`中的v0寄存器则是为了向用户态传递返回值

### 4.2

在该函数中，我们是这样从 `envs`数组中取出进程控制块的：

```
e = &envs[ENVX(envid)];
```

而使用到的 `ENVX`宏以及用来生成进程id的 `mkenvid`函数是这样实现的：

```
#define ENVX(envid) ((envid) & (NENV - 1))

u_int mkenvid(struct Env *e) {
	static u_int i = 0;
	return ((++i) << (1 + LOG2NENV)) | (e - envs);
}
```

也就是一个32位进程编号的结构是：低10位索引该编号对应进程控制块在 `envs`中的偏移量，而高22位则表示该进程被分配的序号，也就是说可能会出现使用一个 `envid`在 `envs`中索引到的进程控制块中的 `envid`和我们预期的不符的情况(哈希冲突)

### 4.3

首先，`envid2env`函数提供了一个重要的功能，即：向该函数传入参数0时，该函数返回指向当前运行进程的进程控制块的指针，具体实现如下：

```
if(envid == 0){ 
	*penv = curenv;
	return 0;
} 
```

而站在用户的角度，在使用诸如 `syscall_mem_map`等系统调用时，会通过传入0这个id来对当前进程进行操作。而若 `mkenvid`在生成进程id时使某个进程的id为0了，这样的调用行为就存在二义性了，即： **无法明确用户制定的进程是当前进程还是那个id为0的进程** 。因此，该函数不会返回0。

### 4.4

应当选择C，`fork`函数的特点就是“调用一次，返回两次”，且在父进程中被调用。

### 4.5

在函数 `env_init`对所有进程控制块的初始化过程中，使用 `map_segment`函数构造了一个名为 `base_pgdir`的“模板页目录”：

```
base_pgdir = (Pde *)page2kva(p);

map_segment(base_pgdir, 0, PADDR(pages), UPAGES, ROUND(npage * sizeof(struct Page), BY2PG),PTE_G);

map_segment(base_pgdir, 0, PADDR(envs), UENVS, ROUND(NENV * sizeof(struct Env), BY2PG),PTE_G);
```

`base_pgdir`作为一个模板，以 `PTE_G`(无写权限)的权限映射了[UENVS, UVPT)之间的地址空间(即UPAGES和UENVS两个段)。

而在创建一个新进程，并进行 `env_setup_vm`的过程中，对于每个新创建进程，我们都要将前面所述的 `base_pgdir`这个模板页目录复制到这个新进程的页目录中来，这是为了保证每个进程都能够只读地访问到 `pages`数组和 `envs`数组：

```
memcpy(e->env_pgdir + PDX(UTOP), base_pgdir + PDX(UTOP),
	       sizeof(Pde) * (PDX(UVPT) - PDX(UTOP)));
```

也就是说，用户空间中的[UPAGES, UVPT)这一段全部进程共享的页面段不需要使用 `duppage`进行复制映射。

### 4.6

* `vpt`和 `vpd`是定义在 `user\lib.h`下的一对宏，具体如下：

  ```
  #define vpt ((volatile Pte *)UVPT)
  #define vpd ((volatile Pde *)(UVPT + (PDX(UVPT) << PGSHIFT)))
  ```

  它们的作用是将用户进程地址空间中的页表基址和页目录基址以指针的形式封装起来，方便编程时使用以访问页表和页目录。使用上可以当作一般的 `Pte*`指针和 `Pde*`指针使用。
* 根据 `mmu.h`中的地址空间布局图，我们可以知道，在所有进程看来，它自身的页表均分布在[UVPT, ULIM)这段地址空间上，更具体地，所有的进程的页表基址均为 `UVPT`。根据这种统一性，我们可以通过这样一个经由宏定义得到的指针来对进程自身的页表进行存取。
* `vpd`的基址是由自映射机制推导而来的，这体现出了自映射的设计。
* 不能，用户态下无权修改页表项。

### 4.7

* 即：在处理A进程的 `tlb_mod`异常时另一进程又触发了该异常。
* 我们的MOS操作系统采用的是**微内核**架构，按照微内核的设计理念，我们应当尽可能地将功能实现在用户空间中，精简内核的功能范围。对于页写入异常的处理，也应当遵循这个设计原则。而将异常现场的 `Trapframe`由内核空间复制到用户空间的异常处理栈 `UXSPACE`上的主要目的就是为了让我们用户空间的处理函数 `cow_entry`能够获取到异常现场信息并进行异常处理。

### 4.8

在用户态处理页写入异常相较于在内核态进行处理的优势在我看来有如下几点：

* 遵循了微内核设计原则，将异常处理的操作交给用户进程自己来完成，精简了内核的功能范围；
* 将处理的核心逻辑从内核下方到用户态，降低了处理失败对整个操作系统造成的危害与损失。

### 4.9

* 因为在父进程调用 `syscall_exofork`的过程中也可能会触发 `tlb_mod`异常。
* 假设这样的情况发生，也就是说父进程的 `env->env_user_tlb_mod_entry`在调用 `syscall_exofork`时未被正确设置为 `cow_entry`，那么当在exofork的过程中发生了 `tlb_mod`异常，根据如下代码段：

  ```
  if (curenv->env_user_tlb_mod_entry) {
  	tf->regs[4] = tf->regs[29];
  	tf->regs[29] -= sizeof(tf->regs[4]);
  	tf->cp0_epc = curenv->env_user_tlb_mod_entry;
  } else {
  	panic("TLB Mod but no user handler registered");
  }
  ```

  操作系统会无法处理这个异常并报错。

## 本次实验的难点

踩的最大的坑是出现了一个因为Lab2里面pmap.c中page_alloc对清零部分的空间大小设置错误！本来应该把一整个物理页都清零，我却写成了sizeof（struct Page），导致Lab4在给进程分配页，查找页表项的时候，恰好和我为清零部分的初始值相同，这部分指向对用户非法的地址，导致fork失败，debug了四五个小时，最后找助教帮忙才解决！

## 心得体会

不要过分依赖于注释，认真阅读，理解代码的用意才是硬道理。
