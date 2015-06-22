---
layout: post
title:  "C++ template traits 初谈"
date:   2013-11-14 16:44:40
categories: C++
---

C++ 模板编程中的Traits是必会的技术了，相当于编译期的多态。

先看代码

{% highlight cpp %}
/**
 * Code by chuanqi @ 2011-9-18
 */
#include <iostream>
using namespace std;

//迭代器的5种分类
struct InputIterator{};
struct OutputIterator{};
struct ForwardIterator : public InputIterator{};
struct BidirectionalIterator : public ForwardIterator{};
struct RandomIteratorIterator : public BidirectionalIterator{};

//类别分别归为5类的迭代器
template<typename T>
struct InIterator
{
    typedef InputIterator iterator_category;
};

template<typename T>
struct OutIterator
{
    typedef OutputIterator iterator_category;
};

template<typename T>
struct ListIterator
{
    typedef ForwardIterator iterator_category;
};

template<typename T>
struct BidirectListIterator
{
    typedef BidirectionalIterator iterator_category;
};

template<typename T>
struct VectorIterator
{
    typedef RandomIteratorIterator iterator_category;
};
//类型Traits
template<typename T>
struct IteratorTraits
{
    typedef typename T::iterator_category iterator_category;
};
//这个偏特化是为了解决和原生指针的兼容性，如果不需要和原生指针的兼容性，真正的traits类在Traits体系中
//的必要性是否应打一个问号？毕竟可以在ShowIteratorType里使用T::iterator_category直接来
template<typename T>
struct IteratorTraits<T *>
{
    typedef RandomIteratorIterator iterator_category;
};
//5个针对不同类型的函数重载
//函数重载简直就是编译期的if选择分支
    template<typename T>
void Show(T &, InputIterator)
{
    cout << "I am a InputIterator" << endl;
}

    template<typename T>
void Show(T &, OutputIterator)
{
    cout << "I am a OutputIterator" << endl;
}

    template<typename T>
void Show(T &, ForwardIterator)
{
    cout << "I am a ForwardIterator" << endl;
}

    template<typename T>
void Show(T &, BidirectionalIterator)
{
    cout << "I am a BidirectionalIterator" << endl;
}

    template<typename T>
void Show(T &, RandomIteratorIterator)
{
    cout << "I am a RandomIteratorIterator" << endl;
}
//这才是惟一公开的函数，会根据迭代器的类别采取不同的措施
    template<typename T>
void ShowIteratorType(T &t)
{
    //Show(t, T::iterator_category()); //如果不需要兼容指针，Traits体系就可以不需要真正的Traits类
    Show(t, IteratorTraits<T>::iterator_category());
}

int main()
{
    ShowIteratorType(InIterator<int>());
    ShowIteratorType(OutIterator<int>());
    ShowIteratorType(ListIterator<int>());
    ShowIteratorType(BidirectListIterator<int>());
    ShowIteratorType(VectorIterator<int>());
    int *p;
    ShowIteratorType(p); //由于traits的偏特化而实现与指针的兼容
}
{% endhighlight %}

Traits手法其实也很简单，归纳一下就是以下几步：

1. 首先定义出所有的类别，即分哪几类，即这里的－迭代器分为5类

{% highlight cpp %}
struct InputIterator{};
struct OutputIterator{};
struct ForwardIterator : public InputIterator{};
struct BidirectionalIterator : public ForwardIterator{};
struct RandomIteratorIterator : public BidirectionalIterator{};
{% endhighlight %}

2. 然后对要进行这些类别区分的模板类统一使用一个相同名字的typedef，即

{% highlight cpp %}
template<typename T>
struct xxxxxxIterator
{
    typedef xxxxxxTypeIterator iterator_category;
};
{% endhighlight %}

3. 可选的一步就是：使用一个真正的traits类来实现对内置类型或指针的兼容性，因为除了使用模板特化外，我们并没有办法对原生类型进行修改，添加typedef之类的。

{% highlight cpp %}
//类型Traits
template<typename T>
struct IteratorTraits
{
    typedef typename T::iterator_category iterator_category;
};
//这个偏特化是为了解决和原生指针的兼容性，如果不需要和原生指针的兼容性，真正的traits类在Traits体系中
//的必要性是否应打一个问号？毕竟可以在ShowIteratorType里使用T::iterator_category直接来
template<typename T>
struct IteratorTraits<T *>
{
    typedef RandomIteratorIterator iterator_category;
};
{% endhighlight %}

因此，虽然此步是可选的，但是由于通常都要实现对内置类型和指针的兼容性，知道这一步也是非常重要的！

4. 再使用函数重载来实现编译期间的选择分支

{% highlight cpp %}
//5个针对不同类型的函数重载
//函数重载简直就是编译期的if选择分支
    template<typename T>
void Show(T &, InputIterator)
{
    cout << "I am a InputIterator" << endl;
}
    template<typename T>
void Show(T &, OutputIterator)
{
    cout << "I am a OutputIterator" << endl;
}
    template<typename T>
void Show(T &, ForwardIterator)
{
    cout << "I am a ForwardIterator" << endl;
}
    template<typename T>
void Show(T &, BidirectionalIterator)
{
    cout << "I am a BidirectionalIterator" << endl;
}
    template<typename T>
void Show(T &, RandomIteratorIterator)
{
    cout << "I am a RandomIteratorIterator" << endl;
}
//这才是惟一公开的函数，会根据迭代器的类别采取不同的措施
    template<typename T>
void ShowIteratorType(T &t)
{
    //Show(t, T::iterator_category()); //如果不需要兼容指针，Traits体系就可以不需要真正的Traits类
    Show(t, IteratorTraits<T>::iterator_category());
}
{% endhighlight %}

5. OK完成，优雅的使用吧，编译期的多态不需要付出运行时间的代价，但是要付出编译时间长的代价

{% highlight cpp %}
ShowIteratorType(InIterator<int>());
ShowIteratorType(OutIterator<int>());
ShowIteratorType(ListIterator<int>());
ShowIteratorType(BidirectListIterator<int>());
ShowIteratorType(VectorIterator<int>());
int *p;
ShowIteratorType(p); //由于traits的偏特化而实现与指针的兼容
{% endhighlight %}
