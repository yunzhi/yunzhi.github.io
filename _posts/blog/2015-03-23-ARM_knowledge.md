---
layout: post
title: ARM 基础知识
description: 介绍ARM的基础知识
category: blog
---

## ARM基础知识

1. ARM有两种工作状态：ARM状态，32位指令模式；Thumb状态，执行16位的指令

2. ARM的对齐方式：Arm是以字做对齐方式。
> 大端，数据的低字节存放在存储空间的高低址。//就是我们看存储空间的值是不用反序。
> 小端，数据的低字节存放在存储空间的低地址。

3. ARM有7种工作模式：
> 1.用户模式（usr）：用于执行正常的程序
> 2.快速中断模式（FIQ）：用于高速数据传输
> 3.外部中断模式（IRQ）：通常的中断处理
> 4.管理模式（SVC）：操作系统使用的保护模式
> 5.数据访问终止模式（abt）：当数据或者指令预取终止时进入该模式
> 6.系统模式（sys）：运行具有特权的操作系统任务
> 7.未定义模式（und）：当未定义的指令执行时进入该模式，用于支持硬件

4. 工作模式切换：通过软件修改，或者通过外部中断或异常处理改变。一般的，ARM运行在用户模式，除用户模式之外，其他6种模式称之为非用户模式，或者特权模式。其中除用户模式和系统模式之外的5中称为异常模式

5. ARM微处理器共有37个32位寄存器，其中31个位通用寄存器，6个为状态寄存器。任何模式下，通用寄存器R0-R7（成为未分组寄存器）, 程序计数器PC，一个状态寄存器（CPSR）都是可以访问的。
> + sys 和 User模式下 R0-R14    (15个)
> + FIQ模式下R8_fiq-R14_fiq     （7个）
> + Svc模式下R13_svc-R13_svc （2个）
> + Abt模式下R13_abt,R14_abt （2个）
> +IRQ模式下R13_irq,R14_irq  （2个）
>+ und模式下R13_und,R14_und （2个）
>+ R15（PC寄存器）               （1个）
>+ CPSR（程序状态寄存器）     （1个）
>+ SPSR_fiq，SPSR_svc，SPSR_abt，SPSR_irq，SPSR_und  （5个）

![ARM寄存器图](/images/about_arm/arm_register.png)

6.堆栈指针 R13寄存器 R13 通常作为堆栈指针(SP)。在 ARM 指令集当中,由于没有以特殊方式使用 R13 的指令或其它功能,只是习惯上都这样使用。每个异常模式都有其自身的 R13 分组版本,它通常指向由异常模式所专用的堆栈。在入口处,异常处理程序通常将其它要使用的寄存器值保存到这个堆栈。通过返回时将这些值重装到寄存器中,异常处理程序可确保异常发生时的程序状态不会被破坏。
7.链接寄存器 R14寄存器 R14(也称为链接寄存器或 LR)在结构上有两个特殊功能:

+ 在每种模式下,模式自身的R14版本用于保存子程序返回地址。当使用BL或BLX 指令(注意:ARM7TDMI 没有 BLX 这条指令)调用子程序时,R14 设置为子程序返回地址。子程序返回通过将 R14 复制到程序计数器来实现。通常有下列两种方式: 执行下列指令之一:
		MOV PC,LR		BX LR
或是在子程序入口,使用下列形式的指令将 R14 存入堆栈:
	 STMFD SP!,{<registers>,LR}并使用匹配的指令返回:	LDMFD SP!,{<registers>,PC}+ 当发生异常时,将R14对应的异常模式版本设置为异常返回地址(有些异常有一个小常量的偏移)。异常返回的执行类似于子程序返回,只是使用稍微不同的指令来确保被异常中断的程序状态能够完全恢复。 
6.当前状态寄存器（CPSR）

CPSR在用户级编程时用于存储条件码。例如，这些为可以用来记录比较操作的结果和控制条件转移是否发生。寄存器的低位用于控制寄存器的模式，指令集，这中断使能，而且被保护以防止用户级程序改变它们。条件标志码位于寄存器的高四位，意义如下：
> + N:负数Negative
> + Z:零Zero
> + C:进位Carry
> + V:溢出位oVerflow

