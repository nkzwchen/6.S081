# Lab3:  page tables
## Print a page table

vmprint() 函数的写法如下

```c
void vmprint(pagetable_t pagetable, int level){
  if(level == 1)
    printf("page table %p\n", pagetable);
  if(level == 4)
    return;
  for(int i = 0; i < 512; i++){
    pte_t pte = pagetable[i];
    if(pte & PTE_V){
      uint64 child = PTE2PA(pte);
      for(int l = 1; l < level; l++){
      	printf(".. ");
      }
      printf("..%d: pte %p pa %p\n", i, pte, child);
      vmprint((pagetable_t)child, level + 1);
     } 
   }
}

```

## A kernel page table per process

首先在 `proc.h` 文件中添加成员kernelpagetable.

### allocproc() 的修改

我们定义了函数proc_kernelpagetable(proc* p) ， 具体函数如下

```c
pagetable_t 
proc_kernelpagetable(struct proc* p){
	pagetable_t kernelpagetable;
        kernelpagetable = uvmcreate();
        if (kernelpagetable == 0)
        	return 0;
        ukvminit(kernelpagetable);
        ukvmmap(kernelpagetable,KSTACK((int) (p - proc)),
             kvmpa(KSTACK((int) (p - proc))), PGSIZE, PTE_R | PTE_W);
        return kernelpagetable;
}
```

其中的 ukvminit() 在 vm.c 中的定义如下:

```c
void ukvminit(pagetable_t kernelpagetable){
        mappages(kernelpagetable, UART0, UART0, PGSIZE, PTE_R | PTE_W);
        mappages(kernelpagetable, VIRTIO0, VIRTIO0, PGSIZE, PTE_R | PTE_W);
        mappages(kernelpagetable, PLIC, PLIC, 0x400000, PTE_R | PTE_W);
        mappages(kernelpagetable, KERNBASE, KERNBASE, (uint64)etext-KERNBASE, PTE_R| PTE_X);
        mappages(kernelpagetable, (uint64)etext, (uint64)etext, PHYSTOP-(uint64)etext, PTE_R | PTE_W);
	    mappages(kernelpagetable, TRAMPOLINE, (uint64)trampoline, PGSIZE, PTE_R | PTE_X);
}
```


我们没有映射CLINT这一内核页表中的虚拟地址

### freeproc()的修改

我们首先定义了 proc_freekernelpagetable() 这一函数如下:

```c
void
proc_freekernelpagetable(pagetable_t kernelpagetable)
{
      ukvmfree(kernelpagetable, 1);
 }
```

其中的ukvmfree定义如下:

```c
void ukvmfree(pagetable_t pagetable, int level){
   if (level == 3)
   {
        kfree((void*) pagetable);
   	return;
   }
  for (int i = 0; i < 512; i++) {
		pte_t pte = pagetable[i];
		if (pte & PTE_V) {
		        pagetable[i] = 0;
			uint64 child = PTE2PA(pte);
			ukvmfree((pagetable_t)child, level + 1);
		}
	}
   kfree((void*)pagetable);
 }
```

### 对scheduler()函数的修改

对scheduler()函数修改如下

```c
void
scheduler(void)
{
  struct proc *p;
  struct cpu *c = mycpu();
  
  c->proc = 0;
  for(;;){
    // Avoid deadlock by ensuring that devices can interrupt.
    intr_on();
    
    int found = 0;
    for(p = proc; p < &proc[NPROC]; p++) {
      acquire(&p->lock);
      if(p->state == RUNNABLE) {
        // Switch to chosen process.  It is the process's job
        // to release its lock and then reacquire it
        // before jumping back to us.
        p->state = RUNNING;
        c->proc = p;
        w_satp(MAKE_SATP(p -> kernelpagetable));
        sfence_vma();
        swtch(&c->context, &p->context);
        // Process is done running for now.
        // It should have changed its p->state before coming back.
        kvminithart();
        c->proc = 0;
        found = 1;
      }
      
      release(&p->lock);
    }
#if !defined (LAB_FS)
    if(found == 0) {
      intr_on();
      asm volatile("wfi");
    }
#else
    ;
#endif
  }
}
```

## Simplify copyin/copyinstr

我们主要修改fork(), exec(), growproc() 三个函数

