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

### 4、两级页表机制

MOS 中用 `PADDR` 与 `KADDR` 这两个宏可以对位于 `kseg0` 的虚拟地址和对应的物理地址进行转换。
但是，对于位于 `kuseg` 的虚拟地址，MOS 中采用两级页表结构对其进行地址转换。

两级页表也就是给页表再加一个页表，原来的页表是二级页表，二级页表的页表是一级页表，也叫页目录。

那么对于32位的虚拟地址，从低到高从0 开始编号，其31-22 位表示的是一级页表项的偏移量，21-12 位表示的是二级页表项的偏移量，11-0 位表示的是页内偏移量。

访问虚拟地址时，先通过一级页表基地址和一级页表项的偏移量，找到对应的一级页表项，得到对应的二级页表的物理页号，再根据二级页表项的偏移量找到所需的二级页表项，进而得到该虚拟地址对应的物理页号。

MIPS 4Kc 发出的地址均为虚拟地址，因此如果程序想访问某个物理地址，需要通过映射到该物理地址的虚拟地址来访问。对页表进行操作时硬件处于内核态，因此使用宏 `KADDR` 获得其位于 `kseg0` 中的虚拟地址即可完成转换。

无论是一级页表还是二级页表，它们的结构都一样，只不过每个页表项记录的物理页号含义有所不同。每个页表均由1024 个页表项组成，每个页表项由32 位组成，包括20 位物理页号以及12 位标志位。其中，12 位标志位包含高6 位硬件标志位与低6 位软件标志位。
当页表项需要借助EntryLo 寄存器填入TLB 时，页表项会被右移6 位，移出所有软件标志位，仅将高20 位物理页号以及6 位硬件标志位填入TLB 使用。每个页表所占的空间为4KB，恰好为一个物理页面的大小。

### 5、两级页表项的12位标识位

高6 位硬件标志位用于存入 `EntryLo` 寄存器中，供硬件使用；
低6 位软件标志位不会被存入 TLB 中，仅供软件使用。

以下是六位硬件标志位：

```c
PTE_V 有效位，若某页表项的有效位为1，则该页表项有效，其中高20 位就是对应的物理页号。

PTE_D 可写位，若某页表项的可写位为1，则允许经由该页表项对物理页进行写操作。

PTE_G 全局位，若某页表项的全局位为1，则TLB 仅通过虚页号匹配表项，而不匹配ASID，将在Lab3 中用于映射pages 和envs 到用户空间。本Lab 中可以忽略。

PTE_C_CACHEABLE 可缓存位，配置对应页面的访问属性为可缓存。通常对于所有物理页面，都将其配置为可缓存，以允许CPU 使用cache 加速对这些页面的访存请求。

PTE_COW 写时复制位，将在Lab4 中用到，用于实现 fork 的写时复制机制。本Lab中可以忽略。

PTE_LIBRARY 共享页面位，将在Lab6 中用到，用于实现管道机制。本Lab 中可以忽略。
```

### 6、4Kc 中与内存管理相关的CP0 寄存器：

```c
8 		BadVaddr 				保存引发地址异常的虚拟地址
10、2、3 	EntryHi、EntryLo0、EntryLo1		所有读写TLB 的操作都要通过这三个寄存器，详见下一小节
0 		Index 					TLB 读写相关需要用到该寄存器
1 		Random					随机填写TLB 表项时需要用到该寄存器
```

### 7、EntryHi、EntryLo0、EntryLo1

EntryHi、EntryLo0、EntryLo1 都是CP0 中的寄存器，他们只是分别对应到TLB 的Key与两组Data，并不是TLB 本身。

其中 EntryLo0、EntryLo1 拥有完全相同的位结构，EntryLo0 存储 Key 对应的偶页而EntryLo1 存储Key 对应的奇页。

### 8、TLB相关指令

* `tlbr`：以 `Index` 寄存器中的值为索引，读出 `TLB` 中对应的表项到 `EntryHi` 与 `EntryLo0`、`EntryLo1`。
* `tlbwi`：以 `Index` 寄存器中的值为索引，将此时 `EntryHi` 与 `EntryLo0`、`EntryLo1` 的值写到索引指定的 `TLB` 表项中。
* `tlbwr`：将 `EntryHi` 与 `EntryLo0`、`EntryLo1` 的数据随机写到一个 `TLB` 表项中（此处使用 `Random` 寄存器来“随机”指定表项，`Random` 寄存器本质上是一个不停运行的循环计数器）。
* `tlbp`：根据 `EntryHi` 中的 `Key`（包含 `VPN` 与 `ASID`），查找 `TLB` 中与之对应的表项，并将表项的索引存入 `Index` 寄存器（若未找到匹配项，则 `Index` 最高位被置1）。

---

---

## Exercise笔记

![B3log Logo](/Users/mawenban/Desktop/CPU-TLB-Memory.png)

这次Lab的题目关联性比较强，没法按题号来写，所以就直接按Boot调用流程来写

在跳转 `init.c`之后，调用 `mips_init`函数，这个函数调用三个函数：

* **`mips_detect_memory(u_int _memsize)`** ，作用是探测硬件可用内存，并对一些和内存管
  理相关的变量进行初始化。
