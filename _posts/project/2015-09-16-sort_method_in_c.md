---
layout: post
title: 排序算法练习
category: project
description: 本篇用c语言记录排序算法，涉及到冒泡排序，选择排序
---

## 冒泡排序算法

如下结果是升序排列

    #include<stdio.h>
    
    int bubble_sort(int *array, int len)
    {
      int i,j;
      int tmp;
    
      for(i = 0; i< len-1; i++)
        for(j = i+1; j < len; j++)
          if( *(array+i) > *(array+j) ){
            tmp = *(array+i);
            *(array+i) = *(array+j);
            *(array+j) = tmp;
          }
    
      return 0;
    }
    
    int main()
    {
      int i;
      int test_array[5] = {6,9,5,1,3};
    
      bubble_sort(test_array, sizeof(test_array)/sizeof(test_array[0]));
    
      for(i=0;i<5 ;i++)
        printf("%d ",test_array[i]);
    
      printf("\n");
    
      return 0;
    }

# 选择排序

    #include<stdio.h>
    
    int test_select_sort(int *array,int len)
    {
      int i,j;
      int tmp;
      int index;
    
      for(i=0;i<len-1;i++){
        index = i;
        for(j=i+1;j<len;j++)
          if( *(array+index)>*(array+j) )
              index = j;
    
        if( index != i){
          tmp = *(array+i);
          *(array+i) = *(array+index);
          *(array+index) = tmp;
        }
      }
    
      return 0;
    }
    
    int main()
    {
      int i;
      int test_array[5] = {17,8,27,89,1};
    
      test_select_sort(test_array,5);
    
      for(i=0;i<5;i++)
        printf("%d ",test_array[i]);
    
      printf("\n");
      return 0;
    }

## 插入排序

代码理解方法：从后面一个数和前的数比较，如果比较小，就把它暂存起来，然后把前面所有和它相比小的数向后位移一位，最后把暂存大数放到移出来得空位上，陆续循环到数列末尾。

    #include<stdio.h>
    
    int insert_sort(int *array, int len)
    {
        int i,j,tmp;
    
        for(i=1;i<len;i++){
            if( *(array+i-1) > *(array+i) ){
                tmp = *(array+i);
                j = i;
                while( (j>0) && ( *( array+j-1) > tmp ) ){
                    *(array+j) = *(array+j-1);
                    j--;
                }
                *(array+j) = tmp;
            }
        }
        return 0;
    }
    
    int main()
    {
        int i;
        int array[5] = {5,3,4,6,2};
    
        insert_sort(array,5);
    
        for(i=0;i<5;i++)
            printf("%d ",array[i]);
    
        printf("\n");
        return 0;
    }

上面的排序时间复杂度都是O(n*n)

## 希尔排序

该方法的基本思想是：先将整个待排元素序列分割成若干个子序列（由相隔某个“增量”的元素组成的）分别进行直接插入排序，然后依次缩减增量再进行排序，待整个序列中的元素基本有序（增量足够小）时，再对全体元素进行一次直接插入排序。

    #include <stdio.h>
    
    int shell_sort(int a[], int n)
    {
        int i,j,k,gap;
        int temp;
    
        for( gap = n/2; gap > 0; gap /=2 ){
            for( i= 0; i < gap; i++){ //gap组插入排序
                for(j = i+gap; j < n; j+=gap){  //直接插入排序
                    if( a[j] < a[j-gap] ){
                        temp = a[j];
                        k = j - gap;
                        while( (k >= 0) && ( a[k] > temp)){
                            a[k+gap] = a[k];
                            k -= gap;
                        }
                        a[k + gap] = temp;
                    }
                }
            }
        }
        return 0;
    
    }
    
    int main()
    {
        int i;
        int array[10] = {9,2,8,3,6,7,5,1,4,0};
    
        shell_sort(array,10);
    
        for(i=0;i<10;i++)
            printf("%d ",array[i]);
        printf("\n");
    
        return 0;
    }

## 堆排序

    #include <stdio.h>
    
    int heap_adjust(int a[],int s,int m)
    {
        int temp,j;
        temp = a[s];
    
        for( j = 2*s + 1 ; j < m ; j *= 2 ){
            if( j < (m-1) && a[j] < a[j+1] )
                j++;
            if(temp > a[j])
                break;
            a[s] = a[j];
            s = j;
        }
        a[s] = temp;
    
        return 0;
    }
    
    int swap(int a[],int pos,int n)
    {
        int tmp;
        tmp = a[pos];
        a[pos] = a[n];
        a[n] = tmp;
    
        return 0;
    }
    
    int heap_sort(int a[], int n)
    {
        int i;
        for( i = n/2 -1 ; i >= 0; i--)
            heap_adjust(a,i,n);
    
        for( i = n-1 ; i > 1 ; i-- ){
            swap(a,0,i);
            heap_adjust(a,0,i-1);
        }
    
        return 0;
    }
    
    int main()
    {
        int i;
        int array[10] = {9,2,8,3,6,7,5,1,4,0};
    
        heap_sort(array,10);
    
        for(i=0;i<10;i++)
            printf("%d ",array[i]);
        printf("\n");
    
        return 0;
    }