### 对 fork() 函数的修改

```c
  // Cause fork to return 0 in the child.
  np->trapframe->a0 = 0;
  
  if (ukvmcopy(np -> pagetable, np -> kernelpagetable, np -> sz) < 0){
  	freeproc(np);
  	release(&np -> lock);
  	return -1;
  }
  
```

其中 ukvmcopy()的定义如下：

```c
int ukvmcopy(pagetable_t pagetable, pagetable_t kernelpagetable, uint64 sz){
  if (sz >= PLIC)
  	panic("ukvmcopy: out of range");	
  pte_t *pte;
  uint64 pa, i;
  uint flags;
   for(i = 0; i < sz; i += PGSIZE){
   if((pte = walk(pagetable, i, 0)) == 0)
   	panic("ukvmcopy: pte should not exsit");
   if((*pte & PTE_V) == 0)
   	panic("ukvmcopy: pte should not exsit");
   pa = PTE2PA(*pte);
   flags = PTE_FLAGS(*pte) & (~PTE_U);
   if(mappages(kernelpagetable, i, PGSIZE, (uint64)pa, flags) != 0){
   	return -1;
   }
  }
  return 0;
}
```
### 对于exec()函数的修改

对于exec()函数的修改如下:

```c
  // Commit to the user image.
  oldpagetable = p->pagetable;
  p->pagetable = pagetable;
  p->sz = sz;
  p->trapframe->epc = elf.entry;  // initial program counter = main
  p->trapframe->sp = sp; // initial stack pointer
  proc_freepagetable(oldpagetable, oldsz);
  if(p->pid==1) 
    vmprint(p->pagetable, 1);
  pagetable_t oldkernelpagetable = p -> kernelpagetable;
  p -> kernelpagetable = proc_kernelpagetable(p);
  ukvmcopy(p -> pagetable, p -> kernelpagetable, p -> sz);
  w_satp(MAKE_SATP(p -> kernelpagetable));
  sfence_vma();
  proc_freekernelpagetable(oldkernelpagetable);
  return argc; // this ends up in a0, the first argument to main(argc, argv)

```

### 对于 growproc() 函数的修改：

主要添加ukvmalloc 以及 ukvmdealloc 两个函数，其定义如下：
 ```c
 uint64
ukvmalloc(pagetable_t pagetable, pagetable_t kernelpagetable, uint64 oldsz, uint64 newsz)
{
  
  if(PGROUNDDOWN(newsz) > PLIC)
  	return 0;
  char* mem;
  uint64 a;
  if(newsz < oldsz)
	return oldsz;
  oldsz = PGROUNDUP(oldsz);
  for(a = oldsz; a < newsz; a += PGSIZE){
  	mem = kalloc();
  	if(mem == 0){
  	   ukvmdealloc(pagetable, kernelpagetable, a, oldsz);
  	   return 0;
  	}
  	memset(mem, 0, PGSIZE);
        if(mappages(pagetable, a, PGSIZE, (uint64)mem, PTE_W|PTE_X|PTE_R|PTE_U) != 0){
  		kfree(mem);
  		ukvmdealloc(pagetable, kernelpagetable, a, oldsz);
  		return 0;
  	}
  	else if ((mappages(kernelpagetable, a, PGSIZE, (uint64)mem, PTE_W|PTE_X|PTE_R) != 0)){
        	printf("kernelpagetable alloc error\n");
        	kfree(mem);
        	uvmdealloc(pagetable, a + PGSIZE, a);
  		ukvmdealloc(pagetable, kernelpagetable, a, oldsz);
  		return 0;
        }
  }
  return newsz;
}
 ```
 ```c
uint64
ukvmdealloc(pagetable_t pagetable,pagetable_t kernelpagetable, uint64 oldsz, uint64 newsz)
{
  if(newsz >= oldsz)
    return oldsz;

  if(PGROUNDUP(newsz) < PGROUNDUP(oldsz)){
    int npages = (PGROUNDUP(oldsz) - PGROUNDUP(newsz)) / PGSIZE;
    uvmunmap(pagetable, PGROUNDUP(newsz), npages, 1);
    uvmunmap(kernelpagetable, PGROUNDUP(newsz), npages, 0);
  }

  return newsz;
}
 ```