7.ARM指令集特点：

+ Load-Store体系结构
+ 3地址的数据处理指令
+ 每条指令都是条件执行
+ 包含非常强大的多寄存器Load和Store指令
+ 能用在单时钟周期内执行的单条指令来完成一项普通的位移操作和一项普通的ALU操作
+ 通过协处理器指令集来扩展ARM指令集，包括在编程模式中增加了新的寄存器和数据类型
+ 在Thumb体系结构中以高密度16位压缩形式表示的指令集


## [ARM CP15协处理器](http://www.cnblogs.com/gaomaolin_88/archive/2010/07/16/1779183.html)

1）主标识符寄存器

访问主标识符寄存器的指令格式如下所示：

	mrc p15, 0, r0, c0, c0, 0       ；将主标识符寄存器C0,0的值读到r0中

2）cache类型标识符寄存器

访问cache类型标识符寄存器的指令格式如下所示：

	mrc p15, 0, r0, c0, c0, 1       ；将cache类型标识符寄存器C0,1的值读到r0中

CP15的寄存器C1
访问主标识符寄存器的指令格式如下所示：

	mrc p15, 0, r0, c1, c0{, 0}     ；将CP15的寄存器C1的值读到r0中

	mcr p15, 0, r0, c1, c0{, 0}     ；将r0的值写到CP15的寄存器C1中


## A&T 内嵌汇编

GCC支持在C/C++代码中嵌入汇编代码，这些汇编代码被称作GCC Inline ASM——GCC内联汇编。这是一个非常有用的功能，有利于我们将一些C/C++语法无法表达的指令直接潜入C/C++代码中，另外也允许我们直接写C/C++代码中使用汇编编写简洁高效的代码。

