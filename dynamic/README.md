# Dynamic Linker

## References

http://nicephil.blinkenshell.org/my_book/ch07s02.html

## 基本思想

把程序按照模块拆分为各个相对独立部分，在程序运行时才将他们链接在一起。涉及到存储管理，内存共享，进程线程等机制。
当程序被装载时，系统的动态链接器会将程序所需要的所有动态链接库装载到进程的地址空间，并将程序中所有未决议的符号绑定到相应的动态链接库中，并进行重定位工作。
动态链接会导致一些程序在性能的损失，使用延迟绑定（Lazy Binding）等方法优化，动态链接比静态链接损失了%5以下。

```
gcc -fPIC -shared -o lib.so lib.c
gcc -o program1 program1.c ./lib.so
```

在链接时动态库还是需要的，因为目标文件引用的外部定义的符号，链接器不知道应该是静态的还是动态的，如果发现这个符号是定义在动态库中的动态符号，链接器对其进行特殊用途。

```
$ cat /proc/9781/maps
08048000-08049000 r-xp 00000000 08:04 8926205    /home/phil/repos/my_bible/program1
08049000-0804a000 rw-p 00000000 08:04 8926205    /home/phil/repos/my_bible/program1
b7579000-b757a000 rw-p 00000000 00:00 0
b757a000-b7715000 r-xp 00000000 08:03 130333     /lib/libc-2.15.so
b7715000-b7716000 ---p 0019b000 08:03 130333     /lib/libc-2.15.so
b7716000-b7718000 r--p 0019b000 08:03 130333     /lib/libc-2.15.so
b7718000-b7719000 rw-p 0019d000 08:03 130333     /lib/libc-2.15.so
b7719000-b771c000 rw-p 00000000 00:00 0
b7732000-b7733000 rw-p 00000000 00:00 0
b7733000-b7734000 r-xp 00000000 08:04 8926204    /home/phil/repos/my_bible/lib.so
b7734000-b7735000 rw-p 00000000 08:04 8926204    /home/phil/repos/my_bible/lib.so
b7735000-b7736000 rw-p 00000000 00:00 0
b7736000-b7737000 r-xp 00000000 00:00 0          [vdso]
b7737000-b7757000 r-xp 00000000 08:03 130342     /lib/ld-2.15.so
b7757000-b7758000 r--p 0001f000 08:03 130342     /lib/ld-2.15.so
b7758000-b7759000 rw-p 00020000 08:03 130342     /lib/ld-2.15.so
bfd83000-bfda4000 rw-p 00000000 00:00 0          [stack]
```

可知，动态链接器也被映射到了进程的地址空间，在系统开始运行program1之前，首先会把控制权交给动态链接器，由它完成所有的动态链接工作以后再把控制权交给program1,然后开始执行。

```
phil~/repos/my_bible $ readelf -l lib.so

Elf file type is DYN (Shared object file)
Entry point 0x420
There are 6 program headers, starting at offset 52

Program Headers:
  Type           Offset   VirtAddr   PhysAddr   FileSiz MemSiz  Flg Align
  LOAD           0x000000 0x00000000 0x00000000 0x00638 0x00638 R E 0x1000
  LOAD           0x000638 0x00001638 0x00001638 0x00120 0x00124 RW  0x1000
  DYNAMIC        0x000644 0x00001644 0x00001644 0x000e0 0x000e0 RW  0x4
  NOTE           0x0000f4 0x000000f4 0x000000f4 0x00024 0x00024 R   0x4
  GNU_EH_FRAME   0x0005b8 0x000005b8 0x000005b8 0x0001c 0x0001c R   0x4
  GNU_STACK      0x000000 0x00000000 0x00000000 0x00000 0x00000 RW  0x4

 Section to Segment mapping:
  Segment Sections...
   00     .note.gnu.build-id .gnu.hash .dynsym .dynstr .gnu.version .gnu.version_r .rel.dyn .rel.plt .init .plt .text .fini .rodata .eh_frame_hdr .eh_frame
   01     .init_array .fini_array .jcr .dynamic .got .got.plt .data .bss
   02     .dynamic
   03     .note.gnu.build-id
   04     .eh_frame_hdr
   05
```

共享对象的最终装载地址在编译时是不确定的，而是在装载时，装载器根据当前地址空间的空闲情况，动态分配一块足够大小的虚拟地址空间给相应的共享对象。

## 地址无关代码

共享对象在编译时不能假设自己在进程虚拟地址空间中的位置。
为了能够使共享对象在任意地址装载，需要装载时重定位。
Linux GCC支持装载时重定位，如果只使用-shared那么输出的共享对象就使用装载时重定位。

地址无关：PIC pic PIE pie:
装载时重定位无法支持代码部分在多个进程间共享，那么我们希望代码段在装载时不需要因为装载地址的变化而变化，如果将指令中变化的部分提取出来作为数据的一部分，那么指令部分大家共享，而数据部分每个进程有自己的副本，地址无关技术。

