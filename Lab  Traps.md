#  Lab : Traps



##   Backtrace



 这里只列出 backtrace() 函数的代码



     ```c
     void
     backtrace()
     {
     	uint64 fp = r_fp();
     	uint64 fpup = PGROUNDUP(fp);
             printf("backtrace:\n");
     	while (fp < fpup){
     		printf("%p\n", *((uint64*)(fp - 8)));
     		fp = *((uint64*)(fp - 16));
     	} 
     }
     ```



## Alarm

首先列出对 `kernel/proc.h` 文件修改的部分

```c
struct trapframe {
  /*   0 */ uint64 kernel_satp;   // kernel page table
  /*   8 */ uint64 kernel_sp;     // top of process's kernel stack
  /*  16 */ uint64 kernel_trap;   // usertrap()
  /*  24 */ uint64 epc;           // saved user program counter
  /*  32 */ uint64 kernel_hartid; // saved kernel tp
  /*  40 */ uint64 ra;
  /*  48 */ uint64 sp;
  /*  56 */ uint64 gp;
  /*  64 */ uint64 tp;
  /*  72 */ uint64 t0;
  /*  80 */ uint64 t1;
  /*  88 */ uint64 t2;
  /*  96 */ uint64 s0;
  /* 104 */ uint64 s1;
  /* 112 */ uint64 a0;
  /* 120 */ uint64 a1;
  /* 128 */ uint64 a2;
  /* 136 */ uint64 a3;
  /* 144 */ uint64 a4;
  /* 152 */ uint64 a5;
  /* 160 */ uint64 a6;
  /* 168 */ uint64 a7;
  /* 176 */ uint64 s2;
  /* 184 */ uint64 s3;
  /* 192 */ uint64 s4;
  /* 200 */ uint64 s5;
  /* 208 */ uint64 s6;
  /* 216 */ uint64 s7;
  /* 224 */ uint64 s8;
  /* 232 */ uint64 s9;
  /* 240 */ uint64 s10;
  /* 248 */ uint64 s11;
  /* 256 */ uint64 t3;
  /* 264 */ uint64 t4;
  /* 272 */ uint64 t5;
  /* 280 */ uint64 t6;
  /*  40 */ uint64 rra;
  /*  48 */ uint64 rsp;
  /*  56 */ uint64 rgp;
  /*  64 */ uint64 rtp;
  /*  72 */ uint64 rt0;
  /*  80 */ uint64 rt1;
  /*  88 */ uint64 rt2;
  /*  96 */ uint64 rs0;
  /* 104 */ uint64 rs1;
  /* 112 */ uint64 ra0;
  /* 120 */ uint64 ra1;
  /* 128 */ uint64 ra2;
  /* 136 */ uint64 ra3;
  /* 144 */ uint64 ra4;
  /* 152 */ uint64 ra5;
  /* 160 */ uint64 ra6;
  /* 168 */ uint64 ra7;
  /* 176 */ uint64 rs2;
  /* 184 */ uint64 rs3;
  /* 192 */ uint64 rs4;
  /* 200 */ uint64 rs5;
  /* 208 */ uint64 rs6;
  /* 216 */ uint64 rs7;
  /* 224 */ uint64 rs8;
  /* 232 */ uint64 rs9;
  /* 240 */ uint64 rs10;
  /* 248 */ uint64 rs11;
  /* 256 */ uint64 rt3;
  /* 264 */ uint64 rt4;
  /* 272 */ uint64 rt5;
  /* 280 */ uint64 rt6;
            uint64 repc;
};
struct proc {
  struct spinlock lock;

  // p->lock must be held when using these:
  enum procstate state;        // Process state
  struct proc *parent;         // Parent process
  void *chan;                  // If non-zero, sleeping on chan
  int killed;                  // If non-zero, have been killed
  int xstate;                  // Exit status to be returned to parent's wait
  int pid;                     // Process ID
  int interval;
  int lefttime;
  int tag;
  uint64 hd;
  // these are private to the process, so p->lock need not be held.
  uint64 kstack;               // Virtual address of kernel stack
  uint64 sz;                   // Size of process memory (bytes)
  pagetable_t pagetable;       // User page table
  struct trapframe *trapframe; // data page for trampoline.S
  struct context context;      // swtch() here to run process
  struct file *ofile[NOFILE];  // Open files
  struct inode *cwd;           // Current directory
  char name[16];               // Process name (debugging)
};
```

​	