* **`mips_vm_init()`** ，作用是为内存管理机制作准备，建立一些用于管理的数据结构。
* **`page_init()`** ，实现位于kern/pmap.c 中，作用是初始化pages 数组中的Page 结构体以
  及空闲链表。这个函数的具体功能会在后面详细描述。

---

### 首先是mips_detect_memory(u_int _memsize)

```c
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

物理页字节数为4K，除物理内存总字节数即可求得 `npage`。

---

### 接下来是mips_vm_init()

```c
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

```c
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

`extern char end[]` 是Lab1中连接器 `kernel.lds`脚本所初始化的

```
. = 0x80400000;
end = . ;
```

即初始化内核的时候给内核所分配的结束地址（虚拟地址），接下来将从物理地址 0x400000 开始分配物理内存，用于建立管
理内存的数据结构 `pages`。

* `alloc`函数中用一个 `char`数组去获取 `end`并用强制类型转换取其地址到 `freedom`。
* 之后将 `freedom`按 `allign`对齐，`ROUND`函数其实就是一个向上取整的高斯函数，使得 `freedom`被 `allign`整除。
* 已经被分配的内存 `alloced_mem`即为此时的 `freedom`，
* 随后在 `freedom`之后分配大小为 `n`字节的内存并置0，
* 最后返回所分配的内存首地址指针 `alloced_mem`。

这段代码里面的 `PADDR`引出了一系列虚拟地址和物理地址之间转换的函数

```
//大概是UserLimit，即给用户分配空间的极限值，此地址之后是内核所占用的。
#define ULIM 0x80000000

//将内核虚拟地址转换为物理地址kernel->physical
#define PADDR(kva)                                                                                 \
	({                                                                                         \
		u_long _a = (u_long)(kva);                                                         \
		if (_a < ULIM)                                                                     \
			panic("PADDR called with invalid kva %08lx", _a);                          \
		_a - ULIM;                                                                         \
	})

//将物理地址转换为内核虚拟地址physical->kernel
#define KADDR(pa)                                                                                  \
	({                                                                                         \
		u_long _ppn = PPN(pa);                                                             \
		if (_ppn >= npage) {                                                               \
			panic("KADDR called with invalid pa %08lx", (u_long)pa);                   \
		}                                                                                  \
		(pa) + ULIM;                                                                       \
	})

```

那么现在回头看mips_vm_init()中所调用的

```c
pages = (struct Page *)alloc(npage * sizeof(struct Page), PAGE_SIZE, 1);
```

深入研究后我们发现，这条语句底层是在
将虚拟内存中代表空闲空间截止地址的 `freedom`增加 `npage——物理总页数`个页的大小，即将其后移对应大小的位移，
同时，按照 `PAGE_SIZE——物理页大小`来对齐，并对所分配的内存清零。

---

### 接下来是page_init()

这个函数的作用是初始化空闲列表页

在开始这个函数的解析之前，就不得不先来探究这个 **页结构体** 。

MOS 中维护了 `npage` 个页控制块，也就是 `Page` 结构体。每一个页控制块对应一页的物理内存，MOS 用这个结构体来按页管理物理内存的分配。

在 **一、3、** 中，已经列出来了大部分有关这些结构体的宏，但是其实最难理解的是页结构体本身！！！

***LIST_ENTRY(type)！！！***

链表实体：

```
#define LIST_ENTRY(type)                                                                           \
	struct {                                                                                   \
		struct type *le_next;  /* next element */                                          \
		struct type **le_prev; /* address of previous next element */                      \
	}
```

这里其实类似Java等高级语言中的泛型概念，只不过是用宏（替换）来实现的。

这里有两个指针，字面意思，一个指针指向下一个链表实体，另一个指针指向上一个链表实体。

但是，最烧脑的来了！其中 `le_prev`这个指针，它的值并不是链表中上个结构体的地址，而是链表中上一个结构体指针 `le_next`的地址，
更进一步说，`*le_prev`的值是链表中上一个结构体的 `le_next`，即 `**le_prev`的值对应的是当前链表项。

如图：

![img](/Users/mawenban/Desktop/LIST_ENTRY.png)

**但是！！！**

这还没有结束：

```c
typedef LIST_ENTRY(Page) Page_LIST_entry_t;

struct Page {
	Page_LIST_entry_t pp_link; /* free list link */

	u_short pp_ref;
};
```

这段代码的意思是：

```c
struct {                                                                                   \
		struct type *le_next;  /* next element */                                  \
		struct type **le_prev; /* address of previous next element */              \
	} Page_LIST_entry_t;

struct Page {
	Page_LIST_entry_t pp_link; /* free list link */

	u_short pp_ref;
};
```

这里的 `Page_LIST_entry_t`相当于 `Page`结构体双向链表中的两个指针，只不过实现方式略有不同，
也就是说，`Page`结构体就是一个合格的链表项形式，
其中的 `pp_ref`是后面要用到的标记数字，代表这一页物理内存被引用的次数，它等于有多少虚拟页映射到该物理页。

---

#### 趁热打铁，恰好下一道练习题就是经典的链表插入。

```
#define LIST_INSERT_AFTER(listelm, elm, field) 						\
 	do { 										\
	 	if ((LIST_NEXT((elm), field) = LIST_NEXT((listelm), field)) != NULL)    \
		LIST_NEXT((listelm), field)->field.le_prev = &LIST_NEXT((elm), field);  \
	 	LIST_NEXT((listelm), field) = (elm); 					\
	 	(elm)->field.le_prev = &LIST_NEXT((listelm), field); 			\
 	} while (0)
