继续上一篇Blog对于链接的介绍，上一篇中提及了SHT(节头表)，本篇Blog将细致讲解SHT

我们可以通过

```shell
readelf -S elf.o
```

来查看elf文件的SHT

![](https://tva1.sinaimg.cn/large/008i3skNly1gwh2e693lej30py0lc77v.jpg)

SHT的每个表项，即每个Section Header的结构如下所示(也是在usr/include/elf.h中定义的)

<img src="https://tva1.sinaimg.cn/large/008i3skNly1gwh3ffjr25j30d80dqwfl.jpg" style="zoom:50%;" />

根据SHT中的.strtab这个表项可以找到string table，每个section header里的sh_name变量(上图中结构体的第一个成员变量，数据类型是uint32_t)被用来在string table里索引，索引得到的字符串是对应的section name，string table如下图所示，里面都是ascii编码的字符串

![](https://tva1.sinaimg.cn/large/008i3skNly1gwh3t71if0j30pe04wjsa.jpg)

在实际的ELF解析中，会根据section header中的sh_flags和sh_type来判断对应的section的属性，sh_addr是这个section应该被装载到的虚拟地址，不过这个只对于被链接好的EOF文件有效，sh_offset的值是对应的section在整个ELF文件中的偏移量，sh_size是这个section的大小，如果这个section是一个表(比如说符号表，symbol table这个section，string table严格意义上不算一个表，因为它的每个表项，即其中的每个字符串，长度不同），那么sh_entsize记录的就是这个table的每个entry的大小(如果当前section不是一个表的话，sh_entsize的值就是0)

Reference:

https://www.bilibili.com/video/BV1AU4y1s7Je?spm_id_from=333.999.0.0