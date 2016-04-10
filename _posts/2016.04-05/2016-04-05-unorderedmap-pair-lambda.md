---
layout: post
title: C++中unordered_map如何使用pair作键值(key)
author: hawker
permalink: /2015/12/unorderedmap-pair-funtor.html
date: 2016-04-5 23:00:00
category:
    - 编程
tags:
    - cpp
    - lambda
    - stl
---
C++ STL中的unordered_map底层是通过Hash实现的，当使用pair作为键值(Key)时，需要手动传入Hash实例类型。

### 函数对象的实现方式

参考网上的解决方案，通过一个函数对象向unordered_map传递Hash实例类型。具体实现如下面的代码：

{% highlight C++ linenos%}
#include <unordered_map>
using namespace std;
struct hashfunc {
	template<typename T, typename U>
	size_t operator()(const pair<T, U> &i) const {
		return hash<T>()(x.first) ^ hash<U>()(x.second);
	}
};

int main() {
	unordered_map<pair<int, int>, int, hashfunc> func_map;
	func_map[make_pair(1, 2)]++;
	return 0;
}
{% endhighlight %}
在这中解决方案中，我们创建了一个函数对象类`hashfunc`，并将它作为`unordered_map`的Hash实例类型传入，成功构造以pair为Key的无序容器。

### Lambda如何实现呢？
Lambda是C++的一种新特性，既然可以通过函数对象实现上面的功能。自然，我们会联想用lambda表达式去实现上面的功能。最初的实现想法：

{% highlight C++ linenos%}
int main() {
	auto hashlambda = [](const pair<int, int>& i) -> size_t{return i.first ^ i.second;};
	unordered_map<pair<int, int>, int, decltype(hashlambda)> lam_map;
	lam_map[make_pair(1, 2)]++;
	return 0;
}
{% endhighlight %}
进行编译出错，提示“lambda默认构造函数是删除的(deleted)”。为什么会这样，同样是可调用对象，为何lambda实现时无法通过编译呢？

自己一翻折腾并大牛的热心帮助下终于有所明白，简单说来，`unordered_map`继承自_Hash类型，_Hash用到一个_Uhash_compare类来封装传入的hash函数，如果`unordered_map`构造函数没有显示的传入hash函数实例引用，则`unordered_map`默认构造函数使用第三个模板参数指定的Hash类型的默认构造函数，进行hash函数实例的默认构造。在第一种情况中，编译为函数类型合成默认构造函数也就是hash_fun()，所以我们在定义`unordered_map`时即使不传入函数对象实例，也能通过默认构造函数生成。但是，对于lambda对象来说，虽然编译时会为每个lambda表达式产生一个匿名类，但是这个匿名类时***不含有默认构造函数(=deleted)***。因此，如果实例化`unordered_map`时，不传入lambda对象实例引用，默认构造函数不能为我们合成一个默认的hash函数实例。所以，编译时产生了上面的错误。明白了这些，自然知道如何去修改了。

{% highlight C++ linenos%}
int main() {
	auto hashlambda = [](const pair<int, int>& i) -> size_t{return i.first ^ i.second;};
	unordered_map<pair<int, int>, int, decltype(hashlambda)> lam_map(10, hashlambda);
	lam_map[make_pair(1, 2)]++;
	return 0;
}
{% endhighlight %}
我们在创建`unordered_map`对象时，手动指定了两个参数；第一参数是“桶”的数量，第二个就是hash实例引用了。在这里需要留意的是，lambda虽然与函数类型功能相似，但在构造函数、赋值运算符、默认析构函数的限制是不同的。此外，还应该留意每个lambda表达都是一种类型，就是值引用、参数类型，返回值都一样，但是它们的类型是不同的。

### Functional可以通过编译，但存在运行时错误

针对上面lambda功能实现时存在的编译错误，有一种方法也可以避免编译出错。用function对象来保存lambda表达式：

{% highlight C++ linenos%}
int main() {
	function<size_t (const pair<int, int>&)> hashfuna = [](const pair<int, int>& i) -> size_t{return i.first ^ i.second;};
	unordered_map<pair<int, int>, int, decltype(hashfuna)> lam_map;
	lam_map[make_pair(1, 2)]++;
	return 0;
}
{% endhighlight %}
可以发现，此时编译一切正常，但执行时报错。这是为什么呢？其实结合上面的解释，结论也很显然了。编译出错是因为我们指定的模版Hash类型，无法默认构造实例；但是用function保存lambda表达式后，这个function对象对映的类型是`function<size_t (const pair<int, int>&)>`，它是有默认构造函数的，故而`unordered_map`可以默认实例化成功，编译正确。但是，`function<size_t (const pair<int, int>&)>`的默认构造函数只会构造一个空的function，所以我们还是要如对待lambda对象那样，手动传入function对象引用（hashfuna）。

{% highlight C++ linenos%}
unordered_map<pair<int, int>, int, decltype(hashfuna)> lam_map(10, hashfuna);
{% endhighlight %}
你可能会奇怪为何`function<size_t (const pair<int, int>&)>`只构造了空的function对象呢，其实这也很显然，functional只是为了通用的存储“可调用对象”，所以它只能默认构造为空function。

{% highlight C++ linenos%}
function<size_t (const pair<int, int>&)> a = [](const pair<int, int>& i) -> size_t{return i.first ^ i.second;};

function<size_t (const pair<int, int>&)> b = [](const pair<int, int>& i) -> size_t{return 5;};

function<size_t (const pair<int, int>&)> c = 合法的函数指针;	
{% endhighlight %}
几种可调用对象的不同之处希望对大家有所帮助，Lambda类型的解释也希望对大家有所启发！
