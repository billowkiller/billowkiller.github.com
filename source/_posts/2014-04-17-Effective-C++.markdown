---
layout: post
title: "Effective C++ Rework"
date: 2014-04-17 02:18
comments: true
categories: book
tags: [effective, insight, c++]
---


##让自己习惯C++

###Item 01： View C++ as a federation of languages

C++同时支持过程形式、面对对象形式、函数形式、泛型形式、元编程形式。

次语言有

*	C
*	Object-Oriented C++
*	Template C++
*	STL

C++高效编程守则视状况而变化，取决于你使用C++的那一部分。

<!--more-->

###Item 02： Prefer consts, enums and inlines to #defines

- `#define`的变量没有进入记号表。并且没有作用域，不能提供任何封装性。
- `#define`定义宏，需要为所有实参加上小括号，且不能够使用`++`和`--`。

`#define`难以调试、行为无法预料、类型不安全。

	class GamePlayer{
	private:
		static const int NumTurns = 5; //常量申明式
	};

通常C++要求你对你所使用的任何东西提供一个定义式，但那如果它是`class`专属常量又是`static`且为**整数型**(ints, chars, bools)则需特殊处理。只要是不取它们的地址，你可以申明并使用它们而无需提供定义式。如果需要取址，必须提供定义式：
	
	const int GamePlayer::NumTurns; //申明时获取了初值，定义不必赋值

"The enum hack"表示一个属于枚举类型的数值可权充int被使用，于是`GamePlayer`定义为

	class GamePlayer{
	private:
		enum { NumTurns = 5 };
		static const int NumTurns = 5;
	};


- 取一个`enum`地址是非法的。`enum`和`#define`一样不会导致非必要的内存分配。
- `enum hack`是`template metaprogramming`的基础技术。

`template inline`可以提供宏带来的效率以及一般函数的所有可预料行为和类型安全。遵守作用域和访问规则。

###Item 03： Use const whenever possible

令函数返回一个常量值，往往可以降低因客户错误而造成的意外，而不至于放弃安全性和高效性。

	const Rational operator* (const Rational& lhs, const Rational& rhs);
	Rational a, b, c;
	...
	(a * b) = c;  //错误

重载`operator[]`并对不同的版本给予不同的返回类型，就可以令`const`和`non-const`获得不同的处理。返回 `char&`也是必要的。

	const char& operator[](std::size_t position) const; //operator[] for const Object
	char& operator[](std::size_t position); //operator[] for non-const object



- 成员函数式`const`有两个流行的概念：`bitwise constness`, `logical constness`。
- `mutable`变量成员可以再const成员函数中改变。
- `static_cast`将`non-const`对象转为`const`对象。`const_cast`相反。


###Item 04： Make sure that objects are initialized before they're used

- `C++`对定义与不同编译单元(文件)内的`non-local static`对象的初始化次序并无明确定义。
- 函数内的`local static`对象会在该函数被调用期间首次遇上该对象的定义式时被初始化。

为内置对象进行手工初始化，`C++`不保证初始化它们。
构造函数使用`成员初值列`，不要在函数内使用赋值操作。其排列次序应该和申明次序相同。
为免除跨编译单元初始化次序问题。以`local static`对象替代`non-local static`对象。


##Constructors, destructors, and Assignment Operators
###Item 05： Know what functions C++ silently writes and calls

- `default`构造函数和析构函数调用`base classes`和`none-static`成员变量的构造函数和析构函数。且只有base是`virtual`析构时，它才是`virtual`的。
- 如果类内含`reference`或`const`成员，或者`base classes`将`copy assignment`操作符申明为`private`，则需要自己定义`copy assignment`

###Item 06: Explicitly disallow the use of compiler-generated functions you do not want

不实现`copy`或`copy assignment`的方法：

1. 将成员函数声明为private而且故意不实现它们。
2. 设计一个专门为了组织copying动作的`base class`， 将`copy`和`copy assigment`声明为`private`。接着私有继承base class
3. `Boost`提供的class，`nonecopyable`