```

这里和初学数据结构时候处理双向链表插入很相似，需要考虑的很完全，
这里的LIST_NEXT()其实就是取指针换了个写法：

```
#define LIST_NEXT(elm, field) ((elm)->field.le_next)
```

这里后续传入的 `elm`都是指针类型，比如 `Page*`，所以有 `(elm)->field.le_prev`这样的写法，
以及 `LIST_NEXT((listelm), field) = (elm);`这样的语句。

---

接着，我们回到了物理内存管理。

**Note：** `npage` 个 `Page` 和 `npage` 个物理页面一一顺序对应。具体来说，用一个数组存放这些 `Page` 结构体，首个 `Page` 的地址为 `P`，则 `P[i] `对应从0 开始计数的第 `i`个物理页面。`Page` 与其对应的物理页面地址的转换可以使用 `include/pmap.h` 中的 `page2pa` 和 `pa2page` 这两个函数。

下面我们来看 `include/pmap.h` 中这几个地址之间相互转换的函数

```c
static inline u_long page2ppn(struct Page *pp) {
	return pp - pages;
}

static inline u_long page2pa(struct Page *pp) {
	return page2ppn(pp) << PGSHIFT;
}

static inline struct Page *pa2page(u_long pa) {
	if (PPN(pa) >= npage) {
		panic("pa2page called with invalid pa: %x", pa);
	}
	return &pages[PPN(pa)];
}

static inline u_long page2kva(struct Page *pp) {
	return KADDR(page2pa(pp));
}
```

* `page2ppn`：由于 `pp`和 `pages`都是 `Page*`类型，所以二者相减就是 `pp`在数组中的索引，又由于 `pages`是物理页控制块数组，其中元素与物理页一一对应，所以 `pp`的索引就是物理页号。
* `page2pa`：这里的 `PGSHIFT`定义在 `mmu.h`：`#define PGSHIFT 12`，将页经过上一个函数来获得页号，再左移十二位，就取到了前二十位，也就是这个页的页号（物理页共1M个，即2^20个）。
* `pa2page`：通过页地址取得页号，在不超过物理总页数的情况下，在 `pages`数组中取得对应的页。
* `page2kva`：先将页通过 `page2pa`函数转化为页物理地址，再通过 `KADDR`宏，转化为内核态使用的虚拟地址。

以下是重要结构体 `Page_list`定义：

```c
struct Page_list{
	struct Page{
		struct {                                                                                   \
			struct type *le_next;  /* next element */                                  \
			struct type **le_prev; /* address of previous next element */              \
		} pp_link;
		u_short pp_ref;
	}* lh_first;
}
```

---

#### 现在终于开始page_init()函数，话不多说，先上代码：

```c
void page_init(void) {
	/* Step 1: Initialize page_free_list. */
	LIST_INIT(&page_free_list);

	/* Step 2: Align `freemem` up to multiple of PAGE_SIZE. */
	freemem = ROUND(freemem, PAGE_SIZE);

	/* Step 3: Mark all memory below `freemem` as used (set `pp_ref` to 1) */
	u_long usedpage = PPN(PADDR(freemem));

	for (u_long i = 0; i < usedpage; i++) {
		pages[i].pp_ref = 1;
	}

	/* Step 4: Mark the other memory as free. */
	for (u_long i = usedpage; i < npage; i++) {
		pages[i].pp_ref = 0;
		LIST_INSERT_HEAD(&page_free_list, &pages[i], pp_link);
	}
}
```

* 首先是初始化空闲物理块数组 `page_free_list`，这里需要注意的是，`page_free_list`经过层层包装，并不是一个指针类型，而 `LIST_INIT`这个宏需要使用 `->`来获取域，即需要传入指针类型，故要加上取址符号 `&`。
* 然后和在 `alloc`函数中一样，对可用空间起始位置 `freedom`按 `PAGE_SIZE`对齐。
* 随后获取 `freedom`地址对应的页号，将这个页号之前的页都标记为已经使用，即 `pp_ref`置1。
* 最后把剩下的页设置为未使用，即 `pp_ref`置0，并插入到 `page_free_list`数组中。

在有了上面全部的预备知识之后，这道题便可以迎刃而解了，只需要注意各个函数以及宏的传参规范。

---

### 接下来是page_alloc()函数，继续上代码：

```c
int page_alloc(struct Page **new) {
	/* Step 1: Get a page from free memory. If fails, return the error code.*/
	struct Page *pp;

	if(LIST_EMPTY(&page_free_list)) {
		return -E_NO_MEM;
	}
	pp=LIST_FIRST(&page_free_list);

	LIST_REMOVE(pp, pp_link);

	// Step 2: Initialize this page with zero.
	memset((void *)(page2kva(pp)), 0, sizeof(struct Page));

	*new = pp;
	return 0;
}
```