模块间的地址引用方式：
* 第一种：模块内的函数调用，跳转等: 不需要重定位
* 第二种：模块内部的数据访问，比如模块中定义的全局变量，静态变量: 指令中不能直接包含数据的绝对地址，任何一条指令与它访问的模块内部数据间的相对位置是固定的，ELF得到当前的PC值，然后再加上一个偏移量来达到访问响应变量的目的。

```
0000054c <bar>:
 54c:   55                      push   %ebp
 54d:   89 e5                   mov    %esp,%ebp
 54f:   e8 40 00 00 00          call   594 <__x86.get_pc_thunk.cx>
 554:   81 c1 20 12 00 00       add    $0x1220,%ecx
 55a:   c7 81 24 00 00 00 01    movl   $0x1,0x24(%ecx) //a = 1
 561:   00 00 00
 564:   8b 81 ec ff ff ff       mov    -0x14(%ecx),%eax
 56a:   c7 00 02 00 00 00       movl   $0x2,(%eax)
 570:   5d                      pop    %ebp
 571:   c3                      ret

00000594 <__x86.get_pc_thunk.cx>:
 594:   8b 0c 24                mov    (%esp),%ecx
 597:   c3                      ret
```

变量a的地址是当前PC值加上两个偏移。

* 第三种：模块外部的函数调用，跳转等

在数据段里建立一个指向目标函数的指针数组，当模块需要调用目标函数时，可以通过GOT中的项进行间接跳转。
先得到但前指令地址PC，然后加上一个偏移得到函数地址在GOT中的偏移，然后一个间接调用

```
54f:   e8 40 00 00 00          call   594 <__x86.get_pc_thunk.cx>
554:   81 c1 20 12 00 00       add    $0x1220,%ecx
55a:   c7 81 24 00 00 00 01    movl   $0x1,0x24(%ecx)
561:   00 00 00
564:   8b 81 ec ff ff ff       mov    -0x14(%ecx),%eax
56a:   c7 00 02 00 00 00       movl   $0x2,(%eax)
```

* 第四种：模块外部的数据访问，比如其它模块定义的全局变量

模块间数据的访问地址要等到转载时才知道，那么把跟地址相关的部分放到数据段，在数据段里建立一个指向这些变量的指针数组，全局偏移表GOT，当代码需要引用这些全局变量时，通过GOT中相应的项间接引用。

各种地址引用

模块内部   相对跳转和调用        相对地址访问
模块外部   间接跳转和调用（GOT） 间接访问（GOT）

* -fpic 指示GCC产生地址无关代码，产生代码相对较快，较小，但跟硬件平台相关
* -fPIC 通常使用这个产生地址无关代码

