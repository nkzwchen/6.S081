# Lab: Copy-on-Write Fork

主要修改了 kernel/vm.c 中的 copyout() ，uvmcopy()，mappages() , uvmunmap()以及 kernel/trap.c 中的 usertrap() , riscv.h , kalloc.c文件，修改内容如下

```c
//vm.c
int
mappages(pagetable_t pagetable, uint64 va, uint64 size, uint64 pa, int perm)
{
  uint64 a, last;
  pte_t *pte;

  a = PGROUNDDOWN(va);
  last = PGROUNDDOWN(va + size - 1);
  for(;;){
    if((pte = walk(pagetable, a, 1)) == 0)
      return -1;
    if(*pte & PTE_V)
      panic("remap");
    *pte = (PA2PTE(pa) | perm | PTE_V) & ~PTE_C;
    if(a == last)
      break;
    a += PGSIZE;
    pa += PGSIZE;
  }
  return 0;
}
```



```c
//vm.c
int
uvmcopy(pagetable_t old, pagetable_t new, uint64 sz)
{
  pte_t *pte;
  uint64 pa, i;
  uint flags;
  //char *mem;

  for(i = 0; i < sz; i += PGSIZE){
    if((pte = walk(old, i, 0)) == 0)
      panic("uvmcopy: pte should exist");
    if((*pte & PTE_V) == 0)
      panic("uvmcopy: page not present");
    pa = PTE2PA(*pte);
    *pte = MASK_W(*pte);
    flags = PTE_FLAGS(*pte);
    if((*pte & PTE_C) == 0){
    	*pte = *pte | PTE_C;
    	PAGEREF[pa/PGSIZE]++;
    }
    if(mappages(new, i, PGSIZE, (uint64)pa, flags) != 0){
	goto err;
    }
    pte = walk(new, i, 0);
    *pte = *pte | PTE_C;
    PAGEREF[pa/PGSIZE]++;
   }
  return 0;

 err:
  uvmunmap(new, 0, i / PGSIZE, 0);
  return -1;
}
```

```c
//vm.c
void
uvmunmap(pagetable_t pagetable, uint64 va, uint64 npages, int do_free)
{
  uint64 a;
  pte_t *pte;

  if((va % PGSIZE) != 0)
    panic("uvmunmap: not aligned");

  for(a = va; a < va + npages*PGSIZE; a += PGSIZE){
    if((pte = walk(pagetable, a, 0)) == 0)
      panic("uvmunmap: walk");
    if((*pte & PTE_V) == 0)
      panic("uvmunmap: not mapped");
    if(PTE_FLAGS(*pte) == PTE_V)
      panic("uvmunmap: not a leaf");
    if(do_free){
      uint64 pa = PTE2PA(*pte);
      if ((*pte & PTE_C) == 0 || --PAGEREF[pa/PGSIZE] == 0){
      	    kfree((void*)pa);
      }
    }
    *pte = 0;
  }
}
```



```c
//vm.c
int
copyout(pagetable_t pagetable, uint64 dstva, char *src, uint64 len)
{
  uint64 n, va0, pa0;
  while(len > 0){
    va0 = PGROUNDDOWN(dstva);
    if(va0 >= MAXVA - PGSIZE)
       return -1;
    pte_t* pte = walk(pagetable, va0, 0);
    if (pte == 0)
    	return -1;

    if((*pte & PTE_W) == 0 && (*pte & PTE_C) != 0){
        uint flags = PTE_FLAGS(*pte)|PTE_W;
        uint64 pa = PTE2PA(*pte);
        char* mem = kalloc();
    	if (mem == 0)
    		return -1;
    	uvmunmap(pagetable, va0, 1, 0);
    	memmove(mem, (char*) pa, PGSIZE);
        if(mappages(pagetable, va0, PGSIZE, (uint64)mem, flags) != 0){
  		kfree(mem);
  		return  -1;
  	}
  	else if(--PAGEREF[pa/PGSIZE] == 0){
  		kfree((char *)pa);
  	}	
    }
    
    pa0 = walkaddr(pagetable, va0);
    if(pa0 == 0)
       return -1;
    n = PGSIZE - (dstva - va0);
    if(n > len)
      n = len;
    memmove((void *)(pa0 + (dstva - va0)), src, n);
    len -= n;
    src += n;
    dstva = va0 + PGSIZE;
  }
  return 0;
}
```

```c
//trap.c
void
usertrap(void)
{
  int which_dev = 0;

  if((r_sstatus() & SSTATUS_SPP) != 0)
    panic("usertrap: not from user mode");

  // send interrupts and exceptions to kerneltrap(),
  // since we're now in the kernel.
  w_stvec((uint64)kernelvec);

  struct proc *p = myproc();
  
  // save user program counter.
  p->trapframe->epc = r_sepc();
  
  if(r_scause() == 8){
    // system call

    if(p->killed)
      exit(-1);

    // sepc points to the ecall instruction,
    // but we want to return to the next instruction.
    p->trapframe->epc += 4;

    // an interrupt will change sstatus &c registers,
    // so don't enable until done with those registers.
    intr_on();

    syscall();
  } else if((which_dev = devintr()) != 0){
    // ok
  } else if(r_scause() == 13 || r_scause() == 15){
  	struct proc* p = myproc();
	pte_t *pte;
  	uint64 va0 = PGROUNDDOWN(r_stval());
  	if((pte = walk(p->pagetable, va0, 0)) != 0 && ((*pte & PTE_W) == 0) && (*pte & PTE_C) != 0){
  		char* mem;
  		if((mem = kalloc()) !=  0){
  			
  			if(va0 <= MAXVA - PGSIZE){
  				uint64 pa = PTE2PA(*pte);
  				uint flags = PTE_FLAGS(*pte) | PTE_W;
  				uvmunmap(p->pagetable, va0, 1, 0);
  				memmove(mem, (char*) pa, PGSIZE);
  				if(mappages(p->pagetable, va0, PGSIZE, (uint64)mem, flags) != 0){
  					kfree(mem);
  					p->killed = 1;
  				}
  				else if(--PAGEREF[pa/PGSIZE] == 0){
  				      kfree((char *)pa);
  				}	
  			}

			else
				p->killed = 1;
  				
  		}
  		else
  			p->killed = 1;
  		
  	}
  	else
  		p->killed = 1;
  }
  else {
    printf("usertrap(): unexpected scause %p pid=%d\n", r_scause(), p->pid);
    printf("            sepc=%p stval=%p\n", r_sepc(), r_stval());
    p->killed = 1;
  }

  if(p->killed)
    exit(-1);

  // give up the CPU if this is a timer interrupt.
  if(which_dev == 2)
    yield();

  usertrapret();
}

```

```c
//kalloc.c
int PAGEREF[(uint64)PHYSTOP/PGSIZE + 1]={0};
```

```c
// riscv.h
#define PTE_C (1L << 8)
#define MASK_W(pte) (pte & (~PTE_W))
```