* 首先查看空闲页列表是否为空
* 若非空则用 `pp`取链表第一项
* 再利用 `LIST_REMOVE`宏把这一项从链表从移除
* 最后用 `memset`将这段地址置0

接下来是用到的这几个宏和函数

```c
#define LIST_FIRST(head) ((head)->lh_first)

#define LIST_EMPTY(head) ((head)->lh_first == NULL)

#define LIST_REMOVE(elm, field)                                                                    \
	do {                                                                                       \
		if (LIST_NEXT((elm), field) != NULL)                                               \
			LIST_NEXT((elm), field)->field.le_prev = (elm)->field.le_prev;             \
		*(elm)->field.le_prev = LIST_NEXT((elm), field);                                   \
	} while (0)

void *memset(void *dst, int c, size_t n) {
	void *dstaddr = dst;
	void *max = dst + n;
	u_char byte = c & 0xff;
	uint32_t word = byte | byte << 8 | byte << 16 | byte << 24;

	while (((u_long)dst & 3) && dst < max) {
		*(u_char *)dst++ = byte;
	}

	// fill machine words while possible
	while (dst + 4 <= max) {
		*(uint32_t *)dst = word;
		dst += 4;
	}

	// finish the remaining 0-3 bytes
	while (dst < max) {
		*(u_char *)dst++ = byte;
	}
	return dstaddr;
}
```

前两个其实就是取传入结构体指针的lh_first域并检查其是否为空。
`LIST_REMOVE`还是用双向链表那一套逻辑。
`memset`要注意的是传入 `void*`类型参数。

---

### page_free

之后是 `page_decref(struct Page *pp)`函数和 `page_free(struct Page *pp)`函数，其中
`page_free`函数需要我们填写，其作用是将 `pp` 指向的页控制块重新插入到 `page_free_list`中。
此外需要先确保 `pp` 指向的页控制块对应的物理页面引用次数为0。

上代码：

```c
void page_free(struct Page *pp) {
	assert(pp->pp_ref == 0);
	/* Just insert it into 'page_free_list'. */
	/* Exercise 2.5: Your code here. */
	pp->pp_ref=0;
	LIST_INSERT_HEAD(&page_free_list, pp, pp_link);

}
```

第一句类似于日志输出，之后将 `pp->ref`置0，并插入空闲链表头部（`LIST_INSERT_HEAD`）。

---

### 随后来到了虚拟内存管理——两级页表

两级页表即页目录（PD）和页表（PT），他们的页表项都有自己的类型

```c
typedef u_long Pde;
typedef u_long Pte;
```

其实他们的页表项就是一个32位的数。

#### 接下来是两级页表操作的第一个函数 `pgdir_walk(Pde *pgdir, u_long va, int create, Pte **ppte)`

上代码：

```c
static int pgdir_walk(Pde *pgdir, u_long va, int create, Pte **ppte) {
	Pde *pgdir_entryp;
	struct Page *pp;

	/* Step 1: Get the corresponding page directory entry. */
	pgdir_entryp=pgdir+PDX(va);
	/* Step 2: If the corresponding page table is not existent (valid) then:
	 *   * If parameter `create` is set, create one. Set the permission bits 'PTE_C_CACHEABLE |
	 *     PTE_V' for this new page in the page directory. If failed to allocate a new page (out
	 *     of memory), return the error.
	 *   * Otherwise, assign NULL to '*ppte' and return 0.
	 */
	if(!(*pgdir_entryp&PTE_V)){
		if(create) {
			if(page_alloc(&pp)!=0) {
				return -E_NO_MEM;
			}
			pp->pp_ref++;
			*pgdir_entryp = page2pa(pp) | PTE_C_CACHEABLE | PTE_V;
			Pte *pgtable = (Pte *)KADDR(PTE_ADDR(*pgdir_entryp));
			*ppte = pgtable + PTX(va);
			return 0;
		}
		*ppte = NULL;
		return 0;
	}

	/* Step 3: Assign the kernel virtual address of the page table entry to '*ppte'. */
	/* Exercise 2.6: Your code here. (3/3) */
	Pte *pgtable = (Pte *)KADDR(PTE_ADDR(*pgdir_entryp));
	*ppte = pgtable + PTX(va);
	return 0;
}
```

个人觉得这个函数比较难理解（😭）

首先定义一个临时页目录项指针和页指针，之后通过取虚拟地址 `va`的高十位，来找到它对应页目录里面的第几个页目录项，

之后用页目录起始地址 `pgdir`来加上这个偏移即得到了 `va`对应的页目录项的地址 `pgdir_entryp`，由于页目录一定存在，而这个页目录项对应的页表不一定存在，所以此时判断 `*pgdir_entryp`的有效位即该值（对应页表物理地址）的有效性，如果并非有效，而且需要创建，那么就通过之前所写的 `page_alloc`函数对 `pp`这个页指针进行分配，那么就可以得到 `pp`，即所分配的页表对应页（别忘了页表大小恰好是一个页）。