如何判断一个动态文件是否是PIC，只要产看有没有任何代码重定位段，TEXTREL表示代码段重定位表地址。
地址无关技术也可应用于普通可执行程序,-fPIE或-fpie`

共享模块的全局变量：

如果当一个模块应用了一个定义在共享对象的全局变量时，由于程序主模块的代码并不是地址无关代码，它引用全局变量的方式跟普通数据访问一样，由于可执行程序在运行时不进行代码重定位，为了使链接过程正常进行，链接器会在创建可执行程序时，在.bss段创建一个变量的副本

那么全局变量定义在原先的共享对象中，而可执行文件的.bss段也有一个副本，那么就产生了冲突。
为了解决这个问题，共享库在编译时，默认都把定义在模块内部的全局变量当作定义在其它模块的全局变量，通过GOT来实现变量的访问，带共享模块加载时，如果某个全局变量在可执行文件中有副本，那么动态链接器就把GOT中相应的地址指向该副本，如果变量在共享库中被初始化，那么动态链接器还需要把初始化值复制到程序主模块中的变量副本。

假设是共享对象的一部分，那么GCC在-fPIC情况下，会把变量的调用按照跨模块方式产生代码，因为编译器无法确定对全局变量的引用是跨模块的还是模块内部的，即使是模块内部的全局变量，还是会按照跨模块的引用方式，因为可能被可执行程序引用。


数据段地址无关性：

```
static int a;
static int *p = &a;
```

那么p就是一个绝对地址，会随着共享对象的装载地址变化而变化。
对于数据段，它在每个进程中都有一份独立的副本，可以在装载时重定位来解决数据段的绝对地址引用问题。对于共享对象，如果数据段有绝对地址引用，那么编译器和链接器会产生一个重定位表，R_386_RELATIVE,当动态链接器加载共享对象时，如果发现该共享对象有这样的重定位入口，那么动态链接器会对该对象重定位。

可以使用不带-fPIC的选项产生装载时重定位的代码段，

```
gcc -shared pic.o -o pic.so
```

因为是地址相关代码，所有多个进程间不能共享此代码段，造成内存的浪费，但是地址相关代码段不需要每次访问全局变量和函数时的地址计算和间接选址，因此会比地址无关代码效率高。
对于可执行文件来说，默认下，如果可执行程序是动态链接的，那么GCC选用PIC方式产生可执行文件的代码段，以便不同的进程能够共享代码段，节约内存。所以动态链接的可执行文件存在.got段。

## 延迟绑定（PLT）

动态链接比静态链接灵活，但牺牲了性能，据统计ELF程序在静态链接下比动态库快大约1%~5%。
主要原因是，动态链接下对于全局和静态数据的访问都要进行复杂的GOT定位，然后间接寻址，对于模块间的调用也要先定位GOT，然后进行间接跳转。
另外，动态链接的链接过程是在运行时完成的，动态链接器会寻找并转载所需要的对象，然后进行符号查找地址重定位等工作。

延迟绑定实现：

因为很多函数可能在程序执行完时都不会被用到，比如错误处理函数或一些用户很少用到的功能模块等，那么一开始就把所有函数都链接好实际是一种浪费，因此ELF采用了一种延迟绑定（Lazy Binding），就是在当函数第一次被用到时才进行绑定（符号查找，重定位等），如果没有用到则不进行绑定。
ELF使用PLT（Procedure Linkage Table）的方法来实现，使用一些很精妙的指令序列来完成。
Glibc提供了地址绑定的函数_dl_runtime_resolve()，当我们调用某个外部模块的函数时，按照正常做法应该通过GOT中的相应的项进行间接跳转。PLT为了实现延迟绑定，在这个过程中邮增加了一层间接跳转。调用函数并不直接通过GOT跳转而是使用PLT项的结构来进行跳转。每个外部函数在PLT中都有一个相应的项。比如bar()函数：

```
bar@plt:
jmp *(bar@GOT)
push n
push moduleID
jump _dl_runtime_resove
```

第一条指令通过GOT间接跳转指令。bar@GOT表示GOT中保存bar()函数地址的相应项。如果初始化阶段已经初始化该项，并且将bar()地址填入该项，那么结果是跳转到bar()。为了实现延迟绑定，链接器在初始化阶段并没有将bar()地址填入该项，而是将上面第二条指令"push n"的地址填入到bar@GOT中，这个步骤不需要查找任何符号，所以代价很小，那么第一条指令相当于跳转到第二条指令，第二条指令将数字n压栈，n是bar这个符号应用在重定位表".rel.plt"中的下标，接着一条push指令将模块ID压栈，让后跳转到_dl_runtime_resolve。实际是：先将所需要决议的符号的下标压入栈，后将模块ID入栈，然后调用动态链接器的_dl_runtime_resolve()函数完成符号解析和重定位工作。一旦函数被解析完毕，当再次调用bar@plt时，第一条jmp指令就能够跳转到真正的bar()函数。

ELF将GOT拆成两个部分".got",".got.plt"，其中.got保存全局变量引用的地址，.got.plt函数引用地址。所有对于外部函数的引用都分离出来放在.got.plt中。它的前三项有特殊用途：

* 第一项保存的是.dynamic的地址，这个段描述了本模块动态链接相关信息
* 第二项保存的是本模块ID
* 第三项是_dl_runtime_resolve()的地址。

其中第二项和第三项由动态链接器在装载共享模块时负责将他们初始化。.got.plt的其余项分别对应每个外部函数的引用。

```
.got.plt:
Address of .dynamic
Module ID "Lib.so"
_dl_runtime_resolve()
import func1
import func2
...
...
```

## ELF在Linux下的动态链接实现

动态链接下，可执行文件的装载和静态链接情况基本一样，操作系统先读取可执行文件头部，检查文件合法性，然后从头部中的Program Header中读取每个Segment的虚拟地址，文件偏移和属性，并将他们映射到程序虚拟空间相应位置。
在动态链接下，操作系统将控制权交给了动态链接器ld.so，操作系统通过映射的方式将它加载到进程地址空间中.将控制权交给动态链接器的入口地址。
然后动态链接器进行一系列自身初始化操作，然后根据当前环境参数，开始对可执行文件进行动态链接工作，当所有动态链接工作完成后，将控制权交给可执行程序入口，程序开始执行。

.interp段：
指定了动态链接器的位置。
/lib/ld-linux.so.2
动态链接器在linux下是Glibc中的一部分，属于系统库级别的，

```
readelf -l program1 | grep interpreter      [Requesting program interpreter: /lib/ld-linux.so.2]i
```

### .dynamic段：

动态链接ELF中最重要的段是.dynamic段，这个段里面保存了动态链接器所需要的基本信息，比如依赖于哪些共享对象，动态链接符号表的位置，动态链接重定位表的位置，共享对象初始化代码的地址等。

```
Elf32_Dyn由一个类型值加上一个附加的数值或指针。
DT_SYMTAB，动态链接符号表的地址，d_ptr表示.dynsym的地址
DT_STRTAB，动态链接字符串表地址，d_ptr表示.dynstr的地址
DT_HASH，动态链接哈希表大小，d_val表示大小
DT_SONAME，本共享对象的SO-NAME
DT_RPATH，动态链接共享对象搜索路径
DT_INIT，初始化代码地址
DT_FINIT，结束代码地址
DT_NEED，依赖的共享对象文件，d_ptr表示依赖的共享对象文件名
DT_REL，动态链接重定位表地址
DT_RELA，
DT_RELENT，动态重读位表入口数量
DT_RELANET]
```

phil~/repos/my_bible $ readelf -d lib.so

```
Dynamic section at offset 0x644 contains 24 entries:
  Tag        Type                         Name/Value
 0x00000001 (NEEDED)                     Shared library: [libc.so.6]
 0x0000000c (INIT)                       0x3a4
 0x0000000d (FINI)                       0x588
 0x00000019 (INIT_ARRAY)                 0x1638
 0x0000001b (INIT_ARRAYSZ)               4 (bytes)
 0x0000001a (FINI_ARRAY)                 0x163c
 0x0000001c (FINI_ARRAYSZ)               4 (bytes)
 0x6ffffef5 (GNU_HASH)                   0x118
 0x00000005 (STRTAB)                     0x234
 0x00000006 (SYMTAB)                     0x154
 0x0000000a (STRSZ)                      193 (bytes)
 0x0000000b (SYMENT)                     16 (bytes)
 0x00000003 (PLTGOT)                     0x1738
 0x00000002 (PLTRELSZ)                   32 (bytes)
 0x00000014 (PLTREL)                     REL
 0x00000017 (JMPREL)                     0x384
 0x00000011 (REL)                        0x344
 0x00000012 (RELSZ)                      64 (bytes)
 0x00000013 (RELENT)                     8 (bytes)
 0x6ffffffe (VERNEED)                    0x314
 0x6fffffff (VERNEEDNUM)                 1
 0x6ffffff0 (VERSYM)                     0x2f6
 0x6ffffffa (RELCOUNT)                   3
 0x00000000 (NULL)                       0x0
