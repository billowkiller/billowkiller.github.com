---
layout: post
title: "Nginx architecture(4)--Modularity"
date: 2013-11-26 16:52
comments: true
categories: rework
tags: Nginx architecture 
---

##1. 概述

这章的题目说的有些不对，nginx的模块化只是模块化编程的一个案例，我用它来分析所谓的模块化编程，接下来在其他的例子中也有体现模块化，但是为了形成nginx架构这一系列文章，我还是把它命名为nginx架构——模块化。

<img src="http://i1113.photobucket.com/albums/k512/billowkiller/LinkSource/nginx-logo_zpsabde8e46.png"/>

<!--more-->

好了，言归正传。现代编程的一大特点是：模块化编程，具体来说就是好几个人来完成一个大的的功能：这一个大的功能又拆分为好几个子功能，每一个人做一块子功能，然后导出子功能的API,而大功能的运行实际上就变成子功能导出的API的相互调用。

换言之，模块化编程并不像我们以前开发的强耦合系统，从头到尾一步步或结构化或对象化的把系统功能添加到代码中，各个子功能之间协商好直接通信的接口——可以互相调用的API，子功能和子功能集成后，调试，运行成功就成了完整的系统。这样的系统可以做到软件代码紧凑，开发效率高，如果是互相熟悉，经验丰富实力强的开发团队，可以达到理想的开发速度，和代码质量。

但如果开发人员数量庞大且良莠不齐，那么核心开发者无法兼顾每个子系统的代码质量，一个子系统的失败也许会导致整个项目的失败，这种损失是无法估量的；如果需要开发的子系统纷杂繁多，这样的设计也无法满足日益增多的个性化子功能。

在这样的背景下，模块化编程是十分必要的。优秀的系统，生命周期长的系统，有革命意义的系统，如Linux、eclipse、NetBeans、nginx，它们所利用的都是模块化设计带来的红利，因为模块化系统有一下几个特点：

1. 而模块化应用由小块的、分散的代码块组成，每一块都是独立的。于是，这些代码块可以由不同的团队进行开发，而他们都有各自的生命周期和时间表。最终的成果则可以由另一个独立的个体，即发行者，进行集成。
2. 应用程序（或操作系统）的源代码不再处于某一个开发者完全的掌控之中。源代码被散布至世界各地。毫无疑问，构建这样的软件与传统那种源代码完全在代码库的应用构建是完全不同的。
3. 管理者无需对整个项目的时间表有完全的掌控。不单单是源代码，开发者们也遍布世界各地，并以他们自己的时间表工作。每个人都有这样一个权利：使用一个新版本或旧版本类库的自由。
4. 使用外部库并使用它们组建应用程序，这意味着人们能够花费更少的时间和精力创造更复杂的软件。比如说，一个模块系统中的组件加入一个XML解析器，或者加入某种数据库驱动。

ok，有了这样统一的一个认识后，再来具体谈谈什么是模块化编程。

## 2.模块化编程

百度百科对模块化设计有这样的定义
> 模块化设计是指在对一定范围内的不同功能或相同功能不同性能、不同规格的产品进行功能分析的基础上，划分并设计出一系列功能模块，通过模块的选择和组合可以构成不同的产品，以满足市场的不同需求的设计方法。

它的概念是
> 首先用主程序、子程序、子过程等框架把软件的主要结构和流程描述出来，并定义和调试好各个框架之间的输入、输出链接关系。逐步求精的结果是得到一系列以功能块为单位的算法描述。以功能块为单位进行程序设计，实现其求解算法的方法称为模块化。模块化的目的是为了降低程序复杂度，使程序设计、调试和维护等操作简单化。

这些说法似乎都不够接地气，但它们又能够很好的表现出模块化的特点：总体框架的制定，分功能块的设计减少耦合性，降低代码逻辑复杂性。在我看来，实现一个具有明显物理结构的软件是模块化的，例如带可扩展`插件`的`eclipse`；具有明显分层结构的软件是模块化的，例如`android`手机操作系统；具有明显封装性的软件是模块化的，例如图形引擎`OGRE`。

