.mod.c 是什么文件
============


转自：[http://www.cnblogs.com/felixjia/archive/2011/09/15/2178126.html](https://www.cnblogs.com/felixjia/archive/2011/09/15/2178126.html)

我们可以为代码清单4.1的模板编写一个简单的Makefile：

obj-m := hello.o

并使用如下命令编译Hello World模块：

       make -C /usr/src/linux-2.6.15.5/ M=/driver\_study/ modules

       如果当前处于模块所在的目录，则以下命令与上述命令同等：

         make –C /usr/src/linux-2.6.15.5 M=$(pwd) modules

       其中-C后指定的是Linux内核源代码的目录，而M=后指定的是hello.c和Makefile所在的目录，编译结果如下：

\[root@localhost driver\_study\]# make -C /usr/src/linux-2.6.15.5/ M=/driver\_study/ modules

make: Entering directory \`/usr/src/linux-2.6.15.5'

CC \[M\] /driver\_study/hello.o

/driver\_study/hello.c:18:35: warning: no newline at end of file

Building modules, stage 2.

MODPOST

CC      /driver\_study/hello.mod.o

LD \[M\] /driver\_study/hello.ko

make: Leaving directory \`/usr/src/linux-2.6.15.5'

从中可以看出，编译过程中，经历了这样的步骤：先进入Linux内核所在的目录，并编译出hello.o文件，运行MODPOST会生成临时的hello.mod.c文件，而后根据此文件编译出hello.mod.o，之后连接hello.o和hello.mod.o文件得到模块目标文件hello.ko，最后离开Linux内核所在的目录。

       中间生成的hello.mod.c文件的源代码如代码清单4.7所示。

**代码清单**4.7 模块编译时生成的.mod.c文件

1    #include <linux/module.h>

2    #include <linux/vermagic.h>

3    #include <linux/compiler.h>

4   

5    MODULE\_INFO(vermagic, VERMAGIC\_STRING);

6   

7    struct module \_\_this\_module

8    \_\_attribute\_\_((section(".gnu.linkonce.this\_module"))) = {

9    .name = KBUILD\_MODNAME,

10    .init = init\_module,

11    #ifdef CONFIG\_MODULE\_UNLOAD

12    .exit = cleanup\_module,

13    #endif

14    };

16    static const char \_\_module\_depends\[\]

17    \_\_attribute\_used\_\_

18    \_\_attribute\_\_((section(".modinfo"))) =

19    "depends=";

hello.mod.o产生了ELF（Linux所采用的可执行/可连接的文件格式）的2个节，即modinfo和.gun.linkonce.this\_module。

如果一个模块包括多个.c文件（如file1.c、file2.c），则应该以如下方式编写Makefile：

obj-m := modulename.o

modulename-objs := file1.o file2.o   

\-----------------------------------------------------------------

[http://blog.csdn.net/zhaokugua/archive/2007/11/02/1862500.aspx](http://blog.csdn.net/zhaokugua/archive/2007/11/02/1862500.aspx %204.9)

4.9模块的编译

\----------------------------------------------------------------------

2.4内核中，模块的编译只需内核源码头文件；需要在包含linux/modules.h之前定义MODULE；编译、连接后生成的内核模块后缀为.o。

2.6内核中，模块的编译需要配置过的内核源码；编译、连接后生成的内核模块后缀为.ko；编译过程首先会到内核源码目录下，读取顶层的Makefile文件，然后再返回模块源码所在目录。

**清单2：2.4 内核模块的Makefile模板**

#Makefile2.4
KVER=$(shell uname -r)
KDIR=/lib/modules/$(KVER)/build
OBJS=mymodule.o
CFLAGS=-D\_\_KERNEL\_\_ -I$(KDIR)/include -DMODULE -D\_\_KERNEL\_SYSCALLS\_\_ -DEXPORT\_SYMTAB
  -O2 -fomit-frame-pointer  -Wall  -DMODVERSIONS -include $(KDIR)/include/linux/modversions.h
all: $(OBJS)
mymodule.o: file1.o file2.o
 ld -r -o $@ $^
clean:
 rm -f \*.o

在2.4 内核下，内核模块的Makefile与普通用户程序的Makefile在结构和语法上都相同，但是必须在CFLAGS中定义-D\_\_KERNEL\_\_- DMODULE，指定内核头文件目录-I$(KDIR)/include。有一点需注意，之所以在CFLAGS中定义变量，而不是在模块源码文件中定义，一方面这些预定义变量可以被模块中所有源码文件可见，另一方面等价于将这些预定义变量定义在源码文件的起始位置。在模块编译中，对于这些全局的预定义变量，一般在CFLAGS中定义。

  
**清单3：2.6 内核模块的Makefile模板**

\# Makefile2.6
ifneq ($(KERNELRELEASE),)
#kbuild syntax. dependency relationshsip of files and target modules are listed here.
mymodule-objs := file1.o file2.o
obj-m := mymodule.o 
else
PWD  := $(shell pwd)
KVER ?= $(shell uname -r)
KDIR := /lib/modules/$(KVER)/build
all:
 $(MAKE) -C $(KDIR) M=$(PWD) 
clean:
rm -rf .\*.cmd \*.o \*.mod.c \*.ko .tmp\_versions
endif

KERNELRELEASE是在内核源码的顶层Makefile中定义的一个变量，在第一次读取执行此Makefile时， KERNELRELEASE没有被定义，所以make将读取执行else之后的内容。如果make的目标是clean，直接执行clean操作，然后结束。当make的目标为all时，-C $(KDIR) 指明跳转到内核源码目录下读取那里的Makefile；M=$(PWD) 表明然后返回到当前目录继续读入、执行当前的Makefile。当从内核源码目录返回时，KERNELRELEASE已被被定义，kbuild也被启动去解析kbuild语法的语句，make将继续读取else之前的内容。else之前的内容为kbuild语法的语句, 指明模块源码中各文件的依赖关系，以及要生成的目标模块名。mymodule-objs := file1.o file2.o表示mymoudule.o 由file1.o与file2.o 连接生成。obj-m := mymodule.o表示编译连接后将生成mymodule.o模块。

补充一点，"$(MAKE) -C $(KDIR) M=$(PWD)"与"$(MAKE) -C $(KDIR) SUBDIRS =$(PWD)"的作用是等效的，后者是较老的使用方法。推荐使用M而不是SUBDIRS，前者更明确。

通过以上比较可以看到，从Makefile编写来看，在2.6内核下，内核模块编译不必定义复杂的CFLAGS，而且模块中各文件依赖关系的表示简洁清晰。

  
**清单4： 可同时在2.4 与 2.6 内核下工作的Makefile**

＃Makefile for 2.4 & 2.6
VERS26=$(findstring 2.6,$(shell uname -r))
MAKEDIR?=$(shell pwd)
ifeq ($(VERS26),2.6)
include $(MAKEDIR)/Makefile2.6
else
include $(MAKEDIR)/Makefile2.4
endif

$(function() { setTimeout(function () { var mathcodeList = document.querySelectorAll('.htmledit\_views img.mathcode'); if (mathcodeList.length > 0) { var testImg = new Image(); testImg.onerror = function () { mathcodeList.forEach(function (item) { $(item).before('<span class="img-codecogs">\\\\(' + item.alt + '\\\\)</span>'); $(item).remove(); }) MathJax.Hub.Queue(\["Typeset",MathJax.Hub\]); } testImg.src = mathcodeList\[0\].src; } }, 1000) })

[![](https://profile.csdnimg.cn/8/1/E/3_njuitjf)njuitjf](https://blog.csdn.net/njuitjf)