```

inux还提供了查看程序主模块或一个共享库依赖哪些共享库：

```
phil~/repos/my_bible $ ldd ./program1
        linux-gate.so.1 =>  (0xb7734000)
        ./lib.so (0xb7731000)
        libc.so.6 => /lib/libc.so.6 (0xb7578000)
        /lib/ld-linux.so.2 (0xb7735000)
```

其中linux-gate.so.1比较特殊，是一个内核虚拟共享对象

### 动态符号表：

为了表示动态链接这些模块之间的符号导入导出关系，ELF专门有个动态符号表.dynsym。它只保存与动态链接相关的符号，对于哪些模块内部的符号，比如模块私有变量则不保存。很多动态链接模块同时拥有.symtab和.dynsym。.symtab保存了所有符号，包含.dynsym中的符号。

对应还有动态符号字符串表.dynstr和为了加快符号查找的符号哈希表.
可以使用readelf查看ELF文件的动态符号表和他的哈希表：

```
phil~/repos/my_bible $ readelf -sD lib.so

Symbol table of `.gnu.hash' for image:
  Num Buc:    Value  Size   Type   Bind Vis      Ndx Name
    8   0: 00001758     0 NOTYPE  GLOBAL DEFAULT ABS _edata
    9   0: 0000175c     0 NOTYPE  GLOBAL DEFAULT ABS _end
   10   1: 00001758     0 NOTYPE  GLOBAL DEFAULT ABS __bss_start
   11   1: 000003a4     0 FUNC    GLOBAL DEFAULT   9 _init
   12   2: 00000588     0 FUNC    GLOBAL DEFAULT  12 _fini
   13   2: 0000054c    57 FUNC    GLOBAL DEFAULT  11 foobar
```

### 动态链接重定位表：

共享对象需要重定位主要原因是导入符号的存在。PIC模式下的共享对象一样需要重定位。
对于PIC共享对象，他的代码段不需要重定位，但是数据段包含了绝对地址引用，因为代码段的绝对地址相关部分分离出来放在了GOT，而GOT实际是数据段的一部分。

动态链接器重定位相关结构：
在静态链接时有.rel.text表示代码段重定位段。.rel.data表示数据段重定位段。

在动态链接下.rel.dyn和.rel.plt分别相当于.rel.data和.rel.text。其中.rel.dyn是对数据段应用的修正，它所修正的位置位于.got和数据段。
而.rel.plt是对函数引用的修正，它所修正的位置位于.got.plt。

```
phil~/repos/my_bible $ readelf -r lib.so

Relocation section '.rel.dyn' at offset 0x344 contains 8 entries:
 Offset     Info    Type            Sym.Value  Sym. Name
00001638  00000008 R_386_RELATIVE
0000163c  00000008 R_386_RELATIVE
00001754  00000008 R_386_RELATIVE
00001724  00000106 R_386_GLOB_DAT    00000000   _ITM_deregisterTMClone
00001728  00000406 R_386_GLOB_DAT    00000000   __cxa_finalize
0000172c  00000506 R_386_GLOB_DAT    00000000   __gmon_start__
00001730  00000606 R_386_GLOB_DAT    00000000   _Jv_RegisterClasses
00001734  00000706 R_386_GLOB_DAT    00000000   _ITM_registerTMCloneTa

Relocation section '.rel.plt' at offset 0x384 contains 4 entries:
 Offset     Info    Type            Sym.Value  Sym. Name
00001744  00000207 R_386_JUMP_SLOT   00000000   printf
00001748  00000307 R_386_JUMP_SLOT   00000000   sleep
0000174c  00000407 R_386_JUMP_SLOT   00000000   __cxa_finalize
00001750  00000507 R_386_JUMP_SLOT   00000000   __gmon_start__

  [20] .got              PROGBITS        00001724 000724 000014 04  WA  0   0  4
  [21] .got.plt          PROGBITS        00001738 000738 00001c 04  WA  0   0  4
  [22] .data             PROGBITS        00001754 000754 000004 00  WA  0   0  4
```

