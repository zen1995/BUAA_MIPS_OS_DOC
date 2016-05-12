lab3 report
============================

## 实验内容分析

### exercise 3.1
>   
• 修改pmap.c/mips_vm_init 函数来为envs 数组分配空间。  
• envs 数组包含NENV 个Env 结构体成员，你可以参考pmap.c 中已经写过
的pages 数组空间的分配方式。  
• 除了要为数组envs 分配空间外，你还需要使用pmap.c 中你填写过的一个内
核态函数为其进行段映射，envs 数组应该被UENVS 区域映射，你可以参
考./include/mmu.h。

关于分配env空间的代码如下
```
envs = (struct Env *)alloc(NENV * sizeof(struct Env), BY2PG, 1);  // NENV :20¸ö
n = ROUND(NENV * sizeof(struct Env), BY2PG);
boot_map_segment(pgdir, UENVS, n, PADDR(envs), PTE_R);
```

其中，我们需要注意的有两点
1. boot_map_segment 的原型：
  `void boot_map_segment(Pde *pgdir, u_long va, u_long size, u_long pa, int perm)`
  其中 va和pa,size都是需要4k对齐的

  在 include/mmu.h 中 我们找到了对 UENVS 的定义

  ```
  o      ULIM     -----> +----------------------------+------------0x8000 0000-------    
  o                      |         User VPT           |     PDMAP                /|\
  o      UVPT     -----> +----------------------------+------------0x7fc0 0000    |
  o                      |         PAGES              |     PDMAP                 |
  o      UPAGES   -----> +----------------------------+------------0x7f80 0000    |
  o                      |         ENVS               |     PDMAP                 |
  o  UTOP,UENVS   -----> +----------------------------+------------0x7f40 0000    |
  o  UXSTACKTOP -/       |     user exception stack   |     BY2PG                 |
  o                      +----------------------------+------------0x7f3f f000    |
  o                      |       Invalid memory       |     BY2PG                 |
  o      USTACKTOP ----> +----------------------------+------------0x7f3f e000    |
  o                      |     normal user stack      |     BY2PG                 |
  o                      +----------------------------+------------0x7f3f d000    |
  a                      |                            |                           |
  a                      ~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~                           |
  a                      .                            .                           |
  a                      .                            .                         kuseg
  a                      .                            .                           |
  a                      |~~~~~~~~~~~~~~~~~~~~~~~~~~~~|                           |
  a                      |                            |                           |
  o       UTEXT   -----> +----------------------------+                           |
  o                      |                            |     2 * PDMAP            \|/
  a     0 ------------>  +----------------------------+ -----------------------------
```
UENVS 和envs 本来就是4k对齐的。这里我们只需要用 ROUND 宏将size对齐即可

### exercise 3.3
> 根据上面的提示与代码注释，填写env_ alloc 函数。

我们先了解一下其他的相关函数：
```
/* Overview:
 *  Initialize the kernel virtual memory layout for environment e.
 *  Allocate a page directory, set e->env_pgdir and e->env_cr3 accordingly,
 *  and initialize the kernel portion of the new environment's address space.
 *  Do NOT map anything into the user portion of the environment's virtual address space.
 */
static int
env_setup_vm(struct Env *e)
{

	int i, r;
	struct Page *p = NULL;
	Pde *pgdir;

    /*Step 1: Allocate a page for the page directory and add its reference.
     *pgdir is the page directory of Env e. */
	if ((r = page_alloc(&p)) < 0) {
		panic("env_setup_vm - page_alloc error\n");
		return r;
	}
	p->pp_ref++;
	pgdir = (Pde *)page2kva(p);

    /*Step 2: Zero pgdir's field before UTOP. */
	for (i = 0; i < PDX(UTOP); i++) {
		pgdir[i] = 0;
	}

    /*Step 3: Copy kernel's boot_pgdir to pgdir. */

    /* Hint:
     *  The VA space of all envs is identical above UTOP
     *  (except at VPT and UVPT, which we've set below).
     *  See ./include/mmu.h for layout.
     *  Can you use boot_pgdir as a template?
     */
	for (i = PDX(UTOP); i <= PDX(~0); i++) {
		pgdir[i] = boot_pgdir[i];
	}
	e->env_pgdir = pgdir;
	e->env_cr3   = PADDR(pgdir);

    /*Step 4: VPT and UVPT map the env's own page table, with
     *different permissions. */
    e->env_pgdir[PDX(VPT)]   = e->env_cr3;
    e->env_pgdir[PDX(UVPT)]  = e->env_cr3 | PTE_V | PTE_R;
	return 0;
}
```
env_setup_vm 的作用是初始化进程共有的相关信息。
包括:
1.  分配给进程一个页目录  
    这里我们使用page2kva 找到这个页所对应的虚拟地址。为什么是page2kva呢？
    因为尽管这个页目录是属于进程的，但它同样是系统分页系统的一部分。存在于内核系统中，所以用page2kva是恰当的。
