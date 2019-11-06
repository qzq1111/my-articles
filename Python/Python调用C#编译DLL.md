# Python调用C#编译DLL

tips:
- 本文代码基于python3编写
- [代码仓库](https://github.com/qzq1111/python-summary)

## 问题背景
   
最近在做点云相关的内容，由于涉及到桌面端的应用，学习了相关C#的内容，也做出了一些实验性的产品。在这过程中，也思考python能否调用C#？毕竟python一大别称是“胶水语言”，能够组合其他语言，进而实现不同的功能需要，像numpy底层使用C语言编写等等。最终在网上找到了pythonnet这个库。

## pythonnet使用

>Python for .NET(pythonnet)是一个软件包，可为Python程序员提供与Windows上的.NET4.0+公共语言运行时（CLR）以及Linux和OSX上的Mono运行时几乎无缝的集成。Python for .NET为.NET开发人员提供了功能强大的应用程序脚本工具。使用此软件包，可以使用.NET服务和以任何针对CLR的语言（C＃，VB.NET，F＃，C ++ / CLI）编写的组件来使用.NET应用程序编写脚本或使用Python构建整个应用程序。

下面写一段Python调用C#编译成Dll的测试代码：

C#代码：

```csharp
using System;

namespace TestDll
{
    public class MyTest
    {

        public void Print()
        {

            Console.WriteLine("Hello world!!!");
        }

        public void Print(string msg)
        {

            Console.WriteLine($"Hello {msg}!!!");
        }

        public double Add(double x, double y)
        {
            return x + y;

        }
    }

}
```

Python代码：

```python
"""
使用须知：
1. 安装包：pip install pythonnet
2. 文档地址：https://pythonnet.github.io/ 或者 https://github.com/pythonnet/pythonnet
"""
# coding:utf-8
import clr

# 引用Dll，不需要添加后缀
clr.AddReference("TestDll")

# 引入DLL命名空间，并导入定义的类
from TestDll import MyTest

# 实例化类
instance = MyTest()
# 无输入及无返回
instance.Print()
# 有输入及无返回
instance.Print("qin")
# 有输入及输出
add_data = instance.Add(1, 1)
print(add_data)
```

## 总结  

- 简单展示了Python调用C#编译的Dll
- 更多的使用方法，大家可以参考[官方文档](https://github.com/pythonnet/pythonnet)。
