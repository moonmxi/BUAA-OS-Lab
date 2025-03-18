# Lab1

## 指导书学习笔记

### 一、Makefile命令：

#### 1、赋值

* `=` :自动推导将最终的赋值作为该变量的值
* `:=` :覆盖式赋值，覆盖**之前**有过的赋值
* `?=` :某变量若前面已经定义赋值过，则不执行本次定义赋值，否则执行本次赋值

#### 2、.PHONY写法：

* 可以在 `.PHONY`后面直接写所有伪目标，不用每个伪目标都写一次，如：

```shell
PHONY: all $(modules) clean
```

#### 3、几种$的用法：

* **`$(...)`** :变量引用，`$(variable_name)` 表示引用名为 `variable_name` 的变量。
* **`$@` **:Makefile中的自动变量，表示当前规则的目标。
  `$(MAKE) --dirtctory=$@`的执行效果与手动切换到模块目录再调用make 是一致的。
* **`$$`:**在Makefile 中，`$` 是一个特殊字符，如果想在命令中使用真正的 `$` 符号，需要用 `$$` 来转义它。
* **`$<` **:Makefile 中的另一个自动变量，表示当前规则的第一个依赖项。
* **`$^`** :Makefile 中的另一个自动变量，表示当前规则的所有依赖项。
* **`$(shell ...)`** :Makefile 中的 shell 函数，用于执行 shell 命令并返回结果。
* **`$*`** :Makefile 中的自动变量，表示当前规则的目标文件名（不包括扩展名）。

---

### 二、ELF中段和节：

#### 1、定义

* 节：如 `.data`、`.section`等，是一个可重定向（`.o`）文件或其他文件里面的一部分
* 段：由**多个节**组成，在可执行文件中，是由链接前的可重定向文件的节经过链接组成的
* 节头表：组成可重定位文件，参与可执行文件和可共享文件的链接。此时使用节头表。
* 段头表：组成可执行文件或者可共享文件，在运行时为加载器提供信息。此时使用段头表。

#### 2、节的对齐

在指导书里面的例子

```shell
user@debian ~/Desktop $ gcc -o test test.c -T test.lds -nostdlib -m32
user@debian ~/Desktop $ readelf -S test
共有 11 个节头，从偏移量 0x2164 开始：

节头：
[Nr] Name Type Addr Off Size ES Flg Lk Inf Al
[ 2] .text PROGBITS 00010000 001000 000018 00 AX 0 0 1
[ 5] .data PROGBITS 08000000 002000 00000e 00 WA 0 0 1
[ 6] .bss NOBITS 08000010 00200e 000004 00 WA 0 0 4
```

* 其中 **.data** 的 **Addr** 被定位到了 `08000000`，**Size** 为 `00000e`，下一个节应该从 `080000e`开始，但是 **.bss** 是 **四字节对齐** ，导致只能从 `080000e`的下一个四字节倍数的地址开始，即 `08000010`。

---

### 三、Linker Scripts

* 文件开头：

```
/*
 * Set the architecture to mips.
 */
OUTPUT_ARCH(mips)

/*
 * Set the ENTRY point of the program to _start.
 */
ENTRY(_start)
```

其中 **`OUTPUT_ARCH(mips)`** 设置了最终生成的文件采用的架构，对于 MOS 来说就是 mips。而 **`ENTRY(_start)`** 便设置了程序的入口函数。因此 MOS 内核的入口即 **`_start`**。这是一个符号，对应的是 init/start.S 中的 **`EXPORT(_start)`**。

* Section部分代码示例：

```
SECTIONS
{
	. = 0x10000;
	.text : { *(.text) }
	. = 0x8000000;
	.data : { *(.data) }
	.bss : { *(.bss) }
}
```

* 注意Linker Script 文件编辑**冒号、等号**的两边需要有**空格**，否则可能会在编译的时候报语法错误。
* `.` 的作用：用来做定位计数器，根据输出节的大小增长，在SECTIONS
  命令开始的时候，它的值为0。通过设置“.”即可设置接下来的节的起始地址。
* ***!!!重点注意（debug好久）***
  **需要分号**：赋值、符号定义、表达式等独立语句。
  **不需要分号**：块结构、节定义等复合语句。


---

### 四、Boot相关MIPS代码编写：

#### 1、函数-->符号：

在 `Note 1.4.3`中写着

* mips_init 函数虽然为C 语言所书写，但是在被编译成汇编之后，其入口点
  成为一个符号。

故可以在 `start.S`中直接调用该C语言函数。

#### 2、预处理代码：

```
1 EXPORT(_start)
2 .set at
3 .set reorder
4 /* disable interrupts */
5 mtc0 zero, CP0_STATUS
```

* 本段代码中的 `EXPORT` 是一个宏，它将 `_start` 函数导出为一个符号，使得链
  接器可以找到它。可以简单的理解为，它实现了一种在汇编语言中的函数定义。
* 首先是 `_start` 函数的声明，随后2、3 行的 `.set`允许汇编器使用 `at`寄存器，也允许对接下来的代码进行重排序，第5 行禁用了外部中断。

---

### 五、C语言函数中变长参数

