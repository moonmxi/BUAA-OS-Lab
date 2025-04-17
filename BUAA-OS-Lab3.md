# Lab3

## 指导书学习笔记

### 一、本实验中的进程

#### 1、本实验中进程作用

由于本实验没有实现线程，所以进程既是基本分配单元，又是基本执行单元（本应该是线程）。

#### 2、进程控制块PCB

* `PCB`是系统专门设置用来管理进程的数据结构，可以记录进程的外部特征和变化过程。
* MOS中，`PCB`由一个 `Env`结构体实现，代码如下:

```c
struct Env {
	struct Trapframe env_tf;	 // saved context (registers) before switching
	LIST_ENTRY(Env) env_link;	 // intrusive entry in 'env_free_list'
	u_int env_id;			 // unique environment identifier
	u_int env_asid;			 // ASID of this env
	u_int env_parent_id;		 // env_id of this env's parent
	u_int env_status;		 // status of this env
	Pde *env_pgdir;			 // page directory
	TAILQ_ENTRY(Env) env_sched_link; // intrusive entry in 'env_sched_list'
	u_int env_pri;			 // schedule priority
};
```

* 首先是 `env_tf`，它的作用是在发生进程调度，或当陷入内核时，会将当时的进程上下文环境保存下来。
* `env_link`：它的类型在lab2里面已经详细解析，这里是让 `PCB`成为了链表项。
* `env_id`和 `env_parent_id`：是进程和其父进程的 `id`，唯一标识符。
* `env_status`：代表进程的状态，分别是空闲 `ENV_FREE`，阻塞 `ENV_NOT_RUNNABLE`，就绪 `ENV_RUNNABLE`。
* `env_pgdir`：进程页目录的内核虚拟地址。
* `env_sched_link`：用于构造调度队列 `env_sched_list`。
* `env_pri`：进程优先级。

在实验中，存放进程控制块的物理内存在系统启动后就已经分配好，就是 `envs`数组。

和Lab2 对页控制块数组的管理类似，我们使用链表管理进程控制块数组。struct Env 中的链表项共涉及调度队列 `env_sched_list`和空闲队列 `env_free_list`两个队列：

* `env_sched_list`：管理已经被分配的进程控制块和对进程的调度。在进程创建时需要为其分配进程控制块并加入 `env_sched_list`，在进程被释放时需要将其对应的进程控制块从 `env_sched_list`移出。

其类型 `TAILQ_ENTRY(Env)`涉及到另一个宏定义的数据结构 `TALIQ`，是一种双端队列，代码如下：

```c
#define _TAILQ_ENTRY(type, qual)                                                                   \
	struct {                                                                                   \
		qual type *tqe_next;	   /* next element */                                      \
		qual type *qual *tqe_prev; /* address of previous next element */                  \
	}
#define TAILQ_ENTRY(type) _TAILQ_ENTRY(struct type, )
```

在构造时，qual是空值，所以相当于构造了一个结构体，其中的两个指针分别指向前一个元素和后一个元素。

* `env_free_list`：空闲进程控制块序列。

### 二、进程的标识

#### 1、识别进程和地址空间

`env.c` 文件中实现了一个 `mkenvid` 函数，作用就是生成一个新的进程 `env_id`。虽然 `env_id`已经能够唯一标识进程，但是还需要引入 `env_asid`来唯一标识虚拟地址空间，用于TLB刷新。











---

---

## Exercise笔记

### 3.1

题目：

```
完成env_init 函数。
实现 Env 控制块的空闲队列和调度队列的初始化功能。请注意，你需要按倒序将所有
控制块插入到空闲链表的头部，使得编号更小的进程控制块被优先分配。
```

我的代码：

```c
void env_init(void) {
	int i;
	/* Step 1: Initialize 'env_free_list' with 'LIST_INIT' and 'env_sched_list' with* 'TAILQ_INIT'. */
	LIST_INIT(&env_free_list);
	TAILQ_INIT(&env_sched_list);

	/* Step 2: Traverse the elements of 'envs' array, set their status to 'ENV_FREE' and insert
	 * them into the 'env_free_list'. Make sure, after the insertion, the order of envs in the
	 * list should be the same as they are in the 'envs' array. */
	for(i=NENV-1;i>=0;i--) {
		env[i].status=ENV_FREE;
		LIST_INSERT_HEAD(&env_free_list, &env[i], env_tf);
	}

	struct Page *p;
	panic_on(page_alloc(&p));
	p->pp_ref++;

	base_pgdir = (Pde *)page2kva(p);
	map_segment(base_pgdir, 0, PADDR(pages), UPAGES,
		    ROUND(npage * sizeof(struct Page), PAGE_SIZE), PTE_G);
	map_segment(base_pgdir, 0, PADDR(envs), UENVS, ROUND(NENV * sizeof(struct Env), PAGE_SIZE),
		    PTE_G);
}
```