在静态链接中遇到了R_386_32和R_386_PC32，这里有了几种新的重定位入口类型：
R_386_RELATIVE, R_386_GLOB_DAT和R_386_JUMP_SLOT。

对于R_386_GLOB_DAT和R_386_JUMP_SLOT，被修正的位置只需要直接填入符号地址即可。比如printf这个重定位入口，类型是R_386_JUMP_SLOT，偏移是0x00001744,它实际上在.got.plt中。.got.plt的前三项被系统占据的，第四项开始才是真正存放导入函数的地址的地方。而第四项刚好是0x00001724 + 4 * 3 = 0x00001750,即__gmon_start__，,第六项是sleep，第七项是sleep
当动态链接器需要进行重定位时，先查找printf的地址，假设是0x08801234,那么链接器将这个地址填入.got.plt中偏移为0x00001744的位置去，实现了地址重定位。
R_386_GLOB_DAT对.got的重定位跟R_386_JUMP_SLOT一样。

对于R_386_RELATIVE类型的重定位入口，实际就是基址重置，共享对象的数据段是无法做到地址无关的，它可能包含绝对地址，需要在装载时重定位。
例如
static int a;
static int *p = &a;
在编译时共享对象起始位置是0,假设静态变量a相对于起始地址0的偏移是B，那么p的值是B，一旦共享对象被装载到地址A，那么a的地址是B+A，那么p的值需要加上A。
如果ELF文件以PIC编译，并调用一个外部函数bar()，那么bar会出现在.rel.plt中，如果不是以PIC模式编译的，则bar将出现在.rel.dyn中。

```
phil~/repos/my_bible $ gcc -shared lib.c -o lib.so
phil~/repos/my_bible $ readelf -r lib.so

Relocation section '.rel.dyn' at offset 0x344 contains 11 entries:
 Offset     Info    Type            Sym.Value  Sym. Name
0000053c  00000008 R_386_RELATIVE
00001600  00000008 R_386_RELATIVE
00001604  00000008 R_386_RELATIVE
0000171c  00000008 R_386_RELATIVE
00000541  00000202 R_386_PC32        00000000   printf
0000054d  00000302 R_386_PC32        00000000   sleep
000016f4  00000106 R_386_GLOB_DAT    00000000   _ITM_deregisterTMClone
000016f8  00000406 R_386_GLOB_DAT    00000000   __cxa_finalize
000016fc  00000506 R_386_GLOB_DAT    00000000   __gmon_start__
00001700  00000606 R_386_GLOB_DAT    00000000   _Jv_RegisterClasses
00001704  00000706 R_386_GLOB_DAT    00000000   _ITM_registerTMCloneTa

Relocation section '.rel.plt' at offset 0x39c contains 2 entries:
 Offset     Info    Type            Sym.Value  Sym. Name
00001714  00000407 R_386_JUMP_SLOT   00000000   __cxa_finalize
00001718  00000507 R_386_JUMP_SLOT   00000000   __gmon_start__
```

可以看到两个导入函数printf, sleep从.rel.plt到了.rel.dyn中，并从类型R_386_JUMP_SLOT变成了R_386_PC32。
而R_386_RELATIVE类型多了一个偏移0x0000171c，它是printf的第一个参数。在PIC时，这个字符串可以看作普通全局变量，地址可以通过PIC中的相对当前指令的位置加上固定偏移计算出来。在非PIC中，代码段采用绝对地址寻址，所以它需要重定位。

动态链接时进程堆栈初始化信息：
进程初始化时，堆栈保存了关于进程执行环境和命令行参数等信息，同时保存了动态链接器需要的一些辅助信息数组（Auxiliary Vector），
先是32位的类型值，后面32位数据部分。
AT_NULL 0 表述辅助数组结束
AT_EXEFD 2 可执行文件的文件句柄
AT_PHDR 3 可执行文件中程序头表在进程中的地址
AT_PHENT 4 可执行文件头中程序头表中每个入口大小
AT_PHNUM 5 可执行文件头中程序头表入口的数量
AT_BASE 7 表示动态链接器本身的装载地址
AT_ENTRY 9 可执行文件入口地址

它位于环境变量指针的后面

```
int main (int argc, char **argv)
{
    int *p = (int *)argv;
    int i;
    Elf32_auxv_t *aux;

    printf ("Argument count: %d\n", *(p-1));

    for (i = 0; i < *(p-1); ++i) {
        printf ("Argument %d: %s\n", i, *(p+i));
    }

    p += i;

    p ++;

    printf ("Environment:\n");
    while (*p) {
        printf ("%s\n", *p);
        p++;
    }

    p++;

    printf ("Auxiliary Vectors:\n");
    aux = (Elf32_auxv_t *)p;
    while (aux->a_type != AT_NULL) {
        printf ("Type: %02d Value:%x\n", aux->a_type, aux->a_un.a_val);
        aux++;
    }
    return 0;
}
```

