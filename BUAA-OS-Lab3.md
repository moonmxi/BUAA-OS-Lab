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

* 首先是 `env_tf`，它的作用是在发生进程调度，或当陷入内核时，会将当时的进程上下文环境保存下来。其类型 `Trapframe`代码如下：

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