这一题的要求先是利用两个 `INIT`相关宏初始化空闲进程表和调用进程表，之后利用宏将 `envs`数组里面的元素倒序插入空闲进程列表（这样能保证两个表中的元素顺序相同）。

### 3.2

可以看到 `env_init`函数中在设置空闲进程列表后使用 `page_alloc`函数为模版页表 `base_pgdir`分配了一页物理内存，将其转换为内核虚拟地址，并使用 `map_segment` 函数在该页表中将内核数组 `pages` 和 `envs` 映射到了用户空间的 `UPAGES` 和 `UENVS` 处。在之后的 `env_setup_vm` 函数中，我们会将这部分模板页表复制到每个进程的页表中。

那么这道题的要求就是：

```c
请你结合env_init 中的使用方式，完成map_segment 函数。
```

我的代码：

```c
static void map_segment(Pde *pgdir, u_int asid, u_long pa, u_long va, u_int size, u_int perm) {

	assert(pa % PAGE_SIZE == 0);
	assert(va % PAGE_SIZE == 0);
	assert(size % PAGE_SIZE == 0);

	/* Step 1: Map virtual address space to physical address space. */
	for (int i = 0; i < size; i += PAGE_SIZE) {
		/*
		 * Hint:
		 *  Map the virtual page 'va + i' to the physical page 'pa + i' using 'page_insert'.*/
		page_insert(pgdir, asid, pa2page(pa+i), va+i, perm);
	}
}
```

这里已经有提示，按照提示直接调用 `page_insert`函数即可。

### 3.3+3.4

在初始化后，就可以开始创建进程：

* 从 `env_free_list`中获取一个空闲 `PCB`块
* 手动设置 `PCB`属性
* 初始化新进程页目录
* 从空闲链表中摘出使用

**以下是 `env_alloc`函数，用于分配并设置 `PCB`属性：**

```c
int env_alloc(struct Env **new, u_int parent_id) {
	int r;
	struct Env *e;

	/* Step 1: Get a free Env from 'env_free_list' */
	if(LIST_EMPTY(&env_free_list)){
		return -E_NO_FREE_ENV;
	}
	e=LIST_FIRST(&env_free_list);

	/* Step 2: Call a 'env_setup_vm' to initialize the user address space for this new Env. */
	if((r=env_setup_vm(e))!=0){
		return r;
	}

	/* Step 3: Initialize these fields for the new Env with appropriate values:*/
	e->env_user_tlb_mod_entry = 0; // for lab4
	e->env_runs = 0;	       // for lab6
	/* Exercise 3.4: Your code here. (3/4) */
	e->env_id=mkenvid(e);
	e->env_parent_id=parent_id;
	if((r=asid_alloc(&(e->asid)))!=0){
		return r;
	}

	/* Step 4: Initialize the sp and 'cp0_status' in 'e->env_tf'.*/
	e->env_tf.cp0_status = STATUS_IM7 | STATUS_IE | STATUS_EXL | STATUS_UM;
	// Reserve space for 'argc' and 'argv'.
	e->env_tf.regs[29] = USTACKTOP - sizeof(int) - sizeof(char **);

	/* Step 5: Remove the new Env from env_free_list. */
	LIST_REMOVE(e, env_link);
	*new = e;
	return 0;
}
```

这里在每次调用函数或者操作列表的时候都要注意判断，列表是否为空，调用函数是否成功，我开始的时候漏下了好几处。

* 第一步，根据提示，取空闲 `PCB`列表第一块
* 第二步，调用下面要写的函数 `env_setup_vm`
* 第三、四步，设置 `PCB`结构体属性
* 第五步，将这个 `PCB`移出空闲列表并返回

其中第四步较为难懂，对 `e->env_tf`进行了设置，其类型结构体定义如下