随后，将刚才页目录中的那个页目录项的值分配为 `pp`通过 `page2pa`函数所转化出来的物理地址，并设置有效位和权限位，
那么此时定义一个页表项指针作为分配出来页表的第一项指针 `pgtable`（因为是第一项，其实 `pgtable`的值也就是这个页表的起始物理地址），
用 `pgtable`去接收 `*pgdir_entryp`的值的前二十位（`PTE_ADDR`函数），由于我们是在内核态下进行操作的，所以还需要将物理地址转化为内核态虚拟地址（`KADDR`函数），最后将 `va`对应的页目录偏移与 `pgtable`相加，就得到了在新分配的页表中 `va`指向的页表项的地址，将这个值赋给 `*ppte`即可。
千万别忘了把 `pp_ref`加一，当然在指导书里也有这个提示。

如果页表有效，直接赋值即可。

---

#### 接下来是增加地址映射函数page_insert

上代码：

```c
int page_insert(Pde *pgdir, u_int asid, struct Page *pp, u_long va, u_int perm) {
	Pte *pte;

	/* Step 1: Get corresponding page table entry. */
	pgdir_walk(pgdir, va, 0, &pte);

	if (pte && (*pte & PTE_V)) {
		if (pa2page(*pte) != pp) {
			page_remove(pgdir, asid, va);
		} else {
			tlb_invalidate(asid, va);
			*pte = page2pa(pp) | perm | PTE_C_CACHEABLE | PTE_V;
			return 0;
		}
	}

	/* Step 2: Flush TLB with 'tlb_invalidate'. */
	tlb_invalidate(asid, va);

	/* Step 3: Re-get or create the page table entry. */
	/* If failed to create, return the error. */
	if (pgdir_walk(pgdir, va, 1, &pte)) {
		return -E_NO_MEM;
	}

	/* Step 4: Insert the page to the page table entry with 'perm | PTE_C_CACHEABLE | PTE_V'
	 * and increase its 'pp_ref'. */
	*pte = page2pa(pp) | perm | PTE_C_CACHEABLE | PTE_V;
	pp->pp_ref++;

	return 0;
}
```

这个函数的作用是作用是将一级页表基地址 `pgdir` 对应的两级页表结构中虚拟地址 `va` 映射到页控制块 `pp` 对应的物理页面，并将页表项权限为设置为 `perm`。

首先利用 `pgdir_walk`函数将 `pte`的值设置为 `va`对应的二级页表项地址，那么 `*pte`的值自然就是 `va`对应的物理地址。

如果 `va`有对应的二级页表项映射，即 `pte`非空且 `*pte`有效，则检查 `*pte`物理地址对应的物理页是否是 `pp`，若不同则移除该物理页与 `va`的映射，若相同则刷新 `TLB`，并将 `pte`位置的页表项设置要求的权限。

如果 `va`没有对应的二级页表项映射，则创建该映射（创建失败直接返回错误值），并设置权限。

---

### 接下来是访问内存与TLB重填

#### 首先是 `TLB`旧表项无效化函数 `tlb_out`

```c
void tlb_invalidate(u_int asid, u_long va) {
	tlb_out((va & ~GENMASK(PGSHIFT, 0)) | (asid & (NASID - 1)));
}
```

由这个 `tlbex.c`中的函数，调用 `tlb_out`函数

```
LEAF(tlb_out)
.set noreorder
	mfc0    t0, CP0_ENTRYHI
	mtc0    a0, CP0_ENTRYHI
	nop
	/* Step 1: Use 'tlbp' to probe TLB entry */
	/* Exercise 2.8: Your code here. (1/2) */
	tlbp

	nop
	/* Step 2: Fetch the probe result from CP0.Index */
	mfc0    t1, CP0_INDEX
.set reorder
	bltz    t1, NO_SUCH_ENTRY
.set noreorder
	mtc0    zero, CP0_ENTRYHI
	mtc0    zero, CP0_ENTRYLO0
	mtc0    zero, CP0_ENTRYLO1
	nop
	/* Step 3: Use 'tlbwi' to write CP0.EntryHi/Lo into TLB at CP0.Index  */
	/* Exercise 2.8: Your code here. (2/2) */
	tlbwi

.set reorder

NO_SUCH_ENTRY:
	mtc0    t0, CP0_ENTRYHI
	j       ra
END(tlb_out)
```

当 `tlb_out` 被 `tlb_invalidate` 调用时, `a0` 寄存器中存放着传入的参数，其值为旧表项的 `Key`（由虚拟页号和 `ASID` 组成）。我们使用 `mtc0` 指令将 `Key` 写入 `EntryHi`，随后使用 `tlbp` 指令，根据 `EntryHi` 中的 `Key` 查找对应的旧表项，将表项的索引存入 `Index`。如果索引值大于等于0（即 `TLB` 中存在 `Key` 对应的表项），我们向 `EntryHi` 和 `EntryLo0`、`EntryLo1` 中写入0, 随后使用 `tlbwi` 指令，将 `EntryHi` 和 `EntryLo0`、`EntryLo1` 中的值写入索引指定的表项。此时旧表项的 `Key` 和 `Data`被清零，实现将其无效化。

---

#### 接下来是TLB重填

`tlbex.c`中代码