2.  在实验的mips系统中，进程的内存分为两个部分 2G+2G。后2G的地址存放的是内核的相关信息，当进程处于陷入内核状态的时候
    就会对进程开放后2G的空间。  
    然而，为什么程序中用的是UTOP 而不是ULIM？
    因为UTOP-ULIM这一段虽然不属于内核程序，但是也是公共空间，所以用UTOP

看到这里 我们就可以根据注释很轻松地填好 env_alloc 函数了。
env_alloc 函数主要有三个步骤
1.  从空闲env链表中取出一个元素，并且从链表中移除它
2.  调用 env_setup_vm 初始化进程的页表
3.  手动设置trapframe 的其他信息  
    1.  e->env_tf.regs[29] = 0x7f3f e000;
        29号寄存器是栈的位置，栈是向下增长的，因此设置为 0x7f3f e000
    2.  e->env_tf.cp0_status = 0x10001004;  
        这条代码的解释在指导书中已经写的很详细了，这里就不多说了

### exercise 3.4
> 通过上面补充的知识与注释，填充load\_icode\_mapper 函数。    

```
/* Overview:
 *   This is a call back function for kernel's elf loader.
 * Elf loader extracts each segment of the given binary image.
 * Then the loader calls this function to map each segment
 * at correct virtual address.
 *
 *   `bin_size` is the size of `bin`. `sgsize` is the
 * segment size in memory.
 *
 * Pre-Condition:
 *   va aligned 4KB and bin can't be NULL.
 *
 * Post-Condition:
 *   return 0 on success, otherwise < 0.
 */
```
 正如注释所说，这个函数的作用将bin文件拷贝到va的地址即可
 那么上来就可以很容易次想到方法：
 1. 用一个循环`pgdir_walk`创建页
 2. 用`bcopy`复制数据。

 然而，你以为这样就可以了?too naive!!!  
在程序运行的过程中你会发现很多问题：
1.  `pagir_walk`的作用是创建一个页表，并且将页表映射到相应的页目录
    然而并不能如你所愿，这里必须用 page\_insert 其实pageinsert在进程初始化的时候的执行基本一致，只不过page\_insert多了一个更新块表的功能，//!!
2.  虽然说注释中说va是4kb对齐的，然而，事实并不是这样的。   
    既然如此，那么我们就手工对齐好啦    
    手工对齐首先需要将bin分为两个部分，前半部分是没有对齐的部分，后半部分是对齐的部分。   
    那么现在我们面临三个问题：
    1.  怎么将bin 分为两份？    
        我们可以用ROUNDDOWN 取得va向下BY2PG对齐的第一个地址（a），offset = va-a。
        那么a-a+4KB的空间就是第一部分，剩下的就是第二部分。
    2.  然而，还会有一个问题，如果第bin_size 足够小，笑到不足以填充第二部分，甚至不足以填充完毕第一部分怎么办呢？
        判断一下就好啦，取BY2PG-offset 与 bin_size较小的一部分进行bcopy
    3.  现在我们解决了加载二进制文件前半部分对齐的问题了，那么剩下的问题是bin_size不一定等于sgsize，也就是说二进制文件结束地址也需要4KB对齐。