```c
struct Trapframe {
	/* Saved main processor registers. */
	unsigned long regs[32];

	/* Saved special registers. */
	unsigned long cp0_status;
	unsigned long hi;
	unsigned long lo;
	unsigned long cp0_badvaddr;
	unsigned long cp0_cause;
	unsigned long cp0_epc;
};
```

`env_alloc`函数对其中的 `cp0_stauts`寄存器和 `reg[29]`进行了修改：

```c
/* Step 4: Initialize the sp and 'cp0_status' in 'e->env_tf'.
	 *   Set the EXL bit to ensure that the processor remains in kernel mode during context
	 * recovery. Additionally, set UM to 1 so that when ERET unsets EXL, the processor
	 * transitions to user mode.
	 */
	e->env_tf.cp0_status = STATUS_IM7 | STATUS_IE | STATUS_EXL | STATUS_UM;
	// Reserve space for 'argc' and 'argv'.
	e->env_tf.regs[29] = USTACKTOP - sizeof(int) - sizeof(char **);
```

* 所修改的29号寄存器是 `$sp`寄存器，记录程序栈顶，为了给 `main`函数的argc和 `argv`留下空间，减去了这两段空间。
* `IE`位表示中断是否开启，为 1 表示开启，否则不开启。我们需要将 `IE`及 `IM7`设置为1，表示中断使能，且7 号中断（时钟中断）可以被响应。
* `EXL`以及 `UM`表示了处理器当前的运行状态。当且仅当 `EXL` 被设置为0 且 `UM` 被设置为1 时，处理器处于用户模式，其它所有情况下，处理器均处于内核模式下。每当异常发生的时候，`EXL` 会被自动设置为1，并由异常处理程序负责后续处理。

由于只有 `UM==1 && EXL==0`的时候处于用户态，而在系统调用的时候总会执行：

```c
RESTORE_ALL
eret
```

其中第一句是用来完成对处理器寄存器状态的恢复的，会将TF中存储的 `Status`写入系统 `Status`需要在内核态下运行，而 `eret`执行时会将 `EXL`置0，所以令 `EXL=1`，`UM=1`是为了在 `RESTORE_ALL`时保持内核态，防止陷入异常，并且在执行 `eret`之后里面进入用户态。

**下面来补全env_setup_vm：**

```c
static int env_setup_vm(struct Env *e) {
	/* Step 1:
	 *   Allocate a page for the page directory with 'page_alloc'.
	 *   Increase its 'pp_ref' and assign its kernel address to 'e->env_pgdir'.*/
	struct Page *p;
	try(page_alloc(&p));
	p->pp_ref++;
	e->env_pgdir=(Pde*)page2kva(p);
	/* Step 2: Copy the template page directory 'base_pgdir' to 'e->env_pgdir'. */
	memcpy(e->env_pgdir + PDX(UTOP), base_pgdir + PDX(UTOP),
	       sizeof(Pde) * (PDX(UVPT) - PDX(UTOP)));

	/* Step 3: Map its own page table at 'UVPT' with readonly permission.
	 * As a result, user programs can read its page table through 'UVPT' */
	e->env_pgdir[PDX(UVPT)] = PADDR(e->env_pgdir) | PTE_V;
	return 0;
}
```

只需按要求，给 `p->pp_ref`也就是该页的映射次数加一，并且将这个所分配的页的内核态虚拟地址赋值给 `e->env_pgdir`即可。

### 3.5+3.6

在创建进程之后，我们要将对应的程序加载到新进程的地址空间中，这里使用的函数是load_icode：

```c
static void load_icode(struct Env *e, const void *binary, size_t size) {
	/* Step 1: Use 'elf_from' to parse an ELF header from 'binary'. */
	const Elf32_Ehdr *ehdr = elf_from(binary, size);
	if (!ehdr) {
		panic("bad elf at %x", binary);
	}

	/* Step 2: Load the segments using 'ELF_FOREACH_PHDR_OFF' and 'elf_load_seg'.
	 * As a loader, we just care about loadable segments, so parse only program headers here.
	 */
	size_t ph_off;
	ELF_FOREACH_PHDR_OFF (ph_off, ehdr) {
		Elf32_Phdr *ph = (Elf32_Phdr *)(binary + ph_off);
		if (ph->p_type == PT_LOAD) {
			// 'elf_load_seg' is defined in lib/elfloader.c
			// 'load_icode_mapper' defines the way in which a page in this segment
			// should be mapped.
			panic_on(elf_load_seg(ph, binary + ph->p_offset, load_icode_mapper, e));
		}
	}

	/* Step 3: Set 'e->env_tf.cp0_epc' to 'ehdr->e_entry'. */
	e->env_tf.cp0_epc=edhr->e_rntey;
}
```