当函数参数列表末尾有省略号时，该函数即有变长的参数表。由于需要定位变长
参数表的起始位置，函数需要含有至少一个固定参数，且变长参数必须在参数表的末尾。

`stdarg.h` 头文件中为处理变长参数表定义了一组宏和变量类型如下：

* `va_list`：变长参数表的变量类型
* `va_start(va_list ap, lastarg)`：用于初始化变长参数表的宏
* `va_arg(va_list ap, 类型)`：用于取变长参数表下一个参数的宏
* `va_end(va_list ap)`：结束使用变长参数表的宏

其中`lastarg` 为该函数最后一个命名的形式参数。

---



## Exercise笔记

### 1.1

题目：

```
阅读tools/readelf 目录下的elf.h、readelf.c 和main.c 文件，并补全
readelf.c 中缺少的代码。readelf 函数需要输出ELF 文件中所有节头中的地址信息，对
于每个节头，输出格式为"%d:0x%x\n"，其中的%d 和%x 分别代表序号和地址。
```

我的作答：

```c
#include "elf.h"
#include <stdio.h>

/* Overview:
 *   Check whether specified buffer is valid ELF data.
 *
 * Pre-Condition:
 *   The memory within [binary, binary+size) must be valid to read.
 *
 * Post-Condition:
 *   Returns 0 if 'binary' isn't an ELF, otherwise returns 1.
 */
int is_elf_format(const void *binary, size_t size) {
	Elf32_Ehdr *ehdr = (Elf32_Ehdr *)binary;
	return size >= sizeof(Elf32_Ehdr) && ehdr->e_ident[EI_MAG0] == ELFMAG0 &&
	       ehdr->e_ident[EI_MAG1] == ELFMAG1 && ehdr->e_ident[EI_MAG2] == ELFMAG2 &&
	       ehdr->e_ident[EI_MAG3] == ELFMAG3;
}

/* Overview:
 *   Parse the sections from an ELF binary.
 *
 * Pre-Condition:
 *   The memory within [binary, binary+size) must be valid to read.
 *
 * Post-Condition:
 *   Return 0 if success. Otherwise return < 0.
 *   If success, output the address of every section in ELF.
 */

int readelf(const void *binary, size_t size) {
	Elf32_Ehdr *ehdr = (Elf32_Ehdr *)binary;

	// Check whether `binary` is a ELF file.
	if (!is_elf_format(binary, size)) {
		fputs("not an elf file\n", stderr);
		return -1;
	}

	// Get the address of the section table, the number of section headers and the size of a
	// section header.
	const void *sh_table;
	Elf32_Half sh_entry_count;
	Elf32_Half sh_entry_size;
	/* Exercise 1.1: Your code here. (1/2) */
	sh_table = binary + ehdr->e_shoff;
	sh_entry_size = ehdr->e_shentsize;
	sh_entry_count = ehdr->e_shnum;
	// For each section header, output its index and the section address.
	// The index should start from 0.
	for (int i = 0; i < sh_entry_count; i++) {
		const Elf32_Shdr *shdr;
		unsigned int addr;
		/* Exercise 1.1: Your code here. (2/2) */
		shdr = sh_table + i*sh_entry_size;
		addr = (int)(shdr->sh_addr);
		printf("%d:0x%x\n", i, addr);
	}

	return 0;
}

```

* 解题心得：
  在开始做这道题的时候，一直没认真看结构体定义，以为文件头表之后就是节头表，就想用文件头表的地址加上文件头表的大小去找节头表的地址😭，但是其实在 `Elf32_Ehdr`结构体里面描述的很清楚——节头表起始位置，节头表大小，节头表数量都有变量来表示，按照描述以及题意，循环输出就可以了，以下是文件头表：

```c
typedef struct {
	unsigned char e_ident[EI_NIDENT]; /* Magic number and other info */
	Elf32_Half e_type;		  /* Object file type */
	Elf32_Half e_machine;		  /* Architecture */
	Elf32_Word e_version;		  /* Object file version */
	Elf32_Addr e_entry;		  /* Entry point virtual address */
	Elf32_Off e_phoff;		  /* Program header table file offset */
	Elf32_Off e_shoff;		  /* Section header table file offset */
	Elf32_Word e_flags;		  /* Processor-specific flags */
	Elf32_Half e_ehsize;		  /* ELF header size in bytes */
	Elf32_Half e_phentsize;		  /* Program header table entry size */
	Elf32_Half e_phnum;		  /* Program header table entry count */
	Elf32_Half e_shentsize;		  /* Section header table entry size */
	Elf32_Half e_shnum;		  /* Section header table entry count */
	Elf32_Half e_shstrndx;		  /* Section header string table index */
} Elf32_Ehdr;
```

### 1.2

题目：

```c
填写kernel.lds 中空缺的部分，在Lab1 中，只需要填补.text、.data
和.bss 节，将内核调整到正确的位置上即可。
```

我的作答：