再列出对  `kernel/sysproc.c` 文件中的修改部分

    ```c
    extern uint64 handlerset[];
    extern int handlerfree[];
    extern int handlernum;
    void restorereg(struct proc* p){
      p -> trapframe -> epc = p -> trapframe -> repc;
      p -> trapframe -> ra = p -> trapframe -> rra;
      p -> trapframe -> sp = p -> trapframe -> rsp;
      p -> trapframe -> gp = p -> trapframe -> rgp;
      p -> trapframe -> tp = p -> trapframe -> rtp;
      p -> trapframe -> t0 = p -> trapframe -> rt0; 
      p -> trapframe -> t1 = p -> trapframe -> rt1;
      p -> trapframe -> t2 = p -> trapframe -> rt2;
      p -> trapframe -> s0 = p -> trapframe -> rs0;
      p -> trapframe -> s1 = p -> trapframe -> rs1;
      p -> trapframe -> a0 = p -> trapframe -> ra0;
      p -> trapframe -> a1 = p -> trapframe -> ra1;
      p -> trapframe -> a2 = p -> trapframe -> ra2;
      p -> trapframe -> a3 = p -> trapframe -> ra3;
      p -> trapframe -> a4 = p -> trapframe -> ra4;
      p -> trapframe -> a5 = p -> trapframe -> ra5;
      p -> trapframe -> a6 = p -> trapframe -> ra6;
      p -> trapframe -> a7 = p -> trapframe -> ra7;
      p -> trapframe -> s2 = p -> trapframe -> rs2;
      p -> trapframe -> s3 = p -> trapframe -> rs3;
      p -> trapframe -> s4 = p -> trapframe -> rs4;
      p -> trapframe -> s5 = p -> trapframe -> rs5;
      p -> trapframe -> s6 = p -> trapframe -> rs6;
      p -> trapframe -> s7 = p -> trapframe -> rs7;
      p -> trapframe -> s8 = p -> trapframe -> rs8;
      p -> trapframe -> s9 = p -> trapframe -> rs9;
      p -> trapframe -> s10 = p -> trapframe -> rs10;
      p -> trapframe -> s11 = p -> trapframe -> rs11;
      p -> trapframe -> t3 = p -> trapframe -> rt3;
      p -> trapframe -> t4 = p -> trapframe -> rt4;
      p -> trapframe -> t5 = p -> trapframe -> rt5;
      p -> trapframe -> t6 = p -> trapframe -> rt6;
    }
    uint64
    sys_sigalarm(void){
     
      struct proc* p = myproc();
      int interval;
      if (argint(0, &interval) < 0)
      	return -1;
      p -> interval = interval;
      p -> lefttime = interval;
      uint64 hd;
      if (argaddr(0, &hd) < 0)
      	return -1;
      p -> hd = hd;
      p -> tag = -1;
      for (int i = 0; i < handlernum; i++){
      	if (handlerset[i] == p -> hd)
      		p -> tag = i; 
      }
      if (p -> tag < 0){
      	p -> tag = handlernum++;
      	handlerset[p -> tag] = p -> hd;
      	handlerfree[p -> tag] = 1;
      }
      return 0;
    }
    uint64
    sys_sigreturn(void){
            struct proc* p = myproc();
            restorereg(p);
            handlerfree[p -> tag] = 1; 
            return 0;
    }
    ```



最后列出对 `trap.c` 文件中修改的部分

```c
#define maxhandlernum  10
uint64 handlerset[maxhandlernum];
int handlerfree[maxhandlernum];
int handlernum = 0;
void storereg(struct proc* p){
  p -> trapframe ->repc = p -> trapframe -> epc;
  p -> trapframe -> rra = p -> trapframe -> ra;
  p -> trapframe -> rsp = p -> trapframe -> sp;
  p -> trapframe -> rgp = p -> trapframe -> gp;
  p -> trapframe -> rtp = p -> trapframe -> tp;
  p -> trapframe -> rt0 = p -> trapframe -> t0; 
  p -> trapframe -> rt1 = p -> trapframe -> t1;
  p -> trapframe -> rt2 = p -> trapframe -> t2;
  p -> trapframe -> rs0 = p -> trapframe -> s0;
  p -> trapframe -> rs1 = p -> trapframe -> s1;
  p -> trapframe -> ra1 = p -> trapframe -> a1;
  p -> trapframe -> ra2 = p -> trapframe -> a2;
  p -> trapframe -> ra3 = p -> trapframe -> a3;
  p -> trapframe -> ra4 = p -> trapframe -> a4;
  p -> trapframe -> ra5 = p -> trapframe -> a5;
  p -> trapframe -> ra6 = p -> trapframe -> a6;
  p -> trapframe -> ra7 = p -> trapframe -> a7;
  p -> trapframe -> rs2 = p -> trapframe -> s2;
  p -> trapframe -> rs3 = p -> trapframe -> s3;
  p -> trapframe -> rs4 = p -> trapframe -> s4;
  p -> trapframe -> rs5 = p -> trapframe -> s5;
  p -> trapframe -> rs6 = p -> trapframe -> s6;
  p -> trapframe -> rs7 = p -> trapframe -> s7;
  p -> trapframe -> rs8 = p -> trapframe -> s8;
  p -> trapframe -> rs9 = p -> trapframe -> s9;
  p -> trapframe -> rs10 = p -> trapframe -> s10;
  p -> trapframe -> rs11 = p -> trapframe -> s11;
  p -> trapframe -> rt3 = p -> trapframe -> t3;
  p -> trapframe -> rt4 = p -> trapframe -> t4;
  p -> trapframe -> rt5 = p -> trapframe -> t5;
  p -> trapframe -> rt6 = p -> trapframe -> t6;
  p -> trapframe -> ra0 = p -> trapframe -> a0;
 }
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
  } else {
    printf("usertrap(): unexpected scause %p pid=%d\n", r_scause(), p->pid);
    printf("            sepc=%p stval=%p\n", r_sepc(), r_stval());
    p->killed = 1;
  }

  if(p->killed)
    exit(-1);

  // give up the CPU if this is a timer interrupt.
  if(which_dev == 2){
    yield();
    if (p -> interval != 0){
               if (p -> lefttime > 1){
    		p -> lefttime--;
    	}
    	else if (handlerfree[p -> tag]){
    		 
    		storereg(p);
    		p -> trapframe -> epc = p -> hd;
    		p -> trapframe -> sp -= 16;
    		p -> lefttime = p -> interval;
    		handlerfree[p -> tag] = 0;
    	   
    	   }
   	}   
   }

  usertrapret();
}
```