```c
void _do_tlb_refill(u_long *pentrylo, u_int va, u_int asid) {
	tlb_invalidate(asid, va);
	Pte *ppte;
	/* Hints:
	 *  Invoke 'page_lookup' repeatedly in a loop to find the page table entry '*ppte'
	 * associated with the virtual address 'va' in the current address space 'cur_pgdir'.
	 *
	 *  **While** 'page_lookup' returns 'NULL', indicating that the '*ppte' could not be found,
	 *  allocate a new page using 'passive_alloc' until 'page_lookup' succeeds.
	 */

	/* Exercise 2.9: Your code here. */
	struct Page *p;
	while ((p = page_lookup(cur_pgdir, va, &ppte)) == NULL) {
    	passive_alloc(va, cur_pgdir, asid);
	}

	ppte = (Pte *)((u_long)ppte & ~0x7);
	pentrylo[0] = ppte[0] >> 6;
	pentrylo[1] = ppte[1] >> 6;

}
```

其中TODO部分的提示：尝试在循环中调用 `page_lookup`以查找虚拟地址 `va` 在当前进程页表中对应的页表项 `*ppte`如果 `page_lookup`回 `NULL`，表明 `*ppte`找不到，使用 `passive_alloc`为 `va` 所在的虚拟页面分配物理页面，直至 `page_lookup`返回不为 `NULL`则退出循环。你可以在调用函数时，使用全局变量 `cur_pgdir` 作为其中一个实参。

需要补全的代码按提示写即可，函数最后将 `TLB`奇偶页均用对应的二级页表项右移6位后填写

下面对调用的函数进行解释

* 这里的 `page_lookup`函数：

```c
struct Page *page_lookup(Pde *pgdir, u_long va, Pte **ppte) {
	struct Page *pp;
	Pte *pte;

	/* Step 1: Get the page table entry. */
	pgdir_walk(pgdir, va, 0, &pte);

	/* Hint: Check if the page table entry doesn't exist or is not valid. */
	if (pte == NULL || (*pte & PTE_V) == 0) {
		return NULL;
	}

	/* Step 2: Get the corresponding Page struct. */
	/* Hint: Use function `pa2page`, defined in include/pmap.h . */
	pp = pa2page(*pte);
	if (ppte) {
		*ppte = pte;
	}

	return pp;
}
```

其作用就是找到虚拟地址 `va`对应的物理页并返回。

* 这里的passive_alloc函数：

```c
static void passive_alloc(u_int va, Pde *pgdir, u_int asid) {
	struct Page *p = NULL;

	if (va < UTEMP) {
		panic("address too low");
	}

	if (va >= USTACKTOP && va < USTACKTOP + PAGE_SIZE) {
		panic("invalid memory");
	}

	if (va >= UENVS && va < UPAGES) {
		panic("envs zone");
	}

	if (va >= UPAGES && va < UVPT) {
		panic("pages zone");
	}

	if (va >= ULIM) {
		panic("kernel address");
	}

	panic_on(page_alloc(&p));
	panic_on(page_insert(pgdir, asid, p, PTE_ADDR(va), (va >= UVPT && va < ULIM) ? 0 : PTE_D));
}
```

实际上就是给va这个虚拟地址分配了一个物理页，并确定其页表项。

---

`tlbex.S`中代码

```
NESTED(do_tlb_refill, 24, zero)
	mfc0    a1, CP0_BADVADDR
	mfc0    a2, CP0_ENTRYHI
	andi    a2, a2, 0xff /* ASID is stored in the lower 8 bits of CP0_ENTRYHI */
.globl do_tlb_refill_call;
do_tlb_refill_call:
	addi    sp, sp, -24 /* Allocate stack for arguments(3), return value(2), and return address(1) */
	sw      ra, 20(sp) /* [sp + 20] - [sp + 23] store the return address */
	addi    a0, sp, 12 /* [sp + 12] - [sp + 19] store the return value */
	jal     _do_tlb_refill /* (Pte *, u_int, u_int) [sp + 0] - [sp + 11] reserved for 3 args */
	lw      a0, 12(sp) /* Return value 0 - Even page table entry */
	lw      a1, 16(sp) /* Return value 1 - Odd page table entry */
	lw      ra, 20(sp) /* Return address */
	addi    sp, sp, 24 /* Deallocate stack */
	mtc0    a0, CP0_ENTRYLO0 /* Even page table entry */
	mtc0    a1, CP0_ENTRYLO1 /* Odd page table entry */
	nop
	/* Hint: use 'tlbwr' to write CP0.EntryHi/Lo into a random tlb entry. */
	/* Exercise 2.10: Your code here. */
	tlbwr

	jr      ra
END(do_tlb_refill)

```

这里直接白给了，注释吧答案给出来了。

下面回顾一下这个过程：

