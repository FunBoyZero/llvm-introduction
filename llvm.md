对应Alex平台的LLVM后端
===
alex是一种32位定长指令集，用于清华大学操作系统课程的教学。整个alex工程由三个部分组成，alex模拟器，基于LLVM的alex编译器以及运行于alex之上的操作系统。其中，模拟器、编译器都只是完成了大部分工作。在清华开始的接近3周时间里，我对alex的模拟器以及编译器后端进行了一定的学习，下面是一个简单的学习总结。

### alex模拟器部分
- alex模拟器是基于nodejs的模拟器。（暂时有点懒得写，先将llvm简单写一下，模拟器部分后面再补充吧）
	

### alex编译器后端
	alex编译器后端基于llvm所写，根据llvm架构，操作系统的代码会先被llvm前端（此处使用clang）翻译为llvm IR中间语言，而后由alex编译器后端翻译为目标机器码。此项目致力于基于llvm框架下开发alex后端，即从IR开始，翻译为最终的alex代码的部分。

1. 重要参考文档：
	- http://jonathan2251.github.io/lbd/ ，里面是CPU0框架（仿MIPS框架）编译器后端的实现，也是此Alex框架实现时的重要参考对象，里面内容比较详细，重点看前两章，这两章是llvm实现的总体介绍以及框架搭建，后面的几章都是具体指令及功能的实现，看懂前两章后基本就已经入门，后面的会比较好理解，可以作为参考手册使用
	- https://github.com/wuye9036/ChsLLVMDocs/blob/master/CodeGen.md 某作者翻译的llvm文档中最重要的Target-Indepenent Code Generator，中文的毕竟好理解，同时也有作者自己的理解注释
	- http://llvm.org/devmtg/2014-10/Videos/Building%20an%20LLVM%20backend-720.mov 一个介绍llvm后端实现的视频，比较有用，只是有一个讲者的口音略难懂，但ppt已经很好了
	- http://llvm.org/docs/doxygen/html/classllvm_1_1Function.html 众多llvm类内函数源码，有注释，可帮助理解众多llvm函数
	- llvm.org 系列，里面的document有所有的llvm需要的文档，比较重要的有 http://llvm.org/docs/LangRef.html （讲解IR语言） http://llvm.org/docs/LangRef.html （tablegen语言） http://llvm.org/docs/CodeGenerator.html （上面已附翻译版） http://llvm.org/docs/WritingAnLLVMBackend.html#basic-steps （backend 写法，基于SPARC平台，较为范范，可以当做参考，结合CPU0文档第二章使用很有效果