![android architecture](https://community.freescale.com/servlet/JiveServlet/showImage/38-1135-1687/android_fig1.png)

可以看出，这些软件都有一个特点：**高对比度**，不会和其他软件代码交杂混合，这可以带来很多显而易见的好处。在开发期，一个模块化的设计有利于程序员实现， 使其在实现过程中一直保持清晰的思路，减少潜伏的BUG；而在维护期，则有利于其他程序 员的理解。

从上面的几个例子中也可以看出，良好模块设计的代码，至少分为两种形式：纵向的模块化和横向的模块化。

横向：整个代码架构是由一个核心和若干外围的模块化代码构成。其中核心构成了软件的业务逻辑，子模块负责子功能的实现。核心和子模块之间有直接的接口，子模块间不需要或很少有交互需求。这样在功能的扩展上具有明显的优势，但对逻辑层的设计带来了更高的要求。

纵向：整个库/软件拥有明显的层次之分，从最底层，与应用业务毫无相关的一层，到最顶层，完全对应用进行直接实现的那一层，每一个相对高层的软件层依赖于更底层的软件层，逐层构建。同一层之中也有独立的子模块，子模块彼此之间耦合甚少，这些子模块构成了一个软件层，共同为上层应用提供服务。

下面就来举两个具体的例子。

## 3.nginx的模块化

网上看到的有关C语言模块化程序设计的一些概念，摘抄一下：

> 1. 模块即是一个.c 文件和一个.h 文件的结合，头文件中是对于该模块接口的声明；
> 2. 某模块提供给其它模块调用的外部函数及数据需在.h 中文件中冠以`extern`关键字声明；
> 3. 模块内的函数和全局变量需在.c 文件开头冠以`static`关键字声明；
> 4. 永远不要在.h 文件中定义变量！定义变量和声明变量的区别在于定义会产生内存分配的操作，是汇编阶段的概念；而声明则只是告诉包含该声明的模块在连接阶段从其它模块寻找外部函数和变量。

意思是暴露在头文件中的信息，则可能被当作该头文件所描述模块的接口描述。所以， 在C语言中任何置于头文件中的信息都需要慎重考虑。

C中默认的作用域是全局的，`static`用于限定其修饰对象的作用域，用它去修饰某个函数或变量，旨在告诉：这个函数或变量仅被当前文件（模块）使用，它仅用于本模块实现所依赖，它不是提供给模块外的接口！ **封装内部实现 ，暴露够用的接口，也是保持模块清晰的方式之一。**

C语言较缺乏模块设计的语言机制——良好的接口封装。所以，在C语言中，良好的设计更依赖于程序员自己的功底。

nginx就是一个十分优秀的模块化编程的C语言实现，在nginx中，除了少量的核心代码，其他一切皆为模块。并且模块对使用用户只是暴露了很少的接口或者说指令更合适。nginx的每个模块都可以实现一些自定义的指令，这些指令写在配置文件的适当配置项中，每一个指令在源码中对应着一个 `ngx_command_t`结构的变量，nginx会从配置文件中把模块的指令读取出来放到模块的commands指令数组中，这些指令一般是把配置项的参数值赋给一些程序中的变量或者是在不同的变量之间合并或转换数据。

并且nginx的模块与模块间基本上没有任何的通信措施，拿最常用的http模块来说（这里的http模块是一个泛类，nginx中将模块分为core、event、http和mail四类，用宏定义标识四个分类）。根据使用的情况不同，http模块可以分为处理模块和过滤模块，处理模块用于处理请求并输出内容，过滤模块用于过滤处理模块的输出内容。处理模块在一个请求中只能使用1种，而过滤模块可以使用多个。那么这里要考虑的是，处理模块和核心代码，过滤模块和核心代码，处理模块和过滤模块，过滤模块和过滤模块之间的交互。实际上，过滤模块是用链表串联起来的，而处理模块与过滤模块在不同的阶段处理数据内容，所以现在只需要考虑前两者的交互手段。

以上的特点和需求都可以在nginx的模块化架构最基本的数据结构为`ngx_module_t`中得到体现。这个数据结构是所有模块的共同基础，类似于Java中的接口。

先来看看nginx的模块化架构最基本的数据结构为`ngx_module_t`，在系列文章1中有简略的介绍，现在看看具体的代码。

	struct ngx_module_s{
	   ngx_uint_t    ctx_index;  //分类模块计数器
	   ngx_uint_t    index;      //模块计数器
	
	   ngx_uint_t    spare0;
	   ngx_uint_t    spare1;
	   ngx_uint_t    spare2;
	   ngx_uint_t    spare3;
	
	   ngx_uint_t    version;    //版本
	
	   void          *ctx;       //该模块的上下文，每个种类的模块有不同的上下文
	   ngx_command_t *commands;  //该模块的命令集，指向一个ngx_command_t结构数组
	   ngx_uint_t    type;       //该模块的种类，为core/event/http/mail中的一种
	   //以下是一些callback函数
	   ngx_uint_t    (*init_master)(ngx_log_t *log);      //初始化master
	
	   ngx_uint_t    (*init_module)(ngx_cycle_t *cycle);  //初始化模块
	
	   ngx_uint_t    (*init_process)(ngx_cycle_t *cycle); //初始化工作进程
	   ngx_uint_t    (*init_thread)(ngx_cycle_t *cycle);  //初始化线程
	   void          (*exit_thread)(ngx_cycle_t *cycle);  //退出线程
	   void          (*exit_process)(ngx_cycle_t *cycle); //退出工作进程
	
	   void          (*exit_master)(ngx_cycle_t *cycle);  //退出master
	
	   uintptr_t     spare_hook0;  //这些字段貌似没用过
	   uintptr_t     spare_hook1;
	   uintptr_t     spare_hook2;
	   uintptr_t     spare_hook3;
	   uintptr_t     spare_hook4;
	   uintptr_t     spare_hook5;
	   uintptr_t     spare_hook6;
	   uintptr_t     spare_hook7;
	};

在这些回调函数中有初始化模块的回调函数，在这个回调函数中可以注册模块自身实现的其他函数。在核心代码中，http框架将http的处理阶段分为多个，每个阶段都有phase_handler成员，可以通过初始化时赋值给这个成员来达到注册的效果。这样在处理这个阶段内容的时候，自然就会调用模块实现的方法。

对于过滤模块来说实现机制却又有些不同，前面提到过滤模块是用链表串联起来的，核心代码在遍历过滤链表的时候触发回调函数。每个过滤模块需要实现两个回调函数，类似于：

	ngx_int_t
	ngx_http_send_header(ngx_http_request_t *r)
	{
	    ...
		return ngx_http_top_header_filter(r);
	}
	
	static int
	ngx_http_example_body_filter(ngx_http_request_t *r, ngx_chain_t *in)
	{
	    ...
	    return ngx_http_next_body_filter(r, in);
	}

分别用来过滤响应头和内容。注意这里面的`ngx_http_top_header_filter`和`ngx_http_next_body_filter`，这里可以看出来，各个模块是由它们是由链表串联起来的，那么是怎么实现这个过程的呢，也就是如何注册过滤模块到核心代码中的链表的。

编译后可以找到`ngx_modules.c`文件，里面规定了一个数组

	ngx_module_t *ngx_modules[] = {
	        ...
	        &ngx_http_write_filter_module,
	        &ngx_http_header_filter_module,
	        &ngx_http_chunked_filter_module,
	        &ngx_http_range_header_filter_module,
	        &ngx_http_gzip_filter_module,
	        &ngx_http_postpone_filter_module,
	        &ngx_http_ssi_filter_module,
	        &ngx_http_charset_filter_module,
	        &ngx_http_userid_filter_module,
	        &ngx_http_headers_filter_module,
	        &ngx_http_copy_filter_module,
	        &ngx_http_range_body_filter_module,
	        &ngx_http_not_modified_filter_module,
	        NULL
	};

模块的执行顺序是反向的。也就是说最早执行的是`not_modified_filter`，然后各个模块依次执行。所有第三方的模块只能加入到`copy_filter`和`headers_filter`模块之间。

Nginx执行的时候是怎么安照次序依次来执行各个模块呢？它采用了一种很隐晦的方法，通过局部的全局变量。比如，在每个filter模块，很可能看到如下代码：

	static ngx_http_output_header_filter_pt  ngx_http_next_header_filter;
	static ngx_http_output_body_filter_pt    ngx_http_next_body_filter;
	...
	ngx_http_next_header_filter = ngx_http_top_header_filter;
	ngx_http_top_header_filter = ngx_http_example_header_filter;
	
	ngx_http_next_body_filter = ngx_http_top_body_filter;
	ngx_http_top_body_filter = ngx_http_example_body_filter;

`ngx_http_top_header_filter`是一个全局变量。当编译进一个filter模块的时候，就被赋值为当前filter模块的处理函数。而`ngx_http_next_header_filter`是一个局部全局变量，它保存了编译前上一个filter模块的处理函数。所以整体看来，就像用全局变量组成的一条单向链表。

每个模块想执行下一个过滤函数，只要调用一下`ngx_http_next_header_filter()`这个局部变量。而整个过滤模块链的入口，只要调用`ngx_http_top_header_filter`这个全局变量就可以了。`ngx_http_top_body_filter`的行为与header fitler类似。

所以nginx的模块化可以概括为

1. 核心代码+模块，核心代码负责处理流程的业务逻辑，模块代码负责实现功能。
2. 核心代码在处理业务逻辑时候用全局字段保留信息，用统一的结构体传递信息，用相同的名称调用回调函数。
2. 模块代码采用统一的数据结构，结构中包括统一变量和回调函数，以及与模块上下文相关的保留字段。
