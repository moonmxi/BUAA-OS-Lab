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

现在终于开始page_init()函数，话不多说，先上代码：

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

接下来是page_alloc()函数，继续上代码：


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

问题：请阅读`include/queue.h` 以及`include/pmap.h`, 将`Page_list` 的结构梳理清楚，选择正确的展开结构。

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
