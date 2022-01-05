# Lab: system calls

## System call tracing
### 添加桩代码  
第一步我们首先要声明 trace() 这一系统指令的存在，在 user 文件夹下， 我们在 `user.h` , `usys.pl` 文件中添加如下的代码：
 ```c
 int trace(int);
 ```
 ```c
entry("trace");
 ```
在 kernel 文件夹下，我们在 `syscall.h` 中添加如下的代码
```c
 #define SYS_trace  22
```
###  实现 sys_trace() 系统指令
首先我们需要在 `proc.h` 文件下修改如下的代码，在 proc 结构体中添加 mask 变量  
```c
struct proc {
  struct spinlock lock;

  // p->lock must be held when using these:
  enum procstate state;        // Process state
  struct proc *parent;         // Parent process
  void *chan;                  // If non-zero, sleeping on chan
  int killed;                  // If non-zero, have been killed
  int xstate;                  // Exit status to be returned to parent's wait
  int pid;                     // Process ID
  int mask;
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
之后我们需要在 `sysproc.c` 文件中添加 sys_trace() 命令来设置mask变量，具体代码如下：
```c
uint64
sys_trace(void){
  int mask;
  if(argint(0, &mask) < 0)
  	return -1;
  struct proc *p = myproc();
  p -> mask = mask;
  return mask;
  
}
```
为了在调用某一系统指令时打印出要求字段，我们还需要修改 `syscall.c` 文件，具体代码如下：
```c
extern uint64 sys_trace(void);
static uint64 (*syscalls[])(void) = {
[SYS_fork]    sys_fork,
[SYS_exit]    sys_exit,
[SYS_wait]    sys_wait,
[SYS_pipe]    sys_pipe,
[SYS_read]    sys_read,
[SYS_kill]    sys_kill,
[SYS_exec]    sys_exec,
[SYS_fstat]   sys_fstat,
[SYS_chdir]   sys_chdir,
[SYS_dup]     sys_dup,
[SYS_getpid]  sys_getpid,
[SYS_sbrk]    sys_sbrk,
[SYS_sleep]   sys_sleep,
[SYS_uptime]  sys_uptime,
[SYS_open]    sys_open,
[SYS_write]   sys_write,
[SYS_mknod]   sys_mknod,
[SYS_unlink]  sys_unlink,
[SYS_link]    sys_link,
[SYS_mkdir]   sys_mkdir,
[SYS_close]   sys_close,
[SYS_trace]   sys_trace,
};
void
syscall(void)
{
  int num;
  struct proc *p = myproc();

  num = p->trapframe->a7;
  
  if(num > 0 && num < NELEM(syscalls) && syscalls[num]) {
   p->trapframe->a0 = syscalls[num]();
   if((p -> mask > 0) && (p -> mask & (1 << num))){
      printf("%d: syscall %s -> %d \n",p -> pid, sysnames[num], p -> trapframe -> a0);
   }
  } else {
    printf("%d %s: unknown sys call %d\n",
            p->pid, p->name, num);
    p->trapframe->a0 = -1;
  }
 
```
### 修改 fork() 函数
为了可以跟踪子进程的命令调用情况，我们修改 `proc.c` 文件中的 fork() 函数，具体代码如下：
```c
int
fork(void)
{
  int i, pid;
  struct proc *np;
  struct proc *p = myproc();

  // Allocate process.
  if((np = allocproc()) == 0){
    return -1;
  }

  // Copy user memory from parent to child.
  if(uvmcopy(p->pagetable, np->pagetable, p->sz) < 0){
    freeproc(np);
    release(&np->lock);
    return -1;
  }
  np->sz = p->sz;
  np -> mask = p -> mask;
  np->parent = p;

  // copy saved user registers.
  *(np->trapframe) = *(p->trapframe);

  // Cause fork to return 0 in the child.
  np->trapframe->a0 = 0;

  // increment reference counts on open file descriptors.
  for(i = 0; i < NOFILE; i++)
    if(p->ofile[i])
      np->ofile[i] = filedup(p->ofile[i]);
  np->cwd = idup(p->cwd);

  safestrcpy(np->name, p->name, sizeof(p->name));

  pid = np->pid;

  np->state = RUNNABLE;

  release(&np->lock);

  return pid;
}

```
## Sysinfo
### 添加桩代码
添加桩代码的步骤如systemtrace中一致，这里不再赘述
### 获取空闲内存
首先在 `kalloc.c` 中添加函数 get_freemem() 获取空闲内存，具体代码如下：
```c
void
get_freemem(uint64* freemem){
  struct run* r;
  uint64 num = 0;
  acquire(&kmem.lock);
  r = kmem.freelist;
  while(r){
  	num += PGSIZE;
  	r = r->next;
  }
 release(&kmem.lock);
 *freemem = num;
}

```
### 获取进程总数

我们需要在 `proc.c` 中添加函数 get_nproc() , 具体代码如下 ：
```c
void
get_nproc(uint64* nproc){
  struct proc* p;
  uint64 num = 0;
  for(p = proc; p < &proc[NPROC]; p++){
  	if (p->state != UNUSED)
  		num++; 
  }
  *nproc = num;
}
```

### 实现sys_sysinfo()
首先，我们需要在 `defs.h` 文件中声明 get_freemem() , 和 get_nproc() 两个函数，然后在 `sysproc.c` 这实现 sys_sysinfo() 函数，具体代码如下：

```c
uint64
sys_sysinfo(void){
  struct sysinfo S;
  uint64 Sinfo;
  if(argaddr(0, &Sinfo) < 0)
  	return -1;
  get_freemem(&S.freemem);
  get_nproc(&S.nproc);
  struct proc *p = myproc();
  if(copyout(p->pagetable, Sinfo, (char *)&S, sizeof(S)) < 0)
      return -1;
  return 1;
}
```
实现 sys_sysinfo() 函数后，我们仍需在 `syscall.c` 中声明该函数。 