* 从 `BadVAddr` 中取出引发 `TLB` 缺失的虚拟地址。
* 从 `EntryHi` 的0 – 7 位取出当前进程的 `ASID`。在Lab3 的代码中，会在进程切换时修改 `EntryHi` 中的 `ASID`，以标识访存所在的地址空间。
* 先在栈上为返回地址、待填入 `TLB` 的页表项以及函数参数传递预留空间，并存入返回地址。以存储奇偶页表项的地址、触发异常的虚拟地址和 `ASID` 为参数，调用 `_do_tlb_refill`函数。该函数是 `TLB` 重填过程的核心，其功能是根据虚拟地址和 `ASID` 查找页表，将对应的奇偶页表项写回其第一个参数所指定的地址。
* 将页表项存入 `EntryLo0`、`EntryLo1` ，并执行 `tlbwr` 将此时的 `EntryHi` 与 `EntryLo0`、`EntryLo1` 写入到 `TLB` 中（在发生 `TLB` 缺失时，`EntryHi` 已经由硬件写入了虚拟页号等信息，无需修改）。

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

第一问——使用宏来实现链表的一个显著好处是**提高了代码的可重用性**。这有几个方面：

#### 1.抽象性：

* 宏能够把链表的操作封装起来，如插入、删除、遍历等，提供统一的接口，而不关心链表的具体实现细节。这样做的好处是，我们不需要重复写相同的链表操作代码，避免了冗余的代码编写。
* 链表的类型和结构不再局限于特定数据结构，宏可以适用于不同类型的链表（单链表、双链表等）。

#### 2.代码模块化：

* 宏将链表操作与具体实现细节分开，使得在链表操作发生变化时，只需要修改宏定义部分，而不需要修改链表使用的其他地方。这也促进了代码的模块化和可维护性。

#### 3.灵活性：

* 宏通过参数化，提供了灵活的功能，例如支持不同数据类型的链表、不同链表的插入位置（头部、尾部、特定位置），这使得链表操作可以适应更多的使用场景。

#### 4.减少错误：

* 通过宏实现链表操作，避免了手动实现的过程中容易出现的错误。例如，手动实现链表操作可能会漏掉某些指针的更新，导致链表破裂，而通过宏来实现链表操作，通常会使得链表的插入、删除操作更加一致和健壮。

第二问——比较单向链表、循环链表和双向链表在插入与删除操作上的性能差异

* 插入性能：

单向链表：只能从头部快速插入，尾部插入需要遍历。

循环链表：从尾部插入比单向链表更高效，但删除操作仍然需要遍历。

双向链表：无论是头部、尾部还是中间，插入和删除都可以在常数时间内完成。

* 删除性能：

单向链表和循环链表：删除中间节点时需要遍历链表，效率较低。

双向链表：支持快速的双向访问，删除任何节点（包括中间节点）都可以在 O(1) 时间内完成。

* 总结：

双向链表：对于频繁插入和删除操作，尤其是在两端操作的场景，双向链表表现最好。

循环链表：适合循环访问的场景，但插入和删除时仍然受到一定限制。

单向链表：在结构简单、访问模式较为单一时最为高效。

### 2.3

问题：请阅读 `include/queue.h` 以及 `include/pmap.h`, 将 `Page_list` 的结构梳理清楚，选择正确的展开结构。

答案应该选C，

因为首先内层 `Page`里面的 `le_prev`是双指针，其次外层在定义时就已经写了 `struct type *lh_first;`，故应为

```c
struct Page_list{
	struct Page{
		struct {                                                                                   \
			struct type *le_next;  /* next element */                                  \
			struct type **le_prev; /* address of previous next element */              \
		} pp_link;
		u_short pp_ref;
	}* lh_first;
}
```

### 2.4

#### ASID的必要性

在多进程操作系统中，每个进程都有自己的虚拟内存地址空间。为了有效管理虚拟内存，操作系统通常使用TLB（Translation Lookaside Buffer）来加速虚拟地址到物理地址的转换。由于TLB的缓存是基于虚拟地址的，因此在多进程环境下，不同进程可能会使用相同的虚拟地址，这就可能导致缓存冲突，影响系统性能。为了避免这种问题，操作系统引入了ASID（Address Space Identifier），即地址空间标识符。

#### MIPS 4Kc 中的ASID段位数和最大地址空间数量

MIPS 4Kc的ASID使用 8位 来标识。这样，ASID可以表示 2^8 = 256 个不同的地址空间。

最大地址空间数量：

每个进程在TLB中都有一个独特的ASID，通过这个ASID，4Kc能够管理最多 256个不同的地址空间。这意味着操作系统可以在同一系统上同时运行多达256个进程，每个进程有自己的虚拟内存地址空间，互不干扰。

总结：

在MIPS 4Kc中，ASID段占据8位，因此系统可以支持最多 256个不同的虚拟地址空间，即最多支持256个进程同时运行，而不会出现TLB冲突的问题 。

### 2.5

**1. tlb_invalidate 和 tlb_out 的调用关系**

**tlb_invalidate** 函数用于无效化指定的 TLB 条目。它通过调用 **tlb_out** 来刷新 TLB 中的条目。在 **tlb_invalidate** 中，虚拟地址和 ASID 被传递到 **tlb_out**，从而在 **tlb_out** 中执行 TLB 无效化操作。具体来说，**tlb_invalidate** 会调用 **tlb_out** 来清除和刷新 TLB 中与给定虚拟地址和 ASID 相关的条目。

**2. 一句话概括 tlb_invalidate 的作用**

**tlb_invalidate** 的作用是根据给定的 ASID 和虚拟地址来无效化相应的 TLB 条目。

