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

编译及结果

	gcc -Wall test.c
	a.exe
	Hello，yunzhi！



## 一个小小的编程练习

假设某电影院卖票给男人，女人，孩子时票价不同，结算时，知道收入，票数，和各个单价。求可能的男人，女人，孩子组合数。

这是一个高中时代的3元一次方程，列两个方程，求解即可。

编程如下：

    #include<stdio.h>
    #include<malloc.h>
    #include<string.h>
    #include<stddef.h>
    #include <assert.h>
    
    //The most least number should be defined in CHILDCOST, or will be error
    //the three cost should not equal
    
    #define MANCOST			3
    #define WOMANCOST		2
    #define CHILDCOST		1
    
    #define MANCOST_MINUS_CHILDCOST ((MANCOST)-(CHILDCOST))
    #define WOMANCOST_MINUS_MANCOST ((WOMANCOST)-(MANCOST))
    #define WOMANCOST_MINUS_CHILDCOST ((WOMANCOST)-(CHILDCOST))
    
    #define size_t int
    
    /************************************************************************
    *	Algorithm:
    *	money	= man * MANCOST + woman *WOMANCOST + children * CHILDCOST
    *	people	= man + woman + children
    *
    *************************************************************************/
    void flush();
    size_t find_woman_men_children(size_t w_offset,size_t c_offset,size_t threshold);
    
    int main(int argc,char *argv[])
    {
    	size_t money,people;
    	size_t a,b;	//temp variable
    	size_t method = 0; // to get the method numbers
    	size_t i;
    	size_t *order[3];
    	size_t t_women,t_children,t_women_left,t_children_left;
    	size_t final_m; // temp max number
    
    	assert(MANCOST_MINUS_CHILDCOST);
    	assert(WOMANCOST_MINUS_CHILDCOST);
    
		printf("pls input money:");
		while( ( scanf("%d",&money) != 1 ) || ( money < 6 ) ){
			printf("\nMoney should be a number and not less than 6,pls input again:");
			flush();
		}

		printf("pls input people:");
		while( ( scanf("%d",&people) !=1 ) || ( people < 3 ) ){
			printf("\npeople should be a number and not less than 3,pls input again:");
			flush();
		}
	
    
    	a = money - CHILDCOST*people;
    	b = people*WOMANCOST - money;
    	final_m = (a-WOMANCOST_MINUS_CHILDCOST)/MANCOST_MINUS_CHILDCOST;
    
    	method = find_woman_men_children(a,b,final_m);
    
    	printf("we have %d method to handle this problem!\n",method);
    	for(i=0;i<3;i++){
    		order[i] = (size_t *)malloc(sizeof(size_t)*method); // to save the order
    		if(order[i] == NULL){
    			printf("failed to alloc space\n");
    			return -1;
    		}
    		memset(order[i],0,method);
    	}
    
    	for(i=0;i<method;i++){
    		t_women = (a - MANCOST_MINUS_CHILDCOST*(i+1))/WOMANCOST_MINUS_CHILDCOST;
    		t_women_left =  (a -MANCOST_MINUS_CHILDCOST*(i+1))%WOMANCOST_MINUS_CHILDCOST;
    		t_children = (b - WOMANCOST_MINUS_MANCOST*(i+1))/WOMANCOST_MINUS_CHILDCOST;
    		t_children_left = (b - WOMANCOST_MINUS_MANCOST*(i+1))%WOMANCOST_MINUS_CHILDCOST;
    		if( (t_women_left == 0) && (t_children_left== 0) \
    				&& (t_women > 0) && (t_children > 0) ){
    			*(order[0]+i) = i + 1;
    			*(order[1]+i) = t_women;
    			*(order[2]+i) = t_children;
    		}
    	}
    
    	printf("  MAN WOMEN CHILDREN\n");
    	for(i=0;i<method;i++){
    		printf("%5d%5d%5d\n",*(order[0]+i),*(order[1]+i),*(order[2]+i));
    	}
    
    	return 0;
    }
    
	void flush()
	{
		char c;
		while((c=getchar()) != '\n' && c != EOF);
	}

    size_t find_woman_men_children(size_t w_offset,size_t c_offset,size_t threshold)
    {
    	size_t t_women,t_children,t_women_left,t_children_left;
    	size_t i;
    	size_t length=0;
    
    	printf("  MAN WOMEN CHILDREN\n");
    	for(i=0;i<threshold;i++){
    		t_women = (w_offset - MANCOST_MINUS_CHILDCOST*(i+1))/WOMANCOST_MINUS_CHILDCOST;
    		t_women_left =  (w_offset -MANCOST_MINUS_CHILDCOST*(i+1))%WOMANCOST_MINUS_CHILDCOST;
    		t_children = (c_offset - WOMANCOST_MINUS_MANCOST*(i+1))/WOMANCOST_MINUS_CHILDCOST;
    		t_children_left = (c_offset - WOMANCOST_MINUS_MANCOST*(i+1))%WOMANCOST_MINUS_CHILDCOST;
    		if( (t_women_left == 0) && (t_children_left== 0) \
    				&& (t_women > 0) && (t_children > 0) ){
    			//printf("%5d%5d%5d\n",i+1,t_women,t_children);
    			length++;
    		}
    	}
    
    	return length;
    }
    
说明：
1.出于简单输入的缘故，程序中宏定义了男人，女人，孩子的票价。导致如果要变更价格需要重新编译。另外，某两个人的票价相同也可以。还需要假设孩子票价最低，如果不是可以将最低的票价放到孩子处，修改打印标题栏即可。
2.由于想将总结果存在一个动态数组中，先计算可能性，然后再进行存储。从代码上看有些重复，因为没有想到什么更好的办法。
3.用到的几个C语言知识：scanf,assert,动态数组，指向数组的指针。当然，还可以继续加ifdef这种宏来加入输入价格这样子的代码。顺便还看了看局部未初始化变量的存储。




