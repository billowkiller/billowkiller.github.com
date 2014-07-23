---
layout: post
title: "C++ string copy-on-write"
date: 2014-07-12 02:18
comments: true
categories: language
tags: [string, c++]
---

不同版本的C++ string实现时不同的，sizeof(string)得出来的结果也不尽相同，其中有种实现采用copy-on-write的方式来共享内存。

引用计数就是string类中写时才拷贝的原理！当第一个类构造时，string的构造函数会根据 传入的参数从堆上分配内存，当有其它类需要这块内存时，这个计数为自动累加，当有类析构时，这个计数会减一，直到最后一个类析构时，此时的RefCnt为1或是0，此时，程序才会真正的Free这块从堆上分配的内存。

string类的内存是在堆上动态分配的，共享内存的各个类指向的是同一个内存区。这块内存区域还包括引用计数，其位于字符串内存区域的上方，通过`_Ptr[-1]`来访问。这样所有共享这块内存的类都可以访问到引用计数，也就知道这块内存的引用者有多少了。

于是，有了这样一个机制，每当我们为string分配内存时，我们总是要多分配一个空间用来存放这个引用计数的值，只要发生拷贝构造可是赋值时，这个内存的值就会加一。而在内容修改时，string类为查看这个引用计数是否为0，如果不为零，表示有人在共享这块内存，那么自己需要先做一份拷贝，然后把引用计数减去一，再把数据拷贝过来。

<!--more-->
具体做法如下：

	//构造函数（分存内存）
	string::string(const char* tmp)
	{
		_Len = strlen(tmp);
		_Ptr = new char[_Len+1+1];
		strcpy( _Ptr, tmp );
		_Ptr[-1]=0;  // 设置引用计数  
	}

	//拷贝构造（共享内存）
	string::string(const string& str)
	{
		if (*this != str){
		this->_Ptr = str.c_str();   //共享内存
		this->_Len = str.szie();
		this->_Ptr[-1] ++;  //引用计数加一
	}

	//写时才拷贝Copy-On-Write
	char& string::operator[](unsigned int idx)
	{
		if (idx > _Len || _Ptr == 0 ) {
			static char nullchar = 0;
			return nullchar;
		}

		_Ptr[-1]--;   //引用计数减一
		char* tmp = new char[_Len+1+1];
		strncpy( tmp, _Ptr, _Len+1);
		_Ptr = tmp;
		_Ptr[-1]=0; // 设置新的共享内存的引用计数
		
		return _Ptr[idx];
	}


	//析构函数的一些处理
	~string()
	{ 
		_Ptr[_Len+1]--;   //引用计数减一
		// 引用计数为0时，释放内存
		if (_Ptr[_Len+1]==0) {
			delete[] _Ptr;
		}
	}

但这样的方式也会造成一些麻烦。容易造成内存访问异常。
**两个例子：**

###动态库
动态链接库中有一个函数返回string类：

	string GetIPAddress(string hostname)
	{
		static string ip;
		……
		return ip;
	}

主程序中动态地载入这个动态链接库，并调用其中的这个函数：

	main()
	{
		//载入动态链接库中的函数
		void(*pTest)();
		void*pdlHandle = dlopen("libtest.so", RTLD_LAZY);   
		pTest = dlsym(pdlHandle, "GetIPAddress");
		//调用动态链接库中的函数
		string ip = (*pTest)(“host1”);
		……
		//释放动态链接库
		 dlclose(pdlHandle);
		……
		cout << ip << endl;
	}

当主程序释放了动态链接库后，那个共享的内存区也随之释放。所以，以后对ip的访问，必然做造成内存地址访问非法，造成程序crash。即使你在以后没有使用到ip这个变量，那么在主程序退出时也会发生内存访问异常，因为程序退出时，ip会析构，在析构时就会发生内存访问异常。

### 多线程 ###

在多线程中，对于多线程来说，引用计数就是一个全局变量。指向同一个buffer的多个string的引用计数有可能变得混乱，从而导致delete异常。尤其是在.h中定义const string A = "XXXX"， 如果多个对象都引用了A，则可能在多线程中出现问题。