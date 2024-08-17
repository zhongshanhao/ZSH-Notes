系统调用

fork系统调用执行过程

<img src="D:\_temp\网络图片\853921-20201231231343930-160569876.png" alt="853921-20201231231343930-160569876" style="zoom: 50%;" />

系统调用入口

将系统fork调用号加载到寄存器，用ecall指令由用户态**陷入**到内核态，根据SYS_fork调用相应的中断处理函数，SYS_fork是一个数字，与内核系统fork的调用号一致

usys.S

```
.global fork
fork:
 li a7, SYS_fork
 ecall
 ret
```

trampoline.S

用来在内核态和用户态之间切换的代码，做了两件事

- uservec：保存用户线程寄存器在TRAPFRAME中，有一些寄存器中保存了系统调用参数
- userret：恢复状态

然后到trap.c  中的usertrap，usertrap用来处理来自用户空间的中断、异常和系统调用，这也是用户获得机器资源的三种方式

syscall.c 根据系统调用号SYS_fork找到相应的处理函数

sysproc.c 中的sys_fork函数，这个程序相当于一个装饰器，将系统调用所需的参数从中断帧中取出，然后调用proc.c 中的fork函数去创建进程