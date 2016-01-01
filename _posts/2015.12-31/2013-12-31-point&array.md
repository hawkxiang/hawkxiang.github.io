---
layout: post
title: C语言中指针与数组内存分配
date: 2015-02-01 23:00:00
category: C/C++
tags: c
---
######年末岁初，未出门远行；一个人待着，感觉应该写的什么。想起自己前不久新搭建的博客，希望自己在博客中留下学习生活的点滴，更希望自己能够坚持写下去，也算做是对自己新的一年的寄语与期盼。

进入主题，前些日子学习语言时，遇到解释`C`结构体中指针与数组内存分配的解释。进而自己思考很多，同时也查资料验证了一些想法，总结如下。

&nbsp;

###指针与数组的区别
还是，先说说指针与数组的区别吧！看一段代码
{% highlight c %}
    #include <stdio.h>
    int main(){
        char* ptr = "aaaa";
        char arr[5] = "bbbb";
        printf("直接访问指针: %x\n",ptr);
        printf("取直接地址: %x\n",&ptr);
        printf("直接访问数组: %x\n",arr);
        printf("取数组地址: %x\n",&arr);
        return 0;
    }
{% endhighlight %}
其输出的结果是：

{% highlight bash %}
    直接访问指针: 400604
    取直接地址: fb4008d8
    直接访问数组: fb4008d0
    取数组地址: fb4008d0
{% endhighlight %}
显然对于数组来说，数组名`arr`与`&arr`是一样都是变量地址。但是直接使用指针变量，实际是操作指针变量所存储的内存地址;"&ptr"才是指针变量自己的地址。
指针与数组这种差异，通过GDB查看汇编代码可以更加清晰的了解。

对于数组arr直接操作，汇编代码使用`lea`指令：{% highlight bash %}lea    -0x10(%rbp),%rax{% endhighlight %}

对于指针ptr直接操作，汇编代码使用`mov`请指令：{% highlight bash %}mov    -0x8(%rbp),%rax{% endhighlight %}

LEA指令的功能是将存储单元的有效地址（偏移地址）传送到目的操作数；MOV指令将变量中的值放到新的寄存器中。所以访问数组名其实就是访问数组的地址，而访问指针实质是获取它保存的变量。

&nbsp;

###结构体中成员地址
首先，说一说结构体中成员变量的内存地址分配。所谓变量，其实就是内存地址的一个抽象名字罢了。机器只知道数字的地址，变量命名是为了可读性。
用一个简单的例子来看看，结构体中变量的地址。
{% highlight c %}
    #include <stdio.h>
    struct memoryassign{
        int a;
        char* p;
        short b;
    }
    int main(){
        struct memoryassign test;
        return 0;
    }
{% endhighlight %}
通过`gdb`调试工具来看下结构相关变量的内存地址。
{% highlight bash %}
    (gdb) p test
    $1 = {a = 4195536, p = 0x4003c0 <_start> "1\355I\211\321^H\211\342H\203\344\360PTI\307\300@\005@", b = -7520}
{% endhighlight %}
我们发现`test`的成员变量被编译器进行了初始化，但是都是一些奇怪的值。由此，明白显示初始化的必要性，它能有效的避免一些bug。
{% highlight bash %}
    (gdb) p &test
    $2 = (struct memory *) 0x7fffffffe1a0
    (gdb) p &(test.a)
    $3 = (int *) 0x7fffffffe1a0
    (gdb) p &(test.p)
    $4 = (char **) 0x7fffffffe1a8
    (gdb) p &(test.b)
    $5 = (short *) 0x7fffffffe1b0
{% endhighlight %}
接着，查看各个变量对应的内存地址。显然，结构体的内存地址与其第一个成员变量的地址相同。结构体中的变量内存地址是连续的，说明结构体的内存是连续分配的。
结构体中的变量访问其实就是结构体基地址加上相应的偏移量。此外，所谓的指针变量它就是一个普通的变量类型，大小与`int`相同，只不过它代表的内存中存储了另一个数据的内存地址。

