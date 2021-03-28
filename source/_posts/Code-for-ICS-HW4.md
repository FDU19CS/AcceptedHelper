---
title: Code for ICS HW4
date: 2021-03-28 14:21:07
index_img: https://ss0.bdstatic.com/70cFuHSh_Q1YnxGkpoWK1HF6hhy/it/u=1418123777,2258928929&fm=26&gp=0.jpg
category: [ICS]
tags: [Linker]
---



本文提供了ICS HW4补充作业中需要的初始代码。

#### main.c

```c
#include"add.h"
#include"BubbleSort.h"
#include"printResult.h"
#include<time.h>

#define bool char
#define true 1
#define false 0
#define LENGTH 10

int main()
{
    int a[LENGTH],i;
    int b[LENGTH];
    int randValue = 0;
    srand(time(NULL));
    for(i=0;i<LENGTH;i++){
        randValue = 1 + (int)rand()%LENGTH;
        a[i] = randValue;
        b[i] = a[i];
    }
    printResult(a,LENGTH,"\nrandom array: ");
    bool flag = true;
    while(flag){
        printf("\n1.Bubble Sort\n2.sum\n3.print result\n4.exit");
        printf("\nchoose a number:");
        int number = 0;
        scanf("%d",&number);
        int sum = 0;
        switch(number)
        {
            case 1:
                BubbleSort(a,LENGTH);
                break;
            case 2:
                sum = add(a,LENGTH);
                printf("\nresult of sum: %d\n",sum);
                break;
            case 3:
                printResult(b,LENGTH,"\noriginal array:\t");
                printResult(a,LENGTH,"\nsorted array:\t");
                break;
            case 4:
                flag = false;
                break;
            default:
                printf("\nplease choose a correct number and continue!");
                break;
        }
        printf("\nDone!\n\n");
    }
    return 0;
}
```



#### add.c

```c
#include"add.h"
int add(int s[],int n){
    //Write your code here
}
```



#### add.h

```c
#include<stdio.h>
int add(int s[],int n);
```



#### BubbleSort.c

```c
#include"BubbleSort.h"
void BubbleSort(int s[], int n){
    //Write your code here
}
```



#### BubbleSort.h

```c
#include<stdio.h>
void BubbleSort(int s[], int n);
```



#### printResult.c

```c
#include"printResult.h"
void printResult(int s[],int n,char* str){
    printf("%s",str);
    int i;
    for(i = 0;i<n;i++){
        printf("%5d",s[i]);
    }
    printf("\n");
}
```



#### printResult.h

```c
#include<stdio.h>
void printResult(int s[],int n,char* str);
```



~~愿天堂没有用word图片发布的作业代码。~~