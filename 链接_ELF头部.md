程序本质上是一系列的指令，我们假定是*I*1,*I*2,*I*3,...,*I*n-1这样的一个指令序列，其中的某条指令调用了函数f，但f函数的代码不在这个指令序列内，也就是说程序即将跳转到这个指令序列以外的指令去执行，但当前只有这样一个指令序列，没有其他的指令，因此我们在这段指令序列之后再加一条指令*I*n，其功能和nop类似，什么也不做，在每次执行跳转类的指令(我们假设跳转到*I*x)的时候做一个类似于%rip=min(*I*n,*I*x)的保护措施，使得程序执行流不会脱离这个指令序列，链接器以及链接的过程要做的就是将*I*n指令和*I*x指令之间建立起一种连接，使得程序跳转到*I*n的时候可以立即跳转到到*I*x去执行，就像下面这段程序

```c
#include<stdio.h>
int main()
{
	printf("hello world");
  return 0;
}
```

在预编译时，stdio.h的内容会被复制到我们的代码中，但是stdio.h里只有对printf的声明，而没有对它的定义，我们执行到含有printf的那行代码时，之后应该执行printf的指令，因此需要链接器将我们的上面这段程序和实现printf的指令建立起一种连接，能让我们由自己的程序成功跳转到printf函数内部去执行

静态链接器的工作分为如下三个步骤：

1.依据ELF的机制对被引用的外部函数进行定位

2.符号解析

3.重定位

ELF文件(一般来说就是Linux系统下编译生成的可执行文件，全称是**可执行与可链接格式**)中有它所定义的数据和函数，我们可以通过得到它们在ELF文件内的偏移量和它们自身的大小从而完成对它们的定位，如果我们创建一个C程序文件，其中只声明数据和函数，像下面这样

<img src="https://tva1.sinaimg.cn/large/008i3skNly1gwfpubi3etj30o805cgm4.jpg" style="zoom:50%;" />

然后将它编译为.o后缀的ELF文件

在一个ELF文件最开始的头部是ELF header，用于说明该文件是ELF文件(通过ELF header里的magic number)，以及包含关于这个ELF文件的各种信息(Linux系统ELF相关规则规定，整个ELF header会占用ELF文件最开始的0x3f个字节)

在Linux系统的usr/include/elf.h中有对于elf头部的数据结构的定义，如下图：

<img src="https://tva1.sinaimg.cn/large/008i3skNly1gwfq7zwpe4j30pk0gq0vq.jpg" style="zoom:50%;" />

这个结构体最开始用于记录magic number和其他信息的e_ident字符数组的长度EI_NIDENT是16，之后使用Linux的工具hexdump可以观察这个elf文件的二进制信息，如下图

<img src="https://tva1.sinaimg.cn/large/008i3skNly1gwfqbmaip7j30ms03y74u.jpg" style="zoom:50%;" />

可以看到图中被选中的ELF文件的前16个字节就是magic number，ELF header中的其他变量也和ELF文件最开始的二进制信息一一对应，其中e_shoff是节头表在ELF文件中的偏移量，我们依据这个偏移量就能定位到ELF文件的节头表，ELF后面的变量e_phentsize记录每个节头表表项的大小，e_phnum记录节头表里有多少个表项，节头表(SHT, section header table)位于每个ELF文件的结尾，因此也可以通过SHT在ELF内的偏移量+SHT表项数*SHT表项大小计算出整个ELF文件的大小，在ELF header和SHT中间是一个ELF文件的各个section(节)，节头表的作用就是用来描述这些节的信息，每个表项用来描述一个节

CSAPP中也给出了ELF文件的全貌的表示图，如下

<img src="https://tva1.sinaimg.cn/large/008i3skNly1gwfrdcjo15j30u80hswg2.jpg" style="zoom:200%;" />



Reference:

https://www.bilibili.com/video/BV1MX4y1576E?spm_id_from=333.999.0.0

