---
layout: post
title: "New features in C++11"
date: 2014-04-27 02:18
comments: true
categories: language
tags: [feature, c++]
---

modified from [http://coolshell.cn/articles/5265.html](http://coolshell.cn/articles/5265.html)

----------

C++11在2011年8月通过ISO用来替换C++03。是对目前C++语言的扩展和修正，C++11不仅包含核心语言的新机能，而且扩展了C++的标准程序库（STL），并入了大部分的C++ Technical Report 1（TR1）程序库(数学的特殊函数除外)。

C++11包括大量的新特性：包括lambda表达式，类型推导关键字auto、decltype，和模板的大量改进。下面就来介绍下这些新特性。

<!--more-->
----------

## Lambda 表达式 ##

Lambda表达式来源于函数式编程，说白就了就是在使用的地方定义函数，有的语言叫“闭包”。表达式的简单语法如下，

	[capture](parameters)->return_type {body}

来看个计数某个字符序列中有几个大写字母的例子：

``` c++
int main()
{
   char s[]="Hello World!";
   int Uppercase = 0; //modified by the lambda
   for_each(s, s+sizeof(s), [&Uppercase] (char c) {
      if (isupper(c)) Uppercase++; });
   cout<< Uppercase<<" uppercase letters in: "<< s<<endl;
}
```

其中 `[&Uppercase]` 中的 `&` 的意义是 lambda 函数体要获取一个 `Uppercase` 引用，以便能够改变它的值，如果没有 `&`，那就 `Uppercase` 将以传值的形式传递过去。

C++引入Lambda的最主要原因就是1）可以定义匿名函数，2）编译器会把其转成函数对象**。这种函数限制了别人的访问，更私有。也可以认为是一次性的方法。Lambda表达式应该是简洁的，极私有的，为了更易的代码和更方便的编程。

## 自动类型推导 auto decltype ##

### auto ###

在 C++03 中，声明对象的同时必须指明其类型，其实大多数情况下，声明对象的同时也会包括一个初始值，C++11 在这种情况下就能够让你声明对象时不再指定类型了：

``` c++
auto x=0; //0 是 int 类型，所以 x 也是 int 类型
auto c='a'; //char
auto d=0.5; //double
auto national_debt=14400000000000LL;//long long
```

这个特性在对象的类型很大很长的时候很有用，特别是迭代器类型。但是，我们不应该把其滥用。

``` c++
auto obj = ProcessData(someVariables);
```

我们完全不知道ProcessData返回什么，这种代码的可读性就降低很多了，但是下面程序就没有问题，因为pObject的型别在后面的new中有了。

``` c++
auto pObject = new SomeType<OtherType>::SomeOtherType();
```

### decltype ###

关于 decltype 是一个操作符，其可以评估括号内表达式的类型，其规则如下：

1. 如果表达式e是一个变量，那么就是这个变量的类型。
1. 如果表达式e是一个函数，那么就是这个函数返回值的类型。
1. 如果不符合1和2，如果e是左值，类型为T，那么decltype(e)是T&；如果是右值，则是T。

``` c++
const vector<int> vi;
typedef decltype (vi.begin()) CIT;
CIT another_const_iterator;
```

还有一个适合的用法是用来typedef函数指针，也会省很多事。比如：

``` c++
decltype(&myfunc) pfunc = 0;
typedef decltype(&A::func1) type;
```

### auto 和 decltype 的差别和关系 ###

Wikipedia 上是这么说的（关于decltype的规则见上）

``` c++
int main()
{
    const std::vector<int> v(1);
    auto a = v[0];        // a 的类型是 int
    decltype(v[0]) b = 1; // b 的类型是 const int&, 因为函数的返回类型是
                          // std::vector<int>::operator[](size_type) const
    auto c = 0;           // c 的类型是 int
    auto d = c;           // d 的类型是 int
    decltype(c) e;        // e 的类型是 int, 因为 c 的类型是int
    decltype((c)) f = c;  // f 的类型是 int&, 因为 (c) 是左值
    decltype(0) g;        // g 的类型是 int, 因为 0 是右值
}
```

如果auto 和 decltype 在一起使用会是什么样子？能看下面的示例，下面这个示例也是引入decltype的一个原因——让C++有能力写一个 “ forwarding function 模板”，

``` c++
template< typename LHS, typename RHS>
  auto AddingFunc(const LHS &lhs, const RHS &rhs) -> decltype(lhs+rhs)
{return lhs + rhs;}
```

这个函数模板看起来相当费解，其用到了auto 和 decltype 来扩展了已有的模板技术的不足。怎么个不足呢？在上例中，我不知道AddingFunc会接收什么样类型的对象，这两个对象的 + 操作符返回的类型也不知道，老的模板函数无法定义AddingFunc返回值和这两个对象相加后的返回值匹配，所以，你可以使用上述的这种定义。

## 统一的初始化语法 ##

C/C++的初始化的方法比较，C++ 11 用大括号统一了这些初始化的方法。

``` c++
C c {0,0}; //C++11 only. 相当于: C c(0,0);
int* a = new int[3] { 1, 2, 0 }; //C++11 only
class X {
    int a[4];
    public:
        X() : a{1,2,3,4} {} //C++11, member array initializer
};

// C++11 container initializer
vector<string> vs={ "first", "second", "third"};
map singers =
{ {"Lady Gaga", "+1 (212) 555-7890"},
{"Beyonce Knowles", "+1 (212) 555-0987"}};

//class member initializer
class C
{
   int a=7; //C++11 only
   public: C();
};
```

## Delete 和 Default 函数 ##

我们知道C++的编译器在你没有定义某些成员函数的时候会给你的类自动生成这些函数，比如，构造函数，拷贝构造，析构函数，赋值函数。有些时候，我们不想要这些函数，比如，构造函数，因为我们想做实现单例模式。传统的做法是将其声明成private类型。

在新的C++中引入了两个指示符，delete意为告诉编译器不自动产生这个函数，default告诉编译器产生一个默认的。下面两个例子：

``` c++
struct A
{
    A()=default; //C++11
    virtual ~A()=default; //C++11
};
```

再如delete

``` c++
struct NoCopy
{
    NoCopy & operator =( const NoCopy & ) = delete;
    NoCopy ( const NoCopy & ) = delete;
};
NoCopy a;
NoCopy b(a); //compilation error, copy ctor is deleted
```

这里，我想说一下，为什么我们需要default？我什么都不写不就是default吗？不全然是，比如构造函数，因为只要你定义了一个构造函数，编译器就不会给你生成一个默认的了。所以，为了要让默认的和自定义的共存，才引入这个参数，如下例所示：

``` c++
struct SomeType
{
 SomeType() = default; // 使用编译器生成的默认构造函数
 SomeType(OtherType value);
};
```

关于delete还有两个有用的地方是

1. 让你的对象只能生成在栈内存上：

		struct NonNewable {
		    void *operator new(std::size_t) = delete;
		};

2. 阻止函数的其形参的类型调用：（若尝试以 double 的形参调用 f()，将会引发编译期错误， 编译器不会自动将 double 形参转型为 int 再调用f()，如果传入的参数是double，则会出现编译错误）

		void f(int i);
		void f(double) = delete;

## nullptr ##

nullptr 是一个新的 C++ 关键字，它是空指针常量，它是用来替代高风险的 NULL 宏和 0 字面量的。nullptr 是强类型的：

``` c++
void f(int); //#1
void f(char *);//#2
//C++03
f(0); //调用的是哪个 f?
//C++11
f(nullptr) //毫无疑问，调用的是 #2
```

所有跟指针有关的地方都可以用 nullptr，包括函数指针和成员指针：

``` c++
const char *pc=str.c_str(); //data pointers
if (pc!=nullptr)
  cout<<pc<<endl;
int (A::*pmf)()=nullptr; //指向成员函数的指针
void (*pmf)()=nullptr; //指向函数的指针
```

## 委托构造 ##

C++11 中构造函数可以调用同一个类的另一个构造函数：

``` c++
class M //C++11 delegating constructors
{
	int x, y;
	char *p;
public:
	M(int v) : x(v), y(0),  p(new char [MAX])  {} //#1 target
	M(): M(0) {cout<<"delegating ctor"<<end;} //#2 delegating
```

但是，为了方便并不足让“委托构造”这个事出现，最主要的问题是，基类的构造不能直接成为派生类的构造，就算是基类的构造函数够了，派生类还要自己写自己的构造函数：

``` c++
class BaseClass
{
public:
  BaseClass(int iValue);
};
 
class DerivedClass : public BaseClass
{
public:
  using BaseClass::BaseClass;
};
```

上例中，派生类手动继承基类的构造函数， 编译器可以使用基类的构造函数完成派生类的构造。 而将基类的构造函数带入派生类的动作 无法选择性地部分带入， 所以，要不就是继承基类全部的构造函数，要不就是一个都不继承(不手动带入)。 此外，若牵涉到多重继承，从多个基类继承而来的构造函数不可以有相同的函数签名(signature)。 而派生类的新加入的构造函数也不可以和继承而来的基类构造函数有相同的函数签名，因为这相当于重复声明。（所谓函数签名就是函数的参数类型和顺序不）

## 右值引用和move语义 ##

**左值和右值**概念的本质区别就是，左值是用户显示声明或分配内存的变量，能够直接用变量名访问，而右值主要是临时变量。

**std::move**是一个用于提示优化的函数，过去的c++98中，由于无法将作为右值的临时变量从左值当中区别出来，所以程序运行时有大量临时变量白白的创建后又立刻销毁，其中又尤其是返回字符串std::string的函数存在最大的浪费。因为并不是所有情况下，C++编译器都能进行返回值优化，所以在C++11中，编码者可以主动提示编译器，返回的对象是临时的，可以被挪作他用. 

``` c++
std::string fileContent = “oldContent”;
s = std::move(readFileContent(fileName));
```
	
对象s在被赋值的时候，方法std::string::operator =(std::string&&)会被调用，符号&&告诉std::string类的编写者，传入的参数是一个临时对象，可以挪用其数据，于是std::string::operator =(std::string&&)的实现代码中，会置空形参，同时将原本保存在中形参中的数据移动到自身。

**std::forward**是用于模板编程中的.当一个临时变量传入形参为T&& t的模板函数时，T被推演为U，参数t所引用的临时变量因为开始能够被据名访问了，所以它变成了左值。这也就是std::forward存在的原因！当你以为实参是右值所以t也应该是右值时，它跟你开了个玩笑，它是左值！如果你要进一步调用的函数会根据左右值引用性来进行不同操作，那么你在将t传给其他函数时，应该先用std::forward恢复t的本来引用性，恢复的依据是模板参数T的推演结果。虽然t的右值引用行会退化，变成左值引用，但根据实参的左右引用性不同，T会被分别推演为U&和U.

使用std::forward的原因：**由于声明为f(T&& t)的模板函数的形参t会失去右值引用性质，所以在将t传给更深层函数前，可能会需要回复t的正确引用行**，当然，修改t的引用性办不到，但根据t返回另一个引用还是可以的。

## C++ 11 STL 标准库 ##

C++ STL库在2003年经历了很大的整容手术 Library Technical Report 1 (TR1)。 TR1 中出现了很多新的容器类 (`unordered_set`, `unordered_map`, `unordered_multiset`, 和 `unordered_multimap`) 以及一些新的库支持诸如：正则表达式， `tuples`，函数对象包装，等等。 C++11 批准了 TR1 成为正式的C++标准，还有一些TR1 后新加的一些库，从而成为了新的C++ 11 STL标准库。这个库主要包含下面的功能：

### 线程库 ###

这就不多说了，以前的STL饱受线程安全的批评。现在好 了。C++ 11 支持线程类了。这将涉及两个部分：第一、设计一个可以使多个线程在一个进程中共存的内存模型；第二、为线程之间的交互提供支持。第二部分将由程序库提供支持。大家可以看看[promises and futures](http://en.wikipedia.org/wiki/Futures_and_promises)，其用于对象的同步。 [async()](http://www.stdthread.co.uk/doc/headers/future/async.html) 函数模板用于发起并发任务，而 [thread_local](http://www.devx.com/cplus/10MinuteSolution/37436) 为线程内的数据指定存储类型。更多的东西，可以查看 Anthony Williams的 [Simpler Multithreading in C++0x](http://www.devx.com/SpecialReports/Article/38883).

### 新型智能指针 ###

C++98 的智能指针是 `auto_ptr`， 在C++ 11中被废弃了。C++11  引入了两个指针类： `shared_ptr` 和 `unique_ptr`。 `shared_ptr`只是单纯的引用计数指针，`unique_ptr` 是用来取代`auto_ptr`。 `unique_ptr` 提供 `auto_ptr` 大部份特性，唯一的例外是 `auto_ptr` 的不安全、隐性的左值搬移。不像 `auto_ptr`，`unique_ptr` 可以存放在 C++0x 提出的那些能察觉搬移动作的容器之中。

### 新的算法 ###

定义了一些新的算法： `all_of()`, `any_of()` 和 `none_of()`。

``` c++
//are all of the elements positive?
all_of(first, first+n, ispositive()); //false
//is there at least one positive element?
any_of(first, first+n, ispositive());//true
// are none of the elements positive?
none_of(first, first+n, ispositive()); //false
```

使用新的`copy_n()`算法，你可以很方便地拷贝数组。

``` c++
int source[5]={0,12,34,50,80};
int target[5];
copy_n(source,5,target); //copy 5 elements from source to target
```

使用 `iota()` 可以用来创建递增的数列。如下例所示：

``` c++
int a[5]={0};
char c[3]={0};
iota(a, a+5, 10); //changes a to {10,11,12,13,14}
iota(c, c+3, 'a'); //{'a','b','c'}
```