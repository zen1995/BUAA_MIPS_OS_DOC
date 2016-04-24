lab2 实验报告
===========================
14061094 陈泽年


# 实验内容解析

##  exercise 3.2
>完成page_init 函数，使用include/queue.h 中定义的宏函数将未分配
的物理页加入到空闲链表page_free_list 中去。思考如何区分已分配的内存块和未
分配的内存块，并注意内核可用的物理内存上限

page_init 函数的作用是初始化空闲链表。

freemem这个变量并不是指剩余多少的内存，而是指当前freemem以上的地址空间都是空闲的，freemem一下的地址空间都是处于使用中的。

这次实验虽然采用二级页表的形式，但是由于有了Page pages[] 这个数组，所有的页表都是连续排列的（这和直接一级页表貌似没有什么区别，完全体现不出二级页表的优越性）。

这个函数中还需要用到PADDR和LIST的相关宏定义。

     // translates from kernel virtual address to physical address.
    #define PADDR(kva)						\
	({								\
		u_long a = (u_long) (kva);				\
		if (a < ULIM)					\
			panic("PADDR called with invalid kva %08lx", a);\
		a - ULIM;						\
	})

     // translates from physical address to kernel virtual address.
    #define KADDR(pa)						\
	({								\
		u_long ppn = PPN(pa);					\
		if (ppn >= npage)					\
			panic("KADDR called with invalid pa %08lx", (u_long)pa);\
		(pa) + ULIM;					\
	})


因为我们的虚拟地址空间大小4G,并且我们涉及的都是内核操作。所以我们的数据操作应该都是在0x80000000-0x9FFFFFFF(512MB)之间。这一部分的转化到实际地址，只需高位清零即可即减去0x80000000.从物理地址转化到虚拟地址，只需加上0x80000000 即可。这就是PADDR和KADDR的作用。

这个函数中，我们以freemem为分界线，由PADDR(freemem)/BY2PG获得页实际的分割地址。小于它的为使用中的地址，pp_ref = 1大于它的pp_ref = 0;并且加入到空闲链表当中。

##  exercise 3.3
>完成mm/pmap.c 中的page_alloc 和page_free 函数，基于空闲内存
链表page_free_list ，以页为单位进行物理内存的管理

page_alloc 和 page_free 两个函数的使用情况是建立在系统的分页系统已经建立完成的情况下。因此每次分配也，以及释放页是不需要alloc函数的，直接操纵page_free_list即可。因为alloc的作用是在分页系统还未建立的情况下使用的。

写 page_alloc 和 page_free 中有这么一句话需要注意：
>  hint :pp_ref should not increment

因此，在 page_alloc 的时候 pp_ref 不须加1 page_free的时候不用减1.

这个exercise中运用到了 page2kva这个函数。下面就顺带说明一下pmap.h中其他的一些必要的函数

    static inline u_long
    page2ppn(struct Page *pp)
    {
	     return pp - pages;
     }

     /* Get the physical address of Page 'pp'.
     */
    static inline u_long
    page2pa(struct Page *pp)
    {
    	return page2ppn(pp) << PGSHIFT;
    }

    /* Get the Page struct whose physical address is 'pa'.
     */
    static inline struct Page *
    pa2page(u_long pa)
    {
    	if (PPN(pa) >= npage) {
    		panic("pa2page called with invalid pa: %x", pa);
    	}

    	return &pages[PPN(pa)];
    }

    /* Get the kernel virtual address of Page 'pp'.
     */
    static inline u_long
    page2kva(struct Page *pp)
    {
    	return KADDR(page2pa(pp));
    }

static inline u_long page2ppn(struct Page *pp) 返回 pp 是第几个页

static inline u_long page2pa(struct Page *pp) 根据page返回所对应的物理地址

static inline struct Page * pa2page(u_long pa) 根据物理地址获得页指针

static inline u_long page2kva(struct Page *pp) 获取页所对应的虚拟地址。


当page_alloc 的时候，分配页需要清零
    bzero(page2kva(*pp),BY2PG);
注意，bzero中使用的是虚拟地址。因此需要使用page2kva 而不是page2pa

## exercise 3.4
> 完成mm/pmap.c 中的boot_pgdir_walk 和pgdir_walk 函数，实现
虚拟地址到物理地址的转换以及创建页表的功能。

static Pte* boot_pgdir_walk(Pte *pgdir , u_long va , int create)

这里 Pde(page directory entry) Pte(page table entry) 实际上都是u_long 数据类型。va(virtual address) pgdir 是页目录的起始地址

Pde *targetPde = (Pde *)(&pgdir[PDX(va)]);

