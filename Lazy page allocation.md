# Lazy page allocation



我们主要修改了四个函数，分别是 `kernel/file.c`中的 `sys_sbrk()` ，`kernel/vm.c` 中的 `copyout()` 和 `copyin()`, 以及 `kernel/trap.c`中的 `usertrap()`



```c
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
        uint64 va0 = PGROUNDDOWN(r_stval());
  	if (va0 + PGSIZE > PGROUNDUP(p->sz) ||va0 < PGROUNDDOWN(p->trapframe->sp) || va0 > MAXVA ||va0 + PGSIZE > MAXVA ){
  	     p->killed = 1;
  	}
  	else{
	    char* mem = kalloc();
	    if(mem != 0){
        	    memset(mem, 0, PGSIZE);
	    if(mappages(p -> pagetable, va0, PGSIZE, (uint64)mem, PTE_W|PTE_X|PTE_R|PTE_U) != 0){
	       kfree(mem);
              p->killed = 1;
	    }
	    }
	    else{
	    	    	p -> killed = 1;
	    }
      }
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
int
copyout(pagetable_t pagetable, uint64 dstva, char *src, uint64 len)
{
  uint64 n, va0, pa0;
  while(len > 0){
    va0 = PGROUNDDOWN(dstva);
    pa0 = walkaddr(pagetable, va0);
    if(pa0 == 0){
       if (va0 + PGSIZE > PGROUNDUP(myproc()->sz) ||va0 < PGROUNDDOWN(myproc()->trapframe->sp) || va0 > MAXVA || va0 + PGSIZE > MAXVA){
            return -1;
        }
  	else{
  	   char* mem = kalloc();
           if(mem != 0){
	    memset(mem, 0, PGSIZE);
	   if(mappages(myproc() -> pagetable, va0, PGSIZE, (uint64)mem, PTE_W|PTE_X|PTE_R|PTE_U) != 0){
              kfree(mem);
	      return -1;
	    }
	    }
	    else{
	    	return -1;
	    }
      }
      pa0 = walkaddr(pagetable, va0);
      if (pa0 == 0)
      	return -1;
    }
    n = PGSIZE - (dstva - va0);
    if(n > len)
      n = len;
    memmove((void *)(pa0 + (dstva - va0)), src, n);

    len -= n;
    src += n;
    dstva = va0 + PGSIZE;
  }
 return 0;
```

```c
int
copyin(pagetable_t pagetable, char *dst, uint64 srcva, uint64 len)
{
  uint64 n, va0, pa0;

  while(len > 0){
   
    va0 = PGROUNDDOWN(srcva);
    pa0 = walkaddr(pagetable, va0);
     if(pa0 == 0){
       if (va0 + PGSIZE > PGROUNDUP(myproc()->sz) ||va0 < PGROUNDDOWN(myproc()->trapframe->sp) || va0 > MAXVA || va0 + PGSIZE > MAXVA){
  	    return -1;
        }
  	else{
  	   char* mem = kalloc();
           if(mem != 0){
	    memset(mem, 0, PGSIZE);
	   if(mappages(myproc() -> pagetable, va0, PGSIZE, (uint64)mem, PTE_W|PTE_X|PTE_R|PTE_U) != 0){
              kfree(mem);
	      return -1;
	    }
	    }
	    else{
	    	return -1;
	    }
      }
      pa0 = walkaddr(pagetable, va0);
      if (pa0 == 0)
      	return -1;
    }
    n = PGSIZE - (srcva - va0);
    if(n > len)
      n = len;
    memmove(dst, (void *)(pa0 + (srcva - va0)), n);

    len -= n;
   dst += n;
    srcva = va0 + PGSIZE;
  }
  return 0;
}
```

```c
uint64
sys_sbrk(void)
{
  int addr;
  int n;

  if(argint(0, &n) < 0)
    return -1;
  addr = myproc()->sz;
  if (n >= 0){
      if (addr + n > MAXVA || addr > MAXVA)
        return -1;
      else
      myproc() -> sz = myproc() -> sz + n;
  }
  else{
  	if(growproc(n) < 0)
             return -1;
  }
  return addr;
}
```