2. llvm后端开发流程简介
	为了开发后端，llvm为我们提供了大量的类和函数。从IR代码开始，使代码不断通过各种pass，从而“lower”至更接近最终机器码的形式。具体流程如下
	
	1）首先，C语言翻译为的IR是SSA的形式，即静态单赋值，而后翻译为IR DAG的形式
	2）IR DAG翻译为Machine Intruction DAG（instruction selection）
	3）Machine Instruction翻译为MCInst并打印
	
	上述流程只是最重要的部分，其实中间包含了很多如寄存器分配、各种优化的内容（alex并没有做很多优化，只是继承了llvm自己提供的一些优化手段）在具体步骤开始之前，需要先写代码搭建好整个alex后端实现的大体框架，具体函数可以在后面再实现。（框架搭建方式可参考 http://jonathan2251.github.io/lbd/ 中Backend Structure一章）所有alex平台编译的代码都放在llvm/Target/Alex中，里面要做的事情包括：
	
	- 定义alex的寄存器，编写AlexRigisterInfo.td，llvm会在之后自动使用tablegen工具将它生成为AlexGenRegisterInfo.inc，通过简单的td语法(tablegen语法可参考	http://llvm.org/docs/TableGen/LangIntro.html )定义好所有的alex寄存器。而后编写AlexRegisterinfo.cpp和.h，里面包含上面的inc文件，于是所有寄存器相关的信息及处理函数被包含在此类里面。
	- 定义alex指令集，编写AlexInstrInfo.td，同样会在tablegen的帮助下生成AlexGenInstrinfo.inc，然后编写AlexInstrinfo.cpp和.h包含.inc来提供接口
	- 编写AlexTargetMachine.cpp和.h，这个文件是整个llvm生成的入口，它包含两个重要的类：AlexTargetMachine和AlexSubTarget，AlexTargetMachine包含AlexSubTarget类的指针，而AlexSubTarget类内部则包含AlexInstrInfo类指针、AlexRegisterInfo指针、AlexTargetLowering等等类的指针，这些类都是用来完成llvm具体的各种功能。
	- 编写AlexISelLowering.cpp和.h和AlexISelDagToDag.cpp和.h，后者很好理解，就是指挥AlexInstrInfo.td内定义的各种pattern将IR DAG（IR DAG就我理解是机器自动生成的）翻译为Machine Instruction DAG的类（使用select和selectaddr函数），而前者则负责将一些高层内容，如函数形参，函数返回值，跳转表、全局变量等机器无法直接生成IR DAG的内容手动转化为IR DAG，好令AlexISelDagToDag进行处理
	- 编写AlexInstPrinter等MC层（machine code）的函数，包括很多，这些函数负责生成最终的机器指令以及汇编码以及ELF文件，实现的方式较为固定，代码大部分可以直接复制。在此没有细看，上面提供的中文文档中对于MC层有一个大概的讲解，具体代码及实现过程可参照CPU0第二章的文档。（其中很重要的一个部分是需要在MCTargetDesc.cpp中进行各种MC层函数的注册）
	- 编写AlexTargetObjectFile.cpp和.h，注册整个平台后端编译器，并将targetMachine和它的MC层结合在一起，至此，Instruction Selection层和MC层合二为一，可以正常工作。
	- 注：有了上述函数及文件，llvm后端框架理论上已经搭建结束，后面要做的是加入对于各种IR指令的翻译支持，但是实际上在结束之前，需要立刻加入对于IR函数返回语句的翻译支持，因为整个C语言处处都是函数，所以翻译出来的IR更是以各种call和ret组成，如果没有加入对于ret和call指令的处理，实际上是没有办法成功编译任何一个简单的C文件至Alex机器码的。
	
	下面是以增加llvm对于函数调用的支持为例，进行llvm后端开发的分步介绍（由于东西较多，此处会略讲，只包括我觉得上述参考文档没有详细说明的部分）

1） SSA翻译为IR DAG
	
	（1）首先需要定义一个AlexCallingConv.td文件，里面定义为函数形参及返回值分配location的方式。
	（2） 在AlexISelLowering.cpp中内部定义了AlexCC类，里面保存了对于函数形参以及返回值如何分配的信息，并具有一个重要的私有对象CCState类型的CCInfo，里面包含了函数的形参及返回值的location分配信息（通过CCState私有对象Locs和ByValRegs）。重要的函数包括analyzeReturn等各种analyze函数以及handleByValArg函数，前者会使用AlexCallingConv.td中定义的参数分配方式为return函数分配存储location（栈+offset或寄存器），后者会为ByVal类型的参数分配空间。（此处各种自定义对象和自定义类型很多，需要慢慢看，推荐还是使用clion或者eclipse之类的IDE查看代码而不是vim，函数之间位置切换效率较高，此处详解略）
	（3） 在AlexISelLowering.cpp中定义了AlexTargetLowering类，最重要的lowerFormalArguments以及lowerReturn两个函数，分别在内部调用了AlexCC类中的analyze系列函数来进行函数形参以及返回值location的分配，然后更新了Chain节点，最后调用dag类的getNode为形参以及返回值创建了IR的SDNode节点。至此，lower函数参数成功。（这里要注意每个node的命名空间，不同命名空间代表了不同阶段的node，如AlexISD：：空间必定是IR node）
	  注：函数的翻译是最重要的指令翻译，一定要细看，虽然内容较为乱和复杂。可参考CPU0第二章以及参考资料中的视频来整理思路

2） Instruction Selection
	
	上一步中，所有的IR DAG已经构建完成，这一部分基本上就是在写AlexInstrInfo.td进行到Machine Dag的匹配并完成AlexISelDagToDag.cpp了，后者基本上就是select函数完成的问题。至于前者，虽然内容众多，但是文档中讲解比较详细了，此处只解释几个我觉得有些卡壳的地方：
	（1）PatFrag基本上就是匹配用，将一种类型通过匹配转化为另一种类型，如一种SDNode转为另一种SDNode；而PatFrag的子类PatLeaf以及ImmLeaf则是限制用，基本上是为了指明操作数的类型
	（2）所有伪指令都是需要再一次在AlexInstrInfo.cpp中通过函数转化的，包括此处的ret，在.td中匹配后会再次回到.cpp中通过expandPostRAPseudo函数转化，再通过调用get函数获取Ret对应的非伪指令（因为伪指令是没有操作码的，非伪指令才是真正的指令）
	

（待续）
	