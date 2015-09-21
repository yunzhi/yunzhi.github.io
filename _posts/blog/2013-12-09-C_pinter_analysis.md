---
layout: post
title: C语言知识点
category: blog
description: 我所理解和掌握的C语言知识点总结
---

## 一、基础知识概括
从概念上整理下c语言涉及的要点。

### 1. 标准头文件及相关函数
* <stdio.h> fopen, fclose, fgetc, fputc, scanf, printf, fscanf, fprintf, sscanf, sprintf, fread, fwrite
* <math.h> sin, cos, tan, asin, acos, atan, exp, log, log10, sqrt, fabs, pow, fmod, ceil, floor
* <ctype.h> isalpha, isdigit, isalnum, isspace, isupper, islower, iscntrl, isprint, isxdigit, ispunct, tolower, toupper
* <string.h> strlen, strcpy, strncpy, strcat, strcmp, strchr, strrchr, strstr, memcpy, memmove, memcmp, memchr, memset
* <stdlib.h> rand, srand, calloc, malloc, realloc, free, abs, labs, div, ldiv, atof, atoi, atol, abort, exit, system, getenv, bsearch, qsort
* <asset.h> assert, perror, strerror
* <locale.h> setlocale, localeconv
* <setjmp.h> setjmp, longjmp
* <signal.h> signal, raise
* <stdarg.h> va_list, ca_start, va_arg, va_end
* <time.h> localtime, asctime, ctime, gmtime, time, tzset, difftime
* <stddef.h>标准头文件都有包含,定义了false,true,NULL,offsetof
* <errno.h>
* <float.h>
* <limits.h>

### 2. 数组用法
* 1). 定义
* 2). 初始化:memcpy,memset
> 参考stackoverflow上的这篇文章：[How to initialize an array in C](http://stackoverflow.com/questions/201101/how-to-initialize-an-array-in-c)。
> 有一个比较有意思的语法是` int array[100] = {[0 ... 99] = 1};//注意 ... 前后都需要一个空格，否则编译会报错 `。这种初始化方式在内核中比较常见，文章也出说了该方法是gcc编译器加入的对c的扩展特性。
* 3). 使用
> 注意数组名是一个指针常量，也就是说不能指向别的地址。

		int array[10];
		int *ap = array + 2;
> 下面这些使用都是正确的：

		ap：也就是 array+2，即 &array[2]
		*ap : array[2]
		ap[0]：array[2]
		ap + 6：&arrray[8]
		*(ap +6)：array[8]
		ap[6]：array[8]
		&ap：&ap[0]，&array[2]
		ap[-1]：array[1]
> 编译器也接受数组形式的形参。如main函数的原型`int main(int argc,char* argv[]);`。所以下面这两个函数原型是相等的：

		int strlen(char* string);
		int strlen(char string[]);		
> 数组的长度。

		int test[8];// sizeof(test) = 32 ; ARRAY_SIZE = (sizeof(test)/testof(test[0]))

### 3. 指针
* 1). 定义：

		int* ptr; 
		struct TEST* my_test;
* 2). 初始化:指向已分配好空间的变量. 

		ptr = (int*)malloc(sizeof(int));
		my_test = (struct TEST*)malloc(sizeof(struct TEST));

* 3). 使用

		ptr++;
		ret = scanf("%d",ptr);
		memset(my_test,0,sizeof(struct TEST));

### 4. struct,union,enum
* 1). struct使用:

> (1). 结构体成员.和->的使用原则：
>> 点操作符接受两个操作数，左操作数就是**结构变量的名字**，右操作数就是需要访问的成员的名字；
>> 箭操作符接受两个操作数，左操作数**必须是一个指向结构的指针**，右操作数就是需要访问的成员的名字；

> (2). 结构体对齐,长度;
> (3). typedef.

		struct hello{
			int w;
			int o;
			char r;
			float l;
			double d;
		};
		typedef struct{
			int w;
			int o;
			char r;
			float l;
			double d;
		}hello;
> (4). 结构体的初始化
> (5). 使用：a.作为函数参数时传递的是结构体的指针，这样效率比传值高；

