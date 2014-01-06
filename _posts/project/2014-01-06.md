---
layout: post
title: 2014练习1
category: project
description: memcpy函数的实现
---

##  memcpy函数

功能简介：该函数是标准的C库函数。提供了一般内存的复制，函数原型如下

	void memcpy(void *dest,const void *src,size_t count)

从原型中可以看到第二个函数是一个只读（const）的指针常量，即该指正指向的内容在函数体中是不会被改变的。第三个参数是复制内存的个数（类型为size_t，这是一个和机器位数相关的类型，32位系统被定义成unsigned int,64位系统是long unsigned int,C语言中位于头文件stddef.h中）。Linux kernle实现

	void *memcpy(void *dest, const void *src, size_t count)
	{
		assert(dest != NULL && src != NULL);
		char *tmp = dest;
		const char *s = src;
		while (count--)
		{
			*tmp++ = *s++ ;
		}
		return dest;
	}

若断言（assert）被证明非真，程序将在标准错误流输出一条适当的提示信息，并且执行异常终止。

写个小程序验证下功能。


	"""
	#include<stdio.h>
	#include<string.h>

	void *memcpy(void *dest, const void *src, size_t count)
	{
		if( dest == NULL && src == NULL){
			printf("Null pointer in src or dest,pls check!\n");
			return NULL;
		}

		char *tmp = dest;
		const char *s = src;
		while (count--)
		{
			*tmp++ = *s++ ;
		}
		return dest;
	}

	int main(int argc,char* argv[])
	{
		char *s="Hello,yunzhi!";
		char d[20]={ [0 ... 18] = 'a','\0'};
		memcpy(d,s,(strlen(s)+1) );
		printf("%s",d);
		return 0;
	}
	"""

编译及结果
gcc -Wall test.c
a.exe
Hello，yunzhi！