PDX(va):获取虚拟地址va到底是属于哪个页目录
Pde* targetPde = (Pde *)(&pgdir[PDX(va)]); 这句话的意思就是指针targetPde指向这个页目录项的物理地址。

Pte* pageTable = (Pte *)KADDR(PTE_ADDR(*targetPde));
    #define PTE_ADDR(pte)	((u_long)(pte)&~0xFFF)  //第12位清零(4k对齐)
PTE_ADDR(*targetPde) :获取页目录中指向的二级页表的物理地址。
Pte *pageTable = (Pte *)KADDR(PTE_ADDR(*targetPde));这句话的意思就是由targetPde获取二级页表的虚拟地址。

return (Pte *)(&pageTable[PTX(va)]);
返回申请的4k空间的地址。

int pgdir_walk(Pde *pgdir , u_long va , int create , Pte **ppte)

这个函数与上个函数类似。不同点在于，执行这个函数的时候，分页机制已经建立完成，因此当创建页表的时候，需用page_alloc 而不是alloc


##  exercse 3.5
> 实现mm/pmap.c 中的boot_map_segment 函数，实现将制定的物理
内存与虚拟内存建立起映射的功能

看懂这个函数需要先了解一下整个系统初始化的过程

首先，在 init.c  有如下代码：
  mips_detect_memory();
  mips_vm_init();
  page_init();
  page_check();

在 pmap.c 中有如下代码：

    void mips_vm_init()
    {
    	extern char end[];
    	extern int mCONTEXT;
    	extern struct Env *envs;

    	Pde *pgdir;
    	u_int n;

    	/* Step 1: Allocate a page for page directory(first level page table). */
    	pgdir = alloc(BY2PG, BY2PG, 1);
    	printf("to memory %x for struct page directory.\n", freemem);
    	mCONTEXT = (int)pgdir;

    	boot_pgdir = pgdir;

    	/* Step 2: Allocate proper size of physical memory for global array `pages`,
    	 * for physical memory management. Then, map virtual address `UPAGES` to
    	 * physical address `pages` allocated before. For consideration of alignment,
    	 * you should round up the memory size before map. */
    	pages = (struct Page *)alloc(npage * sizeof(struct Page), BY2PG, 1);
    	printf("to memory %x for struct Pages.\n", freemem);
    	n = ROUND(npage * sizeof(struct Page), BY2PG);
    	boot_map_segment(pgdir, UPAGES, n, PADDR(pages), PTE_R);

    	/* Step 3, Allocate proper size of physical memory for global array `envs`,
    	 * for process management. Then map the physical address to `UENVS`. */
    	envs = (struct Env *)alloc(NENV * sizeof(struct Env), BY2PG, 1);
    	n = ROUND(NENV * sizeof(struct Env), BY2PG);
    	boot_map_segment(pgdir, UENVS, n, PADDR(envs), PTE_R);

    	printf("pmap.c:\t mips vm init success\n");
    }

其中
  #define PDMAP		(4*1024*1024)	// bytes mapped by a page directory entry
  #define ULIM 0x80000000
  #define UVPT (ULIM - PDMAP)//2G-4M
  #define UPAGES (UVPT - PDMAP)//2G-8M 系统分页的pages存放地址
  #define UENVS (UPAGES - PDMAP)//

在 mips_vm_init 函数中，我们首先分配了一个4k大小的页目录。然后分配了npage个连续的二级页表

boot_map_segment 函数的要求是建立非kseg0部分的页映射关系。

nToMap变量是需要分配多少个页面才能装下 u_long size 大小的数据

建立虚拟地址和物理地址相关的映射时，只需用boot_pgdir_walk 函数，获得页表项，并将页表项的内容指向虚拟地址即可



# 实验思考题

##  thinking 2.1
> 们注意到我们把宏函数的函数体写成了do { // ... } while(0)
的形式，而不是仅仅写成形如{// ... } 的语句块，这样的写法好处是什么？

若写成 {//...} 则在如下情况中，会编译不过：
  #define BPP(b)\
  {\
    b++;\
    b++;\
  }

  if(1)
    BPP(i);
  else
    ....
编译后则是
if(1)
  {
    b++;
    b++;
  };
  else
    ...
if 和else之间不能有其他语句 所以编译不会过


# thinking 2.2
> 了解了二级页表页目录自映射的原理之后，我们知道，Win2k 内核的
虚存管理也是采用了二级页表的形式，其页表所占的4M 空间对应的虚存起始地址
为0xC0000000，那么，它的页目录的起始地址是多少呢？

0xC0000000+0xc0000000 >> 10 = 0xC0300000

# thinking 2.3

k1 存进 CP0_ENTRYHI
a0 存进CP0_ENTRYHI
当k0 小于 0 时 跳转到 NOFOUND