* 2). union使用
	
		struct{
			struct test{
				int a:8;
				int b:8;
				int c:8;
				int d:18;	
			};
			char e[8];
		}test_union;

* 3). enum的使用:	
> (1). 枚举值是常量，不是变量，如果没有定义第一个值，则默认为0;
> (2). 枚举类型的长度是4字节，sizeof(test)=4; 
> (3). 枚举值只能是整数

		typedef enum{                                                                    
    			test1=-10,                                                                   
    			test2=0,                                                                     
    			test3,                                                                       
    			test4=5,                                                                     
    			test5,                                                                       
    			test6=10                                                                     
		}test; 

### 5. 位域,位操作
* 1). “位域”是把一个字节中的二进位划分为几个不同的区域， 并说明每个区域的位数。每个域有一个域名，允许在程序中按域名进行操作。 这样就可以把几个不同的对象用一个字节的二进制位域来表示。位域用结构体struct来表示,例如:

		struct test{
			int a:2;
			int b:4;
			int c:7;
		};
> (1).一个位域必须存储在同一个字节中，不能跨两个字节;
> (2).位域的长度不能大于一个字节的长度;
> (3).位域可以无位域名，这时它只用来作填充或调整位置。无名的位域是不能使用的。
> (4).位域也有正负，指有符号属性的，就是最高位表示的

### 6. static关键字,volatile,const
* 1). static关键字两个用法:
> (1）,用于函数/全局变量表示该函数/全局变量在该文件内有效,外部文件无法引用;
> (2）,用于局部变量,表示无论进入多少次包含该局部变量的函数,该值只初始化一次,可以理解为只能通过该函数进行改变的全局变量.

* 2). volatile用来修饰被不同线程访问和修改的变量.与volatile变量有关的运算，不要进行编译优化
* 3). const表示变量或指针是不能改变的常量,如果在程序中确定该变量不能改变,宜用该关键字加以限定,以避免未知的改动.编译器通常不为普通const常量分配存储空间，而是将它们保存在符号表中，这使得它成为一个编译期间的常量，没有了存储与读内存的操作，使得它的效率也很高。
> 指针常量: int const *p;表是p是一个指向的int型的指针,该指针指向的值是不能改变的.譬如不能做 *p=10 这样的操作.但可以进行 p=&a这样的操作
> 常量指针: int * const P;表是p是一个指向的int型的常量指针,该指针不能在指向别的变量.譬如不能做 p=&a 这样的操作.但可以进行 *p=10 这样的操作

### 7. 宏定义与预处理
\#ifdef #define #undef #endif
\#include

### 8. 变量的生命周期与存储
变量(局部,全局,静态)的生命周期及其编译存储的data,code,bss,stack段分布
* 1). 局部变量(函数内定义的不带static关键字的局部变量,只在该函数生命周期内有效,初始化在stack段)
* 2). 局部变量(函数内定义的带static关键字的未初始化或者初始化为0的局部变量,相当与全局变量,保存在bss段,但只能在该函数内部改变)
* 3). 全局变量,整个文件生命周期内存在,未初始化或者初始化为0的存在于bss段,不占用程序文件的大小.
* 4). 全局变量,整个文件生命周期内存在,初始化不为0的存在于data段,占用程序文件的大小.

另，[C语言全局变量那些事儿](http://coolshell.cn/articles/10115.html)这篇文章提到了全局变量的使用中可能引起的一些问题值得在编程中引起重视。
> 编译器对多重定义的全局符号的解析和链接。在编译阶段，编译器将全局符号信息隐含地编码在可重定位目标文件的符号表里。这里有个“强符号(strong)”和“弱符号(weak)”的概念——前者指的是定义并且初始化了的变量，后者指的是未定义或者定义但未初始化的变量。当符号被多重定义时，GNU链接器(ld)使用以下规则决议：

* 不允许出现多个相同强符号。
* 如果有一个强符号和多个弱符号，则选择强符号。
* 如果有多个弱符号，那么先决议到size最大的那个，如果同样大小，则按照链接顺序选择第一个。
	
### 9. 扩展关键字 
\_\_attribute\_\_,aligned,packed,#pragma
* 1). \_\_attribute\_\_((packed))为变量或者结构体成员使用最小的对齐方式，即对变量是一字节对齐，对域（field）是位对齐.
* 2). 使用伪指令#pragma pack(n)，C编译器将按照n个字节对齐。
* 3). 使用伪指令#pragma pack()，取消自定义字节对齐方式,同__attribute__((aligned(4)))

