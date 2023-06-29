- `stvec`: 存储trap handler， tvec= trap vec?  发生trap时，把pc = stvec
- `sepc`: 当trap发生时，把pc存在这里， 终端返回sret把pc=sepc
- `scause`: trap reason
- `sscratch`: 进入trap前保存user模式的一些状态的地址，代码里面就是trapframe
- `sstatus`:  SIE 中断控制  SPP:发生trap前在那个模式
- `satp`:页表root物理地址
- `stval`:保存page-fault 


# uservec是哪里设置的？
```
userinit-->allocproc
    p->context.ra = (uint64)forkret; //这个设置会让这个process switch的ra变成forkret
//ra 寄存器存的是ret的返回地址，也就是ret要跳转的地方  ra=return address
//这样在switch函数内
.globl swtch
swtch:
...
        ld s9, 88(a1)
        ld s10, 96(a1)
        ld s11, 104(a1)

        ret  //这里就直接开始往ra地方跳转，也就是从scheduler-->p 的时候先跳到forkret,这样到user mode后uservec就被设置了
        //一旦user mode发生了trap,就会选择uservec来进行下一步动作

 
// A fork child's very first scheduling by scheduler()
// will swtch to forkret.
forkret
   usertrapret(void)
   {
      // send syscalls, interrupts, and exceptions to trampoline.S
      w_stvec(TRAMPOLINE + (uservec - trampoline));  这里设置了uservec给trap register


scheduler 和 sched都调用了swtch这个函数使其在scheduler和其他进程间切换，其他进程也可能在内核，也可能在用户态
   2    493  kernel/proc.c <<scheduler>>
             swtch(&c->context, &p->context);
   3    540  kernel/proc.c <<sched>>
             swtch(&p->context, &mycpu()->context);

sscratch是怎样被设置成trap frame的,由上面可以知道创建新的进程后首先会调用到forkret，而在后续的调用里面就有设置sscratch的步骤，其实userret是会被
首先调用的,userret也是用来进程跳到用户空间的步骤，之后用户空间发生了trap, cpu就会找到uservec然后执行，在uservec得到执行时sscratch就是前面说设置的trapframe.
forkret-->usertrapret-->((void (*)(uint64,uint64))fn)(TRAPFRAME, satp) //这个其实就是userret(TRAPFRAME,satp)-->
        # put the saved user a0 in sscratch, so we
        # can swap it with our a0 (TRAPFRAME) in the last step.
        ld t0, 112(a0)
        csrw sscratch, t0   <====
```


# trapframe
```
trapframe是trampoline.S. 的data部分，创建proc的时候把他map到了下面地方。
  // map the trapframe just below TRAMPOLINE, for trampoline.S.
  if(mappages(pagetable, TRAPFRAME, PGSIZE,
              (uint64)(p->trapframe), PTE_R | PTE_W) < 0){
    uvmunmap(pagetable, TRAMPOLINE, 1, 0);
    uvmfree(pagetable, 0);
    return 0;
  }
```