PLT结构也有点不同，为了避免代码的重复，ELF把最后两条指令放在了PLT中的第一项。

```
PLT0:
push *(GOT + 4)
jump *(GOT + 8)
...
bar@plt:
jmp *(bar@GOT)
push n
jump PLT0
```

PLT在ELF文件中放在.plt段，因为本身是地址无关代码因此，和代码段一起放在可读可执行的Segment被装载进内存。

## 动态链接的步骤和实现

### 动态链接器自举：

动态链接器本身也是一个共享对象，但是它不能依赖任何其他共享对象，本身所需要的全局和静态变量的重定位工作由它本身完成。自举（Bootstrap）。
自举代码首先找到自己的GOT，而GOT的第一个入口是.dynamic段偏移地址，通过.dynamic中的信息，自举代码可以获得动态链接器本身的重定位表和符号表等，从而得到动态链接器本身的重定位入口，先将他们全部重定位，然后，动态链接器代码才可以使用自己的全局变量和静态变量。

在自举代码中不可以使用全局变量和静态变量，甚至不能调用函数，即使动态链接器本身的函数也不能使用，因为PIC模式下的共享对象，对于模块内部的函数调用也是采用模块外部函数调用一样使用GOT/PLT方式，所以不能调用函数。

### 装载共享对象：

完成基本自举以后，动态链接器将可执行文件和链接器本身的符号表合并到一个符号表，全局符号表。然后链接器开始寻找可执行文件所依赖的共享对象，在.dynamic段中DT_NEEDED，指出所依赖的共享对象。列出所有可执行文件所需要的所有共享对象，并将这些共享对象的名字放入到一个装载集合中。

然后链接器开始从集合里取一个所需要的共享对象后打开该文件，读取相应的ELF文件头和.dynamic段，然后将它相应的代码段和数据段映射到进程空间中，如果这个共享对象还依赖其他共享对象，那么将所依赖的共享对象的名字放到装载集合中，装载过程就是一个图的遍历过程，一般算法使用广度优先。

当一个新的共享对象被装载，它的符号表会被合并到全局符号表，当所有的共享对象都被装载进去时，全局符号表里面将包含进程中所有的动态链接所需要的符号。

### 符号

-XLiner -rpath ./ 表示链接器在当前路径寻找共享对象，否则链接器会报无法找到库
一个共享对象里的全局符号被另外一个共享对象的同名全局符号覆盖的现象称为共享对象全局符号介入
当一个符号需要被加入全局符号表时，如果相同的符号名已经存在，则后加入的符号被忽略。
为了调高模块内部函数调用的效率，需要将函数变成私有的使用static定义函数。


### 重定位和初始化：

链接器开始重新遍历可执行文件和每个共享对象的重定位表，将他们的GOT/PLT的每个需要重定位的位置进行修正。
完成重定位后，如果某个共享对象有.init段，那么动态链接器会执行.init段的代码，比如C++的全局/静态对象的构造。
如果可执行程序也有.init段，动态链接器不会执行它，因为.init段和.finit段由程序初始化部分代码负责执行。
当完成了重定位和初始化之后，动态链接器将进程的控制权交给程序的入口。

## Linux动态链接器的实现

Linux动态链接器本身是个共享对象：

```
catherine~/repos/my_bible $ /lib/ld-linux.so.2
Usage: ld.so [OPTION]... EXECUTABLE-FILE [ARGS-FOR-PROGRAM...]
You have invoked `ld.so', the helper program for shared library executables.
This program usually lives in the file `/lib/ld.so', and special directives
in executable files using ELF shared libraries tell the system's program
loader to load the helper program from this file.  This helper program loads
the shared libraries needed by the program executable, prepares the program
to run, and runs it.  You may invoke this helper program directly from the
command line to load and run an ELF executable file; this is like executing
that file itself, but always uses this helper program from the file you
specified, instead of the helper program file specified in the executable
file you run.  This is mostly of use for maintainers to test new versions
of this helper program; chances are you did not intend to run this program.

  --list                list all dependencies and how they are resolved
  --verify              verify that given object really is a dynamically linked
                        object we can handle
  --library-path PATH   use given PATH instead of content of the environment
                        variable LD_LIBRARY_PATH
  --inhibit-rpath LIST  ignore RUNPATH and RPATH information in object names
                        in LIST
  --audit LIST          use objects named in LIST as auditors
```

Linux的动态链接器是Glibc的一部分，源码位于sysdeps/i386/dl-machine.h中的_start，_start()在sysdeps/i386/elf/start.S

_start调用位于elf/rtld.c的_dl_start()函数，它首先对ld.so进行重定位，完成自举后就可以调用其他函数并访问全局变量了。调用_dl_start_final，收集一些基本运行参数，进入_dl_sysdep_start，这个函数进行一些平台相关的处理之后进入_dl_main,这个就是动态链接器真正的主函数。