### 10. 变量类型长度:
32位机器上，以下变量的长度是，括号里位64位系统中得不同之处

变量类型|.  占用的字符数   .|  备注
-----------|-----------------|-----
	char|	1|
	short|	2|
	int|		4|
	long|	4|	(64位系统为8)
	float|	4|
	*ptr|	4|	(64位系统为8)
	double|	8|

另，**sizeof()一个字符串时算上了最后的NUL(\0)，而strlen没有计算。**
	
### 11.动态内存分配
malloc 和 free 的函数原型如下，在头文件stdlib.h中声明

	void *malloc(size_t size);
	void free(void *pointer)

malloc 所分配的是一块连续的内存。如果操作系统无法向malloc提供更多的内存，malloc就返回一个NULL指针

## 二、进阶

### 1. 指针

C语言中最神奇，重要和麻烦的东西就是这个。工欲善其事,必先利其器。

#### （1）.指针的声明定义
这里先看一个错误（可以通过编译，但运行出错）指针声明

	#include<stdio.h>
	int main(int argc,char *argv[])
	{
		int *p;
		*p = 12;
		printf("*p=%d\n",*p);	

		return 0;
	}

第一行首先定义了一个指针变量p，按字面的意思，p这个地址存储了一个常量值 10。那么p这个地址在哪呢？因为没有给他分配地址，在编译时，都会给出警告。但win7下可以运行，ubuntu和mac上打印出**核心段已转储**的错误，至于这个“Segmentation fault”产生的原因，待以后有时间研究下反编译后汇编代码的分析在来补充。

win7中运行情况：

	$ gcc -Wall test.c
	test.c: 在函数‘main’中:
	test.c:6:4: 警告：此函数中的‘p’在使用前未初始化 [-Wuninitialized]
	
	*p=12;
      ^
	./a.exe
	*p=12

mac中编译及运行情况：

	$ gcc -Wall test.c -o test.bin
	test.c:5:4: warning: variable 'p' is uninitialized when used here [-Wuninitialized]
	*p = 12;
	 ^
	 test.c:4:9: note: initialize the variable 'p' to silence this warning
	 int *p;
               ^
                 = NULL
	1 warning generated.
	$ ./test.bin
	Segmentation fault: 11

因此，在指针的使用前，需要给它分配内存空间，或者将它指向别的地址。

	#include<stdio.h>
	#include<malloc.h>

	int main(int argc,char *argv[])
	{
		int *p;
		p = (int *)malloc(sizeof(int));
		*p = 12;
		printf("*p=%d\n",*p);	
		free（p）;
		
		return 0;
	}

或者

	#include<stdio.h>

	int main(int argc,char *argv[])
	{
		int *p;
		int a = 12;
		p = &a;
		printf("*p=%d\n",*p);
		*p = 13;
		printf("*p=%d\n",*p);	

		return 0;
	}

### 2.typedef的妙用
大部分的C语言语法书都有这个例子，说明typedef比直接的申明定义好使。

	typedef char *ptr;
	ptr a,b;
	
上面的代码中，a和b都是指向 char行变量的指针。另外的，在大部分的语法书上还会提到这个声明：

	void (*signal(int sig,void (*func)(int)))(int);
解释如下：signal是一个函数，他的返回值是一个函数指针，这个函数指针有一个int参数，返回值为void型。至于signal这个函数，他有连个参数，第二个参数又是一个参数为int，返回值为void的函数指针。

书上会用typedef 来简化该函数：

	typedef void (*ptr_func)(int);
	//可能比较难理解的是这个申明
	//参考上面的typedef char *ptr；就变得容易理解了
	//于是把ptr_func当成一个类似于 char * 的指针去用就行
	//当然，这里 ptr_func是一个函数指针
	
	ptr_func signal(int sig,ptr_func)；

