---
layout: post
title: 简单的malloc堆内存分配实现
author: hawker
permalink: /2016/07/malloc-heap.html
date: 2016-07-15 22:00:00
category:
    - 编程
tags:
    - cpp
    - malloc
    - heap
---
操作系统中程序可操作的内存主要有两块，一个是栈存储空间，另一个是堆内存空间。C/C++中与后者直接挂钩的接口是malloc，大家对这个库函数都非常熟悉，需要一块堆内存时调用它。大家用了很多，但是很少去了解malloc背后堆内存分配的实际流程。坦白说，malloc只是C的标准库中提供的一个普通函数，结合ELF和操作系统原理实现malloc的基本思想并不复杂。

本文通过一种简单的机制实现malloc，虽然不高效，但是能够浅显的说明C程序堆内存分配背后的故事。相关的代码可以参见：[传送门](https://github.com/hawkxiang/Berkeley-CS162/tree/master/hw3)

### 一、Linux程序的内存结构

大家或多或少两件ELF载入内存之后的程序内存布局，在详细说明malloc之前，我们回归一下相关知识。

##### 1.1 虚拟内存地址和物理内存地址
可以打趣的说计算机领域有一句金句：没有虚拟化解决不了的问题，如果有再加一层虚拟化。为了简单，现代操作系统在处理内存地址时，普遍采用虚拟内存地址技术。即在汇编程序层面，采用虚拟内存地址技术，每个进程角度都是独享整个机器最大的内存空间。

这种虚拟地址空间的作用主要是简化程序的编写及方便操作系统对进程间内存的隔离管理，真实中的进程不太可能（也用不到）如此大的内存空间，实际能用到的内存取决于物理内存大小。

当然，虚拟内存肯定是要映射到实际的物理内存地址。目前，主要采用内存页机制，详细的映射说明可以查看操作系统叙述。

##### 1.2 内存排布
那么在一个程序中，实际的虚拟内存布局是什么样呢？以Linux64位系统为例。理论上，64bit内存地址可用空间为0x0000000000000000 ~ 0xFFFFFFFFFFFFFFFF，这是个相当庞大的空间，Linux实际上只用了其中一小部分（256T）。实际用到的地址为空间为0x0000000000000000 ~ 0x00007FFFFFFFFFFF和0xFFFF800000000000 ~ 0xFFFFFFFFFFFFFFFF，其中前面为用户空间（User Space），后者为内核空间（Kernel Space）。图示如下：
![Alt text](/upload/2016/11/memory.jpg "内存布局")

对于用户来说，可见的主要是User Space，Kernel Space主要是操作系统拥有权限，user -> kernel一般通过系统调用这种软中断实现。详细看看User Space分为的几段内容。

1. 静态代码段：这里是低地址空间，主要存放只读的代码段，一般不会从0开始地址空间。
2. 数据段：这里主要存放初始化后的全局变量。
3. BSS：这里主要存放未初始化的全局变量，实际并不占空间。
4. Heap：堆内存，自低向高地址增加，也是本文重点说明的地方。
5. Mapping Area：这里是与mmap系统调用相关的区域。大多数实际的malloc实现会考虑通过mmap分配较大块的内存区域，本文不讨论这种情况。这个区域自高地址向低地址增长。
6. Stack：函数栈区，自高地址向低地址增长，其上方一般是内核地址；相关的指针是esp和ebp。

##### 1.3 Heap内存模型
大家的一个错误理解是heap的内存已经分配好了，程序使用时只需要取一段就好。其实，并不是这样。Linux维护一个break指针，这个指针指向堆空间的某个地址。从堆起始地址到break之间的地址空间为映射好的，可以供进程访问；而从break往上，是未映射的地址空间，如果访问这段空间则程序会报错。所以，我们可以简单的理解，heap的内存也是要建立虚拟空间到实际内存的关系，当已分配的内存不满足程序要求时，需要移动break指针申请新的内存，这里其他需要进行内核。涉及的函数接口是brk和sbrk。两个系统调用的原型如下：

{% highlight C linenos%}
int brk(void *addr);
void *sbrk(intptr_t increment);
{% endhighlight %}
brk将break指针直接设置为某个地址，而sbrk将break从当前位置移动increment所指定的增量。brk在执行成功时返回0，否则返回-1并设置errno为ENOMEM；sbrk成功时返回break移动之前所指向的地址，否则返回(void *)-1。一个小技巧是，如果将increment设置为0，则可以获得当前break的地址。

另外需要注意的是，由于Linux是按页进行内存映射的，所以如果break被设置为没有按页大小对齐，则系统实际上会在最后映射一个完整的页，从而实际已映射的内存空间比break指向的地方要大一些。但是使用break之后的地址是很危险的（尽管也许break之后确实有一小块可用内存地址）。

### 二、 实现malloc
其实，malloc的实现有多种方式：空闲链表、位图map、内存池等。其中空闲链表最为简单，同时也容易让我们了解malloc功能原理，后两者效率更高。本文通过空闲链表技术，来阐述malloc的实现流程。

##### 2.1 基本区块结构
空闲链表的思想比较通俗易懂，我们将堆内存以多个块（Block）的形式进行组织，每个块以链表的形式串起来。那么这里有一个问题，链表块本身获取本块的大小是一个难题；因此将每个块分为meta区和数据区，meta区记录数据块的元信息（数据区大小、空闲标志位、指针等等），数据区是真实分配的内存区域，并且数据区的第一个字节地址即为malloc返回的地址。可以定义block结构体如下：

{% highlight C++ linenos%}
//data meta
struct s_block {
    size_t size;   /* 数据区大小 */
    t_block next;  /* 指向下个块的指针 */
    t_block prev;  /* 指向上个块的指针 */
    int free;      /* 是否是空闲块 */
    int padding;   /* 填充4字节，保证meta块长度为8的倍数 */
    void *ptr;     /* 数据段地址，配合进行地址合法性验证 */
    char data[1];  /* 这是一个虚拟字段，表示数据块的第一个字节，长度不应计入meta */
};
{% endhighlight %}

##### 2.2 寻找候选block
调用malloc时，如果选择合适的block返回内存呢？一般来说两种find算法：

1. First fit: 从头开始找，使用第一个数据区大小大于要求size的块所谓此次分配的块
2. Best fit: 从头开始，遍历所有块，使用数据区大小大于size且差值最小的块作为此次分配的块

前者效率高；后者的内存使用率好，避免内存碎片。本文主要是说明原理，效率上不作太多深究，而且实际的malloc不会使用空闲链表技术，因此本文使用First fit。基本代码如下：

{% highlight C++ linenos%}
//First fit
t_block find_block(t_block *last, size_t s) {
    t_block b = base;
    while (b && !(b->free && b->size >= s)) {
        *last = b;
        b = b->next;
    }
    return b;
};
{% endhighlight %}
find_block从空闲链表头开始查找，第一个符合size要求的block返回它的起始地址，如果找不到这返回NULL。这里在遍历时会更新一个叫last的指针，这个指针始终指向当前遍历的block。这是为了如果找不到合适的block而开辟新block使用的，具体会在接下来的一节用到。

##### 2.3 创建block开辟新内存
如果，空闲链表中所有的block都不满足malloc要求的size，需要抵用sbrk申请内存，需要在链表最后开辟一个新的block。创建的流程如下：

{% highlight C++ linenos%}
t_block extend_heap(t_block last, size_t s) {
    t_block brk = (t_block)sbrk(0);
    if (sbrk(BLOCK_SIZE + s) == (void*)-1)
        return NULL;
    brk->size = s;
    brk->next = NULL;
    if (last) 
        last->next = brk;
    brk->prev = last;
    brk->free = 0;
    brk->ptr = brk->data;
    return brk;
}
{% endhighlight %}
这里首先获取break指针目前的位置，分配size+meta大小的一块内存，将新分配的block串入空闲链表，用的last我们在上一节已经提及。

##### 2.4 分裂block
First_fit有一个比较致命的缺点，就是可能会让很小的size占据很大的一块block，此时，为了提高payload，应该在剩余数据区足够大的情况下，将其分裂为一个新的block，示意如下：
![Alt text](/upload/2016/11/split.jpg "block分裂")

实现分块的代码如下：

{% highlight C++ linenos%}
void split_block(t_block b, size_t s) {
    t_block new_b = (t_block)(b->data + s);
    new_b->size = b->size - BLOCK_SIZE - s;
    new_b->next = b->next;
    new_b->prev = b;
    new_b->free = 1;
    new_b->ptr = new_b->data;
    b->size = s;
    b->next = new_b;
    if (new_b->next)
        new_b->next->prev = new_b;
}
{% endhighlight %}

##### 2.5 malloc的实现
有了上面的工具代码，我们可以利用它们实现一个简单的malloc，提供分配heap内存的功能。内存对齐是一个必须的要求，malloc分配的block数据区按8字节对齐，所以在size不为8的倍数时，我们需要将size调整为大于size的最小的8的倍数：

{% highlight C++ linenos%}
size_t align8(size_t s) {
    if ((s & 0x7) == 0) return s;
    return ((s >> 3) + 1) << 3;
}
{% endhighlight %}
malloc的接口实现如下：

{% highlight C++ linenos%}
void *mm_malloc(size_t size) {
    /* YOUR CODE HERE */
    t_block b, last;
    size_t s = align8(size);
    if (base) {
        last = base;
        b = find_block(&last, s);
        if (b) {
            if ((b->size - s) >= BLOCK_SIZE+8)
                split_block(b, s);
            b->free = 0;
        } else {
            b = extend_heap(last, s);
            if (!b) return NULL;
        }
    } else {
        b = extend_heap(NULL, s);
        if (!b) return NULL;
        base = b;
    }
    return b->data;
}
{% endhighlight %}
这里的`base`是空闲链表的头部，如果NULL很明显是需要直接分配一个block的。

##### 2.6 calloc的实现
calloc与malloc的区别很明显，只需要在malloc分配的内存上，进行初始化为0即可。由于我们的数据区是按8字节对齐的，所以为了提高效率，我们可以每8字节一组置0，而不是一个一个字节设置。我们可以通过新建一个size_t指针，将内存区域强制看做size_t类型来实现。

{% highlight C++ linenos%}
void *mm_calloc(size_t number, size_t size) {
    size_t *new_b;
    size_t s8, i;
    new_b = (size_t*)mm_malloc(number * size);
    if (new_b) {
        s8 = align8(number*size) >> 3;  //有多少个size_t的意思_
        for (i = 0; i < s8; i++)
            new_b[i] = 0;
    }
    return new_b;
}
{% endhighlight %}

##### 2.7 free的实现
有内存分配，就需要内存释放的free函数。我们主要需要解决两个问题：

1. 如何验证所传入的地址是有效地址，即是通过malloc方式分配的block中数据区首地址
2. 如何解决碎片问题

首先我们要保证传入free的地址是有效的，这个有效包括两方面：

1. 地址应该在之前malloc所分配的区域内，即在first_block和当前break指针范围内
2. 这个地址确实是之前通过我们自己的malloc分配的

这里的解决方案是block的meta区实际存储data块的开始指针，即ptr的作用。你可能会问data[1]也是数据块的开始指针，为什么需要冗余变量。注释已经说了data[1]其实并没有存储在meta的中，因此我们需要一个变量存储这个指针变量。这点可以通过meta的大小明确`const int BLOCK_SIZE = sizeof(struct s_block) - sizeof(char*);`：

接下来，我们定义一个检查free传入的地址合法性的函数：

{% highlight C++ linenos%}
t_block get_block(void* p) {
    char *tmp = (char*)p;
    p = (tmp -= BLOCK_SIZE);
    return (t_block)p;
}

int valid_addr(void *p) {
    if (base) {
        if (p > (void*)base && p < sbrk(0)){
            void *tmp = get_block(p)->ptr;
            return p == tmp;
        }
    }
    return 0;
}
{% endhighlight %}

接下来的只需要解决太多空闲块的问题即可，一个简单的解决方式是当free某个block时，如果发现它相邻的block也是free的，则将block和相邻block合并。

{% highlight C++ linenos%}

t_block fusion(t_block b) {
    if (b->next && b->next->free) {
        b->size += b->next->size + BLOCK_SIZE;
        b->next = b->next->next;
        if (b->next)
            b->next->prev = b;
    }
    return b;
}
{% endhighlight %}

有了上述方法，free的实现思路就比较清晰了：首先检查参数地址的合法性，如果不合法则不做任何事；否则，将此block的free标为1，并且在可以的情况下与后面的block进行合并。如果当前是最后一个block，则回退break指针释放进程内存，如果当前block是最后一个block，则回退break指针并设置first_block为NULL。实现如下：

{% highlight C++ linenos%}
void mm_free(void *ptr) {
    /* YOUR CODE HERE */
    t_block b;
    if (valid_addr(ptr)) {
        b = get_block(ptr);
        b->free = 1;
        if (b->prev && b->prev->free)
            fusion(b->prev);
        if (b->next)
            fusion(b);
        else {
            if (b->prev)
                b->prev->next = NULL;
            else
                base = NULL;
            brk(b);
        }
    }
}
{% endhighlight %}

##### 2.8 realloc的实现
为了实现realloc，需要一个是心啊内存复制的方案，基本逻辑如下：

{% highlight C++ linenos%}
void copy_block(t_block src, t_block dst) {
    size_t *sdata, *ddata, i;
    sdata = (size_t*)src->ptr;
    ddata = (size_t*)dst->ptr;
    for (i = 0; (i*8) < src->size && (i*8) < dst->size; i++)
        ddata[i] = sdata[i];
}
{% endhighlight %}
它的主要功能以8字节为单位将内容拷贝到新的内存位置。

realloc的实现如下：

{% highlight C++ linenos%}
	size_t s;
    t_block b, new_b;
    void *new_ptr;
    if (!ptr) return mm_malloc(size);
    if (valid_addr(ptr)) {
        s = align8(size);
        b = get_block(ptr);
        if (b->size >= s) {
            if (b->size - s >= (BLOCK_SIZE + 8))
                split_block(b, s);
        } else {
            if (b->next && b->next->free &&
                     (b->next->size + b->size + BLOCK_SIZE) >= s){
                fusion(b);
                if (b->size - s >= (BLOCK_SIZE + 8))
                    split_block(b, s);
            } else {
                new_ptr = mm_malloc(s);
                if (new_ptr) return NULL;
                new_b = get_block(new_ptr);
                copy_block(b, new_b);
                free(b);
                return new_ptr;
            }
        }
        return ptr;
    }
    return NULL;
{% endhighlight %}
整体的逻辑如下：
1. 如果当前的block的数据区还有剩余空间，即可以满足realloc要求的size，只需要考虑是否需要split。
2. 如果当前block的数据区不能满足size，但是其后继block是free的，并且合并后可以满足，则考虑做合并
3. 如果都不新，只能分配一个新的block，进行新旧内存的copy操作。

### 三、参考

本文参考了[A malloc Tutorial](http://www.inf.udec.cl/~leo/Malloc_tutorial.pdf)，引用了其中的相关内容。
