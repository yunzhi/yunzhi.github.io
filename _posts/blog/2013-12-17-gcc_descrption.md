---
layout: post
title: gcc用法简介
description: 初步介绍和记录目前常用的gcc编译命令，以方便查询
category: blog
---

gcc 作为linux强大的编译工具，了解其用法对日常编程具有很大的作用。windows下安装类unix环境也可以使用，我就在win7上安装了cgwin，把它的bin目录加到系统的环境变量里，然后cmd下就可以用它那一堆强大的工具了，例如ls,grep,sed,gcc等工具了。

下面例子中的环境: x86 win7 | cgwin | gcc version 4.8.2 (GCC)

## 常用gcc编译选项
简单编辑了一个test.c文件。如下：

	#include<stdio.h>

	int main()
	{
		printf("Hello,yunzhi!\n");
		return 0;
	}

### 简单预处理，汇编，编译链接，生成可执行文件的编译命令
	gcc test.c
将会生成a.out(linux)或a.exe（windows下）

### 通过预处理，汇编，编译链接，生成名称为test的可执行文件
	gcc test.c -o test	//linux
	gcc test.c -o test.exe	//windows

### 预处理 -E
	gcc -E test.c -o test.i
会生成一个test.i的文件，其实就是用 stdio.h 文件的内容替换掉 #include<stdio.h>

### 汇编 -S
	gcc —S test.i -o test.s
	或者 gcc -S test.i (-S大写)
将预处理的文件汇编，值得好好分析这个文件：

		.file	"test.c"
		.def	___main;	.scl	2;	.type	32;	.endef
		.section .rdata,"dr"
	LC0:
		.ascii "Hello,yunzhi!\0"
		.text
		.globl	_main
		.def	_main;	.scl	2;	.type	32;	.endef
	_main:
	LFB7:
		.cfi_startproc
		pushl	%ebp
		.cfi_def_cfa_offset 8
		.cfi_offset 5, -8
		movl	%esp, %ebp
		.cfi_def_cfa_register 5
		andl	$-16, %esp
		subl	$16, %esp
		call	___main	
		movl	$LC0, (%esp)
		call	_puts
		movl	$0, %eax
		leave
		.cfi_restore 5
		.cfi_def_cfa 4, 4
		ret
		.cfi_endproc
	LFE7:
		.ident	"GCC: (GNU) 4.8.2"
		.def	_puts;	.scl	2;	.type	32;	.endef


说明：

1. GCC产生的汇编代码中以 "." 开头的行都是指导汇编器和链接器的命令，称为“汇编器命令”。
2. 第2行：gcc编译器为_main插入的初始化函数。只有当gcc检测到当前的系统不支持 constructors 和destructors才会在编译的时候将该函数插入。具体请参见[gcc手册](http://gcc.gnu.org/onlinedocs/gccint/Initialization.html)。如果在编译的时候指定了INIT_SECTION_ASM_OP便不会出现_main函数。
3. 第5行：定义只读字符，字符以'\0'结束。
4. 第8行：用户在C中定义的main函数。gcc自动将其更名为__main。如果直接用汇编语言编写程序，一般是_start.
5. 第12行：main函数在最终的可执行文件中只是一个被调用的函数,在调用函数时需要做一些入栈和对齐操作，还有重要的一点就是传递的参数也在栈中。这句的作用是将%ebp入栈。
6. 第15行：将保存当前的栈指针%esp移入%ebp中。
7. 第17-18行：栈对齐操作
8. 第19行：调用_main函数
9. 第21行：将字符串的地址拷贝到%esp所指的内存中，该地址将会传递给_puts（printf）函数。
10. 第22行：将返回值（立即数0）传递给%eax 
11. 第23行： 相当于movl %ebp, %esp   popl %ebp，与它相对应的命令是enter
12. 第26行：main函数执行完毕，返回

简言之，当函数被调用时，执行操作如下：

1. 将帧指针（ebp）压入栈中。 pushl %ebp ; push后的l说明是一个long型
2. 用ebp保存当前栈指针(esp)。 movl %esp, %ebp;
3. 使得栈指针自减，自减得到的内存应当能够被用来存储被调用函数的本地状态。

### 编译成二进制文件 -c
	gcc -c test.s
	或者 gcc -c test.s -o test.o
来看看test.o。
	$ file test.o
	test.o: 80386 COFF executable not stripped - version 30821

### 连接成目标文件 -o
	gcc test.o -o test	//linux
	gcc test.o -o test.exe	//ubuntu
上面做连接时并未使用优化选项。优化的编译选项如下
	gcc -o2 test.o -o test
	gcc -o1 test.c -o test	//也可以一开始就使用优化的编译连接选项

### 多文件编译 -o
例如 hello.c world.c可通过如下编译命令生成 great.exe文件。
	gcc hello.c world.c -o great.exe

### gcc编译-D选项 传参
练习源代码如下：

	/*
	 *验证 gcc -D 传参的用法
 	*/
	#include<stdio.h>

	int main(int argc,char *argv[])
	{

	#ifdef HAHA
		printf("HAHA=%d\n",HAHA);
	#else
		printf("HAHA was not defined!\n");
	#endif
		return 0;
	}

编译及结果：

> gcc -Wall -DHAHA test.c
> a.exe
> 运行结果：HAHA=1
> 若 gcc -Wall test.c -DHAHA=2
> 运行结果：HAHA=2
> 若 gcc -Wall test.c -DHAHA=good，则无法通过编译
> 原因是good没有申明

再看一个例子，然后来总结用法：

	'''
	#include<string.h>

	#ifdef HAHA
	#undef HAHA
	#define HAHA "haha"
	#endif

	int main()
	{
	#ifdef HAHA
		if(strcmp(HAHA,"haha")==0){
			printf("Hello.yunzhi!\n");
		}
	#else
		printf("HAHA was not defined!\n");
	#endif
		return 0;
	}
	'''
> 使用gcc -DHAHA test.c ，gcc -DHAHA=good test.c 与 gcc -DHAHA=2 test.c 
> 编译结果都相同：Hello,yunzhi!
> 使用 gcc test.c
> 执行 a.exe 结果 HAHA was not defined!

最后对于上面的第二个例子我们执行下这个命令看一下预处理后的文件：

> gcc -E test.c -o test.i -DHAHA=2

通过查看test.i文件可以看到如下预处理结果：

	int main()
	{
	 	if(strcmp("haha","haha")==0){
  			printf("Hello.yunzhi!\n");
 		}
		return 0；
	}

小结：

+ 1. 通过上面的两个例子可以看到，gcc 在使用 -D 传入编译参数的过程仅仅作用于预处理阶段。
+ 2. 当把-D后的宏变量定义为int型时，可以将该int值传递到程序中。而定义为别的值时，只会认为该宏已被定义。如果想传一个字符串给代码中得宏，可以这样定义
	
		-DHAHA=\"good\"

+ 3. 最后，需要提到-U参数刚好完成与-D相反的操作。

-----------------------------------------------------------------------------------------
## 参考

1. [GCC的内嵌汇编语法](http://os.chinaunix.net/a2008/0313/977/000000977964.shtml)
2. [CSDN论坛](http://bbs.csdn.net/topics/320108252)
3. [gcc内嵌汇编](http://www.cnblogs.com/zhuyp1015/archive/2012/05/01/2478099.html)