同样的，就不难理解

	typedef int (*cust_brightness_set)(int level);

这样的形式，在某平台的背光驱动中有这样的一个函数,

	int light（int level）
	{
		……
	}

	struct cust_led{
		……
		int data；
		……
	};

	struct cust_led cust={
		.data=(int)light,
	}

但更有意思的是接下来的代码：
	
	return ((cust_brightness_set)(cust->data))(level);

上面的函数中 dev->data 的原型是int的，在ARM32位上，int和指针长度都是4字节，于是进行了强制转换，将data强制转换为cust_brightness_set指针，同时传入level的参数。函数指针运行的返回值是一个int 类型。为了验证这中用法，写了个程序，code如下：

	#include<stdio.h>
	typedef int (*cust_brightness_set)(int level);

	int light(int level)
	{
		printf("enter in light!\n");
		return level;
	}

	struct test_data{
		char *name;
		int data;
	};

	static struct test_data mytest={
		.name="test",
		.data = (int)light,
	};

	int main(int argc,char *argv[])
	{
		int ret;

		ret = ((cust_brightness_set)(mytest.data))(10);
		printf("ret = %d\n",ret);
		return 0；
	}

编译运行：

	$ gcc -Wall test.c ;./a.exe
	enter in light!
	ret = 10

### 宏和指针的妙用

高通平台的耳机驱动中，有这样一段用了许多宏的代码，使得本来要初始化一堆变量的操作直观看起来较为清晰，对c语言的某些语法和语义理解比较清晰，非常巧妙，这里来分析下。

由于这些代码是开源在codegen上面的，所以这里引起原code的变量等，为了简单的可以验证，这里加入main函数。code见 [这里](/C_language_excise)

### 一些小技巧

为了提升程序运行的效率，优化代码是很有必要的，这里有一些小例子。

#### （1）一个对数组进行初始化的代码 （引自《c和指针》）

	#define SIZE 100
	void try(int x[],int y[])
	{
		int i;
		for(i=0;i<SIZE;i++)
			x[i] = y[i];
	}
	
优化后的代码：

	#define SIZE 100
	void better(int x[],int y[])
	{
		register int *p1,*p2;
		for( p1 = x, p2 = y;p1 < &x[SIZE]; )
			*p1++ = *p2++；
	}
说明，从这两段代码的汇编代码可以看到，第二段代码由于不需要对中间变量i进行操作，省去了这部分代码的执行时间。同时使用寄存器变量，不用再复制指针的值.

#### (2) 宏处理，#号和##号的用法

\#argument 这种结构被预处理器翻译成 argument.于是可以有如下的代码：

	#define PRINT(FOMART,VALUE) printf("the value of " #VALUE " is" FOMART "\n",VALUE)

打印示例：
	x=10
	PRINT("%d", x+3)
	>> the value of x+3 is 13

\#\#结构把位于他两边的符号连接成同一个符号。

#### (3) 浮点数比较和技术的判断

判断两个浮点数 a 和 b 是否相等时,不要用 a==b,应该判断二者之差的绝对值 fabs(a-b) 是否小于某个阈值,例如 1e-9。判断一个整数是否是为奇数,用 x % 2 != 0,不要用 x % 2 == 1,因为 x 可能是负 数。

