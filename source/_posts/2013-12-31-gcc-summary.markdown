---
layout: post
title: "gcc summary"
date: 2013-12-31 18:07
comments: true
categories: tools
tags: [gcc, compiler]
---

本文介绍在Linux平台下应用程序的编译过程，以及编译程序[GCC](http://gcc.gnu.org/)在编译应用程序的过程的具体用法，同时详细说明了GCC的常用选项、模式和警告选项。

<center>![gcc](http://gcc.gnu.org/img/gccegg-65.png)</center>

<!--more-->

### Compile Process

#### Four Steps

对于GUN编译器来说，程序的编译要经历预处理、编译、汇编、连接四个阶段.

<center>![四个阶段](http://new.51cto.com/files/uploadimg/20060926/1714230.jpg)</center>

从功能上分，预处理、编译、汇编是三个不同的阶段，但`gcc`的实际操作上，它可以把这三个步骤合并为一个步骤来执行。下面我们以C语言为例来谈一下不同阶段的输入和输出情况。

在预处理阶段，输入的是C语言的源文件，通常为`*.c`。它们通常带有`.h`之类头文件的包含文件。这个阶段主要处理源文件中的`#ifdef`、 `#include`和`#define`命令。该阶段会生成一个中间文件`*.i`，但实际工作中通常不用专门生成这种文件，因为基本上用不到；若非要生成这种文件不可，可以利用下面的示例命令：

	gcc -E  test.c -o test.i

在编译阶段，输入的是中间文件`*.i`，编译后生成汇编语言文件`*.s` 。这个阶段对应的`gcc`命令如下所示：

	gcc -S test.i -o test.s 

在汇编阶段，将输入的汇编文件`*.s`转换成机器语言`*.o`。这个阶段对应的`gcc`命令如下所示：

	gcc -c test.s -o test.o 

最后，在连接阶段将输入的机器代码文件`*.s`（与其它的机器代码文件和库文件）汇集成一个可执行的二进制代码文件。这一步骤，可以利用下面的示例命令完成：

	gcc test.o -o test 

#### Two Modes

gcc常用的两种模式：编译模式和编译连接模式。下面以一个例子来说明各种模式的使用方法。为简单起见，假设我们全部的源代码都在一个文件`test.c`中，要想把这个源文件直接编译成可执行程序，可以使用以下命令：

	gcc test.c -o test

这里`test.c`是源文件，生成的可执行代码存放在一个名为`test`的文件中（该文件是机器代码并且可执行）。`-o`是生成可执行文件的输出选项。如果我们只想让源文件生成目标文件（给文件虽然也是机器代码但不可执行），可以使用标记`-c`，详细命令如下所示：
	
	gcc -c test.c

默认情况下，生成的目标文件被命名为`test.o`，但我们也可以为输出文件指定名称，如下所示：

	gcc -c test.c -o mytest.o

上面这条命令将编译后的目标文件命名为`mytest.o`，而不是默认的`test.o`。

迄今为止，我们谈论的程序仅涉及到一个源文件；现实中，一个程序的源代码通常包含在多个源文件之中，这该怎么办？没关系，即使这样，用gcc处理起来也并不复杂，见下例：

	gcc -o test  first.c second.c third.c

该命令将同时编译三个源文件，即`first.c`、`second.c`和 `third.c`，然后将它们连接成一个可执行程序，名为`test`。

### Compile Option

#### Debug
在`gcc`编译源代码时指定`-g`选项可以产生带有调试信息的目标代码,`gcc`可以为多个不同平台上帝不同调试器提供调试信息,默认`gcc`产生的调试信息是为`gdb`使用的,可以使用`-gformat`指定要生成的调试信息的格式以提供给其他平台的其他调试器使用.

常用的格式有 

- -ggdb:生成gdb专 用的调试信息,使用最适合的格式(DWARF 2,stabs等)会有一些gdb专用的扩展,可能造成其他调试器无法运行. 
- -gstabs:使用 stabs格式,不包含gdb扩展,stabs常用于BSD系统的DBX调试器. 
- -gcoff:产生COFF格式的调试信息,常用于System V下的SDB调试器; 
- -gxcoff:产生XCOFF格式的调试信息,用于IBM的RS/6000下的DBX调试器; 
- -gdwarf-2:产生DWARF version2 的格式的调试信息,常用于IRIXX6上的DBX调试器.`gcc`会使用DWARF version3的一些特性. 

#### Common Option

--help</br>
--target-help </br>
显示 `gcc` 帮助说明。‘target-help’是显示目标机器特定的命令行选项。

--version </br>
显示 `gcc` 版本号和版权信息 。

-o outfile </br>
输出到指定的文件。

-x language </br>
指明使用的编程语言。允许的语言包括：c c++ assembler none 。 ‘none’意味着恢复默认行为，即根据文件的扩展名猜测源文件的语言。

-v </br>
打印较多信息，显示编译器调用的程序。

-### </br>
与 -v 类似，但选项被引号括住，并且不执行命令。

**-E**</br>
仅作预处理，不进行编译、汇编和链接。如上图所示。

-S </br>
仅编译到汇编语言，不进行汇编和链接。如上图所示。

**-c** </br>
编译、汇编到目标代码，不进行链接。如上图所示。

-pipe </br>
使用管道代替临时文件。

-combine </br>
将多个源文件一次性传递给汇编器。


#### Other Option

更多有用的`gcc`选项：

-l library</br>
-llibrary </br>
进行链接时搜索名为library的库。 </br>
例子： $ `gcc` test.c -lm -o test

-Idir </br>
把dir 加入到搜索头文件的路径列表中。 </br>
例子： $ `gcc` test.c -I../inc -o test

-Ldir </br>
把dir 加入到搜索库文件的路径列表中。 </br>
例子： $ `gcc` -I/home/foo -L/home/foo -ltest test.c -o test

-Dname </br>
预定义一个名为name 的宏，值为1。 </br>
例子： $ `gcc` -DTEST_CONFIG test.c -o test

-Dname =definition </br>
预定义名为name ，值为definition 的宏。

-ggdb </br>
-ggdblevel </br>
为调试器 gdb 生成调试信息。level 可以为1，2，3，默认值为2。

-g </br>
-glevel </br>
生成操作系统本地格式的调试信息。-g 和 -ggdb 并不太相同， -g 会生成 gdb 之外的信息。level 取值同上。

-s </br>
去除可执行文件中的符号表和重定位信息。用于减小可执行文件的大小。

-M </br>
告诉预处理器输出一个适合make的规则，用于描述各目标文件的依赖关系。对于每个 源文件，预处理器输出 一个make规则，该规则的目标项(target)是源文件对应的目标文件名，依赖项(dependency)是源文件中`#include`引用的所有文件。生成的规则可以是单行，但如果太长，就用`\`-换行符续成多行。规则 显示在标准输出，不产生预处理过的C程序。

-C </br>
告诉预处理器不要丢弃注释。配合`-E'选项使用。

-P </br>
告诉预处理器不要产生`#line`命令。配合`-E`选项使用。

-static </br>
在支持动态链接的系统上，阻止连接共享库。该选项在其它系统上无效。

-nostdlib</br> 
不连接系统标准启动文件和标准库文件，只把指定的文件传递给连接器。

#### Warnings Option

-Wall </br>
会打开一些很有用的警告选项，建议编译时加此选项。

-W </br>
-Wextra</br> 
打印一些额外的警告信息。

-w </br>
禁止显示所有警告信息。

-Wshadow </br>
当一个局部变量遮盖住了另一个局部变量，或者全局变量时，给出警告。很有用的选项，建议打开。 -Wall 并不会打开此项。

-Wpointer-arith </br>
对函数指针或者void *类型的指针进行算术操作时给出警告。也很有用。 -Wall 并不会打开此项。

-Wcast-qual </br>
当强制转化丢掉了类型修饰符时给出警告。 -Wall 并不会打开此项。

-Waggregate-return </br>
如果定义或调用了返回结构体或联合体的函数，编译器就发出警告。

-Winline </br>
无论是声明为 inline 或者是指定了-finline-functions 选项，如果某函数不能内联，编译器都将发出警告。如果你的代码含有很多 inline 函数的话，这是很有用的选项。

-Werror </br>
把警告当作错误。出现任何警告就放弃编译。

-Wunreachable-code</br> 
如果编译器探测到永远不会执行到的代码，就给出警告。也是比较有用的选项。

-Wcast-align</br> 
一旦某个指针类型强制转换导致目标所需的地址对齐增加时，编译器就发出警告。

-Wundef</br>
当一个没有定义的符号出现在 #if 中时，给出警告。

-Wredundant-decls</br> 
如果在同一个可见域内某定义多次声明，编译器就发出警告，即使这些重复声明有效并且毫无差别。



#### Standard

-ansi </br>
支持符合ANSI标准的C程序。这样就会关闭GNU C中某些不兼容ANSI C的特性。

-std=c89 </br>
-iso9899:1990 </br>
指明使用标准 ISO C90 作为标准来编译程序。

-std=c99 </br>
-std=iso9899:1999 </br>
指明使用标准 ISO C99 作为标准来编译程序。

-std=c++98 </br>
指明使用标准 C++98 作为标准来编译程序。

-std=gnu9x </br>
-std=gnu99 </br>
使用 ISO C99 再加上 GNU 的一些扩展。

-fno-asm </br>
不把asm, inline或typeof当作关键字，因此这些词可以用做标识符。用 __asm__， __inline__和__typeof__能够替代它们。 `-ansi` 隐含声明了`-fno-asm`。

-fgnu89-inline </br>
告诉编译器在 C99 模式下看到 inline 函数时使用传统的 GNU 句法。

#### C options

-fsigned-char </br>
-funsigned-char </br>
把char定义为有/无符号类型，如同signed char/unsigned char。

-traditional </br>
尝试支持传统C编译器的某些方面。详见GNU C手册。

-fno-builtin </br>
-fno-builtin-function </br>
不接受没有 __builtin_ 前缀的函数作为内建函数。

-trigraphs </br>
支持ANSI C的三联符（ trigraphs）。`-ansi'选项隐含声明了此选项。

-fsigned-bitfields </br>
-funsigned-bitfields </br>
如果没有明确声明`signed`或`unsigned`修饰符，这些选项用来定义有符号位域或无符号位域。缺省情况下，位域是有符号的，因为它们继承的基本整数类型，如`int`，是有符号数。

-Wstrict-prototypes </br>
如果函数的声明或定义没有指出参数类型，编译器就发出警告。很有用的警告。

-Wmissing-prototypes </br>
如果没有预先声明就定义了全局函数，编译器就发出警告。即使函数定义自身提供了函数原形也会产生这个警告。这个选项 的目的是检查没有在头文件中声明的全局函数。

-Wnested-externs </br>
如果某`extern`声明出现在函数内部，编译器就发出警告。

#### C++ options

-ffor-scope </br>
从头开始执行程序，也允许进行重定向。

-fno-rtti </br>
关闭对 dynamic_cast 和 typeid 的支持。如果你不需要这些功能，关闭它会节省一些空间。

-Wctor-dtor-privacy </br>
当一个类没有用时给出警告。因为构造函数和析构函数会被当作私有的。

-Wnon-virtual-dtor </br>
当一个类有多态性，而又没有虚析构函数时，发出警告。-Wall会开启这个选项。

-Wreorder </br>
如果代码中的成员变量的初始化顺序和它们实际执行时初始化顺序不一致，给出警告。

-Wno-deprecated </br>
使用过时的特性时不要给出警告。

-Woverloaded-virtual </br>
如果函数的声明隐藏住了基类的虚函数，就给出警告。

Machine Dependent Options (Intel)</br>

-mtune=cpu-type </br>
为指定类型的 CPU 生成代码。cpu-type 可以是：i386，i486，i586，pentium，i686，pentium4 等等。

-msse </br>
-msse2 </br>
-mmmx </br>
-mno-sse </br>
-mno-sse2 </br>
-mno-mmx </br>
使用或者不使用MMX，SSE，SSE2指令。

-m32 </br>
-m64 </br>
生成32位/64位机器上的代码。

-mpush-args </br>
-mno-push-args </br>
（不）使用 push 指令来进行存储参数。默认是使用。

-mregparm=num </br>
当传递整数参数时，控制所使用寄存器的个数。

#### Optimization Option

-O0 </br>
禁止编译器进行优化。默认为此项。

-O</br>
-O1</br>
尝试优化编译时间和可执行文件大小。

-O2 </br>
更多的优化，会尝试几乎全部的优化功能，但不会进行“空间换时间”的优化方法。

-O3 </br>
在 -O2 的基础上再打开一些优化选项：-finline-functions， -funswitch-loops 和 -fgcse-after-reload 。

-Os </br>
对生成文件大小进行优化。它会打开 -O2 开的全部选项，除了会那些增加文件大小的。

-finline-functions </br>
把所有简单的函数内联进调用者。编译器会探索式地决定哪些函数足够简单，值得做这种内联。

-fstrict-aliasing </br>
施加最强的别名规则（aliasing rules）。

`gcc`默认提供了5级优化选项的集合: 

- -O0:无优化(默认)
- -O和-O1:使用能减少目标文件大小以及执行时间并且不会使编译时间明显增加的优化.在编译大型程序的时候会显著增加编译时内存的使用. 
- -O2: 包含-O1的优化并增加了不需要在目标文件大小和执行速度上进行折衷的优化.编译器不执行循环展开以及函数内联.此选项将增加编译时间和目标文件的执行性 能. 
- -Os:专门优化目标文件大小,执行所有的不增加目标文件大小的-O2优化选项.并且执行专门减小目标文件大小的优化选项. 
- -O3: 打开所有-O2的优化选项并且增加 -finline-functions, -funswitch-loops,-fpredictive-commoning, -fgcse-after-reload and -ftree-vectorize优化选项. 

<pre>
-O1包含的选项-O1通常可以安全的和调试的选项一起使用:
           -fauto-inc-dec -fcprop-registers -fdce -fdefer-pop -fdelayed-branch 
           -fdse -fguess-branch-probability -fif-conversion2 -fif-conversion 
           -finline-small-functions -fipa-pure-const -fipa-reference 
           -fmerge-constants -fsplit-wide-types -ftree-ccp -ftree-ch 
           -ftree-copyrename -ftree-dce -ftree-dominator-opts -ftree-dse 
           -ftree-fre -ftree-sra -ftree-ter -funit-at-a-time 
</pre>

以下所有的优化选项需要在名字前加上-f,如果不需要此选项可以使用-fno-前缀 

- defer-pop:延迟到只在必要时从函数参数栈中pop参数; 
- thread-jumps:使用跳转线程优化,避免跳转到另一个跳转; 
- branch-probabilities:分支优化; 
- cprop-registers:使用寄存器之间copy-propagation传值; 
- guess-branch-probability:分支预测; 
- omit-frame-pointer:可能的情况下不产生栈帧; 

<pre>
-O2:以下是-O2在-O1基础上增加的优化选项: 
           -falign-functions  -falign-jumps -falign-loops  -falign-labels 
           -fcaller-saves -fcrossjumping -fcse-follow-jumps  -fcse-skip-blocks 
           -fdelete-null-pointer-checks -fexpensive-optimizations -fgcse 
           -fgcse-lm -foptimize-sibling-calls -fpeephole2 -fregmove 
           -freorder-blocks  -freorder-functions -frerun-cse-after-loop 
           -fsched-interblock  -fsched-spec -fschedule-insns 
           -fschedule-insns2 -fstrict-aliasing -fstrict-overflow -ftree-pre 
           -ftree-vrp
</pre>
 
cpu架构的优化选项,通常是-mcpu(将被取消);-march,-mtune 


### Warning Interpretation

- unused-function:警告声明但是没有定义的static函数; 
- unusedlabel:声明但是未使用的标签; 
- unused-parameter:警告未使用的函数参数; 
- unused-variable:声明但是未使用的本地变量; 
- unused-value:计算了但是未使用的值; 
- format:printf和scanf这样的函数中的格式字符串的使用不当; 
- implicit-int:未指定类型; 
- implicit-function:函数在声明前使用; 
- charsubscripts:使用char类作为数组下标(因为char可能是有符号数); 
- missingbraces:大括号不匹配; 
- parentheses: 圆括号不匹配; 
- return-type:函数有无返回值以及返回值类型不匹配; 
- sequence-point:违反顺序点的代码,比如 a[i] = c[i++]; 
- switch:switch语句缺少default或者switch使用枚举变量为索引时缺少某个变量的case; 
- strictaliasing=n:使用n设置对指针变量指向的对象类型产生警告的限制程度,默认n=3;只有在-fstrict-aliasing设置的情况下有效; 
- unknow-pragmas:使用未知的#pragma指令; 
- uninitialized:使用的变量为初始化,只在-O2时有效; 
- 以下是在-Wall中不会激活的警告选项: 
- cast-align:当指针进行类型转换后有内存对齐要求更严格时发出警告; 
- signcompare:当使用signed和unsigned类型比较时; 
- missing-prototypes:当函数在使用前没有函数原型时; 
- packed:packed 是`gcc`的一个扩展,是使结构体各成员之间不留内存对齐所需的空间 ,有时候会造成内存对齐的问题; 
- padded:也是`gcc`的扩展,使结构体成员之间进行内存对齐的填充,会 造成结构体体积增大. 
- unreachable-code:有不会执行的代码时. 
- inline:当inline函数不再保持inline时 (比如对inline函数取地址); 
- disable-optimization:当不能执行指定的优化时.(需要太多时间或系统资源). 
- 可以使用 -Werror时所有的警告都变成错误,使出现警告时也停止编译.需要和指定警告的参数一起使用. 

####1. Warning: multi-character character constant

Could be suppressed by -Wno-multichar

There're three kinds of character constants: Normal character constants, Multicharacter constants and Wide-character constants.

    char ch = 'a';
    int mbch = '1234';
    wchar_t wcch = L'ab';

Mbch is of type int(signed), has 4 meaningful characters.

`gcc` compiler evaluates a multi-character character constant a character at a time, shifting the previous value left by the number of bits per target character, and then or-ing in the bit-pattern of the new character truncated to the width of a target character.

'ab' for a target with an 8-bit char would be interpreted as:

(int) ((unsigned char) 'a' * 256 + (unsigned char) 'b') = 97*256+98

'1',x          0x31
'12',x        0x3132
'123',x      0x313233
'1234',x    0x31323334

判断系统是big endian还是little endian的方法：

    if (('1234' >> 24) == '1')
    {
        //Little endian
    }
    else if (('4321' >> 24) == '1')
    {
        //Big endian
    }

####2. Warning: operation on xx may be undefined

序列点问题。为什么 a[i] = i++; 不能正常工作？子表达式 i++ 有一个副作用，它会改变 i 的值。由于 i 在同一表达式的其它地方被引用，这会导致无定义的结果，无从判断该引用(左边的 a[i] 中)是旧值还是新值。(尽管 在 K&R 中建议这类表达式的行为不确定，但 C 标准却强烈声明它是无定义的），具体实现取决于编译器。

####3.  Warning: conversion to xxx from yyy may alter its value

`gcc` promotes unsigned char/uint16_t/uint8_t  to type int for for all arithmetic. Need to apply static_cast<>.

####4.  Warning: function might be possible candidate for attribute 'noreturn'

打开了 -Wmissing-noreturn。A few standard library functions, such as abort and exit, cannot return. `gcc` knows this automatically. With noreturn attribute it can then optimize without regard to what would happen if function ever did return. This makes slightly better code. More importantly, it helps avoid spurious warnings of uninitialized variables. The noreturn keyword does not affect the exceptional path. 给函数加上 __attribute__((noreturn)) 即可消除这个warning。

####5. Warning: deprecated conversion from string constant to "char *"

void SomeFunc (char* str)
{
}

int _tmain(int argc, _TCHAR* argv[])
{
    SomeFunc("Hello!");
    return 0;
}

SomeFunc() 的输入时char*，含义是：给我个字符串，我要修改它。而传给它的字面常量是没办法修改的，将char* 改成 const char*，消除这个warning.

 

####6.  Warning: implicit declaration of function 'malloc'/'free', incompatible implicit declaration of built-in function 'malloc'/'free'

要显示的#include <stdlib.h>

####7.  Warning: Dereferencing type-punned pointer will break strict-aliasing rules

打开了-fstrict-aliasing and -Wstrict-aliasing. Suppress with -fno-strict-aliasing

Strict-aliasing rule: An object of one type is assumed never to reside at the same address as an object of a different type, unless the types are almost the same. 编译器希望不同类型的对象不会指向同一个地址。

####8.  Warning: inlining failed in call to xxx: call is unlikely and code size would grow

打开了-Winline: Warn if a function can not be inlined and it was declared as inline. Even with this option, the compiler will not warn about failures to inline functions declared in system headers.

相关选项：

-fno-inline: Don't compile statement functions inline. Might reduce the size of a program unit--which might be at expense of some speed (though it should compile faster). Note that if you are not optimizing, no functions can be expanded inline.

-finline-functions: Interprocedural optimizations occur. However, if you specify -O0, the default is OFF. Enables function inlining for single file compilation.

 

####9.  Warning: cannot optimize loop, the loop counter may overflow

打开了-Wunsafe-loop-optimizations: Warn if the loop cannot be optimized because the compiler could not assume anything on the bounds of the loop indices. With -funsafe-loop-optimizations warn if the compiler made such assumptions.

相关选项：

-funsafe-loop-optimizations: Enable unsafe loop optimizations, e.g. assume loop indices never overflow, etc