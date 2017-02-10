---
layout: post
title: Go语言内存模型之Channel
author: hawker
permalink: /2016/10/gomem-channel.html
date: 2016-10-14 18:00:00
category:
    - 编程
tags:
    - Golang
    - Memory Model
---
Golang的内存模型比较复杂，同时也非常重要。理解Go的内存模型会就可以明白很多奇怪的竞态条件问题，"The Go Memory Model"的原文在[这里](https://golang.org/ref/mem)，读个四五遍也不算多。

这里并不是要翻译这篇文章，英文原文是精确的，但读起来却很晦涩，尤其是happens-before的概念本身就是不好理解的，很容易跟时序问题混淆。大多数读者第一遍读Go的内存模型时基本上看不懂它在说什么。所以我要做的事情用不怎么精确但相对通俗的语言解释一下。

先用一句话总结，Go的内存模型描述的是"在一个groutine中对变量进行读操作能够侦测到在其他goroutine中对该变量的写操作"的条件。

## 内存模型相关bug一例

为了证明理解内存模型的重要性，先看一个例子。下面一小段代码：

{% highlight go linenos %}
package main

import (
  "sync"
  "time"
)

func main() {
  var wg sync.WaitGroup
  var count int
  var ch = make(chan bool, 1)
  for i := 0; i < 10; i++ {
    wg.Add(1)
    go func() {
      ch <- true
      count++
      time.Sleep(time.Millisecond)
      count--
      <-ch
      wg.Done()
    }()
  }
  wg.Wait()
}
{% endhighlight %}

以上代码有没有什么问题？这里把buffered channel作为semaphore来使用，表面上看最多允许一个goroutine对count进行++和--，但其实这里是有bug的。根据Go语言的内存模型，对count变量的访问并没有形成临界区。编译时开启竞态检测可以看到这段代码有问题：

{% highlight bash linenos %}
go run -race test.go
{% endhighlight %}

编译器可以检测到16和18行是存在竞态条件的，也就是count并没像我们想要的那样在临界区执行。继续往下看，读完本文，回头再来看就可以明白为什么这里有bug了。bug解释：Golang的内存模型只保证15行Channel写一定happens-before19行Channel读（即使在不同Goroutine之间），但是不同Goroutine的16-18行之间根本没保证happens-before关系。因此编译器实际如果排布不同Goroutine间16-18行的代码执行没法保证，因此，上面代码没有形成临界代码段。

## happens-before术语

happens-before是编程语言中的一种术语，并非Golang独有。简单的来说，通常定义如下：

假设A和B表示一个多线程的程序执行的两个操作。如果A happens-before B，那么A操作对内存的影响 将对执行B的线程(且执行B之前)可见。

无论使用哪种编程语言，有一点是相同的：如果操作A和B在相同的线程中执行，并且A操作的声明在B之前，那么A happens-before B。

{% highlight go linenos %}
int A, B;
void foo()
{
  // This store to A ...
  A = 5;
  // ... effectively becomes visible before the following loads. Duh!
  B = A * A;
}
{% endhighlight %}

还有一点是，在每门语言中，无论你使用那种方式获得，happens-before关系都是可传递的：如果A happens-before B，同时B happens-before C，那么A happens-before C。当这些关系发生在不同的线程中，传递性将变得非常有用。

刚接触这个术语的人总是容易误解，这里必须澄清的是，happens-before并不是指时序关系，并不是说A happens-before B就表示操作A在操作B之前发生。它就是一个术语，就像光年不是时间单位一样。具体地说：

1. `A` happens-before `B`并不意味着时序上`A`在`B`之前发生。
2. 时序上`A`在`B`之前发生并不意味着`A` happens-before `B`。

这两个陈述看似矛盾，其实并不是。如果你觉得很困惑，可以多读几篇它的定义。后面我会试着解释这点。记住，happens-before 是一系列语言规范中定义的操作间的关系。它和时间的概念独立。这和我们通常说”A在B之前发生”时表达的真实世界中事件的时间顺序不同。

## `A` happens-before `B`并不意味着时序上`A`在`B`之前发生
这里有个例子，其中的操作具有happens-before关系，但是实际上并不一定是按照那个顺序发生的。下面的代码执行了(1)对A的赋值，紧接着是(2)对B的赋值。

{% highlight go linenos %}
int A = 0;
int B = 0;
void main()
{
    A = B + 1; // (1)
    B = 1; // (2)
}
{% endhighlight %}

根据前面说明的规则，(1) happens-before (2)。但是，如果我们使用gcc -O2编译这个代码，编译器将产生一些指令重排序。有可能执行顺序是这样子的：

{% highlight linenos %}
将B的值取到寄存器
将B赋值为1
将寄存器值加1后赋值给A
{% endhighlight %}

也就是到第二条机器指令(对B的赋值)完成时，对A的赋值还没有完成。换句话说，(1)并没有在(2)之前发生!
那么，这里违反了happens-before关系了吗？让我们来分析下，根据定义，操作(1)对内存的影响必须在操作(2)执行之前对其可见。换句话说，对A的赋值必须有机会对B的赋值有影响.+

但是在这个例子中，对A的赋值其实并没有对B的赋值有影响。即便(1)的影响真的可见，(2)的行为还是一样。所以，这并不能算是违背happens-before规则。

###  时序上`A`在`B`之前发生并不意味着`A` happens-before `B`
下面这个例子中，所有的操作按照指定的顺序发生，但是并能不构成happens-before 关系。假设一个线程调用pulishMessage，同时，另一个线程调用consumeMessage。 由于我们并行的操作共享变量，为了简单，我们假设所有对int类型的变量的操作都是原子的。

{% highlight go linenos %}
int isReady = 0;
int answer = 0;
void publishMessage()
{
  answer = 42; // (1)
  isReady = 1; // (2)
}
void consumeMessage()
{
  if (isReady) // (3) <-- Let's suppose this line reads 1
  printf("%d\n", answer); // (4)
}
{% endhighlight %}

根据程序的顺序，在(1)和(2)之间存在happens-before 关系，同时在(3)和(4)之间也存在happens-before关系。
除此之外，我们假设在运行时，isReady读到1(是由另一个线程在(2)中赋的值)。在这中情形下，我们可知(2)一定在(3)之前发生。但是这并不意味着在(2)和(3)之间存在happens-before 关系!+

happens-before 关系只在语言标准中定义的地方存在，这里并没有相关的规则说明(2)和(3)之间存在happens-before关系，即便(3)读到了(2)赋的值。
还有，由于(2)和(3)之间，(1)和(4)之间都不存在happens-before关系，那么(1)和(4)的内存交互也可能被重排序 (要不然来自编译器的指令重排序，要不然来自处理器自身的内存重排序)。那样的话，即使(3)读到1，(4)也会打印出“0“。

## Golang的同步规则
我们回过头来看看"The Go Memory Model"中关于happens-before的说明。

如果满足下面条件，对变量**v**的读操作<em>r</em>可以侦测到对变量v的写操作<em>w</em>：
1. <em>r</em> does not happen before <em>w</em>.
2. There is no other write <em>w</em> to **v** that happens after <em>w</em> but before <em>r</em>.

为了保证对变量**v**的读操作<em>r</em>可以侦测到某个对**v**的写操作<em>w</em>，必须确保w是<em>r</em>可以侦测到的唯一的写操作。也就是说当满足下面条件时可以保证读操作<em>r</em>能侦测到写操作<em>w</em>：
1. <em>w</em> happens-before <em>r</em>.
2. Any other write to the shared variable **v** either happens-before <em>w</em> or after <em>r</em>.

关于channel的happens-before在Go的内存模型中提到了三种情况：
1. 对一个channel的发送操作 happens-before 相应channel的接收操作完成
2. 关闭一个channel happens-before 从该Channel接收到最后的返回值0
3. 不带缓冲的channel的接收操作 happens-before 相应channel的发送操作完成

先看一个简单的例子

{% highlight go linenos %}
var c = make(chan int, 10)
var a string
func f() {
    a = "hello, world"  // (1)
    c <- 0  // (2)
}
func main() {
    go f()
    <-c   // (3)
    print(a)  // (4)
}
{% endhighlight %}

上述代码可以确保输出"hello, world"，因为(1) happens-before (2)，(4) happens-after (3)，再根据上面的第一条规则(2)是 happens-before (3)的，最后根据happens-before的可传递性，于是有(1) happens-before (4)，也就是a = "hello, world" happens-before print(a)。

再看另一个例子：

{% highlight go linenos %}
var c = make(chan int)
var a string
func f() {
    a = "hello, world"  // (1)
    <-c   // (2)
}
func main() {
    go f()
    c <- 0  // (3)
    print(a)  // (4)
}
{% endhighlight %}

根据上面的第三条规则(2) happens-before (3)，最终可以保证(1) happens-before (4)。

如果我把上面的代码稍微改一点点，将c变为一个带缓存的channel，则print(a)打印的结果不能够保证是"hello world"。

{% highlight go linenos %}
var c = make(chan int, 1)
var a string
func f() {
    a = "hello, world"  // (1)
    <-c   // (2)
}
func main() {
    go f()
    c <- 0  // (3)
    print(a)  // (4)
}
{% endhighlight %}

因为这里不再有任何同步保证，使得(2) happens-before (3)。可以回头分析一下本节最前面的例子，也是没有保证happens-before条件。
