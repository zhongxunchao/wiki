这篇翻译我曾经在csdn和简书上发表.
翻译来源：[AppleScript Introduction](https://developer.apple.com/library/mac/documentation/AppleScript/Conceptual/AppleScriptLangGuide/introduction/ASLR_intro.html)

注：AppleScript准确说是苹果脚本，是Apple公司推出来的支持mac的一种脚本语言，支持mac自带的脚本编辑器。

##AppleScript语言介绍

    这份文档是关于AppleScript的指导－包括它的约定词语，语法，关键字和其他元素。这里主要支持AppleScript 2.0和OS X10.5以上的版本。

   AppleScript2.0可以使用AppleScirpt1.1到1.10.7开发的脚本，任何AppleScript1.5以上的脚本扩展，以及OS X7.1以上的脚本应用程序。同时，如果AppleScript2.0编写的脚本如果没有用到任何之前版本没有用到的特征，那么它也是可以被AppleScript1.1之后的版本使用的。



  AppleScript是什么？

AppleScript是Apple公司推出的一种脚本语言。它允许用户直接控制mac上的应用程序。与此同时，它本身也可以成为OS X系统的一部分。面对一些重复性的任务，你可以编写代码使得它们自动完成。你也可以和应用程序结合起来，去完成更复杂的工作。

脚本化的应用是指可以被脚本控制的应用。具体到AppleScript，对应用程序内的消息的响应称为Apple events.脚本对应用程序发出指令就会触发Apple events.

AppleScript仅仅提供少量的指令，但是它提供一整套框架--那些由应用程序和系统脚本提供的的特殊指令。

在这份指导文件中，绝大部分脚本案例和代码模块都使用了Finder，OS X脚本模块或者脚本化应用程序中的脚本特性。比如说TextEdit。

##本文档的阅读范围

如果你需要编写AppleScript脚本，或者需要创建脚本化应用程序，了解脚本背后的机制，那么你有必要阅读本文档。

《AppleScript 语言指导》假定你对《AppleScript Overview》中的关于AppleScript的高级信息十分了解。

##本文档的组织结构

这份指导在一系列的章节和附录中描述了AppleScript语言。

在最初的五个章节中，我们描述了AppleScript的组成和基本概念，并且提供了基本类型和日常程序的概览：

AppleScript词汇定义描述了特性，符号，关键字以及构成AppleScript脚本的其他语言元素;

AppleScript基础描述了本指南内,术语和规则的基本概念；

变量和属性描述了处理变量和属性时的问题，包括AppleScript对它们的声明和解释范围；

脚本对象描述了对脚本对象的定义，初始化，继承以及向他们发送命令；

关于处理器（此处翻译为函数句柄或许更准确）提供了使用函数的信息，这些函数可以分解和复用代码。

接下来的几个章节提供了AppleScript语言的参考信息：

类参考描述了脚本中通用对象的定义；

命令行参考描述了在任何脚本中都能使用的命令；

参考表单描述了在应用程序及其他容器中，指定一个或一组对象的语法；

操作参考提供了AppleScript中的各种操作和使用方法，以及常见操作的使用细节；

操作声明参考描述了控制其他声明怎么执行，什么时候执行的声明。它包括了标准状态的声明，错误处理以及其他操作的声明；

处理器参考则说明了定义和使用这些函数句柄的语法，也包括其他的一些场景。

接下来的一个章节OS X系统中与AppleScript相关的一些特性：

文件行为参考描述了你怎么编写脚本函数以及怎么把它和特定的文件关联起来，比如当文件内容被修改时就会被调用的函数。

附录部分则提供了AppleScript语言的额外信息，以及你应该怎么处理异常情况：

AppleScript关键字列出了AppleScript中的关键字和它们的简单描述，相关的信息；

异常编号和异常信息列出了你在AppleScript脚本中可能遇到的异常的编号和描述信息；

异常处理提供了运用try方法和error方法处理异常的详细案例；

双括号描述了什么时候你会碰到双括号和怎么运用它们；

不支持条款则描述了那些在AppleScript过去的版本中支持，而现在不再支持的情况。

##本文档约定

所有术语在它们定义的地方都使用黑体。

在语法表达中将用到以下术语：

       基本语言要素	有的元素就用普通字体，那么它的含义就和你想的一样。比如说你看到了像"+"或者"&"这样的字符，那就是它们的本来含义
          占位符	斜体文本表示它是可以用合适的值替代的
          可选值[]
中括号表示内部的元素是可选的
          数组（）	将一组元素放在一块。它本身也可以作为函数参数。
          可选值[]...	后面三个点表示你可以把这组元素重复多次
          a|b|c	你有一组元素可以选择，但是最后必须确定一个。它经常被用在小括号或中括号内
     

      脚本内的文件名

本文档中，大部分用到的文件名都是带扩展名的，比如文本文档就有扩展名rtf。使用扩展名在Finder参考标准中是有必要的。

要使用你电脑上的案例，你可能需要更改它们的文件名。

##你还需要看的东西
下面这些苹果官方的文档也会提供AppleScript的帮助：

快速看一遍Getting Started with AppleScript 对脚本使用者和开发者都是很有用的；

看一下AppleScript Overview，包括Scripting with AppleScript这一章，里面有对AppleScript的高度概括和相关技术；

Getting Started With Scripting & Automation中有OS X系统中常用的脚本技术；

AppleScript Terminology and Apple Event Codes列出了Apple规定的很多脚本场景。

如果你要学习AppleScript语言和脚本的更多信息，可以去看看bookstore和网上的复杂的第三方文档。