## 三、他山之木
[酷壳](http://coolshell.cn)上的一些c语言的帖子，转载到这里，相关的精彩的内容也做一些转载，感谢陈皓老师。

* 1). [如何学好c语言](http://coolshell.cn/articles/4102.html)
* 2). [深入理解c语言](http://coolshell.cn/articles/5761.html)
* 3). [c语言的谜题](http://coolshell.cn/articles/945.html)
* 4). [谁说c语言很简单](http://coolshell.cn/articles/873.html)
* 5). [语言的歧义](http://coolshell.cn/articles/830.html)
* 6). [c语言整形溢出问题](http://coolshell.cn/articles/11466.html)
* 7). [一个“蝇量级”c语言协成库](http://coolshell.cn/articles/10975.html)
* 8). [6个变态的c语言hello，world程序](http://coolshell.cn/articles/914.html)
* 9). [无锁队列的实现](http://coolshell.cn/articles/8239.html)
* 10). [Python调用C语言函数](http://coolshell.cn/articles/671.html)
* 11). [代码执行的效率](http://coolshell.cn/articles/7886.html)

### 1. size_t在64位机器下的问题
在64位下，sizeof的size_t是unsigned long，而32位下是 unsigned int，所以，C99引入了一个专门给size_t用的%zu。这点需要注意。在64位平台下，C/C++ 的编译需要注意很多事。

### 2. 编译指令技巧
编译器warning。 如下代码：
	
	#include <stdio.h>
	int main(void)
	{
    		int a;
    		printf("%d\n", a);
	}

考虑下面两种编译代码的方式 ：

+ cc -Wall a.c
+ cc -Wall -O a.c

前一种是不会编译出a未初化的警告信息的，而只有在-O的情况下，才会有未初始化的警告信息。这点就是为什么我们在makefile里的CFLAGS上总是需要-Wall和 -O。	

### 3. 一个指针用法的问题

	#include <stdio.h>
	int main(void)
	{
    		int a[5];
    		printf("%x\n", a);
    		printf("%x\n", a+1);
    		printf("%x\n", &a);
    		printf("%x\n", &a+1);
	}

假如我们的a的地址是：0Xbfe2e100, 而且是32位机，那么这个程序会输出什么？

+ 第一条printf语句应该没有问题，就是 bfe2e100
+ 第二条printf语句你可能会以为是bfe2e101。那就错了，a+1，编译器会编译成 a+ 1\*sizeof(int)，int在32位下是4字节，所以是加4，也就是bfe2e104
+ 第三条printf语句可能是你最头疼的，我们怎么知道a的地址？我不知道吗？可不就是bfe2e100。那岂不成了a==&a啦？这怎么可能？自己存自己的？也许很多人会觉得指针和数组是一回事，那么你就错了。如果是 int \*a，那么没有问题，因为a是指针，所以 &a 是指针的地址，a 和 &a不一样。但是这是数组啊a[]，所以&a其实是被编译成了 &a[0]。
+ 第四条printf语句就很自然了，就是bfe2e104。还是不对，因为是&a是数组，被看成int(\*)[5]，所以sizeof(a)是5，也就是5\*sizeof(int)，也就是bfe2e114。

### 4. main()问题
在C标准下，如果一个函数不要参数，应该声明成main(void)，而main()其实相当于main(…)，也就是说其可以有任意多的参数。

### 5. 浮点数的比较
下面程序的输出是？

	#include <stdio.h> 
	int main()
	{
    		float f=0.0f;
    		int i;
 
    		for(i=0;i<10;i++)
        	f = f + 0.1f;
 
    		if(f == 1.0f)
        		printf("f is 1.0 \n");
    		else
        		printf("f is NOT 1.0 \n");
 
    		return 0;
	}

解答：
不要让两个浮点数相比较。所以本题的答案是”f is NOT 1.0″，如果你真想比较两个浮点数时，你应该按一定精度来比较，比如你一定要在本题中做比较那么你应该这么做if( (f – 1.0f)<0.1 )

### 6. -(减号)问题

如下这段程序，编译出错，原因何在？

	#include <stdio.h>
 
	void OS_Solaris_print()
	{
    		printf("Solaris - Sun Microsystems\n");
	}
 
	void OS_Windows_print()
	{
    		printf("Windows - Microsoft\n");
	}
 
	void OS_HP-UX_print()
	{
    		printf("HP-UX - Hewlett Packard\n");
	}
 
	int main()
	{
    		int num;
    		printf("Enter the number (1-3):\n");
    		scanf("%d",&num);
 
    		switch(num)
    		{
        		case 1:
            		OS_Solaris_print();
            		break;
        		case 2:
            		OS_Windows_print();
            		break;
        		case 3:
            		OS_HP-UX_print();
            		break;
        		default:
            		printf("Hmm! only 1-3 :-)\n");
        		break;
    		}
    		return 0;
	}

解答：
程序员要以计算机的语言进行思考，不上上面那段程序导致的结果不止是歧义这么简单，而直接的结果是，导致计算机”听不懂”你在说什么。导致计算机听不懂的原因是HP-UX中的’-‘是减号？还是其他什么？

### 7. #和##的用法示例
这段程序的输出结果是？

	#include <stdio.h>
	#define f(a,b) a##b
	#define g(a)   #a
	#define h(a) g(a)
 
	int main()
	{
        printf("%s\n", h(f(1,2)));
        printf("%s\n", g(f(1,2)));
        return 0;
	}

本题的输出是12和f(1,2)，为什么会这样呢？因为这是宏，宏的解开不象函数执行，由里带外。

### 8. 代码作用域
下面这个函数返回值是什么？
	
	int x = 5;
	int f() {
		int x = 3;
  		{
    			extern int x;
    			return x;
  		}
	}

结果是5，如果编译参数加上-Wall -O,会提示 `int x =3 ; warning: unused variable 'x' [-Wunused-variable]`

### 9. 结构体的初始化
初始化可能是ISO C中最难的部分了。无论是MSVC 还是GCC 都没有完全实现。GCC 可能更接近标准。在下面的代码中，i.nested.y 和i.nested.z的最终值是什么？
	
	struct {
   		int x;
   		struct {
       		int y, z;
   		} nested;
	} i = { .nested.y = 5, 6, .x = 1, 2 };

答案：2和6. 
	
### 10. stdout和stderr
下面的程序并不见得会输出 hello-std-out，你知道为什么吗？

	#include <stdio.h>
	#include <unistd.h>
	int main()  
	{
    		while(1)
    		{
        		fprintf(stdout,"hello-std-out");
        		fprintf(stderr,"hello-std-err");
        	sleep(1);
    		}
    		return 0;
	}

参考答案：stdout和stderr是不是同设备描述符。stdout是块设备，stderr则不是。对于块设备，只有当下面几种情况下才会被输入，1）遇到回车，2）缓冲区满，3）flush被调用。而stderr则不会。

### 11. switch后不会对变量进行初始化
请说出下面的程序输出是多少？并解释为什么？（注意，该程序并不会输出 “b is 20″）

	#include <stdio.h>
	int main()  
	{      
    		int a=1;      
    		switch(a)      
    		{   
        		int b=20;          
        		case 1: 
            			printf("b is %d\n",b);
            			break;
        		default:
            			printf("b is %d\n",b);
            			break;
    		}
    	return 0;
	}

参考答案：该程序在编译时，可能会出现一条warning: unreachable code at beginning of switch statement。我们以为进入switch后，变量b会被初始化，其实并不然，因为switch-case语句会把变量b的初始化直接就跳过了。所以，程序会输出一个随机的内存值。
	
### 12. scanf的另类用法
请问下面的程序输出什么？（假设：输入 Hello, World）

	#include <stdio.h>
	int main()  
	{ 
    		char dummy[80];
    		printf("Enter a string:\n");
    		scanf("%[^r]",dummy);
    		printf("%s\n",dummy);
    		return 0;
	}
	
参考答案：本例的输出是“Hello, Wo”，scanf中的”%[^r]”是从中作梗的东西。意思是遇到字符r就结束了。

### 13. 数据输出缓冲区
程序遇到“\n”，或是EOF，或是缓中区满，或是文件描述符关闭，或是主动flush，或是程序退出，就会把数据刷出缓冲区。需要注意的是，标准输出是行缓冲，所以遇到“\n”的时候会刷出缓冲区，但对于磁盘这个块设备来说，“\n”并不会引起缓冲区刷出的动作，那是全缓冲，你可以使用setvbuf来设置缓冲区大小，或是用fflush刷缓存。

### 14. [队列的无锁实现问题](http://coolshell.cn/articles/8239.html)

CAS操作——Compare & Set，或是 Compare & Swap，现在几乎所有的CPU指令都支持CAS的原子操作，X86下对应的是 CMPXCHG 汇编指令。

	EnQueue(x) //进队列
	{
    		//准备新加入的结点数据
    		q = new record();
    		q->value = x;
    		q->next = NULL;
    		
    		do {
    			p = tail; //取链表尾指针的快照
    		} while( CAS(p->next, NULL, q) != TRUE); //如果没有把结点链在尾指针上，再试
 
	    CAS(tail, p, q); //置尾结点
	}

我们可以看到，程序中的那个 do- while 的 Re-Try-Loop。就是说，很有可能我在准备在队列尾加入结点时，别的线程已经加成功了，于是tail指针就变了，于是我的CAS返回了false，于是程序再试，直到试成功为止。这个很像我们的抢电话热线的不停重播的情况。

你会看到，为什么我们的“置尾结点”的操作（第12行）不判断是否成功，因为：

如果有一个线程T1，它的while中的CAS如果成功的话，那么其它所有的 随后线程的CAS都会失败，然后就会再循环，
此时，如果T1 线程还没有更新tail指针，其它的线程继续失败，因为tail->next不是NULL了。
直到T1线程更新完tail指针，于是其它的线程中的某个线程就可以得到新的tail指针，继续往下走了。
这里有一个潜在的问题——如果T1线程在用CAS更新tail指针的之前，线程停掉或是挂掉了，那么其它线程就进入死循环了。下面是改良版的EnQueue()

	EnQueue(x) //进队列改良版
	{
    		q = new record();
    		q->value = x;
    		q->next = NULL;
    		
		p = tail;
		oldp = p
    		do {
        		while (p->next != NULL)
            			p = p->next;
    		} while( CAS(p.next, NULL, q) != TRUE); //如果没有把结点链在尾上，再试
 
	    CAS(tail, oldp, q); //置尾结点
	}

我们让每个线程，自己fetch 指针 p 到链表尾。但是这样的fetch会很影响性能。而通实际情况看下来，99.9%的情况不会有线程停转的情况，所以，更好的做法是，你可以接合上述的这两个版本，如果retry的次数超了一个值的话（比如说3次），那么，就自己fetch指针。

### 15. [Python调用C语言函数](http://coolshell.cn/articles/671.html)

在一些需要高性能的地方(比如加/解密算法)可以用python调用c函数，此时就需要借助C/C++生成库，由python调用，同时由c/c++生成的库能够保证安全。

使用Python的ctypes，我们可以直接调用由C直接编译出来的函数。其实就是调用动态链接库中的函数。

首先，我们用一个乘法来表示一个算法功能。下面是C的程序：

	int multiply(int num1, int num2)
	{
    		return num1 * num2;
	}

然后，把这个C文件编成动态链接库。Linux下的编译：
	
	gcc -c -fPIC libtest.c
	gcc -shared libtest.o -o libtest.so

于是在我们的Python中可以这样使用：

	>>> from ctypes import *
	>>> import os
	>>> libtest = cdll.LoadLibrary(os.getcwd() + '/libtest.so')
	>>> print libtest.multiply(2, 2)
	4

注意：上面的Python脚本中需要把动态链接库放到当前目录中。

### 16. 排好序的数据在遍历时会更快
因为排好序的更容易预测分支，所以在循环中除去if-else语句有利于提高性能。比如：我们把条件语句：

	if (data[j] >= 128)
	sum += data[j];
变成：

	int t = (data[j] - 128) >> 31;
	sum += ~t & data[j];
“没有分叉”的性能也很快

### 17. 0长数组的意义
C语言对结构体的访问就是该结构体成员的首地址加上结构体成员的偏移量。

0长数组不占用数组长度，即sizeof()的计算结果不用计入数组个数为0的成员变量长度，它存在的意义是：

* 第一个意义是，方便内存释放。如果我们的代码是在一个给别人用的函数中，你在里面做了二次内存分配，并把整个结构体返回给用户。用户调用free可以释放结构体，但是用户并不知道这个结构体内的成员也需要free，所以你不能指望用户来发现这个事。所以，如果我们把结构体的内存以及其成员要的内存一次性分配好了，并返回给用户一个结构体指针，用户做一次free就可以把所有的内存也给释放掉。（读到这里，你一定会觉得C++的封闭中的析构函数会让这事容易和干净很多）

* 第二个原因是，这样有利于访问速度。连续的内存有益于提高访问速度，也有益于减少内存碎片。

 ### 18. [C语言中史上最愚蠢的Bug]（http://coolshell.cn/articles/5388.html）
 这篇文章提到的code过程中的一个bug特别无奈，我也犯过这样的bug，当时觉得自己无比愚蠢。

## 四、c的世界很精彩

这几个hello,world程序很有意思

### 1). hello1.c

    #define _________ }
    #define ________ putchar
    #define _______ main
    #define _(a) ________(a);
    #define ______ _______(){
    #define __ ______ _(0x48)_(0x65)_(0x6C)_(0x6C)
    #define ___ _(0x6F)_(0x2C)_(0x20)_(0x77)_(0x6F)
    #define ____ _(0x72)_(0x6C)_(0x64)_(0x21)
    #define _____ __ ___ ____ _________
    #include<stdio.h>
    _____

### 2). hello2.c

    #include<stdio.h>
    main(){
      int x=0,y[14],*z=&y;*(z++)=0x48;*(z++)=y[x++]+0x1D;
      *(z++)=y[x++]+0x07;*(z++)=y[x++]+0x00;*(z++)=y[x++]+0x03;
      *(z++)=y[x++]-0x43;*(z++)=y[x++]-0x0C;*(z++)=y[x++]+0x57;
      *(z++)=y[x++]-0x08;*(z++)=y[x++]+0x03;*(z++)=y[x++]-0x06;
      *(z++)=y[x++]-0x08;*(z++)=y[x++]-0x43;*(z++)=y[x]-0x21;
      x=*(--z);while(y[x]!=NULL)putchar(y[x++]);
    }

### 3). hello3.c

    #include<stdio.h>
    #define __(a) goto a;
    #define ___(a) putchar(a);
    #define _(a,b) ___(a) __(b);
    main()
    { _:__(t)a:_('r',g)b:_('$',p)
      c:_('l',f)d:_(' ',s)e:_('a',s)
      f:_('o',q)g:_('l',h)h:_('d',n)
      i:_('e',w)j:_('e',x)k:_('\n',z)
      l:_('H',l)m:_('X',i)n:_('!',k)
      o:_('z',q)p:_('q',b)q:_(',',d)
      r:_('i',l)s:_('w',v)t:_('H',j)
      u:_('a',a)v:_('o',a)w:_(')',k)
      x:_('l',c)y:_('\t',g)z:___(0x0)}
      
### 4). hello4.c

    int n[]={0x48,
    0x65,0x6C,0x6C,
    0x6F,0x2C,0x20,
    0x77,0x6F,0x72,
    0x6C,0x64,0x21,
    0x0A,0x00},*m=n;
    main(n){putchar
    (*m)!='\0'?main
    (m++):exit(n++);}
    
### 5). 混合编程 hello5.c.sh
直接执行 sh hello5.c.sh，编译运行得到结果

	#if 0
	file=`mktemp`
	gcc -o $file $0
	$file
	rm $file
	exit
	#endif
	#include <stdlib.h>
	#include <stdio.h>
 
	int main(int argc, char *argv[]) 
	{
		puts("Hello from C!");
		return EXIT_SUCCESS;
	}

### 6). C语言中的Duff device，据说效率比memcpy快
 
 	void duff_memcpy( char* to, char* from, size_t count ) 
	{
		size_t n = (count+7)/8;
    		switch( count%8 ) {
    		case 0: do{ *to++ = *from++;
    		case 7:     *to++ = *from++;
    		case 6:     *to++ = *from++;
    		case 5:     *to++ = *from++;
    		case 4:     *to++ = *from++;
    		case 3:     *to++ = *from++;
    		case 2:     *to++ = *from++;
    		case 1:     *to++ = *from++;
            		}while(--n>0);
    		}
	} 



