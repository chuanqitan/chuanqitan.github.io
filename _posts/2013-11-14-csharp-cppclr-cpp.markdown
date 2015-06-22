---
layout: post
title:  "详细实现C#,C++/clr,C++三层架构"
date:   2013-11-14 16:44:40
categories: C++
---

用C++写核心的算法，用C#写界面或者逻辑，再用C++/CLR把它们粘合起来。
这是相当好的架构，在Windows下的程序员非常值得尝试。
C#写界面和逻辑等等对性能没有要求的代码效率还真是很高的，用起来也很顺手！

首先建好一个解决方案CSharpCppClrCppDll，里面包含有三个项目CppDll、CppClrLib、CSharpCall

CppDll是一个Win32应用程序，将其应用程序类型选择为DLL文件。这个CppDll项目不需要CLR的支持，因为它是用纯C++来实现核心代码的
CppClrLib是一个Visual C++ CLR Class Library，这个类用来包装CppDll生成的Dll文件，需要clr的支持，用来生成符合.NET标准的.NET DLL。

1：首先从CppDll中导出一个Dll的公用方法，在CppDll.cpp中添加代码：

{% highlight cpp %}
extern "C" _declspec (dllexport)
void fill_int_array(vector&lt;int&gt;&amp; v)
{
    v.clear();

    for(int i = 0; i &lt; 10; ++i)
    {
        v.push_back(i);
    }
}
{% endhighlight %}

接着在$(SolutionDir)$(Configuration) 目录下生成CppDll.dll以及其它的相当lib等等的文件。

2：再把生成的CppDll.dll等文件导入到CppClrLib项目中去。将CppClrLib项目的属性的Linker的Input设置为$(SolutionDir)$(Configuration)CppDll.lib，加载CppDll项目生成的动态链接库。
再在CppClrLib.h中添加代码来包装CppDll中导出的方法

{% highlight cpp %}
#pragma once

#include <vector>
using namespace std;
using namespace System;
using namespace System::Collections::Generic;

namespace CppClrLib {

    extern "C" _declspec (dllimport)
    void fill_int_array(vector<int> &gt;&amp; v);

    public ref class MyPackage
    {
        // TODO: Add your methods for this class here.
        public:
            static void fill_a_int_array(List&lt;int&gt;^ a)
            {
                vector<int>; v;
                fill_int_array(v);

                for(size_t i = 0; i &lt; v.size(); ++i)
                {
                    a->Add(v[i]);
                }
            }
    };
}
{% endhighlight %}

至此，就在$(SolutionDir)$(Configuration) 目录下生成了一个C++ .NET DLL（CppClrLib.dll），这个.NET DLL可以被所有基于.NET的程序语言所访问。
到这里，通过CppClrLib项目包装好的纯C++项目CppDll就可以完美的被C#等.NET语言所调用了。

3：在CSharpCall项目中加载第2步生成的那个符合.NET规范的DLL（CppClrLib.dll），在CSharpCall项目的引用中添加解决方案中的CppClrLib。这种完全符合.NET规范的程序集被加载之后，就可以非常方便的使用了。
这里，还有一点就是需要设置C#项目的输出目录为“..Debug”，这样所有的输入都在一个目录下，就可以相互调用了。
如果这里没有CppDll生成的非托管的DLL，就完全可以不用设置C#的输出目录了，因为VS会把项目需要的.NET程序集copy到本地目录下的。但是不会复制非引用的.NET DLL（CppDll.dll），所以，这里还是需要把输出目录全设在一起，或者是把CppDll.dll复制到默认的binDebug下去。

![](/images/post/csharp_cppclr_cpp1.png)

再在Program.cs中添加代码来调用引用的程序集中的方法了

{% highlight cpp %}
using System;
using System.Collections.Generic;
using System.Linq;
using System.Text;

namespace CSharpCall
{
    class Program
    {
        static void Main(string[] args)
        {
            List<int> a = new List<int>();
            CppClrLib.MyPackage.fill_a_int_array(a);

            foreach (var item in a)
            {
                Console.WriteLine(item);
            }

            Console.ReadKey();
        }
    }
}
{% endhighlight %}

最后的调用结果：

![](/images/post/csharp_cppclr_cpp3.png)


最后的程序需要三个文件来运行：CSharpCall.exe、CppClrLib.dll、CppDll.dll就可以了，不需要中途的lib等等文件…
场景：现有C++原代码，包装后供C#调用。

C++的原代码，实际上可以直接编译成托管代码。MFC也好ATL也好……这样看起来在.NET中最强大的编程语言就是C++了：它不仅可以编写托管程序，甚至可以将标准C++的代码也编译成托管程序！其实VC++最强大的地方不止如此，它还在于能够编写混合了托管和非托管的代码的程序！！！这样最大的好处不仅可以将关键代码直接编译成非托管的代码，还可以避免被反编译。假设现有C++代码如下：

{% highlight cpp %}
class UnmanagedClass {
    public:
        LPCWSTR GetPropertyA() { return L"Hello!"; }
        void MethodB( LPCWSTR ) {}
};
{% endhighlight %}

我们只要再增加一个包装类到工程文件中：（托管的类公开的方法才会出现在最后的.NET DLL程序集中）

{% highlight cpp %}
namespace wrapper
{
    public ref class ManagedClass {
        public:
            // Allocate the native object on the C++ Heap via a constructor
            ManagedClass() : m_Impl( new UnmanagedClass ) {}// Deallocate the native object on a destructor
            ~ManagedClass() {
                delete m_Impl;
            }protected:
            // Deallocate the native object on the finalizer just in case no destructor is called
            !ManagedClass() {
                delete m_Impl;
            }public:
            property String ^  get_PropertyA {
                String ^ get() {
                    return gcnew String( m_Impl-&gt;GetPropertyA());
                }
            }

            void MethodB( String ^ theString ) {
                pin_ptr&lt;const WCHAR&gt; str = PtrToStringChars(theString);
                m_Impl-&gt;MethodB(str);
            }

        private:
            UnmanagedClass * m_Impl;
    };
}
{% endhighlight %}

然后，改变编译选项为“使用公共语言扩展 /clr”就可以了。这样，我们把代码编译成DLL文件就可以供.NET其它语言调用了。

所以说：C++的原代码，实际上可以直接编译成托管代码。
最后，C#中可以象如下的代码一样调用C++类了：

{% highlight cpp %}
ManagedClass mc = new ManagedClass();
mc.MethoB("Hello");
string s = mc.get_PropertyA;
{% endhighlight %}

总结：
这样，在有源代码的情况下，直接把C++代码编译成托管代码，C#就可以直接调用了，就没有了上述所说的三层架构了。
上述的三层架构在以下情况下比较适用：
1：只有ISO C++的DLL文件，没有源代码时，用C++/CLR包装是最佳选择。
2：接口类型有vector, list, deque或者自定义的复杂类型时，因为C++/CLR可以直接引用C++的头文件，所以这种包装方式安全可靠，而且完美无缺！
3：需要用ISO C++的DLL来保密自己的算法。

否则的话，直接把C++编译成托管的DLL就可以完美的在.NET上被复用了！最后：完整的代码下载 [CSharpCppClrCppDll.rar](/attachment/CSharpCppClrCppDll.rar)