**3. 逐行解释 tlb_out 中的汇编代码**

```shell
LEAF(tlb_out)
.set noreorder
	mfc0    t0, CP0_ENTRYHI      # 从 CP0 的 ENTRYHI 寄存器获取当前的 ASID（地址空间标识符）并存储到 t0 寄存器
	mtc0    a0, CP0_ENTRYHI      # 将 a0 寄存器的值（新的 ASID）写入到 CP0 的 ENTRYHI 寄存器
	nop                           # 无操作（填充指令，用于延迟槽）
	tlbp                           # 执行 TLB 探测指令，检查虚拟地址是否已存在于 TLB 中
	nop                           # 无操作（填充指令，用于延迟槽）
	mfc0    t1, CP0_INDEX        # 从 CP0 的 INDEX 寄存器获取当前 TLB 入口的索引
.set reorder
	bltz    t1, NO_SUCH_ENTRY    # 如果 t1 寄存器的值小于 0，表示未命中 TLB 条目，跳转到 NO_SUCH_ENTRY 标签
.set noreorder
	mtc0    zero, CP0_ENTRYHI    # 将 CP0 的 ENTRYHI、ENTRYLO0、ENTRYLO1 寄存器清零，清除相关的 TLB 条目
	mtc0    zero, CP0_ENTRYLO0
	mtc0    zero, CP0_ENTRYLO1
	nop                           # 无操作（填充指令，用于延迟槽）
	tlbwi                          # 将 CP0 的 ENTRYHI、ENTRYLO0 和 ENTRYLO1 写入到 TLB 中（如果找到了条目）
.set reorder
NO_SUCH_ENTRY:
	mtc0    t0, CP0_ENTRYHI      # 恢复之前存储的 ASID 到 CP0 的 ENTRYHI 寄存器
	j       ra                    # 返回调用者
END(tlb_out)
```

### 2.6

每次函数调用时，CPU 会访问栈内存，分配空间以保存参数、局部变量和返回地址，这些内存访问需要通过虚拟地址到物理地址的转换。

在访问过程中，CPU 会先查找 TLB以加速地址转换；
若 TLB 中没有相应的条目，则会查找页表。
如果发生 TLB miss 或页表缺失，CPU 会触发异常并执行异常处理程序，更新 TLB 或页表。

函数返回时，栈空间会被恢复，类似的内存访问也依赖于虚拟地址到物理地址的转换。

总之，函数调用过程中的栈操作和内存访问需要依赖 TLB 和页表的协同工作，确保虚拟内存能够正确映射到物理内存。

### 2.7

**RISC-V 内存管理机制**

RISC-V 是一种精简指令集计算机（RISC）架构，具有简单的设计，并且支持多种内存管理特性。RISC-V 的内存管理机制主要依赖于虚拟地址到物理地址的映射，使用页表和 MMU（内存管理单元）来进行地址转换。以下是 RISC-V 内存管理的一些关键特性：

**1.虚拟内存支持** ：RISC-V 支持虚拟内存，允许操作系统为每个进程提供独立的虚拟地址空间，这使得每个进程的内存管理更为安全和高效。

**2.页表** ：RISC-V 使用分页机制，采用多级页表来将虚拟地址转换为物理地址。具体实现包括页表的分层结构，例如三级页表（如果使用 64 位虚拟地址）。页表项存储了虚拟页到物理页的映射信息。

**3.TLB** ：为了加速虚拟地址到物理地址的转换，RISC-V 使用 TLB，缓存最近访问的页表项。TLB 通过直接映射虚拟地址到物理地址来提高内存访问速度。

**4.虚拟地址空间** ：RISC-V 支持用户模式和内核模式的虚拟地址空间，确保内核与用户程序的内存隔离，提高系统的安全性。

**5.异常处理** ：当发生页面错误（如缺页）时，RISC-V 会触发一个异常，操作系统负责处理缺页异常并更新页表，从而恢复正常执行。


**RISC-V 与 MIPS 在内存管理上的区别**

**1.页表结构** ：

RISC-V：使用更灵活的多级页表结构，支持更大的虚拟地址空间（如 64 位），并且具有更高的扩展性。

MIPS：通常使用较简单的二级页表，较少支持大范围的虚拟地址空间（通常为 32 位）。

**2.地址空间** ：

RISC-V：提供了更灵活的虚拟地址空间设计，支持 32 位和 64 位地址空间。RISC-V 通过定义不同的模式（如 S模式、M模式等）来支持用户模式和内核模式的内存隔离。

MIPS：较多采用 32 位地址空间，支持用户模式和内核模式，但通常不如 RISC-V 灵活。

**3.虚拟内存与异常处理** ：

RISC-V：更注重扩展性，支持更复杂的虚拟内存机制（如支持更多级别的页表），并通过更细粒度的异常管理来处理虚拟内存异常。

MIPS：虚拟内存和异常处理机制较为简单，但同样能够通过页表和 TLB 提供内存管理和异常处理。

**4.TLB 设计** ：

RISC-V：支持更复杂的 TLB 设计，通常包括多级 TLB，并具有更大的灵活性。

MIPS：TLB 通常比较简单，通常只有一级 TLB，容量较小，设计上较为简化。