###Item 07: Declare destructors virtual in ploymorphic base classes

	Base *pt = new Derived;
	delete pt;

`derived class`对象经由一个`base class`指针被删除，如果`base class`有个k析构函数，则对象的`derived`成分没被销毁。

无端地将所有classes的析构函数声明为`virtual`，就像从未声明它们为`virtual`一样，都是错误的，带来对象体积的增加。只有当class内含有至少一个`virtual`函数才为它声明`virtual`析构函数。

为你希望它成为抽象的那个class(polymorphic base classes)声明一个`pure virtual`析构函数。

	class AWOV {
	public:
		virtual  ~AWOV() = 0;
	};
	AWOV::~AWOV() {}  //pure virtual 析构函数的定义 

然而必须为这个`pure virtual`函数提供一份定义，根据析构函数的运作方式，编译器会在AWOV的`derived classes`的析构函数中创建一个对~AWOV的调用动作，所以必须为这个函数提供一份定义。否则，连接器会发出抱怨。

###Item 08: Prevent exceptions from leaving destructors

- 析构函数绝对不要突出异常。如果一个被析构函数调用的函数可能抛出异常，析构函数应该捕捉任何异常，然后吞下它们（不传播）或结束程序。
- 如果客户需要对某个操作函数运行期间抛出的异常作出反应，那么class应该提供一个普通函数（而非在析构函数中）执行该操作。

###Item 09: Never call virtual functions during construction or destruction

**在base class构造期间，virtual函数不是virtual函数**。因为derived class对象还未构造好，所以base class构造期间virtual函数绝不会下降到derived classes阶层。在derived class对象的base class构造期间，对象类型是base class而不是derived class。

在构造期间，可以借由“令derived classes将必要的构造信息向上传递至base class构造函数”替换之。

**同样的道理也适用于析构函数**。一旦derived class析构函数开始执行，对象内的derived class成员变量变成未定义值，所以C++是它们仿佛不再存在。进入base class析构函数后对象就称为一个base class 对象，而C++的任何部分包括virtual函数、`dynamic_cast`等等也就这么看它。

所以，在构造和析构席间不要调用virtual函数，因为这类调用从不降至derived class（比起当前执行构造函数和析构函数的那层）。

###Item 10: Have assignment operators return a reference to *this

	Widget& operator=(const widget &rhs) {
		...
		return *this;
	}

###Item 11: Handle assignment to self in operator=

容易掉进“在停止使用资源之前意外释放了它”的陷阱。

让`operator=`具备“异常安全性”往往自动获得“自我复制安全”的汇报。
	
	Widget& operator=(const widget &rhs) {
		Bitmap * pOrig = pb;
		pb = new Bitmap(*rhs.pb);
		delete pOrig;
		return *this;
	}

还可以使用`copy and swap`技术。或利用一下事实：(1)某class的`copy assignment`操作符可能被声明`by value`的方式；(2)以`by value`的方式传递东西会造成一份副本。

###Item 12: Copy all parts of an object

- 编写一个copying函数确保(1)复制所有local成员变量，(2)调用所有base classes内的适当的copy函数。
- 不要尝试以某个copying函数实现另一个copying函数。应该讲共同机能放进第三个函数中，并有两个copying函数共同调用。

##资源管理
###Item 13: Use objects to manage resources.

-   获得资源后立即放进资源对象内。`RAII`--Resource Acquisition Is initialization
-   管理对象运用析构函数确保资源被释放。
-   auto_ptr通过copy构造函数或copy assignment操作符复制它们，它们会变成null，而复制所得到的指针将取得资源的唯一拥有权。
-   动态分配得到的`array`身上使用auto_ptr或tr1::shared_ptr是个馊主意。两者再析构函数内做delete而不是delete[]动作。

###Item 14: Think carefully about copying behavior in resource-managing classes

当一个RAII对象被复制，考虑两种可能性：

