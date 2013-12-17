---
layout: post
title: gcc用法简介
description: 初步介绍和记录目前常用的gcc编译命令，以方便查询
category: blog
---

gcc 作为linux强大的编译工具，了解其用法对日常编程具有很大的作用。windows下安装类unix环境也可以使用，我就在win7上安装了cgwin，把它的bin目录加到系统的环境变量里，然后cmd下就可以用它那一堆强大的工具了，例如ls,grep,sed,gcc等工具了。


##常用gcc编译选项

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
		.def	_puts;	.scl	2;	.type	32;	.ende

### 编译成二级制文件 -c
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

### 编译成静态库


### 附录

$ gcc --help
用法：gcc [选项] 文件...
选项：
  -pass-exit-codes         在某一阶段退出时返回最高的错误码
  --help                   显示此帮助说明
  --target-help            显示目标机器特定的命令行选项
  --help={common|optimizers|params|target|warnings|[^]{joined|separate|undocumented}}[,...] 显示特定类型的命令行选项
  (使用‘-v --help’显示子进程的命令行参数)
  --version                显示编译器版本信息
  -dumpspecs               显示所有内建 spec 字符串
  -dumpversion             显示编译器的版本号
  -dumpmachine             显示编译器的目标处理器
  -print-search-dirs       显示编译器的搜索路径
  -print-libgcc-file-name  显示编译器伴随库的名称
  -print-file-name=<库>    显示 <库> 的完整路径
  -print-prog-name=<程序>  显示编译器组件 <程序> 的完整路径
  -print-multi-directory   显示不同版本 libgcc 的根目录
  -print-multi-lib         显示命令行选项和多个版本库搜索路径间的映射
  -print-multi-os-directory 显示操作系统库的相对路径
  -print-sysroot           显示目标库目录
  -print-sysroot-headers-suffix 显示用于寻找头文件的 sysroot 后缀
  -Wa,<选项>               将逗号分隔的 <选项> 传递给汇编器
  -Wp,<选项>               将逗号分隔的 <选项> 传递给预处理器
  -Wl,<选项>               将逗号分隔的 <选项> 传递给链接器
  -Xassembler <参数>       将 <参数> 传递给汇编器
  -Xpreprocessor <参数>    将 <参数> 传递给预处理器
  -Xlinker <参数>          将 <参数> 传递给链接器
  -save-temps              不删除中间文件
  -save-temps=<arg>        不删除中间文件
  -no-canonical-prefixes   生成其他 gcc 组件的相对路径时不生成规范化的前缀
  -pipe                    使用管道代替临时文件
  -time                    为每个子进程计时
  -specs=<文件>            用 <文件> 的内容覆盖内建的 specs 文件
  -std=<标准>              指定输入源文件遵循的标准
  --sysroot=<目录>         将 <目录> 作为头文件和库文件的根目录
  -B <目录>                将 <目录> 添加到编译器的搜索路径中
  -v                       显示编译器调用的程序
  -###                     与 -v 类似，但选项被引号括住，并且不执行命令
  -E                       仅作预处理，不进行编译、汇编和链接
  -S                       编译到汇编语言，不进行汇编和链接
  -c                       编译、汇编到目标代码，不进行链接
  -o <文件>                输出到 <文件>
  -pie                     Create a position independent executable
  -shared                  Create a shared library
  -x <语言>                指定其后输入文件的语言,允许的语言包括：c c++ assembler none
  ‘none’意味着恢复默认行为，即根据文件的扩展名猜测源文件的语言

以 -g、-f、-m、-O、-W 或 --param 开头的选项将由 gcc 自动传递给其调用的
 不同子进程。若要向这些进程传递其他选项，必须使用 -W<字母> 选项。

报告程序缺陷的步骤请参见：
<http://gcc.gnu.org/bugs.html>.

[Yunzhi]:    http://yunzhi.github.io  "Yunzhi"
