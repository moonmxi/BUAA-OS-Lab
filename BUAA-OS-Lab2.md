# Lab2

## 一、指导书学习笔记

### 1、MIPS 4Kc的地址映射

在4Kc 上，软件访存的虚拟地址会先被MMU 硬件映射到物理地址，随后使用物理地址来访问内存或其他外设。与本实验相关的映射与寻址规则（内存布局）如下：

* 若虚拟地址处于 `0x80000000~0x9fffffff` (kseg0)，则将虚拟地址的最高位置0 得到物理地址，通过cache 访存。这一部分用于存放内核代码与数据。
* 若虚拟地址处于 `0xa0000000~0xbfffffff` (kseg1)，则将虚拟地址的最高3位置0 得到物理地址，不通过cache 访存。这一部分可以用于访问外设。
* 若虚拟地址处于 `0x00000000~0x7fffffff` (kuseg)，则需要通过TLB 转换成物理地址，再通过cache 访存。这一部分用于存放用户程序代码与数据。

以上的 **“将虚拟地址最高xx位置0"** 指的是 **二进制下** 的地址，而非写出来的十六进制（这里我想了好久，才发现一直在疑惑的点实际是我一开始就想错了）

在4Kc 中，使用MMU 来完成上述地址映射，MMU 采用硬件TLB 来完成地址映射。TLB需要由软件进行填写，即操作系统内核负责维护TLB 中的数据。所有对低2GB 空间（kuseg）的内存访问操作都需要经过TLB。

### 2、初始化分配内存

在未建立内存管理机制的时候，需要手动分配内存。

根据映射规则，0x80400000 对应的物理地址是0x400000。在物理地址0x400000 的前面，存放着操作系统内核的代码和定义的全局变量或数组（还额外保留了一些空间）。接下来将从物理地址 0x400000 开始分配物理内存，用于建立管理内存的数据结构。

* 将 `a`按 `n`对齐

  ```
  #define ROUND(a, n) (((((u_long)(a)) + (n)-1)) & ~((n)-1))
  ```

### 3、链表宏

* `LIST_HEAD(name, type)`，创建一个名称为name 链表的头部结构体，包含一个指向type类型结构体的指针，这个指针可以指向链表的首个元素。
* `LIST_ENTRY(type)`，作为一个特殊的类型出现，例如可以进行如下的定义：

  ```
  LIST_ENTRY(Page) a;
  ```

  它的本质是一个链表项，包括指向下一个元素的指针le_next，以及指向前一个元素链表项 `le_next` 的指针 `le_prev`。`le_prev` 是一个指针的指针，它的作用是当删除一个元素时，更改前一个元素链表项的 `le_next`。
* `LIST_EMPTY(head)`，判断head 指针指向的头部结构体对应的链表是否为空。
* `LIST_FIRST(head)`，将返回head 对应的链表的首个元素。
* `LIST_INIT(head)`，将head 对应的链表初始化。
* `LIST_NEXT(elm, field)`，返回指针elm 指向的元素在对应链表中的下一个元素的指针。`elm` 指针指向的结构体需要包含一个名为 `field` 的字段，类型是一个链表项 `LIST_ENTRY(type)`，下面出现的 `field` 含义均和此相同。
* `LIST_INSERT_AFTER(listelm, elm, field)`，将elm 插到已有元素listelm 之后。
* `LIST_INSERT_BEFORE(listelm, elm, field)`，将 `elm` 插到已有元素 `listelm` 之前。
* `LIST_INSERT_HEAD(head, elm, field)`，将 `elm`插到 `head` 对应链表的头部。
* `LIST_REMOVE(elm, field)`，将 `elm` 从对应链表中删除。

---

---

## Exercise笔记

![B3log Logo](/Users/mawenban/Desktop/CPU-TLB-Memory.png)

这次Lab的题目关联性比较强，没法按题号来写，所以就直接按Boot调用流程来写

在跳转 `init.c`之后，调用 `mips_init`函数，这个函数调用三个函数：

* mips_detect_memory(u_int _memsize) ，作用是探测硬件可用内存，并对一些和内存管
  理相关的变量进行初始化。
* mips_vm_init() ，作用是为内存管理机制作准备，建立一些用于管理的数据结构。
* page_init()，实现位于kern/pmap.c 中，作用是初始化pages 数组中的Page 结构体以
  及空闲链表。这个函数的具体功能会在后面详细描述。

### 首先是mips_detect_memory(u_int _memsize)

```
u_long npage;	       /* Amount of memory(in pages) */

void mips_detect_memory(u_int _memsize) {
	/* Step 1: Initialize memsize. */
	memsize = _memsize;

	/* Step 2: Calculate the corresponding 'npage' value. */
	/* Exercise 2.1: Your code here. */

	npage=memsize/PAGE_SIZE;

	printk("Memory size: %lu KiB, number of pages: %lu\n", memsize / 1024, npage);
}
```

