---
title: Malloc Lab 动态内存分配器
date: 2021-05-22 22:58:57
index_img: https://tuapi.eees.cc/api.php?category=fengjing
category: [ICS]
tags: [Malloc, VM]
---

# Malloc Lab 

​	个人的实验报告，放上来给大家参考。

​	Malloc lab 需要我们编写一个类似 libc malloc 的动态内存分配器，其主要考察动态内存分配器的原理设计以及堆内存的结构组织，同时需要比较强的 DEBUG 能力。最后在不使用BST以及其他全局数据结构的情况下我的方法达到了 97/100 的分数

[ZiYang-xie/Malloc_Lab: CMU Malloc Lab Repo (github.com)](https://github.com/ZiYang-xie/Malloc_Lab)

## 一、空闲块组织结构

​	在结构设计上我采用了分离存储的显示链表形式来进行组织空闲块，在书上说明了分离存储的思想，但没有具体说明实现方法。在此我使用称为 **Segregated Free List** 的空闲块组织设计，即在堆的低地址分配数量等于 ```SEG_LEN``` 的指针，每个指针分别对应着一个大小类，指向正式堆块中的空闲块，相当于 ```SEG_LEN``` 个链表。

![Segregated Free List](https://tva1.sinaimg.cn/large/008i3skNly1gquqskpijgj30ff07cdg2.jpg)

​	在我的代码设计中，我以2的幂次分割大小类，由于空闲块最小块大小为16 bytes （包括头尾标记以及前后指针）因此其设计为 {2^4 ~ 2^5} \ {2^5 ~ 2^6} \ {2^6 ~ 2^7} ...(类推)

 为了区分某一空闲块应该被放置在哪个类中，我们需要一个 **get_index** 函数，正常设计也十分简单，即通过一个循环右移，计算位数。在这里我参考了 **Bit twiddling hacks** 著名位运算*奇技淫巧* 网站，采用了一个位运算的 log2 方式，其可以在 O(1) 的复杂度计算出 log2(x)

```c
static int get_index(size_t v) 
{
    // 本质上是位运算的 log 2, O(1)复杂度
    // 参考 'Bit twiddling hacks'
    // Linking: https://graphics.stanford.edu/~seander/bithacks.html#IntegerLogLookup
    
    size_t r, shift;
    r = (v > 0xFFFF)   << 4; v >>= r;
    shift = (v > 0xFF) << 3; v >>= shift; r |= shift;
    shift = (v > 0xF)  << 2; v >>= shift; r |= shift;
    shift = (v > 0x3)  << 1; v >>= shift; r |= shift;
                                          r |= (v >> 1);
    // 从 2^4 开始 (空闲块最小 16 bytes)
    int x = (int)r - 4;
    if(x < 0) 
        x = 0;
    if(x >= SEG_LEN) 
        x = SEG_LEN - 1;
    return x;
}
```



## 二、堆内存设计

​	在空闲块指针之上，分配正常的堆块，正常的堆块由**序言块** （一个已分配大小为8的块），以及**结尾块**（一个已分配大小为0的块）前后包围，这样可以很方便的检验边界情况，当后继块大小为0，那么便可判断其达到了结尾。之后便记录下全局的开始地址 ```global_list_start_ptr``` 即可

```c
/* 空闲块 */
for(i = 0; i < SEG_LEN; ++i)
  PUT(heap_listp + i*WSIZE, NULL);	            // 初始化空闲块大小类头指针

/* 分配块 */
PUT(heap_listp + (i+0)*WSIZE, PACK(DSIZE, ALLOCATED));  /* 序言块头部 */
PUT(heap_listp + (i+1)*WSIZE, PACK(DSIZE, ALLOCATED));  /* 序言块尾部 */
PUT(heap_listp + (i+2)*WSIZE, PACK(0, ALLOCATED));      /* 结尾块头部 */

global_list_start_ptr = heap_listp;
heap_listp += (i+1)*WSIZE; // 对齐到起始块有效载荷
```



## 三、具体设计

​	接下来以函数为单位详细介绍实现过程

	### mm_init 初始化堆

​	堆内存设计块节中以及包含大部分，mm_init 代码，在组织完堆初始化的指针之后就可以进行分配栈空间以一个初始化的空闲块，这涉及到了 extend_heap 函数

```c
/* 扩展空栈至 CHUNKSIZE bytes */
    if(extend_heap(CHUNKSIZE) == NULL)
        return -1;
    return 0;
```



### extend_heap 堆扩展

​	对于堆扩展，我们调用 mm_sbrk 函数将lab中设计好的抽象 program breaker 上移扩展堆大小，其返回空闲块的头指针，我们设置好它的头尾标记，并通过 coalesce 函数在进行前后空闲块合并之后插入到空闲块链表中。

```c
static void *extend_heap(size_t asize)
{
    char *bp;
    if((long)(bp = mem_sbrk(asize)) == -1)
        return NULL;
    
    /* 初始化空闲块的头尾和结尾块的头部 */
    PUT(HDRP(bp), PACK(asize, FREE));                /* 空闲块头部 */
    PUT(FTRP(bp), PACK(asize, FREE));                /* 空闲块尾部 */
    PUT(HDRP(NEXT_BLKP(bp)), PACK(0, ALLOCATED));    /* 结尾块头部 */

    return coalesce(bp);
}
```



### coalesce 合并块

​	合并块的模式包含四种情况，并且在我的设计模式中，在合并后将空闲块插入到空闲链表中去，形成一体化操作。

- **Case1: 前后均不空闲**

```c
if(prev_alloc && next_alloc){                   /* 前后非空闲 */
  insert_free_block(bp);
  return bp;
}
```

前后均不空闲的时候就直接插入当前空闲块，并返回bp

- **Case2: 后空闲**

```c
else if(prev_alloc && !next_alloc){             /* 后空闲 */
  size += NEXT_BLKSZ(bp);
  delete_free_block(NEXT_BLKP(bp));
  PUT(HDRP(bp), PACK(size, FREE));
  PUT(FTRP(bp), PACK(size, FREE));
  PUT(PRED(bp), NULL);
  PUT(SUCC(bp), NULL);
}
```

​	后空闲的时候就从空闲链表中删除后方空闲块，并把当前块的头部和后部块的尾部大小设计为扩展后大小 *( 由于 FTRP 中调用了 HDRP，所以先设计HDRP的size之后FTRP能够正确定位到尾部 )* 并且设置空闲块前驱后继指针为NULL做好清理。

- **Case3:** 前空闲

```c
if(!prev_alloc && next_alloc) {            /* 前空闲 */
  size += PREV_BLKSZ(bp);
  delete_free_block(PREV_BLKP(bp));

  PUT(FTRP(bp), PACK(size, FREE));
  PUT(HDRP(PREV_BLKP(bp)), PACK(size, FREE));

  bp = PREV_BLKP(bp);
  PUT(PRED(bp), NULL);
  PUT(SUCC(bp), NULL);
}
```

​	前空闲就从空闲链表中删除前方空闲块，并且注意分配的头部标记是前一块的头部标记，其余逻辑和 Case2类似

- **Case4:** 前后均非空闲

```c
else{	/* 前后均空闲 */
  size += NEXT_BLKSZ(bp) + PREV_BLKSZ(bp);
  delete_free_block(PREV_BLKP(bp));
  delete_free_block(NEXT_BLKP(bp));
  PUT(HDRP(PREV_BLKP(bp)), PACK(size, FREE));
  PUT(FTRP(NEXT_BLKP(bp)), PACK(size, FREE));
  bp = PREV_BLKP(bp);
  PUT(PRED(bp), NULL);
  PUT(SUCC(bp), NULL);
}
```

​	前两种的结合，不多赘述



### insert_free_block 插入空闲链表

​	插入空闲链表算是一个比较重要的函数，其关乎着空闲块的组织结构，在这里我采用的是**地址排序**的策略。

```c
static void insert_free_block(char *fbp)
{
    // 地址排序 - Address Order
    void *succ = root;
    
    while(SUCC_BLKP(succ)){
        succ = (char *)SUCC_BLKP(succ);
        if((unsigned int)succ >= (unsigned int)fbp){
            // 安装地址顺序插入空闲块
            // PRED_BLKP(succ) <-> fbp <-> succ
            char *tmp = succ;
            succ = (char *)PRED_BLKP(succ);
            PUT(SUCC(succ), fbp);
            PUT(PRED(fbp), succ);
            PUT(SUCC(fbp), tmp);
            PUT(PRED(tmp), fbp);
            #ifdef INDEBUG
                printf("succ(PRE): %p \t tmp(SUCC): %p \t", succ, tmp);
                print_free_list("Insert");
            #endif
            return;
        }
    }
    
    // Base Case & Last Case 
    // 当前大小类无空闲块 或者 在地址分配时当前空闲块地址最大被分配在最后
    PUT(SUCC(succ), fbp);
    PUT(PRED(fbp), succ);
    PUT(SUCC(fbp), NULL);
}
```

​	首先获得目标块的 index，即属于二的几次幂，之后通过 ```global_list_start_ptr``` 加上 index 偏移定位到其属于的大小类链表的 root 指针，如果root指针有指向就进行地址顺序的排序，如果找到后部块地址大于插入块，就把该插入块插到上述块的前部。

​	如果root没有指向，即当前该大小类中没有空闲块，或者按地址序，该块地址大小最大则进行直接的分配在succ之后。



### delete_free_block 删除空闲块

​	删除空闲块要注意，这里常见的bug是和 insert_free_block 一同出现的指针维护不良，导致删除不存在的块，或者访问 nullptr 的前后继以及指针越界问题。

```c
static void delete_free_block(char *fbp)
{
    // NORMAL: GOT A SUCCESSOR AND PREDECESSOR
    if(SUCC_BLKP(fbp) && PRED_BLKP(fbp)){
        PUT(SUCC(PRED_BLKP(fbp)), SUCC_BLKP(fbp));
        PUT(PRED(SUCC_BLKP(fbp)), PRED_BLKP(fbp));
    }
    else if(PRED_BLKP(fbp)){ // LAST BLOCK
        PUT(SUCC(PRED_BLKP(fbp)), NULL);
    }

    PUT(SUCC(fbp), NULL);
    PUT(PRED(fbp), NULL);
}
```

​	正常情况是当前块是链表中间节点，重新连接好前后，把其从链表上脱离即可。如果是最后一个节点，就直接把前继节点的后继指针置为空。最后做好当前删除块的清理工作，把其前后继指针置为NULL



### mm_malloc 分配空闲块

​	mm_malloc 是该lab中的主要函数，用于控制分配内存块的工作。

```c
void *mm_malloc(size_t size)
{
    size_t asize = align_size(size);    /* 调整后的块大小 */
    size_t extendsize;                  /* 扩展堆大小 */
    char *bp;

    /* Trivial Case */
    if(size == 0)
        return NULL;

    /* 寻找适配 */
    if((bp = find_fit(asize, get_index(asize))) != NULL)
        return place(bp, asize);

    /* 未找到适配，分配更多堆空间 */
    extendsize = MAX(asize, CHUNKSIZE);
    if((bp = extend_heap(extendsize)) == NULL)
        return NULL;

    return place(bp, asize);
}
```

​	首先要做好分配大小的对齐，这里定义了一个util函数 align_size 用来对齐块大小。

```c
static size_t align_size(size_t size)
{
    /* 调整块大小 */
    if(size <= DSIZE) return 2*DSIZE;
    else return DSIZE * ((size + (DSIZE) + (DSIZE - 1)) / DSIZE);

    // Code Never Went Here
    return 0;
}
```

​	之后逻辑就是通过 find_fit 在空闲链表中寻找适配，如果没找到适配就进行 heap_extend，每次最小扩展 ```CHUNKSIZE``` bytes，这里我将 ```CHUNKSIZE``` 设为 512

 **(有讲究，如果大于520就会导致realloc2的第一次分配 512 就能够成功，这样之后alloc的块就跟在512块后，就不能成功的将 realloc 的 0 号 block 安排在块位，导致无法通过 extend_heap 来提高性能)**

最后放置空闲块，使用place函数进行分配和分割



### find_fit 寻找适配

​	我使用的是简单的首次适配，即从小到大遍历分离空闲链表，找到第一块适合的空闲块。由于每个空闲链表内部是按地址顺序排列而非大小排列，所以其效果并非严格等同于 best_fit 但是由于大小分块的组织结构，其效果又好于完全不按空间大小排序的适配方式。

```c
static void *find_fit(size_t size, int seg_idx)
{
    // First Fit
    char* res;
    while(seg_idx < SEG_LEN){
        char *root = global_list_start_ptr + seg_idx * WSIZE;
        char *bp = (char *)SUCC_BLKP(root);
        while(bp){
            if((size_t)CRT_BLKSZ(bp) >= size)
                return bp;
            
            bp = (char *)SUCC_BLKP(bp);
        }
        // 在这类中未找到适合，在更大类中寻找
        seg_idx++;
    }
    return NULL;
}
```



### place 分配块

​	分配块这里有说法了，第一层是分配空闲块的时候，如果当前适配的块大小比需要分配的大很多（超出最小空闲块大小 16bytes）那么我们就可以通过分割来减小内部碎片。

​	并且这个分割也是很有讲究，我们可以设计当需要分配的空间较大时例如大于64 bytes，我们就将其分配在空闲块的后部，将前部分割出来作为新的空闲块。如果小于就直接分配在当前空闲块的前部，将后部分割出来作为新的空闲块。这样的组织方式有两方面好处，

- 一方面是其进行了大小分类，有利于块的合并
- 另一方面是对于 realloc2 的测试 trace，我们通过前部切分的方式，使 512 块后再次分配的两 128 块占用前部空间，这样可以使 512 块始终是最后一块即其后继块是结尾块，那么在 realloc 它的时候我们就可以直接通过 extend_heap 达到如此一来可以大大提高内存利用率，**将realloc1、2的util提升至近100%！**

```c
static void *place(char *bp, size_t asize)
{
    size_t blk_size = CRT_BLKSZ(bp);
    size_t rm_size = blk_size - asize;

    if(!GET_ALLOC(HDRP(bp)))
        delete_free_block(bp);
    // 剩余空间大于最小块大小的可分割的情况
    if(rm_size >= 2*DSIZE){
        // 当块大小大于 64 时将其有效载荷放在空闲块后部，前部切分出来作为空闲块
        if(asize > 64){
            PUT(HDRP(bp), PACK(rm_size, FREE));
            PUT(FTRP(bp), PACK(rm_size, FREE));
            PUT(HDRP(NEXT_BLKP(bp)), PACK(asize, ALLOCATED));
            PUT(FTRP(NEXT_BLKP(bp)), PACK(asize, ALLOCATED));
            coalesce(bp);
            return NEXT_BLKP(bp);
        }
        else{
            PUT(HDRP(bp), PACK(asize, ALLOCATED));
            PUT(FTRP(bp), PACK(asize, ALLOCATED));
            PUT(HDRP(NEXT_BLKP(bp)), PACK(rm_size, FREE));
            PUT(FTRP(NEXT_BLKP(bp)), PACK(rm_size, FREE));

            coalesce(NEXT_BLKP(bp));
        }
    }
    // 不可分割情况
    else{
        PUT(HDRP(bp), PACK(blk_size, ALLOCATED));
        PUT(FTRP(bp), PACK(blk_size, ALLOCATED));
    }
    return bp;
}
```



### mm_free 释放块

​	直接设置空闲，并释放同时合并，没什么好说的

```c
void mm_free(void *ptr)
{
    #ifdef DEBUG
        printf("Freeing.....\n");
    #endif
    char *bp = ptr;
    size_t size = CRT_BLKSZ(bp);

    PUT(HDRP(bp), PACK(size, FREE));
    PUT(FTRP(bp), PACK(size, FREE));
    coalesce(bp);
}
```



### mm_realloc 重分配块

​	mm_realloc 能否做好是分数能否上 90 的关键，其主要策略有两个

- **空闲块融合**

  一在重分配的时候，如果后方有空闲块可以进行融合，再看空间够不够，如果够了就不用释放再分配了。

  （同时前融合也应当有相应的效果，前融合要注意内部载荷数据的移动，但其实观察 trace 文件下的block组织表现，发现其实前融合很少甚至没有，对性能影响不大，之后便在代码中删除了）

- **尾部堆扩展**

  就是之前提到的如果要重分配的块是尾部块就执行 extend_heap 就行了，不需要释放再分配。同时注意到了 trace 文件中反复 realloc 首次分配的块，于是和 place 中提到的策略相互结合可以达到将首次分配的块移动到末尾的效果。

其余就是一些基础写法，在注释中已经体现，还有需要注意一下**分配大小的对齐**和特殊情况

```c
void *mm_realloc(void *ptr, size_t size)
{
    // 如果 ptr == NULL 直接分配
    if(ptr == NULL)    
        return mm_malloc(size);
    // 如果 size == 0 就释放
    else if(size == 0){
        mm_free(ptr);
        return NULL;
    }
    size_t asize = align_size(size), old_size = CRT_BLKSZ(ptr);
    size_t mv_size = MIN(asize, old_size);
    char *oldptr = ptr;
    char *newptr;

    if(old_size == asize)
        return ptr;
    
    size_t prev_alloc =  GET_ALLOC(FTRP(PREV_BLKP(ptr)));
    size_t next_alloc =  GET_ALLOC(HDRP(NEXT_BLKP(ptr)));
    size_t next_size = NEXT_BLKSZ(ptr);
    char *next_bp = NEXT_BLKP(ptr);
    size_t total_size = old_size;

    if(prev_alloc && !next_alloc && (old_size + next_size >= asize)){    // 后空闲  
        total_size += next_size;
        delete_free_block(next_bp);
        PUT(HDRP(ptr), PACK(total_size, ALLOCATED));
        PUT(FTRP(ptr), PACK(total_size, ALLOCATED));
        place(ptr, total_size);
    }
    else if(!next_size && asize >= old_size){   // 如果后部是结尾块，则直接 extend_heap
        size_t extend_size = asize - old_size;
        if((long)(mem_sbrk(extend_size)) == -1)
            return NULL; 
        
        PUT(HDRP(ptr), PACK(total_size + extend_size, ALLOCATED));
        PUT(FTRP(ptr), PACK(total_size + extend_size, ALLOCATED));
        PUT(HDRP(NEXT_BLKP(ptr)), PACK(0, ALLOCATED)); 
        place(ptr, asize);
    }
    else{   // 直接分配
        newptr = mm_malloc(asize);
        if(newptr == NULL)
            return NULL;
        memcpy(newptr, ptr, MIN(old_size, size));
        mm_free(ptr);
        return newptr;
    }
    return ptr;
}
```



## 关于DEBUG

​	代码中为了 DEBUG 定义了大量 debug util 函数和 Error Handler，如果想清晰的看清楚堆块的组织结构，调用它们是很有帮助的。还有 debug 要善用 gdb...



## 四、实验结果

![](https://tva1.sinaimg.cn/large/008i3skNgy1gqqcdfltwxj309s088aal.jpg)

​	在不使用BST和全局数据结构的情况下达到了 97/100 的分数，还不错。



## 五、结语

​	这个Lab用了我2、3天的时间，是比较难的，需要用心 DEBUG 考验 gdb的使用。Malloc Lab 还是很好玩的，ddl之后我可能会考虑进一步优化，采用BST结构尽量做到接近 100/100