### 1.基本内联汇编
GCC中基本的内联汇编非常易懂，我们先来看两个简单的例子：
    
    __asm__("movl %esp,%eax"); // 看起来很熟悉吧！
    
    或者是
    __asm__("
    movl $1,%eax // SYS_exit
    xor %ebx,%ebx
    int $0x80
    ");
    
    或
    __asm__(
    "movl $1,%eax\r\t" \
    "xor %ebx,%ebx\r\t" \
    "int $0x80" \
    );


基本内联汇编的格式是

    __asm__ __volatile__("Instruction List");

#### a、`__asm__`
    __asm__是GCC关键字asm的宏定义：
    #define __asm__ asm
    __asm__或asm用来声明一个内联汇编表达式，所以任何一个内联汇编表达式都是以它开头的，是必不可少的。

#### b、Instruction List
    Instruction List是汇编指令序列。它可以是空的，比如：`__asm__ __volatile__(""); 或__asm__("");`都是完全合法的内联汇编表达式，只不过这两条语句没有什么意义。但并非所有Instruction List为空的内联汇编表达式都是没有意义的，比如：`__asm__ ("":::"memory"); `就非常有意义，它向GCC声明："我对内存作了改动"，GCC在编译的时候，会将此因素考虑进去。

我们看一看下面这个例子：

    $ cat example1.c
    
    int main(int __argc, char* __argv[])
    {
    int* __p = (int*)__argc;
    
    (*__p) = 9999;
    
    //__asm__("":::"memory");
    
    if((*__p) == 9999)
    return 5;
    
    return (*__p);
    }

在这段代码中，那条内联汇编是被注释掉的。在这条内联汇编之前，内存指针__p所指向的内存被赋值为9999，随即在内联汇编之后，一条if语句判断__p所指向的内存与9999是否相等。很明显，它们是相等的。GCC在优化编译的时候能够很聪明的发现这一点。我们使用下面的命令行对其进行编译：

    $ gcc -O -S example1.c

选项-O表示优化编译，我们还可以指定优化等级，比如-O2表示优化等级为2；选项-S表示将C/C++源文件编译为汇编文件，文件名和C/C++文件一样，只不过扩展名由.c变为.s。我们来查看一下被放在example1.s中的编译结果，我们这里仅仅列出了使用gcc2.96在redhat 7.3上编译后的相关函数部分汇编代码。为了保持清晰性，无关的其它代码未被列出。
    
    $ cat example1.s
    
    main:
    pushl %ebp
    movl %esp, %ebp
    movl 8(%ebp), %eax # int* __p = (int*)__argc
    movl $9999, (%eax) # (*__p) = 9999
    movl $5, %eax # return 5
    popl %ebp
    ret

参照一下C源码和编译出的汇编代码，我们会发现汇编代码中，没有if语句相关的代码，而是在赋值语句`(*__p)=9999`后直接`return 5；`这是因为GCC认为在`(*__p)`被赋值之后，在if语句之前没有任何改变`(*__p)`内容的操作，所以那条if语句的判断条件`(*__p)== 9999`肯定是为true的，所以GCC就不再生成相关代码，而是直接根据为true的条件生成return 5的汇编代码（GCC使用eax作为保存返回值的寄存器）。

我们现在将example1.c中内联汇编的注释去掉，重新编译，然后看一下相关的编译结果。

	$ gcc -O -S example1.c
    
    $ cat example1.s
    
    main:
    pushl %ebp
    movl %esp, %ebp
    movl 8(%ebp), %eax # int* __p = (int*)__argc
    movl $9999, (%eax) # (*__p) = 9999
    #APP
    
    # __asm__("":::"memory")
    #NO_APP
    cmpl $9999, (%eax) # (*__p) == 9999 ?
    jne .L3 # false
    movl $5, %eax # true, return 5
    jmp .L2
    .p2align 2
    .L3:
    movl (%eax), %eax
    .L2:
    popl %ebp
    ret

由于内联汇编语句`__asm__("":::"memory")`向GCC声明，在此内联汇编语句出现的位置内存内容可能了改变，所以GCC在编译时就不能像刚才那样处理。这次，GCC老老实实的将if语句生成了汇编代码。可能有人会质疑：为什么要使用__asm__("":::"memory")向GCC声明内存发生了变化？明明"Instruction List"是空的，没有任何对内存的操作，这样做只会增加GCC生成汇编代码的数量。确实，那条内联汇编语句没有对内存作任何操作，事实上它确实什么都没有做。但影响内存内容的不仅仅是你当前正在运行的程序。比如，如果你现在正在操作的内存是一块内存映射，映射的内容是外围I/O设备寄存器。那么操作这块内存的就不仅仅是当前的程序，I/O设备也会去操作这块内存。既然两者都会去操作同一块内存，那么任何一方在任何时候都不能对这块内存的内容想当然。所以当你使用高级语言C/C++写这类程序的时候，你必须让编译器也能够明白这一点，毕竟高级语言最终要被编译为汇编代码。

你可能已经注意到了，这次输出的汇编结果中，有两个符号：#APP和#NO_APP，GCC将内联汇编语句中"Instruction List"所列出的指令放在#APP和#NO_APP之间，由于__asm__("":::"memory")中"Instruction List"为空，所以#APP和#NO_APP中间也没有任何内容。但我们以后的例子会更加清楚的表现这一点。

关于为什么内联汇编__asm__("":::"memory")是一条声明内存改变的语句，我们后面会详细讨论。

刚才我们花了大量的内容来讨论"Instruction List"为空是的情况，但在实际的编程中，"Instruction List"绝大多数情况下都不是空的。它可以有1条或任意多条汇编指令。当在"Instruction List"中有多条指令的时候，你可以在一对引号中列出全部指令，也可以将一条或几条指令放在一对引号中，所有指令放在多对引号中。如果是前者，你可以将每一条指令放在一行，如果要将多条指令放在一行，则必须用分号（；）或换行符（\n，大多数情况下\n后还要跟一个\t，其中\n是为了换行，\t是为了空出一个tab宽度的空格）将它们分开。比如：

    __asm__("movl %eax, %ebx
    sti
    popl %edi
    subl %ecx, %ebx");
    
    __asm__("movl %eax, %ebx; sti
    popl %edi; subl %ecx, %ebx");
    
    __asm__("movl %eax, %ebx; sti\n\t popl %edi
    subl %ecx, %ebx");

都是合法的写法。如果你将指令放在多对引号中，则除了最后一对引号之外，前面的所有引号里的最后一条指令之后都要有一个分号(；)或(\n)或(\n\t)。比如：

    __asm__("movl %eax, %ebx
    sti\n"
    "popl %edi;"
    "subl %ecx, %ebx");
    
    __asm__("movl %eax, %ebx; sti\n\t"
    "popl %edi; subl %ecx, %ebx");
    
    __asm__("movl %eax, %ebx; sti\n\t popl %edi\n"
    "subl %ecx, %ebx");
    
    __asm__("movl %eax, %ebx; sti\n\t popl %edi;" "subl %ecx,
    %ebx");

都是合法的。

上述原则可以归结为：
+ 任意两个指令间要么被分号(；)分开，要么被放在两行；
+ 放在两行的方法既可以从通过\n的方法来实现，也可以真正的放在两行；
+ 可以使用1对或多对引号，每1对引号里可以放任一多条指令，所有的指令都要被放到引号中。

在基本内联汇编中，"Instruction List"的书写的格式和你直接在汇编文件中写非内联汇编没有什么不同，你可以在其中定义Label，定义对齐(.align n )，定义段(.section name )。例如：
    
    __asm__(".align 2\n\t"
    "movl %eax, %ebx\n\t"
    "test %ebx, %ecx\n\t"
    "jne error\n\t"
    "sti\n\t"
    "error: popl %edi\n\t"
    "subl %ecx, %ebx");

上面例子的格式是Linux内联代码常用的格式，非常整齐。也建议大家都使用这种格式来写内联汇编代码。

#### c、`__volatile__`

`__volatile__`是GCC关键字volatile的宏定义：
    
    #define __volatile__ volatile

`__volatile__`或volatile是可选的，你可以用它也可以不用它。如果你用了它，则是向GCC声明"不要动我所写的Instruction List，我需要原封不动的保留每一条指令"，否则当你使用了优化选项(-O)进行编译时，GCC将会根据自己的判断决定是否将这个内联汇编表达式中的指令优化掉。
    
 那么GCC判断的原则是什么？我不知道（如果有哪位朋友清楚的话，请告诉我）。我试验了一下，发现一条内联汇编语句如果是基本内联汇编的话（即只有"Instruction List"，没有Input/Output/Clobber的内联汇编，我们后面将会讨论这一点），无论你是否使用__volatile__来修饰，GCC2.96在优化编译时，都会原封不动的保留内联汇编中的"Instruction List"。但或许我的试验的例子并不充分，所以这一点并不能够得到保证。

为了保险起见，如果你不想让GCC的优化影响你的内联汇编代码，你最好在前面都加上`__volatile__`，而不要依赖于编译器的原则，因为即使你非常了解当前编译器的优化原则，你也无法保证这种原则将来不会发生变化。而`__volatile__`的含义却是恒定的。

### 2、带有C/C++表达式的内联汇编

GCC允许你通过C/C++表达式指定内联汇编中"Instrcuction List"中指令的输入和输出，你甚至可以不关心到底使用哪个寄存器被使用，完全靠GCC来安排和指定。这一点可以让程序员避免去考虑有限的寄存器的使用，也可以提高目标代码的效率。
    
我们先来看几个例子：
    
    __asm__ (" " : : : "memory" ); // 前面提到的
    
    __asm__ ("mov %%eax, %%ebx" : "=b"(rv) : "a"(foo)
    : "eax", "ebx");
    
    __asm__ __volatile__("lidt %0": "=m" (idt_descr));
    
    __asm__("subl %2,%0\n\t"
    "sbbl %3,%1"
    : "=a" (endlow), "=d" (endhigh)
    : "g" (startlow), "g" (starthigh), "0"
    (endlow), "1" (endhigh));

 带有C/C++表达式的内联汇编格式为：
    
    __asm__　__volatile__("Instruction List"
    : Output : Input : Clobber/Modify);
    
 从中我们可以看出它和基本内联汇编的不同之处在于：它多了3个部分(Input，Output，Clobber/Modify)。在括号中的4个部分通过冒号(:)分开。这4个部分都不是必须的，任何一个部分都可以为空，其规则为：

+ 如果Clobber/Modify为空，则其前面的冒号(:)必须省略。比如`__asm__("mov %%eax, %%ebx" : "=b"(foo) : "a"(inp) : )`就是非法的写法；而`__asm__("mov %%eax, %%ebx" : "=b"(foo) : "a"(inp) )`则是正确的。

+ 如果Instruction List为空，则Input，Output，Clobber/Modify可以不为空，也可以为空。比如
`__asm__( " " : : : "memory" );和__asm__(" ": : );`都是合法的写法。

+ 如果Output，Input，Clobber/Modify都为空，Output，Input之前的冒号(:)既可以省略，也可以不省略。如果都省略，则此汇编退化为一个基本内联汇编，否则，仍然是一个带有C/C++表达式的内联汇编，此时"Instruction List"中的寄存器写法要遵守相关规定，比如寄存器前必须使用两个百分号(%%)，而不是像基本汇编格式一样在寄存器前只使用一个百分号(%)。比如`__asm__( " mov %%eax, %%ebx" : : )；__asm__(" mov %%eax, %%ebx" : )和__asm__( " mov %eax, %ebx" )`都是正确的写法，而`__asm__(" mov %eax, %ebx" : : )；__asm__( " mov %eax, %ebx" : )和__asm__(" mov %%eax, %%ebx" )`都是错误的写法。

+ 如果Input，Clobber/Modify为空，但Output不为空，Input前的冒号(:)既可以省略，也可以不省略。比如`__asm__(" mov %%eax, %%ebx" : "=b"(foo) : )；__asm__( "mov %%eax, %%ebx" : "=b"(foo) )`都是正确的。

+ 如果后面的部分不为空，而前面的部分为空，则前面的冒号(:)都必须保留，否则无法说明不为空的部分究竟是第几部分。比如， Clobber/Modify，Output为空，而Input不为空，则Clobber/Modify前的冒号必须省略（前面的规则），而Output前的冒号必须为保留。如果Clobber/Modify不为空，而Input和Output都为空，则Input和Output前的冒号都必须保留。比如
`__asm__(" mov %%eax, %%ebx" : : "a"(foo) )和__asm__( "mov %%eax, %%ebx" : : : "ebx" )。`
   
从上面的规则可以看到另外一个事实，区分一个内联汇编是基本格式的还是带有C/C++表达式格式的，其规则在于在"Instruction List"后是否有冒号(:)的存在，如果没有则是基本格式的，否则，则是带有C/C++表达式格式的。
    
两种格式对寄存器语法的要求不同：基本格式要求寄存器前只能使用一个百分号(%)，这一点和非内联汇编相同；而带有C/C++表达式格式则要求寄存器前必须使用两个百分号(%%)，其原因我们会在后面讨论。

#### a). Output

Output用来指定当前内联汇编语句的输出。我们看一看这个例子：
    
    __asm__("movl %%cr0, %0": "=a" (cr0));

这个内联汇编语句的输出部分为"=r"(cr0)，它是一个"操作表达式"，指定了一个输出操作。我们可以很清楚得看到这个输出操作由两部分组成：括号括住的部分(cr0)和引号引住的部分"=a"。这两部分都是每一个输出操作必不可少的。括号括住的部分是一个C/C++表达式，用来保存内联汇编的一个输出值，其操作就等于C/C++的相等赋值cr0 = output_value，因此，括号中的输出表达式只能是C/C++的左值表达式，也就是说它只能是一个可以合法的放在C/C++赋值操作中等号(=)左边的表达式。那么右值output_value从何而来呢？

答案是引号中的内容，被称作"操作约束"（Operation Constraint），在这个例子中操作约束为"=a"，它包含两个约束：等号(=)和字母a，其中等号(=)说明括号中左值表达式cr0是一个Write-Only的，只能够被作为当前内联汇编的输入，而不能作为输入。而字母a是寄存器EAX / AX / AL的简写，说明cr0的值要从eax寄存器中获取，也就是说cr0 = eax，最终这一点被转化成汇编指令就是movl %eax, address_of_cr0。现在你应该清楚了吧，操作约束中会给出：到底从哪个寄存器传递值给cr0。

另外，需要特别说明的是，很多文档都声明，所有输出操作的操作约束必须包含一个等号(=)，但GCC的文档中却很清楚的声明，并非如此。因为等号(=)约束说明当前的表达式是一个Write-Only的，但另外还有一个符号——加号(+)用来说明当前表达式是一个Read-Write的，如果一个操作约束中没有给出这两个符号中的任何一个，则说明当前表达式是Read-Only的。因为对于输出操作来说，肯定是必须是可写的，而等号(=)和加号(+)都表示可写，只不过加号(+)同时也表示是可读的。所以对于一个输出操作来说，其操作约束只需要有等号(=)或加号(+)中的任意一个就可以了。

二者的区别是：等号(=)表示当前操作表达式指定了一个纯粹的输出操作，而加号(+)则表示当前操作表达式不仅仅只是一个输出操作还是一个输入操作。但无论是等号(=)约束还是加号(+)约束所约束的操作表达式都只能放在Output域中，而不能被用在Input域中。
    
另外，有些文档声明：尽管GCC文档中提供了加号(+)约束，但在实际的编译中通不过；我不知道老版本会怎么样，我在GCC 2.96中对加号(+)约束的使用非常正常。
    
我们通过一个例子看一下，在一个输出操作中使用等号(=)约束和加号(+)约束的不同。

    $ cat example2.c
    
    int main(int __argc, char* __argv[])
    {
    int cr0 = 5;
    
    __asm__ __volatile__("movl %%cr0, %0":"=a"
    (cr0));
    
    return 0;
    }
    
    $ gcc -S example2.c
    
    $ cat example2.s
    
    main:
    pushl %ebp
    movl %esp, %ebp
    subl $4, %esp
    movl $5, -4(%ebp) # cr0 = 5
    #APP
    movl %cr0, %eax
    #NO_APP
    movl %eax, %eax
    movl %eax, -4(%ebp) # cr0 = %eax
    movl $0, %eax
    leave
    ret

这个例子是使用等号(=)约束的情况，变量cr0被放在内存-4(%ebp)的位置，所以指令mov %eax, -4(%ebp)即表示将%eax的内容输出到变量cr0中。

下面是使用加号(+)约束的情况：
    
    $ cat example3.c
    
    int main(int __argc, char* __argv[])
    {
    int cr0 = 5;
    
    __asm__ __volatile__("movl %%cr0, %0" : "+a" (cr0));
    
    return 0;
    }
    
    $ gcc -S example3.c
    
    $ cat example3.s
    
    main:
    pushl %ebp
    movl %esp, %ebp
    subl $4, %esp
    movl $5, -4(%ebp) # cr0 = 5
    movl -4(%ebp), %eax # input ( %eax = cr0 )
    #APP
    movl %cr0, %eax
    #NO_APP
    movl %eax, -4(%ebp) # output (cr0 = %eax )
    movl $0, %eax
    leave
    ret
    
    
从编译的结果可以看出，当使用加号(+)约束的时候，cr0不仅作为输出，还作为输入，所使用寄存器都是寄存器约束(字母a，表示使用eax寄存器)指定的。关于寄存器约束我们后面讨论。
    
在Output域中可以有多个输出操作表达式，多个操作表达式中间必须用逗号(,)分开。例如：
    
    __asm__(
    "movl %%eax, %0 \n\t"
    "pushl %%ebx \n\t"
    "popl %1 \n\t"
    "movl %1, %2"
    : "+a"(cr0), "=b"(cr1), "=c"(cr2));
    
#### b)、Input

Input域的内容用来指定当前内联汇编语句的输入。我们看一看这个例子：
    
    __asm__("movl %0, %%db7" : : "a" (cpu->db7));

例中Input域的内容为一个表达式"a"[cpu->db7)，被称作"输入表达式"，用来表示一个对当前内联汇编的输入。像输出表达式一样，一个输入表达式也分为两部分：带括号的部分(cpu->db7)和带引号的部分"a"。这两部分对于一个内联汇编输入表达式来说也是必不可少的。

括号中的表达式cpu->db7是一个C/C++语言的表达式，它不必是一个左值表达式，也就是说它不仅可以是放在C/C++赋值操作左边的表达式，还可以是放在C/C++赋值操作右边的表达式。所以它可以是一个变量，一个数字，还可以是一个复杂的表达式（比如`a+b/c*d）`。比如上例可以改为：`__asm__("movl %0, %%db7" : : "a"(foo))`，`__asm__("movl %0, %%db7" : : "a" (0x1000))`或`__asm__("movl %0, %%db7" : : "a"(va*vb/vc))`。

引号号中的部分是约束部分，和输出表达式约束不同的是，它不允许指定加号(+)约束和等号(=)约束，也就是说它只能是默认的Read-Only的。约束中必须指定一个寄存器约束，例中的字母a表示当前输入变量cpu->db7要通过寄存器eax输入到当前内联汇编中。

我们看一个例子：
    
    $ cat example4.c
    
    int main(int __argc, char* __argv[])
    {
    int cr0 = 5;
    
    __asm__ __volatile__("movl %0, %%cr0"::"a" (cr0));
    
    return 0;
    }
    
    $ gcc -S example4.c
    
    $ cat example4.s
    
    main:
    pushl %ebp
    movl %esp, %ebp
    subl $4, %esp
    movl $5, -4(%ebp) # cr0 = 5
    movl -4(%ebp), %eax # %eax = cr0
    #APP
    movl %eax, %cr0
    #NO_APP
    movl $0, %eax
    leave
    ret
    
我们从编译出的汇编代码可以看到，在"Instruction List"之前，GCC按照我们的输入约束"a"，将变量cr0的内容装入了eax寄存器。
    
#### c). Operation Constraint
每一个Input和Output表达式都必须指定自己的操作约束Operation Constraint，我们这里来讨论在80386平台上所可能使用的操作约束。

##### 1、寄存器约束
    
当你当前的输入或输入需要借助一个寄存器时，你需要为其指定一个寄存器约束。你可以直接指定一个寄存器的名字，比如：
    
    __asm__ __volatile__("movl %0, %%cr0"::"eax" (cr0));

也可以指定一个缩写，比如：
    
    __asm__ __volatile__("movl %0, %%cr0"::"a" (cr0));

如果你指定一个缩写，比如字母a，则GCC将会根据当前操作表达式中C/C++表达式的宽度决定使用%eax，还是%ax或%al。比如：
    
    unsigned short __shrt;
    
    __asm__ ("mov %0，%%bx": : "a"(__shrt));
    
由于变量__shrt是16-bit short类型，则编译出来的汇编代码中，则会让此变量使用%ex寄存器。编译结果为：
    
    movw -2(%ebp), %ax # %ax = __shrt
    #APP
    movl %ax, %bx
    #NO_APP
    
无论是Input，还是Output操作表达式约束，都可以使用寄存器约束。
    
下表中列出了常用的寄存器约束的缩写。

约束 Input/Output 意义
    
    r I,O 表示使用一个通用寄存器，由GCC在%eax/%ax/%al, %ebx/%bx/%bl, %ecx/%cx/%cl, %edx/%dx/%dl中选取一个GCC认为合适的。
    q I,O 表示使用一个通用寄存器，和r的意义相同。
    a I,O 表示使用%eax / %ax / %al
    b I,O 表示使用%ebx / %bx / %bl
    c I,O 表示使用%ecx / %cx / %cl
    d I,O 表示使用%edx / %dx / %dl
    D I,O 表示使用%edi / %di
    S I,O 表示使用%esi / %si
    f I,O 表示使用浮点寄存器
    t I,O 表示使用第一个浮点寄存器
    u I,O 表示使用第二个浮点寄存器

##### 2、内存约束
如果一个Input/Output操作表达式的C/C++表达式表现为一个内存地址，不想借助于任何寄存器，则可以使用内存约束。比如：
    
    __asm__ ("lidt %0" : "=m"(__idt_addr)); 或 __asm__ ("lidt %0" : :"m"(__idt_addr));
    
我们看一下它们分别被放在一个C源文件中，然后被GCC编译后的结果：
    
    $ cat example5.c
    
    // 本例中，变量sh被作为一个内存 
 
 
    
## 参考
1. [GCC的内嵌汇编语法](http://os.chinaunix.net/a2008/0313/977/000000977964.shtml)

["Yunzhi made"](http://yunzhi.github.io) &copy;
