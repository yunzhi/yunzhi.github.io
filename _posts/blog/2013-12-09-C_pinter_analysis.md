---
layout: post
title: C语言知识点
category: blog
description: 我所理解和掌握的C语言知识点总结
---


## 指针

C语言中最神奇，重要和麻烦的东西就是这个。工欲善其事,必先利其器。

### 1.指针的声明定义

这里先看一个错误指针声明。

	#include<stdio.h>
	int main(int argc,char *argv[])
	{
		int *p;
		*p = 12;
		printf("*p=%d\n",*p);	

		return 0;
	}

第一行首先定义了一个指针变量p，按字面的意思，p这个地址存储了一个常量值 10。那么p这个地址在哪呢？因为没有给他分配地址，在编译时，都会给出警告。但win7下运行正常，ubuntu上打印出**核心段已转储**的错误。

	$ gcc -Wall test.c
	test.c: 在函数‘main’中:
	test.c:6:4: 警告：此函数中的‘p’在使用前未初始化 [-Wuninitialized]
  	*p=12;
      ^
	./a.exe
	*p=12

因此，在指针的使用前，需要给它分配内存空间，或者将它指向别的地址。

	#include<stdio.h>
	#include<malloc.h>

	int main(int argc,char *argv[])
	{
		int *p;
		p = (int *)malloc(sizeof(int));
		*p = 12;
		printf("*p=%d\n",*p);	

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


[Yunzhi]:    http://yunzhi.github.io  "Yunzhi"