这里有一个需要插入的地方，提示为 **计算相应的 `npage`值** ，即计算总物理页数。
其中 `memsize`为总物理内存对应的字节数，接收传入的 `u_int _memsize`并初始化。

已知物理内存总字节数，求总物理页数，只需要知道每页字节数，那么就开始找（英文确实不好找），
最后在 `mmu.h`中可以找到

```
#define PAGE_SIZE 4096
```

物理页字节数为4K，除物理内存总字节数即可求得`npage`。

### 接下来是mips_vm_init()

```
/* Overview:
    Set up two-level page table. */
void mips_vm_init() {
	/* Allocate proper size of physical memory for global array `pages`,
	 * for physical memory management. Then, map virtual address `UPAGES` to
	 * physical address `pages` allocated before. For consideration of alignment,
	 * you should round up the memory size before map. */
	pages = (struct Page *)alloc(npage * sizeof(struct Page), PAGE_SIZE, 1);
	printk("to memory %x for struct Pages.\n", freemem);
	printk("pmap.c:\t mips vm init success\n");
}
```

其主要功能是用 `alloc(u_intn, u_intalign, intclear)`函数为数组 `pages`分配内存，要考虑对齐。

其内容如下

```
void *alloc(u_int n, u_int align, int clear) {
	extern char end[];
	u_long alloced_mem;

	if (freemem == 0) {
		freemem = (u_long)end; // end
	}

	freemem = ROUND(freemem, align);

	alloced_mem = freemem;

	freemem = freemem + n;

	panic_on(PADDR(freemem) >= memsize);

	if (clear) {
		memset((void *)alloced_mem, 0, n);
	}

	return (void *)alloced_mem;
}
```

此函数的功能就是用于分配内存空间（在建立页式内存管理机制之前使用）。
这段代码的作用是分配n 字节的空间并返回初始的虚拟地址，同时将地址按 align 字节对齐（保证align 可以整除初始虚拟地址），若clear 为真，则将对应内存空间的值清零，否则不清零。

看到这里我们可能有一个疑问：现在还没有建立内存管理机制，那么内核是如何操作内存的呢？
考虑 `kseg0`段的性质，该段上的虚拟地址被线性映射到物理地址，操作系统通过访问该段的地址来直接操作物理内存，从而逐步建立内存管理机制。例如当写虚拟地址 `0x80012340`时，由于 `kseg0` 段的性质，事实上在写物理地址 `0x12340`。

下面逐行来解析这个函数：

`extern char end[]` 是Lab1中连接器`kernel.lds`脚本所初始化的

```
. = 0x80400000;
end = . ;
```

即初始化内核的时候给内核所分配的结束地址（虚拟地址），接下来将从物理地址 0x400000 开始分配物理内存，用于建立管
理内存的数据结构 `pages`。

`alloc`函数中用一个 `char`数组去获取 `end`并用强制类型转换取其地址到 `freedom`。

之后将 `freedom`按 `allign`对齐，`ROUND`函数其实就是一个向上取整的高斯函数，使得 `freedom`被 `allign`整除。

已经被分配的内存 `alloced_mem`即为此时的 `freedom`，

随后在 `freedom`之后分配大小为 `n`字节的内存并置0，

最后返回所分配的内存首地址指针 `alloced_mem`。

这段代码里面的`PADDR`引出了一系列虚拟地址和物理地址之间转换的函数

```
#define ULIM 0x80000000
//大概是UserLimit，即给用户分配空间的极限值，此地址之后是内核所占用的。

#define PADDR(kva)                                                                                 \
	({                                                                                         \
		u_long _a = (u_long)(kva);                                                         \
		if (_a < ULIM)                                                                     \
			panic("PADDR called with invalid kva %08lx", _a);                          \
		_a - ULIM;                                                                         \
	})

// translates from physical address to kernel virtual address
#define KADDR(pa)                                                                                  \
	({                                                                                         \
		u_long _ppn = PPN(pa);                                                             \
		if (_ppn >= npage) {                                                               \
			panic("KADDR called with invalid pa %08lx", (u_long)pa);                   \
		}                                                                                  \
		(pa) + ULIM;                                                                       \
	})

```

---

---

## 思考题

### 2.1

* 题目：

```
请根据上述说明，回答问题：在编写的C 程序中，指针变量中存储的地址被视为虚拟地址，还是物理地址？MIPS 汇编程序中 lw 和sw 指令使用的地址被视为虚拟地址，还是物理地址？
```

* 答案：

```
均被视为虚拟地址
```

### 2.2