具体代码：
```
static int load_icode_mapper(u_long va, u_int32_t sgsize,
							 u_char *bin, u_int32_t bin_size, void *user_data)
{
	struct Env *env = (struct Env *)user_data;
	struct Page *p = NULL;
	u_long i;
	int j;
	int r;
	Pte *pte;
	//int result;
	u_long offset = va - ROUNDDOWN(va, BY2PG);
	/*Step 1: load all content of bin into memory. *//
	int copysize = 0;
	if(offset > 0){
		r = page_alloc(&p);
		if(r < 0){
			return r;
		}
		p->pp_ref++;
		page_insert(env->env_pgdir,p,va-offset,PTE_V);
		if(bin_size < BY2PG-offset){
			copysize = bin_size;
			bcopy((void *)bin,(void *)(page2kva(p)+offset),copysize);

		}
		else{
			copysize = BY2PG-offset;
			bcopy((void *)bin,(void *)(page2kva(p)+offset),copysize);
		}
		va = va + BY2PG - offset;
		bin = bin + BY2PG - offset;
		if(bin_size < BY2PG)
			bin_size = 0;
		else bin_size -= BY2PG-offset;
		if(sgsize < BY2PG)
			sgsize = 0;
		else sgsize -= BY2PG - offset;
	}


	for (i = 0; i < bin_size; i += BY2PG) {//i : offset
		/* Hint: You should alloc a page and increase the reference count of it. */
		r = page_alloc(&p);
		if(r <0){
			return -1;
		}
		page_insert(env->env_pgdir,&p,va+i,PTE_V | PTE_R);
		p->pp_ref++;
		if(bin_size -i >BY2PG)
			//bcopy(bin+i,va+i,BY2PG);
			bcopy(bin+i,(void*)page2kva(p),BY2PG);
		else{
			bcopy(bin+i,(void*)page2kva(p),bin_size-i);
			bzero((void *)(page2kva(p)+bin_size-i),i+BY2PG-bin_size);
		}
	}
	/*Step 2: alloc pages to reach `sgsize` when `bin_size` < `sgsize`.
    * i has the value of `bin_size` now. */
	while (i < sgsize) {
		if(r = page_alloc(&p) < 0){
			return -E_NO_MEM;
		}
		p->pp_ref++;
		page_insert(env->env_pgdir,p,va+i,PTE_V);
		bzero((void*)page2kva(p),BY2PG);
		i += BY2PG;
	}
	return 0;
}
```


### exercise 3.8
> 根据补充说明，填充完成`env_run` 函数

```
/* Overview:
 *  Restores the register values in the Trapframe with the
 *  env_pop_tf, and context switch from curenv to env e.
 *
 * Post-Condition:
 *  Set 'e' as the curenv running environment.
 *
 * Hints:
 *  You may use these functions:
 *      env_pop_tf and lcontext.
 */
 ```
 这里 `env_run`会在两个情况下使用。
 1. 首个程序运行。
 2. 切换进程运行。

初次运行程序和切换到某一程序运行的区别是：
* 当程序初次运行的话，加载程序运行的上下文即可
* 当程序不是初次运行，那么则将之前上下文环境保存起来，然后再去加载新程序的上下文

那么现在还有一个问题：当前的进程的环境数据(Trapframe)保存在哪里呢？
代码中已经明确给出：
```
old= (struct Trapframe *)(TIMESTACK - sizeof(struct Trapframe));
bcopy(old,&(curenv->env_tf),sizeof(struct Trapframe));
```
看到这个，联系到后面的`bcopy`很容易联想到系统在对运行中的程序的trapframe的修该不是直接反映在env的trapframe上，而是操作位于
` (struct Trapframe *)(TIMESTACK - sizeof(struct Trapframe));`的trapframe。
其中TIMESTACK 可在mmu.h中找到定义：
```
#define TIMESTACK 0x82000000
```
但事实上，在思考题中让我们区分TIMESTACK 和kernel_SP的区别。在这一点上，经过实验，将kernel_SP的数据拷贝到curenv->env_tf上，程序依然可以运行。
所以我们得出的结论是位于TIMESTACK的数据是中断是临时拷贝上去的。真正程序在运行的时cpu操纵的trapframe 是位于kernel_SP上的，而TIMESTACK上的数据只是中断是临时拷贝过去的而已。

在保存旧的程序上下文时，不要忘记更新pc：curenv->env_tf.pc = old->env_tf.epc;这样程序切换回来才能顺利地进行。