这里需要填写的部分就是最后一句话，已经很明确的写出来了。

下面来看这个函数的逻辑，首先调用 `elf_from`函数对 `ELF`文件进行解析，得到文件头：

```c
const Elf32_Ehdr *elf_from(const void *binary, size_t size) {
	const Elf32_Ehdr *ehdr = (const Elf32_Ehdr *)binary;
	if (size >= sizeof(Elf32_Ehdr) && ehdr->e_ident[EI_MAG0] == ELFMAG0 &&
	    ehdr->e_ident[EI_MAG1] == ELFMAG1 && ehdr->e_ident[EI_MAG2] == ELFMAG2 &&
	    ehdr->e_ident[EI_MAG3] == ELFMAG3 && ehdr->e_type == 2) {
		return ehdr;
	}
	return NULL;
}
```

这个函数其实就是简单的 `ELF`文件头魔术识别，并返回文件头。

接下来 `load_icode`中使用了一个宏：

```c
#define ELF_FOREACH_PHDR_OFF(ph_off, ehdr)                                                         \
	(ph_off) = (ehdr)->e_phoff;                                                                \
	for (int _ph_idx = 0; _ph_idx < (ehdr)->e_phnum; ++_ph_idx, (ph_off) += (ehdr)->e_phentsize)
```

这个宏在函数中的作用是循环取出程序头，如果其中的 `p_type`类型为 `PT_LOAD`，那么就说明其对应的程序需要被加载到内存里，这个时候就调用 `elf_load_seg`函数来加载。

那么接下来来看 `elf_load_seg`函数：

```c
int elf_load_seg(Elf32_Phdr *ph, const void *bin, elf_mapper_t map_page, void *data) {
	u_long va = ph->p_vaddr;
	size_t bin_size = ph->p_filesz;
	size_t sgsize = ph->p_memsz;
	u_int perm = PTE_V;
	if (ph->p_flags & PF_W) {
		perm |= PTE_D;
	}

	int r;
	size_t i;
	u_long offset = va - ROUNDDOWN(va, PAGE_SIZE);
	if (offset != 0) {
		if ((r = map_page(data, va, offset, perm, bin,
				  MIN(bin_size, PAGE_SIZE - offset))) != 0) {
			return r;
		}
	}

	/* Step 1: load all content of bin into memory. */
	for (i = offset ? MIN(bin_size, PAGE_SIZE - offset) : 0; i < bin_size; i += PAGE_SIZE) {
		if ((r = map_page(data, va + i, 0, perm, bin + i, MIN(bin_size - i, PAGE_SIZE))) !=
		    0) {
			return r;
		}
	}

	/* Step 2: alloc pages to reach `sgsize` when `bin_size` < `sgsize`. */
	while (i < sgsize) {
		if ((r = map_page(data, va + i, 0, perm, NULL, MIN(sgsize - i, PAGE_SIZE))) != 0) {
			return r;
		}
		i += PAGE_SIZE;
	}
	return 0;
}
```

这个函数开始讲程序头表里面的数据提取出来，之后先进行对齐,将开头那部分不对齐的数据先利用回调函数 `load_icode_mapper`映射到页面里。

随后处理中间对齐的部分，最后对于“在内存中大小"大于”在文件中大小"的部分，只加载页面，无需填写数据。

接下来终于到了需要填写的部分——`load_icode_mapper`函数：

```c
static int load_icode_mapper(void *data, u_long va, size_t offset, u_int perm, const void *src,
			     size_t len) {
	struct Env *env = (struct Env *)data;
	struct Page *p;
	int r;

	/* Step 1: Allocate a page with 'page_alloc'. */
	if((r=page_alloc(&p))!=0) {
		return r;
	}

	/* Step 2: If 'src' is not NULL, copy the 'len' bytes started at 'src' into 'offset' at this
	 * page. */
	// Hint: You may want to use 'memcpy'.
	if (src != NULL) {
		/* Exercise 3.5: Your code here. (2/2) */
		memcpy((void*)(page2kva(p)+offset), src, len);
	}

	/* Step 3: Insert 'p' into 'env->env_pgdir' at 'va' with 'perm'. */
	return page_insert(env->env_pgdir, env->env_asid, p, va, perm);
}
```
