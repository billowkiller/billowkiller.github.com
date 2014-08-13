---
layout: post
title: "C++ Template"
date: 2014-04-23 02:18
comments: true
categories: language
tags: [template, c++]
---

modified from [http://www.cnblogs.com/assemble8086/archive/2011/10/02/2198308.html](http://www.cnblogs.com/assemble8086/archive/2011/10/02/2198308.html)

## 模版

`template` 是声明类模板的关键字，表示声明一个模板，模板参数可以是一个，也可以是多个，可以是**类型参数** ，也可以是**非类型参数**。类型参数由关键字`class`或`typename`及其后面的标识符构成。非类型参数由一个普通参数构成，代表模板定义中的一个常量。

**类模板什么时候会被实例化呢？**

1. 当使用了类模板实例的名字，并且上下文环境要求存在类的定义时。
1. 对象类型是一个类模板实例，当对象被定义时。此点被称作类的实例化点。
1. 一个指针或引用指向一个类模板实例，当检查这个指针或引用所指的对象时。

<!--more-->

**非类型参数的模板实参**

1. 绑定给非类型参数的表达式必须是一个常量表达式。
1. 从模板实参到非类型模板参数的类型之间允许进行一些转换。包括左值转换、限定修饰转换、提升、整值转换。
1. 可以被用于非类型模板参数的模板实参的种类有一些限制。

		Template<int* ptr> class Graphics{…….};
		
		Template<class Type,int size> class Rect{……..};
		
		const int size=1024;
		
		Graphics<&size> bp1;//错误:从const int*－>int*是错误的。
		
		Graphics<0> bp2;//错误不能通过隐式转换把０转换成指针值
		
		const double db=3.1415;
		
		Rect<double,db> fa1;//错误：不能将const double转换成int.
		
		unsigned int fasize=255;
		
		Rect<String, fasize> fa2;//错误：非类型参数的实参必须是常量表达式，将unsigned改为const就正确。
		
		Int arr[10];
		
		Graphics<arr> gp;//正确

**类模板的成员函数**

1. 类模板的成员函数可以在类模板的定义中定义(`inline`函数)，也可以在类模板定义之外定义(此时成员函数定义前面必须加上`template`及模板参数)。
1. 类模板成员函数本身也是一个模板，类模板被实例化时它并不自动被实例化，只有当它被调用或取地址，才被实例化。

		template<class type>
		Class Graphics{
			Graphics(){…}//成员函数定义在类模板的定义中
			void out();
		};
		
		template<class type>//成员函数定义在类模板定义之外
		void Graphics<type>::out(){…}

**类模板的友元声明**

1. 非模板友元类或友元函数
2. 绑定的友元类模板或函数模板。

		template<class type>
		void create(Graphics<type>);
		
		template<class type>
		class Graphics{
			friend void create<type>(Graphics<type>);
		};
3. 非绑定的友元模板

		template<class type>
		class Graphics{
			template<class T>
			friend void create(Graphics<T>);
		};

**注意：**当把非模板类或函数声明为类模板友元时，它们不必在全局域中被声明或定义，但将一个类的成员声明为类模板友元，该类必须已经被定义，另外在声明绑定的友元类模板或函数模板时，该模板也必须先声明。

**类模板的静态数据成员**

1. 静态数据成员的模板定义必须出现在类模板定义之外。
1. 类模板静态数据成员本身就是一个模板，它的定义不会引起内存被分配，只有对其实例化才会分配内存。
1. 当程序使用静态数据成员时，它被实例化，每个静态成员实例都与一个类模板实例相对应，静态成员的实例引用要通过一个类模板实例。

**成员模板**

1. 在一个类模板中定义一个成员模板,意味着该类模板的一个实例包含了可能无限多个嵌套类和无限多个成员函数．
1. 只有当成员模板被使用时，它才被实例化.
1. 成员模板可以定义在其外围类或类模板定义之外．

**类模板的编译模式**

1. 包含编译模式

	这种编译模式下，类模板的成员函数和静态成员的定义必须被包含在“要将它们实例化”的所有文件中，如果一个成员函数被定义在类模板定义之外，那么这些定义应该被放在含有该类模板定义的头文件中。

2. 分离编译模式

	这种模式下，类模板定义和其inline成员函数定义被放在头文件中，而非inline成员函数和静态数据成员被放在程序文本文件中。

		//------Graphics.h---------
		
		export template<class type>
		Class Graphics
		{void Setup(const type &);};
		
		//-------Graphics.c------------
		
		#include “Graphics.h”
		Template <class type>
		Void Graphics<type>::Setup(const type &){…}
		
		//------user.c-----
		
		#include “Graphics.h”
		Void main(){
			Graphics<int> *pg=new Graphics<int>;
			Int ival=1;
			//Graphics<int>::Setup(const int &)的实例（下有注解）
			Pg->Setup(ival);
		}

	Setup的成员定义在User.c中不可见,但在这个文件中仍可调用模板实例Graphics<int>::Setup(const int &)。为实现这一点，须将类模声明为可导出的：**当它的成员函数实例或静态数据成员实例被使用时，编译器只要求模板的定义，它的声明方式是在关键字template前加关键字export**

3. 显式实例声明

	当使用包含编译模式时，类模板成员的定义被包含在使用其实例的所有程序文本文件中，何时何地编译器实例化类模板成员的定义，我们并不能精确地知晓，为解决这个问题，标准C++提供了显式实例声明：关键字template后面跟着关键字class以及类模板实例的名字。**显式实例化类模板时，它的所有成员也被显式实例化**。

		#include “Graphics.h”
		Template class Graphics<int>;//显式实例声明

**类模板的特化**

`template<>`成员函数特化定义

1. 只有当通用类模板被声明后，它的显式特化才可以被定义。
1. 若定义了一个类模板特化，则必须定义与这个特化相关的所有成员函数或静态数据成员，此时类模板特化的成员定义不能以符号`template<>`作为打头。(`template<>`被省略)
1. 类模板不能够在某些文件中根据通用模板定义被实例化，而在其他文件中却针对同一组模板实参被特化。

**类模板部分特化**

如果模板有一个以上的模板参数，则有些人就可能希望为一个特定的模板实参或者一组模板实参特化类模板，而不是为所有的模板参数特化该类模板。即，希望提供这样一个模板：它仍然是一个通用的模板，只不过某些模板参数已经被实际的类型或值取代。通过使用类模板部分特化，可以实现这一点。

	template<int hi,int wid>
	Class Graphics{…};
	
	Template<int hi>//类模板的部分特化
	Class Graphics<hi,90>{…};

1. 部分特化的模板参数表只列出模板实参仍然未知的那些参数。
1. 类模板部分特化是被隐式实例化的。编译器选择“针对该实例而言最为特化的模板定义”进行实例化，当没有特化可被使用时，才使用通用模板定义。
1. 类模板部分特化必须有它自己对成员函数、静态数据成员和嵌套类的定义。