-	禁止复制
-	对底层资源采用`引用计数法`， 
	-	`shared_ptr`的缺省行为是“当引用次数为0时删除其所指之物”，允许制定特定的删除器(`deleter`)
-	复制底部资源
-	转移底部资源的拥有权

###Item 15: Provide access to raw resources in resource-managing classes

智能指针重载了指针取值操作符(operator-> 和 operator*)，它们允许隐式转换至底部原始指针。

隐式转换举例：

	class Font {
	public:
		...
		operator FontHandle() const //隐式转换函数
		{ return f; }
		...
	}

-	API往往要求访问原始资源，所以每个RAII class应该提供一个取得原始资源的方法。
-	对原始资源的访问可能经由显示转换或隐式转换。一般而言显示转换比较安全，但隐私转换对客户比较方便。

###Item 16: Use the same form in corresponding uses of new and delete

new对应delete，new[] 对应delete[]

对于typedef,必须要在程序中说明清楚
	
	typedef std::string AddressLines[4];
	std::string * pal = new AddressLines;
	delete [] pal; //delete pal 导致行为未有定义
因此，最好尽量不要对数组形式做typedef动作。

###Item 17: Store newed objects in smart pointers in standalone statements.

假设有个函数解释处理程序的优先权，另一个函数用来在某动态分配所得的Widget上进行某些带有优先权的处理：

	int priority();
	void processWidget(std::tr1::shared_ptr<Widget> pw, int priority);

如果这样调用
	
	processWidget(std::tr1::shared_ptr<Widget>(new Widget), priority());
编译器创建代码，做以下三件事：

- 调用priority
- 执行new Widget
- 调用tr1::shared_ptr构造函数。

只能保证new Widget在shared_ptr构造函数之前被调用。如果priority在两者中间被调用，而且导致异常。那么new Widget返回的指针将会遗失。

##设计与声明
###Item 18: Make interfaces easy to use correctly and hard to use incorrectly

- 好的接口很容易被正确使用，不容易被误用。你应该在你的所有接口中努力达成这些性质。
- “促进正确使用”的办法包括借口的一致性，以及与内置类型的行为兼容。任何接口如果要求客户必须记得做某些事情，就是有着“不正确使用”的倾向，因为客户可能会忘记做那件事。
- “阻止误用”的办法包括建立新类型、限制类型上的操作，束缚对象值，以及消除客户的资源管理责任。
- tr1::shared_ptr支持定制性删除器(custom deleter)。可以防范`cross-DLL problem`(在一个DLL中被new创建，却在另一个DLL中被delete销毁)，自动解除互斥锁等等。

###Item 20: Prefer pass-by-reference-to-const to pass-by-value.

- 尽量以`pass-by-reference-to-const`替换`pass-by-value`。前者通常比较高效，并可以避免切割问题。
- 以上规则并不适用于内置类型，以及STL的迭代器和函数对象。对它们而言，pass-by-value往往比较适当。

###Item 21: Don't try to return a reference when you must return an object

绝不要返回pointer或reference指向一个local stack对象，或返回reference指向一个heap-allocated对象，或返回pointer或reference指向一个local static对象而有可能同时需要多个这样的对象。

###Item 22: Declare data members private.

- 封装的重要性比你最初见到它时还要重要
- 切记将成员变量声明为private。这可赋予客户访问数据的一致性、可细微划分访问控制、允诺约束条件获得保证，并提供class作者以充分的实现弹性。
- protected并不比public更具有封装性。

###Item 23: Prefer non-member non-friend functions to member functions

- 提供更大的封装性
- 比较自然的做法是让 non-member non-friend函数作为便利函数位于类所在的同一个namespace内。并将同类便利函数声明放在同一头文件中。
- 将所有便利函数放在多个头文件内但隶属于同一个命名空间，意味客户可以轻松扩展这一组便利函数。

###Item 24: Declare non-member functions when type conversions should apply to all parameters

	class Rational {
	public:
		const Rational operator* (const Rational &rhs) const;
	};

	Rational oneHalf(1, 2);
	result = oneHalf * 2; // oneHalf.operator*(2)，可以执行
	result = 2 * oneHalf; // 2.operator*(oneHalf)，错误

