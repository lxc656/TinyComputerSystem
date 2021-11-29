继续上一篇Blog的内容，每个符号表的表项中还有st_info这个表项，它的数据类型是unsigned char，即uint8_t，有8个bit位，它的高4位是Bind的信息，低4位是Type的信息，在/usr/include/elf.h中相关的宏定义如下

```c
#define ELF32_ST_BIND(val)		(((unsigned char) (val)) >> 4)
#define ELF32_ST_TYPE(val)		((val) & 0xf)
#define ELF32_ST_INFO(bind, type)	(((bind) << 4) + ((type) & 0xf))
```

前面Blog的实例程序中的符号func1,func2,data1,data2的Bind都是1，代表它们是全局(global)的符号，data1和data2的Type都是1，说明这两个符号是object类型，func1和func2的Type都是2，说明这两个符号是function类型，关于Bind和Type在elf.h中也有相关的宏定义如下图(Bind和Type分别对应STB和STT)

<img src="https://tva1.sinaimg.cn/large/008i3skNly1gwj8tqs6lxj30ta0pin2s.jpg" style="zoom:50%;" />

Bind的值为STB_GLOBAL的符号对于其他的外部文件来说是可见的，如果我们在s1.c和s2.c中都定义了一个全局变量a，之后分别编译成s1.o和s2.o，然后把它们链接在一起，会发生报错，因为出现了两个同名的global的符号，会发生冲突，不知道a到底指的是哪个文件里的变量，如果我们把其中一个a的类型改为静态全局变量，即只在自己的文件里可见，那么它的符号的Bind也会改为STB_LOCAL，这样就不会发生冲突，如下图所示

<img src="https://tva1.sinaimg.cn/large/008i3skNly1gwjaab4pvwj30r20bwjtn.jpg" style="zoom:50%;" />

除了STB_LOCAL和STB_GLOBAL，Bind的常见取值还有STB_WEAK，如果我们在一个文件中如下声明这样一个函数

<img src="https://tva1.sinaimg.cn/large/008i3skNly1gwjah6ymymj30lg03aglm.jpg" style="zoom:50%;" />

那么add这个符号的Bind就是STB_WEAK，即一个weak bind，我们查看符号表可以得到如下结果

<img src="https://tva1.sinaimg.cn/large/008i3skNly1gwjaj8idn5j30rs030gm1.jpg" style="zoom:50%;" />

这种weak bind一般会出现在如下的场景中:

我们假设一个C程序文件的内容如下

```c
void f();
void main()
{
		f();
}
```

其中只是声明并调用了f函数，而并未给出f函数的定义，之后如果将编译后的这个程序和某个静态库链接在一起，但是我们不确定静态库中是否有这个函数的定义，如果没有在静态库中没有这个函数的定义，操作系统会认为这个函数被定义在某个动态库中，即会在程序运行的时候将该函数的定义动态加载到内存中，如果在程序运行的时候也没有找到这个动态库，那么程序就会因为引用了未被定义的函数而crash，我们如果能用下面的方法，使用__ attribute __((weak))关键字，在找不到函数定义的时候，就会执行这个函数的代码，这会便于我们做异常处理，及时发现错误

```c
__attribute__((weak)) void f()
{
  return -1;
}
```

接下来结合实际应用场景分析一下使用weak bind的作用，如下图所示

<img src="https://tva1.sinaimg.cn/large/008i3skNly1gwjb5wbatej30ma0io75b.jpg" style="zoom:50%;" />

这样的函数定义方式也叫做弱定义

我们总结一下，符号表的表项中的st_info的前4个字节，Bind值，用来记录符号对于全局的可见程度，程序里的全局变量的Bind是global的，程序里用static关键字声明的静态全局变量的Bind是local的，采用我们前面使用的特殊关键字声明的函数的Bind是weak的

接下来讲解st_info的后4个字节，Type值，在前面介绍过的elf.h中的Type的宏定义里，常见的有如下三个

```c
#define STT_NOTYPE	0		/* Symbol type is unspecified */
#define STT_OBJECT	1		/* Symbol is a data object */
#define STT_FUNC	2		/* Symbol is a code object */
```

只被声明但没被定义的符号的Type值是STT_NOTYPE，比如说

```c
extern void f();
```

依据这个声明就可以得知，f在外部的文件被定义，这样的话，符号f的Type就是notype，它对应的符号表的表项中的用来标记它所在的section的index(即st_shndx)的值是SHN_UNDEF(在elf.h中被宏定义为0)，指向SHT的0号表项对应的section

像下面这样也是

```c
extern int d;
```

如果没有extern但是在文件中就是没有定义的话，也符合上面的规则，比如说:

```c
void f();
```

对于每一个符号，编译器都会维护关于它的(Bind,Type,Section)这样的一个三元组，通过这样的一个三元组就能得到关于该符号的信息，比如说 

```c
extern void f()
```

对应的符号f，它的这个三元组就是(global,notype,undef)

对于Type是object的符号，即程序中的变量，如果已经被初始化为0，或者以static关键字声明且未被初始化，那么会被存放在.bss节，如果static的变量被初始化为非0了，那么它就会被存放到.data节

接下来讨论一种特别的object，如果我们在程序中做如下声明

```c
int d;
```

有可能在外部文件中变量d已经被初始化为非0，那么经过链接之后d会被存放到.data section，要是在外部被初始化为0，那么链接之后会被存放到.bss section，如果外部没有初始化这个变量，那么链接之后d就会取默认值0并被存放到.bss section，上面代码对d的定义属于临时性的定义(Tentative)，并且在ELF文件中会被存储在.common section，它对应的符号表表项中的section index，即st_shndx的值，是0xfff2(这个数值是固定的，和ELF的SHT无关，并且并不是内存物理意义上的第0xfff2个section，只是这样约定的，这是一个虚拟的index)，同时，由于这个符号对应的变量未来链接后要么被存入.data section，要么被存入.bss section，而我们在当前是不知道的，并且这个变量当前被存放在.common section中，因此，符号对应的符号表表项中的st_value的意义就不是之前那样定义的了(其他的section中的符号的st_value的意义是它对应的数据在对应的section中的偏移量，而我们连.common section中的符号对应的数据将来会被存放在.data节还是.bss节，是否会被初始化都不知道，自然不能以之前的意义来看待st_value)，因此，如果一个符号对应的数据被存放在.common section，那么st_value的意义就是数据在.common section中的对齐方式，比如说，如果数据是int类型，那么st_value就是它的size，即4，采用4字节对齐

这个d变量对应的三元组是(global,object,.common)，对于一个被链接好的可执行的ELF文件，所有的符号对应的数据位于哪个section都被确定下来了，因此不存在common segament(原本的section的概念对于被加载到内存中执行的可执行文件改叫做segament)

如果在声明时采用弱定义关键字，像下面这样

```c
__attribute__((weak)) int d;
```

那么就会变成下面的情况

```c
__attribute__((weak)) int d=0;
```

符号d的三元组是(weak, object,.bss)

如果是

```c
__attribute__((weak)) int d=1;
```

这样的话，那么符号d的三元组是(weak, object,.data)

Reference:

https://www.bilibili.com/video/BV14h411Q7Rz?spm_id_from=333.999.0.0