```c
/*
 * Set the architecture to mips.
 */
OUTPUT_ARCH(mips)

/*
 * Set the ENTRY point of the program to _start.
 */
ENTRY(_start)

SECTIONS {
	/* Exercise 3.10: Your code here. */

	/* fill in the correct address of the key sections: text, data, bss. */
	/* Hint: The loading address can be found in the memory layout. And the data section
	 *       and bss section are right after the text section, so you can just define
	 *       them after the text section.
	 */
	/* Step 1: Set the loading address of the text section to the location counter ".". */
	/* Exercise 1.2: Your code here. (1/4) */
	. = 0x80020000
	/* Step 2: Define the text section. */
	/* Exercise 1.2: Your code here. (2/4) */
	.text : {*(.text)}
	/* Step 3: Define the data section. */
	/* Exercise 1.2: Your code here. (3/4) */
	.data : {*(.data)}
	bss_start = .;
	/* Step 4: Define the bss section. */
	/* Exercise 1.2: Your code here. (4/4) */
	.bss : {*(.bss)}
	bss_end = .;
	. = 0x80400000;
	end = . ;
}

```

* 解题心得：
  和第一题一样，也是没有仔细阅读，在mmu.h文件中已经画好了elf的内存布局图，只要按照布局图，把 `.text`放在 `0x80020000`，之后的 `.data` 和 `.bss`接着后面放即可，以下是mmu.h中的内存布局图。

```c
/*
 o     4G ----------->  +----------------------------+------------0x100000000
 o                      |       ...                  |  kseg2
 o      KSEG2    -----> +----------------------------+------------0xc000 0000
 o                      |          Devices           |  kseg1
 o      KSEG1    -----> +----------------------------+------------0xa000 0000
 o                      |      Invalid Memory        |   /|\
 o                      +----------------------------+----|-------Physical Memory Max
 o                      |       ...                  |  kseg0
 o      KSTACKTOP-----> +----------------------------+----|-------0x8040 0000-------end
 o                      |       Kernel Stack         |    | KSTKSIZE            /|\
 o                      +----------------------------+----|------                |
 o                      |       Kernel Text          |    |                    PDMAP
 o      KERNBASE -----> +----------------------------+----|-------0x8002 0000    |
 o                      |      Exception Entry       |   \|/                    \|/
 o      ULIM     -----> +----------------------------+------------0x8000 0000-------
 o                      |         User VPT           |     PDMAP                /|\
 o      UVPT     -----> +----------------------------+------------0x7fc0 0000    |
 o                      |           pages            |     PDMAP                 |
 o      UPAGES   -----> +----------------------------+------------0x7f80 0000    |
 o                      |           envs             |     PDMAP                 |
 o  UTOP,UENVS   -----> +----------------------------+------------0x7f40 0000    |
 o  UXSTACKTOP -/       |     user exception stack   |     PTMAP                 |
 o                      +----------------------------+------------0x7f3f f000    |
 o                      |                            |     PTMAP                 |
 o      USTACKTOP ----> +----------------------------+------------0x7f3f e000    |
 o                      |     normal user stack      |     PTMAP                 |
 o                      +----------------------------+------------0x7f3f d000    |
 a                      |                            |                           |
 a                      ~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~                           |
 a                      .                            .                           |
 a                      .                            .                         kuseg
 a                      .                            .                           |
 a                      |~~~~~~~~~~~~~~~~~~~~~~~~~~~~|                           |
 a                      |                            |                           |
 o       UTEXT   -----> +----------------------------+------------0x0040 0000    |
 o                      |      reserved for COW      |     PTMAP                 |
 o       UCOW    -----> +----------------------------+------------0x003f f000    |
 o                      |   reversed for temporary   |     PTMAP                 |
 o       UTEMP   -----> +----------------------------+------------0x003f e000    |
 o                      |       invalid memory       |                          \|/
 a     0 ------------>  +----------------------------+ ----------------------------
 o
*/
```

### 1.3

题目：

```
完成init/start.S 中空缺的部分。设置栈指针，跳转到mips_init 函数。
```

我的作答：

```
#include <asm/asm.h>
#include <mmu.h>

.text
EXPORT(_start)
.set at
.set reorder
/* Lab 1 Key Code "enter-kernel" */
	/* clear .bss segment */
	la      v0, bss_start
	la      v1, bss_end
clear_bss_loop:
	beq     v0, v1, clear_bss_done
	sb      zero, 0(v0)
	addiu   v0, v0, 1
	j       clear_bss_loop
/* End of Key Code "enter-kernel" */

clear_bss_done:
	/* disable interrupts */
	mtc0    zero, CP0_STATUS

	/* hint: you can refer to the memory layout in include/mmu.h */
	/* set up the kernel stack */
	/* Exercise 1.3: Your code here. (1/2) */
	la sp KSTACKTOP

	/* jump to mips_init */
	/* Exercise 1.3: Your code here. (2/2) */
	j mips_init

```

* 解题心得：这道题只需要填写两行代码，而且也有明确要求
  * 第一步只需要在mmu.h中的内存布局图找到 `KSTACKTOP`，并检索到文件中对应的宏 `#define KSTACKTOP (ULIM + PDMAP`，之后将这个地址赋值给 `sp`寄存器即可
  * 第二步直接用 `j` 函数跳转即可：`j mips_init`