动态链接器本身是静态链接的，本身应该是PIC的，便于重定位和代码共享，本身ld.so可以直接执行，装载地址是0,作为共享库，内核在装载它时会为其选择一个合适的装载地址。


显式运行时链接：

运行时加载，让程序自己在运行时控制加载制定的模块，并可以在不需要该模块时将其卸载。动态装载库。
在Linux中，动态库实际上跟一般的共享对象没区别。主要区别是共享对象由动态链接器在程序启动之前负责装载和链接的。而动态库是通过一系列API程序自己控制的，打开动态库dlopen(),查找符号dlsym(),错误处理dlerror(),以及关闭动态库dlclose()。<dlfcn.h>

dlopen():
void *dlopen (const char *filename, int flag)
第一个参数是被加载动态库的路径，如果是绝对路径直接尝试打开，如果是相对路径，查找如下：
a。环境变量LD_LIBRARY_PATH指定的一系列目录
b。由/etc/ld.so.cache里面指定的共享库路径
c。/lib,/usr/lib
如果库在多个目录下由不同副本会导致系统极为不可靠。

如果filename是0,那么dlopen会返回全局符号表的句柄，我们可以在运行时找到全局符号表里的任何符号，并可执行他们。全局符号表包含了程序的可执行文件本身，被动态链接器加载到进程中的所有共享模块已经在运行时通过dlopen打开并使用RTLD_GLOBAL方式的模块中的符号。

第二个参数表示函数符号的解析方式，常量RTLD_LAZY表示使用延迟绑定，即PLT机制。而RTLD_NOW表示当模块被加载时即完成所有函数的绑定工作，如果有任何未定义的符号引用的绑定工作无法完成。另外还有RTLD_GLOBAL跟上面两种的一个配合使用，表示被加载的模块的全局符号合并到进程全局符号表中，使得以后加载的模块可以使用这些符号。

dlopen返回被加载的模块的句柄，如果失败返回NULL，如果已经被加载过了返回同一个句柄。如果模块间由依赖关系，需要手动加载被依赖模块。

实际上dlopen还在加载模块时执行模块中的初始化部分代码。.init段的代码。

dlsym():
void *dlsym (void *handle, char *symbol);
第一个参数是dlopen()返回的动态库句柄
第二个参数是所要查找的符号的名字。
如果dlsym()找到相应的符号，返回符号的值，如果没有找到返回NULL。
dlsym()返回值对于不同类型的符号，意义不同。
如果查找函数符号，返回函数地址。如果查找变量符号，返回变量地址。如果查找常量符号，返回该常量的值。但如果该常量刚好是NULL或0,那么需要使用dlerror()函数。
如果符号找到了，dlerror()返回NULL，如果没有，dlerror返回相应的错误信息。

符号优先级：
当多个同名符号冲突时，先装入的符号优先，装载序列。不管是动态链接器装入的还是dlopen()装入的，都采用装载序列。

dlsym()查找符号时优先级分为两种类型。第一种情况，如果我们是在全局符号表中进行符号查找即dlopen的参数filename是NULL，那么使用装载序列。第二种情况，如果对于某个dlopen()打开的共享对象进行符号查找的话，采用依赖序列的优先级。以被dlopen()打开的共享对象为根节点，对它所依赖的共享对象进行广度优先遍历，直到直到符号为止。


dlerror():
判断上次调用是否成功，上次失败返回char *字符串，否则NULL。

dlclose():
将一个已经加载的模块卸载，系统维持一个加载引用计数器，每次使用dlopen()加载某模块，相应计数器加1;每次使用dlclose()卸载某模块，相应计数器减1.当计数器减到0,才真正卸载模块，先执行.fini段的代码，然后将相应符号从符号表中去除，取消进程空间的映射关系，然后关闭模块文件。


运行时装载的程序：
可以通过命令行来执行共享对象里的任意函数：由命令行给出共享对象路径，函数名和相关参数，然后程序通过运行时加载将该模块加载到进程中，查找相应的函数，并执行它，然后将执行结果打印出来。
runso /lib/foobar.so function arg1 arg2 arg3 return_type
为了支持不同的函数参数组合，通过了解函数调用约定，在调用函数之前伪造好相应的栈，为了能够直接操作栈，不得不使用嵌入汇编代码。

## Linux共享库

共享库兼容问题：
往共享库添加一个导出符号，兼容
删除共享库里的一个原有导出符号，不兼容
将原有导出函数添加一个参数，不兼容
删除导出函数的一个参数，不兼容
改变一个导出结构类型的长度，内容，成员类型，不兼容
修正一个导出函数的bug，兼容
修正一个导出函数的bug，但改变了函数语义，功能，行为或接口类型， 不兼容

C++的ABI问题更严重：

如果需要提供一个导出接口为C++的共享库：
不要在接口类中使用虚函数，使用了不要随意删除
不要改变类的任何成员变量的位置和类型
不要删除非内嵌的public和protected成员函数
不要将非内嵌的成员函数改变成内嵌成员函数
不要改变成员函数的访问权限
不要在接口使用模板
不要改变接口的任何部分或干脆不要使用C++提供共享库