所以让函数称为一个`non-member`函数

	const Rational operator* (const Rational &lhs， const Rational &rhs);	
允许编译器在每一个实参身上执行隐私转换类型。

###Item 25: Consider support for a non-throwing swap

	namespace std {
		template<> //全特化
		void swap<Widget>( Widget &a, Widget &b) {
			swap(a.pImpl, b.pImpl); //私有变量，编译不通过，需要创建类的成员函数
		} }

	class Widget {
	public:
		void swap(Widget& other) {
			using std::swap; //声明是有必要的，下个swap调用std版本的
			swap(pImpl, other.pImpl);

- 当`std::swap`对你的类型效率不高时，提供一个`swap`成员函数，并确定这个函数不抛出异常
- 如果你提供一个`member swap`，也应该提供一个`non-member swap`用来调用前者。对于classes（而非templates），也请特化`std::swap`
- 调用`swap`时应针对`std::swap`使用`using`声明式，然后调用`swap`并且不带任何“命名空间资格修饰”。
- 为“用户定义类型”进行`std templates`全特化是好的，但千万不要尝试在`std`内加入某些对`std`而言是全新的东西

##实现
###Item 26: Postpone variable definitions as long as possible

不止应该延后变量的定义，这道非得使用该变量的前一刻为止，甚至应该尝试延后这份定义知道能够给它初值实参为止。

###Item 27: Minimize casting

- `const_cast`被用来将对象的常量性转除，也是唯一有此能力的c++ style转型操作符
- `dynamic_cast`主要用来执行“safe downcasting”，可能需要耗费重大运行成本。
- `reinterpret_cast`执行低级转型，实际动作及结果可能取决于编译器，也就表示它不可移植。
- `static_cast`用来强迫隐式转换。例如将non-const转为const对象，将int转为double。

单一对象(例如一个类型为Derived的对象)`可能拥有一个以上的地址`(例如“以Base* 指向它”时的地址和“以Derived* 指向它”时的地址)。这至少意味着你通常应该避免作出“对象在C++中如何布局”的假设。

- 如果可以，尽量避免转型，特别是在注重效率的代码中避免`dynamic_cast`。如果有个设计需要转向动作，试着发展无需转型的替代设计。
- 如果转型是必要的，试着将它隐藏于某个函数背后。客户随后可以调用该函数，而不需将转型放进他们自己的代码内。
- 宁可使用C++ style转型，不要使用旧式转型。前者很容易辨识出来，而且也比较有着分门别类的职掌。

###Item 28: Avoid returning "handles" to object internals

避免返回handles(包括references、指针、迭代器)指向对象内部。遵守这个条款可增加封装性，帮助const成员函数的行为像个const，并将发生dangling handles的可能性降至最低。

考虑一个例子：
	
	const Point& Rectangle::upperLeft() const { return pData->ulhc; }	

	class GUIObject{};
	const Rectangle boundingBox(const GUIObject &obj);

	GUIObject *pgo;
	const Point* pUpperLeft = &(boundingBox(*pgo).upperLeft()); //指向一个不存在的对象

###Item 29: Strive for exception-safe code.

- 异常安全函数即使发生异常也不会泄露资源或允许任何数据结构败坏。这样的函数区分三种可能的保证：基本型、强烈型、不抛异常型。
- 强烈保证往往能够以copy-and-swap实现出来，但强烈保证并非对所有函数都可实现或具备现实意义。
- 函数提供的异常安全保证通常最高只等于其所调用之各个函数的异常安全保证中的最弱者。

###Item 30: Understand the ins and outs of inlining

`inline`只是对编译器的一个申请，不是强制命令。这项申请可以隐喻提出，也可以明确提出。隐喻方式是将函数定义于class定义内：

	class Person {
	public:
		int age() const { return theAge; } //隐喻的inline申请
	private:
		int the Age;
	};

这样的函数通常是成员函数，`friend`函数也可以被定义于class内，这样它们也是被隐喻为`inline`。

`inline`和`template`函数通常都被定义于头文件内。

	inline void f() {}
	void ( * pf )() = f;
	f(); //这个调用将被inlined
	pf(); //或许不被inlined，因为它通过函数指针达成。

- `inline`函数无法随着程序库升级而升级。调用`inline`函数的程序都必须重新编译。
- 大部分调试器面对`inline`函数都束手无策。

###Item 31: Minimize compilation dependencies between files

以`声明的依存性`替换`定义的依存性`：

- 如果使用`object reference`或`object pointers`可以完成任务，就不要使用objects。
- 如果能够，尽量以class`声明式`替换class`定义式`。 
- 为声明式和定义式提供不同的头文件。这种方法无论是否涉及templates都适用。
	- <iosfwd>内含iostream各组件的声明式，包括<sstream>, <streambuf>, <fstream>和<iostream>

##继承和面向对象设计
###Item 32: Make sure public inheritance models "is-a"

###Item 33: Avoid hiding inherited names

	class Base {
	public:
		virtual void mf1() = 0;
		virtual void mf1(int);
		virtual void mf2();
		void mf3();
		void mf3(double);
	};

	class Derived: public Base {
	public:	
		using Base::mf1;  //让base class内名为mf1和mf3的所有东西
		using Base::mf3;  //在Derived作用域内都可见
		virtual void mf1();
		void mf3();
	};	

	//或者
	class Derived: private Base {
	public:
		virtual void mf1() { Base::mf1(); }  //转交函数，暗自称为inline
	};

- derived class内的名称会遮掩base classes内的名称。在public继承下从来没有人希望如此。
	- 上述规则对不同参数类型也适用，而且不论函数式virtual或non-virtual。
- 为了让被遮掩的名称再见天日，可使用using声明式或转变函数。

###Item 34: Differentiate between inheritance of interface and inheritance of implementation

- 接口继承和实现继承不同。在public继承之下，derived classes总是继承base class的接口。
- pure virtual函数只具体指定接口继承。pure virtual函数必须在derived classes中重新声明，但它们也可以拥有自己的实现。
- 简朴的impure virtual函数具体指定接口继承及缺省实现继承。
- non-virtual函数具体指定接口继承以及强制性实现继承。

###Item 35: Consider alternatives to virtual functions

当你为解决问题而寻找某个设计方法时，不妨考虑virtual函数的替代方案。

- 使用non-virtual interface（NVI）手法，那是Template Method设计模式的一种特殊形式。它以public non-virtual成员函数包裹较低访问性(private或protected)的virtual函数, wrapper。
- 将virtual函数替换为函数指针成员变量，这是Strategy设计模式的一种分解表现形式。
- 以tr1::function成员变量替换virtual函数，因而允许使用任何可调用物(callable entity)搭配一个兼容于需求的签名式。这也是Strategy设计模式的某种形式。
- 将继承体系内的virtual函数替换为另一个继承体系内的virtual函数。这是Strategy设计模式的传统实现手法。

###Item 36: Never redefine an inherited non-virtual function

non-virtual是静态绑定，调用的方法是静态类型所拥有的方法，而不是实际类型所拥有的方法。

###Item 37: Never redefine a function's inherited default parameter value

virtual函数是动态绑定，而缺省参数值确实静态绑定。静态绑定为early binding，动态绑定为late binding。即使子类重新定义了virtual函数的缺省参数，调用还是用了父类的缺省参数。这是为了运行期效率。如果缺省参数值是动态绑定，编译器就必须有某种办法在运行期间为virtual函数决定适当的缺省参数值。

可以使用NVI（non-virtual interface）手法：

	class Shape {
	public:
		enum ShapeColor { Red, Green, Blue};
		void draw(ShapeColor color = Red) const {
			doDraw(color);
		}
	private:
		virtual void doDraw(ShapeColor color) const = 0;//真正的工作在此处
	}；
	class Rectangle: public Shape {
	public:
		...
	private:
		virtual void doDraw(ShapeColor color) const; //不须制定缺省参数值
	};

###Item 39: Use private inheritance judiciously

- 编译器不会自动将一个derived class对象转换为一个base class对象。
- 由private base class继承而来的所有成员，在derived class 中都会变成private 属性。
- private继承在软件设计层面没有意义，只有在软件实现层面有意义。
- Private继承意味is-implemented-in-terms of(根据某物实际出)。它通常比复合的级别低。但是当derived class需要访问protected base class的成员，或需要重新定义继承而来的virtual函数时，这么设计是合理的。
和
- 复合不同，private继承可以造成empty base最优化。这对致力于对象尺寸最小化的程序库开发者而言，可能很重要。

怎样阻止derived classes重新定义virtual函数？

	class Widget {
	private:
		class WidgetTimer: public Timer {
		public:
			virtual void onTick() const;
		};
		WidgetTimer timer;
	};

私有继承空类并不继承空类的空间

	class Empty {}; //sizeof Empty == 1;
	class HoldsAnInt: private Empty { int x; }; //sizeof HoldsAnInt == 4;

###Item 40: Use multiple inheritance judiciously

- 多重继承比单一继承复杂。它可能导致新的歧义性，以及对virtual继承的需要。
- virtual继承会增加大小、速度、
- 初始化（及赋值）复杂度等等成本。如果virutal base classes不带任何数据，将是最具使用价值的情况。Java和.Net的Interfaces指的注意，它在许多方面兼容于C++的virtual base classes，而且也不允许含有任何数据。
- 多重继承的确有正当用途。其中一个情节涉及public继承某个Interface class和private继承某个协助实现的class的两相结合。

##模版与泛型编程

###Item 41: Understand implicit interfaces and compile-time polymorphism

- classes和templates都支持接口和多态。
- 对classes而言接口是显示的，以数字签名为中心。多态则是通过virtual函数发生于运行期。
- 对template参数而言，接口是隐式的，奠基于**有效表达式**。多态则是通过template具现化和函数重载解析发生于编译器。

###Item 42: Understand the two meanings of typename.

在template声明式中，`class`和`typename`不一定相同。

	template<typename C>
	void print2nd(const C& container) {
		if(container.size() >= 2) {
			C::const_iterator iter(container.begin());
			++iter;
			int value = *iter;
		{
	}

`iter`的类型是`C::const_iterator`，实际是什么值取决于`template`参数`C`。`template`内出现的名称如果相依于某个`template`参数，称之为从属名称。如果从属名称在`class`内呈嵌套状，我们称它为嵌套从属名称。`C::const_iterator`就是这样的一个名称。而`value`的类型`int`并不依赖`template`参数的名称，称之为非从属名称。**在缺省情况下，嵌套从属名称不是类型**。

改为`typename C::const_iterator iter(container.begin());`。

一种特列情况为，`typename`不可以出现在`base classes list`内的嵌套从属类型名称之前，也不可以在`member initialization list`中作为`base class`修饰符。

	template<typename T>
	class Derived: public Base<T>::Nested {
	public:
		explict Derived(int x): Base<T>::Nested(x) {
			typename Base<T>::Nested temp;
		}
	};

	最后一个例子：
	template<typename IterT>
	void workWithIterator(IterT iter) {
		typename std::iterator_traits<IterT>::value_type temp(*iter);
	}
	
###Item 43: Know how to access names in templatized base classes

当我们从`Object Oriented C++`进入`Template C++`，继承就不像以前那般顺利。编译器知道`base class templates`有可能被特化，而那个特化版本可能不提供和一般性template相同的接口，因而它往往拒绝在`templatized base classes`内寻找继承而来的名称。

	template<typename Company>
	class LoggingMsgSender: public MsgSender<Company> {
	public:
		void sendClearMsg(const MsgInfo* info) {
			sendClear(info);  //调用base class函数；这段代码无法通过编译
		}
	};

基类`MsgSender<Company>`的特化版本，可能不提供`sendClear()`方法。

解决方法：

1. base class函数调用动作之前加上`this->`
1. 在函数前使用using声明式，`using MsgSender<Company>::sendClear`.
1. 直接使用`MsgSender<Company>：：sendClear(info)`。

第三种做法有缺陷，如果被调用的是`virtual`函数，上述的做法会关闭`virtual绑定行为`。

###Item 44: Factor parameter-independent code out of templates.

- Templates生成多个classes和多个函数，所以任何template代码都不该与某个造成膨胀的template参数产生相依关系。
- 因非类型模版参数而造成的代码膨胀，往往可消除，做法是以函数参数或class成员变量替换template参数。
- 因类型参数而造成的代码膨胀，往往可降低，做法是让带有完全相同二进制表述的具现类型共享实现码。
    
template &lt;T*>可以改为`tempalte<void*>`减少代码膨胀。

###Item 45: Use member function templates to accept "all compatible types"

如果以带有`base-derived`关系的B，D两类型分别具现化某个template，产生出来的两个具现体并不带有`base-derived`关系。

	template<typename T>
	class SmartPtr {
	public:
		tempalte<typename U> //member template, 为了生成copy构造函数
		SmartPtr(const SmartPtr<U>& other); 
	};

这一类构造函数根据SmartPtr&lt;U>创建一个Smart&lt;T>。未加上`explicit`是因为原始指针类型之间的转换（例如从derived转化base）是隐式转换。可以在构造模板实现代码中约束行为：

	template<typename T>
	class SmartPtr {
	public:
		tempalte<typename U> //以other的heldPtr初始化this的heldPtr
		SmartPtr(const SmartPtr<U>& other):heldPtr(other.get()) {}
		T * get() const { return heldPtr; }
	private:
		T* heldPtr; 
	};

成员函数模板的效用不限于构造函数，它们常扮演的另一个角色是支持赋值操作。

	template<typename T>
	class shared_ptr {
	public:
		template<class Y>
		explicit shared_ptr(Y* p);
		template<class Y>
		shared_ptr(shared_ptr<Y> const& r);
		template<class Y>
		explicit shared_ptr(weak_ptr<Y> const& r);
		template<class Y>
		explicit shared_ptr(auto_ptr<Y> const& r);
		template<class Y>
		shared_ptr& operator=(shared_ptr<Y> const& r);
		template<class Y>
		shared_ptr& operator=(auto_ptr<Y> & r);
	};

上述函数的`explict`表示从某个shared\_ptr类型隐式转换至另一个shared\_ptr类型是被允许的，但从某个内置指针或从其他智能指针类型进行隐式转换则不被认可。auto\_ptr不声明const是因为复制一个auto\_ptr，它其实被改动了。

在class内声明泛化copy构造函数并不会阻止编译器生成它们自己的copy构造函数。

###Item 46: Define non-member functions inside templates when type conversions are desired.

将Item24的例子改为模板：

	template<typename T>
	class Rational {
	public:
		Rational(const T& numerator = 0, const T& denominator = 1);
		
		template<typename T>
		const Rational<T> operator* (const Rational<T> &rhs, const Rational<T> &rhs);
	};

	Rational oneHalf(1, 2);
	result = oneHalf * 2; // 无法通过编译，不加模板则可以

这是因为template实参推导过程中从不将隐式类型转换函数考虑在内。可以改为如下：

	friend const Rational operator* (const Rational &rhs, const Rational &rhs); //省略了<T>

当对象oneHalf被声明为一个Rational<int>, 模板被具现化出来，而作为过程的一部分friend函数（接受Rational<int> 参数）也就自动声明出来，后者作为一个函数而非函数模板，因此编译器可以在调用它时使用隐式转换函数。


##定制new和delete
###Item 49: Understand the behavior of the new-handler.

set_new_handler允许客户指定一个函数，在内存分配无法获得满足时候被调用。



 
		
	