保存了旧的进程上下文之后，我们就可以开加载进程新的上下文了，直接调用系统的函数就好了
```
lcontext(KADDR(e->env_cr3));//implemented at lib/env_asm.s
env_pop_tf(&(curenv->env_tf),GET_ENV_ASID(curenv->env_id));
```

##  实验思考题

### thinking 3.1
> 为什么我们在构造空闲进程链表时使用了逆序插入的方式？

因为我们用的宏定义`LIST_INSERT_HEAD`要想插入后依然保持env顺序，就需要逆序插入。

事实上，经过实验，正序插入也不影响什么，逆序插入可能只是为了美观罢了。
我们在之前操纵pmap.c中的page_free_list的时候就是顺序插入的。

### thinking 3.2
> 思考env_ setup_ vm 函数：
*  第三点注释中的问题: 为什么我们要执行pgdir[i] = boot_pgdir[i]这个
赋值操作？换种说法，我们为什么要使用boot_pgdir作为一部分模板？(提
示:mips 虚拟空间布局)

因为mips采用2G+2G的布局。普通程序占用低2G空间，当程序需要访问内核的时候，采取陷入内核状态，在不切换cr3寄存器的情况下，
访问系统空间，就需要将内核页表拷贝到用户页表中。
>*  UTOP 和ULIM 的含义分别是什么，在UTOP 到ULIM 的区域与其他用户区
相比有什么最大的区别？

UTOP是用户态下可读写的最高地址，ULIM是内核地址空间是内核态可读写的最低地址，从UTOP到ULIM区域相比其他区域的最大区别是内核和用户对于地址范围[UTOP, ULIM]有着相同的访问权限，那就是可以读取但是不可以写入。这一个部分的地址空间通常被用于把一些只读的内核数据结构暴露给用户地址空间的代码。

>*  (选做) 我们为什么要让pgdir[PDX(UVPT)]=env_cr3?(提示: 结合系统自映射
机制)

### thinking 3.3
> 思考user_data 这个参数的作用。没有这个参数可不可以？为什
么？（如果你能说明哪些应用场景中可能会应用这种设计就更好了。可以举一个实
际的库中的例子）

因为我们需要`user_data`这个参数作为传入的env.
user_data的类型为void * 的好处是我们可以在不同情况下传入不同的类型。

### thinking 3.5
> 思考一下，要保存的进程上下文中的env_tf.pc的值应该设置为多少？ 为什么要这样设置？

设置为epc的值，因为进程切换时进程处于中断状态，所以设置为epc

### thinking 3.4
> 思考上面这一段话，并根据自己在lab2 中的理解，回答：
• 我们这里出现的” 指令位置” 的概念，你认为该概念是针对虚拟空间，还是物
理内存所定义的呢？
• 你觉得entry_point其值对于每个进程是否一样？该如何理解这种统一或不
同？
• 从布局图中找到你认为最有可能是entry_point的值。

mips没有实模式，所以一切地址都是需要查表/直接映射获得的。

entry_point是程序入口，虽然每个程序地址不一样，但是每个程序页表不同，所以还是有可能entry_point一样的。
并且，在pcb中我们并没有保存程序入口，那么理论上来说entry_point应该是一个定值，否则系统不能找到程序入口。
实验证明，entry_point的值为0x4000b0

### thinking 3.6
> 思考TIMESTACK 的含义，并找出相关语句与证明来回答以下关于
TIMESTACK 的问题：
• 请给出一个你认为合适的TIMESTACK 的定义
• 请为你的定义在实验中找出合适的代码段作为证据(请对代码段进行分析)
• 思考TIMESTACK 和第18 行的KERNEL_SP 的含义有何不同

* TIMESTACK应该是时钟中断发生时寄存器值的存储区的起始地址
* TIMESTACK应该是时钟中断发生时寄存器值的存储区，而KERNEL_SP应该是正常系统运行时所操控的空间

### thinking 3.7
> 思考一下你的调度程序，这种调度方式由于某种不可避免的缺陷而造成对进程的不公平。    
• 这种不公平是如何产生的？    
• 如果实验确定只运行两个进程，你如何改进可以降低这种不公平？     

目前的调度是clock算法遍历envs数组调度。并不是先来先服务。   
若要实现先来先服务可以维护一个进程的创建时间链表，遍历链表来进行调度。