为了解决共享库的兼容问题，使用共享库版本：
libname.so.x.y.z其中lib是前缀，中间是库名称和后缀.so,
最后是三个数字组成的版本号。x表示主版本号，y表示次版本号，z表示发布版本号。
主版本号，表示重大升级，不同主版本号库之间不兼容
次版本号，表示库的增量升级，增加了一些新接口符号，并保持原来符号不变。在主版本号相同下，新的次版本号向后兼容低次版本号的接口。
发布版本号，表示库的一些错误的修正，性能的改进，并不添加任何新的接口，也不对接口进行更正。相同主版本号，次版本号的共享库，不同的发布版本号之间完全兼容。


SO-NAME:
SO-NAME命名机制来记录共享库的依赖关系，每个共享库都有一个对应的SO-NAME，即共享库的文件名去掉次版本号和发布版本号。SO-NAME规定了共享库的接口。

ldconfig会遍历所有默认共享库，然后更新所有软链接，使他们指向最新版本的共享库。

链接名：
ld在-static时，-lc会去找libc.a。使用-Bdynamic时，会找libc.so.x.y.z


基于符号的版本机制：让每个导出和导入的符号都有一个相关联的版本号，类似名字修饰，VER_1.1 VER_1.2

Linux下Glibc中C运行库GLIBC_2.0，GLIBC_2.6
类似GCC_位前缀或GLIBC_PRIVATE这样的符号版本表示用于GCC编译器和GLIBC内部的。

GCC提供了一种叫.symver的汇编宏指令来指定符号的版本。
asm(".symver add, add@VERS_1.1");
int add (int a, int b)
{
  return a + b;
}
这样符号add被指定为符号标签VERS_1.1

GCC允许多个版本的同一个符号存在于一个共享库，在链接层面提供了某种形式的符号重载：
asm(".symver old_printf, printf@VERS_1.1");
asm(".symver new_printf, printf@VERS_1.2");
int old_printf ()
{
}

int new_printf ()
{
}

链接器可以挑选某个符合的符号进行链接。

Linux下的符号版本机制：
在Linux下，使用ld --version-script，或gcc -Xlinker --version-script
gcc -shared -fPIC lib.c -Xlinker --version-script lib.ver -o lib.so
VERS_1.2 {
global:
foo;
local:
*;
}
gcc main.c ./lib.so -o main

共享库系统路径：
/lib
/usr/lib
/usr/local/lib
/lib /usr/lib是一些很常用，成熟的，一般系统本身使用的库。
/usr/local/lib是非系统所需的第三方程序的共享库


共享库查找过程：
DT_NEED类型指定了绝对路径，就从这个路径查找，如果是相对路径，动态链接器从/lib , /usr/lib和由/etc/ld.so.conf指定的目录查找库。
ld.so.conf，ldconfig会将SO-NAME收集起来，存放在/etc/ld.so.cache文件，加快共享库的查找过程。


环境变量：
LD_LIBRARY_PATH：临时改变某个应用程序的共享库查找路径。
或者直接运行动态链接器/lib/ld-linux.so.2 -library-path /home/usr
顺序：
变量LD_LIBRARY_PATH指定的路径，/etc/ld.so.cache指定的路径，默认共享库目录，/usr/lib, /lib
会影响GCC编译时查找库的路径。

LD_PRELOAD：指定预先加载的一些共享库，/etc/ld.so.preload

LD_DEBUG,可以打开动态链接器的调试功能，
LD_DEBUG=files ./a.out
LD_DEBUG可以设置为如下值：
bindings显示动态链接的符号绑定过程
libs显示共享库的查找过程
versions显示符号的版本依赖关系
reloc显示重定位过程
symbols显示符号表查找过程
statistics显示动态链接过程的各种统计信息
all显示以上所有信息
help显示上面的各种选项的帮助信息。



共享库的创建：
gcc -shared -W1,-soname,my_soname -o library_name sourcefiles library_files
-W1可以指定参数传递给链接器，-soname,my_soname指定输出共享库的SO-NAME。

注意：
不要将输出共享库的符号和调试信息去掉，不要使用GCC的-fomit-frame-pointer
测试共享库，又不希望影响现有程序正常运行，LD_LIBRARY_PATH,或ld -rpath, gcc -W1,-rpath
默认情况下，链接器在产生可执行文件时，只会将哪些链接时被其它共享模块引用到的符号放到动态符号表，当程序使用dlopen()动态加载某个共享模块，而该模块又反向引用主模块的符号，可能会导致失败。ld -export-dynamic表示将所有符号表导出到动态符号表。gcc -W1,-export-dynamic

strip 清除符号信息。ld -s 消除所有符号信息， -S消除调试符号信息。

安装共享库：ldconfig -n shared_library_directory

共享库的构造和析构函数：
void __attribute__((constructor(5))) init_function(void);
void __attribute__((destructor(10))) fini_function(void);
数字越小优先级越高,不可以使用gcc -nostartfiles或-nostdlib


共享库脚本：
通过链接脚本可以将几个现在的共享库通过一定方式组合产生新的库
GROUP( /lib/libc.so.6 /lib/libm.so.2 )
