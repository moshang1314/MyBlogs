# 1 编辑器gedit介绍

gedit是一个GNOME桌面环境下兼容UTF-8的文本编辑器。它使用GTK+编写而成，因此它十分简单易用，有良好的语法高亮，对中文支持很好，支持gb2312、gbk在内的多种字符编码。gedit是一个自由软件。

这是一个Linux下的纯文本编辑器，类似于Windows下的写字板程序，在不需要特别复杂的编程环境下，作为基本的文本编辑器比较合适。

示例：

**gedit 1.txt**

# 2 编辑器vim

vi(Visual interface)编辑器是Linux系统中最常用的文本编辑器，vi在Linux界有编辑器之神的美誉，几乎所有的Linux的发行版都包含vi程序。

vi工作在字符模式下，不需要图形界面，非常适合远程及嵌入式工作，是效率很高的文本编辑器，尽管在Linux上也有很多图形界面的编辑器可用，但vi的功能是那些图形编辑器所无法比拟的。

vim是vi的升级版，它不仅兼容vi的所有指令，而且还有一些新的特性，例如vim可以撤销无限次、支持关键词自动完成、可以用不同的颜色来高亮你的代码。vim普遍被推崇为类vi编辑器总最好的一个。

## 2.1 vim修改文件机制

在使用vim打开文件时，会把文件复制一份，编辑副本，最后完成时副本替换正文。

## 2.2 衍生的问题

如果在编辑过程中异常退出（如vim编辑中中途关机，或中途关闭终端）的话，会导致副本没有覆盖原文件，而出现两个文件共存的情况，而对于Linux的描述，就是所谓的交换文件.swap，而下次使用vim编辑该文件时，自然而然会再次创建副本，且不会覆盖。例如，第一次产生.swap，第二次产生.swo文件，第三次为.swn文件。而这中途就会检测到之前到之前异常退出而导致异常存在的副本文件，所以会红框报错，提示。

## 2.3 解决方式

1、临时解决，只为进入文件编辑

点击两次enter键会出现选择提示，点击E或者R可以进入编辑界面，但再次进入文件时依然会有提示

* O：只读打开文件
* E：继续编辑
* R：恢复为保存内容编辑
* D：删除.swap文件进入编辑
* Q：退出当前文件编辑
* A：退出vim程序

2、永久解决

可以先恢复先前修改而为保存的内容：vim -r  .passwd.swp

然后删除未删除的副本文件：rm -rf .passwd.swp

Ctrl + h可以显示隐藏文件

## 2.1 vim工作模式

vi有三种基本工作模式：命令模式、文本输入模式（编辑模式）、末行模式

![image-20220605004458785](C:\Users\moshang\AppData\Roaming\Typora\typora-user-images\image-20220605004458785.png)

### 2.1.1 命令模式

任何时候，不管用户处于何种模式，只要按一下ESC键，即可使vi进入命令模式。我们在shell环境（提示符$）下输入启动vim命令，进入编辑器时，也是处于该模式下。

在命令模式下，用户可以输入各种合法的vi命令，用于管理自己的文档。此时从键盘上输入的任何字符都被当做编辑命令来解释，若输入的字符是合法的vi命令，则vi在接受用户命令之后完成相应的动作。但需要注意的是，所输入的命令并不在屏幕上显示出来。若输入的字符不是vi的合法命令，vi会响铃报警。

### 2.1.2 编辑模式

在命令模式下插入命令i（I）、附加命令a（A）、打开命令o（O）、替换命令s（S）都可以进入文本输入模式，此时vi窗口的最后一行会显示“插入”。

在该模式下，用户输入的任何字符都被vi当做文件内容保存起来，并将其显示在屏幕上。在文本输入过程中，若想回到命令模式下，按键ESC即可。

### 2.1.2 末行模式

末行模式下，用户可以对文件进行一些附加处理。尽管命令模式下的命令可以完成很多功能，但要执行一些如字符串查找、替换、显示行号等操作还是必须要进入末行模式的。

在命令模式下，输入冒号即可进入末行模式。此时vi窗口的状态行会显示出冒号，等待用户输入命令。用户输入完成后，按回车执行，知乎vi编辑器会自动返回命令模式下。

## 2.2 vim教程

vimtutor

# 3 vim 基本操作

## 3.1 打开文件

**vim filename：**打开或新建文件，并将光标置于第一行行首，如果文件不存在，则会新建文件。

## 3.2 编辑文件

如果通过vi打开一个已经存在的文件，首先进入命令模式，此时输入的任何字符都被视为命令，不能输入内容。

在命令模式下输入i

##  3.3 保存文件

一定要先退出插入模式（按ESC）然后（小写状态下），shift + zz（按住“shift” + 按两下“z"键），或者（大写状态下：ZZ）即可保存退出当前文件。

# 4 vim实用操作

## 4.1 命令模式下的操作

### 4.1.1 切换到编辑模

按键功能：

* i：光标位置左边插入文字
* I：光标所在行首插入文字
* o：光标下一行插入文字（新行）
* O：光标上一行插入文字（新行）
* a：光标位置右边插入文字
* A：光标所在行尾插入文字
* s：删除光标后边的字符，从光标当前位置插入
* S：删除光标所在当前行，从行首进行插入

### 4.1.2 光标移动

按键功能：

* Ctrl + f： 向前滚动一个屏幕

* Ctrl + b：向后滚动一个屏幕

* gg： 到文件第一行行首

* G： 到文件最后一行行首

* mG或mgg： 到指定行 ，m为目标行数

* 0（数字）：光标移动到行首（第一个字符位置）

* $：光标移动到行尾

* l(小写L)：向右移动光标

* h：向左移动光标

* k：向上移动光标

* j：向下移动光标

* ^：光标移动到行首（第一个有效字符位置）

### 4.1.3 复制粘贴

按键功能

* [n]yy：复制从当前行开始的n行
* p：把粘贴板上的内容插入到当前行

### 4.1.4 删除

按键功能：

* [n]x：删除光标后n个字符

* [n]X：删除光标前n个字符
* D：删除光标所在行开始到此行行尾的字符
* [n]dd：删除从当前行开始的n行（准确来讲，是剪切，剪切不粘贴即为删除）
* dG：删除光标所在位置到文件尾部的所有字符
* dw：删除光标开始位置的一个字（单词），包含光标所在的字符
* d0（0为数字）：删除光标前本行所有内容，不包含光标所在字符
* dgg：删除光标所在开始到文件首行第一个字符的所有字符

### 4.1.5 撤销和恢复

按键功能

* .（点）：执行上一次操作

* **u：撤销前一个命令**

* Ctrl + r：反撤销

* 100 + . ：执行上一次操作100次

### 4.1.6 保存退出

按键功能：

ZZ：保存退出

### 4.1.7 查找

按键功能：

/字符串：从当前光标位置向下查找（n，N查找内容切换，n下一个，N上一个）

?字符串：从当前光标位置向上查找（n，N查找内容切换）

### 4.1.8 替换

按键功能：

r：替换当前字符

R：替换当前行光标后的字符（ESC退出替换模式）

### 4.1.9 可视模式

按键功能：

* v：按字符移动，选择文本，可配合h、j、k、l选择内容，使用d删除，使用y复制
* Shift + v：行选（以行为单位），可配合h、j、k、l选择内容，使用d删除，使用y复制
* Ctrl + v：列选，选中文本，可配合h、j、k、l选择内容，使用d删除，使用y复制

### 4.20 格式对齐

* gg(把光标定位到最上面)，V（进入VISUAL模式），shift+g(选中整篇文本)，然后=。
* gg=G

### 4.21 其它

* %：跳转到匹配的括号
* *：跳转到调用函数的定义

## 4.2 末行模式下的操作

### 4.2.1 保存退出

按键功能：

* :wq：保存退出  w表示保存，q表示退出
* :x(小写)：保存退出
* **:w filename 保存到指定文件**
* :q：退出，如果文件修改但没有保存，会提示无法退出
* :q!：退出，不保存

### 4.2.2 替换

* :s/abc/123/：光标所在行的第一个abc替换为123
* :s/abc/123/g：光标所在行所有abc替换为123
* :1,10s/abc/123/g：将第一行到第10行的abc全部替换成123
* :%s/abc/123/g：当前文件的所有abc替换为123
* :%s/abc/123/gc：同上，但是每次替换需要用户确认
* :1,$s/abc/123/g：当前文件的所有abc替换为123

### 4.2.3 分屏

按键功能：

* :sp：当前文件水平分屏
* :vsp：当前文件垂直分屏
* :sp 文件名：当前文件和另一个文件水平分屏
* :vsp 文件名：当前文件和另一个文件垂直分屏
* ctrl-w-w：在多个窗口切换光标
* :wall/:wqall/:qall：保存/保存退出/退出所有分屏窗口
* vim -O a.c b.c 垂直分屏
* vim -o a.c b.c 水平分屏

## 4.3 其它扩展用法

* :!man 3 printf：在vim中执行命令（q退出）这里是执行查看手册命令，**：!命令**
* :r !ls -l：将ls执行的结果写入当前文件中 ，**:r 可以把命令结果写到当前文件中**
* :r /etc/passwd：将/etc/passwd文件中的内容写入到当前文件中
* :w /tmp/txt 将当前文件写入到/tmp/txt文件中
* :w! /tmp/txt 强制将当前文件内容写入到/tmp/txt文件中
* :1,10s/^/\/\/g：将第1行到第10行行首添加//(**^表示行首**)/\/\转义字符
* :1,10s#^#//#g：将第1行到第10行首添加//（#可以临时代替/分隔）
* :%s/;/\r{\r\treturn0;\r}\r/g：将；替换成{return 0;}
* :1,10s#//##g：将第1行到10行行首去掉//（#可以临时代替/分隔）

## 4.4 配置文件

局部配置文件（推荐）

vim ~/.vimrc

全局配置文件：

sudo vim /etc/vim/vimrc

## 4.5 快捷键

Ctrl + p：自动补全

选中 + =：自动对齐

Ctrl + ]：代码跳转

# 5 GCC编译器简介

编辑器（如vi、记事本）是指我们用它来写程序的（编辑代码），而我们写的代码语句，电脑是看不懂的，我们需要把它转成电脑能懂的语句，编译器就是这样的转化工具。就是说，我们用编辑器编写程序，右编译器编译后才可以运行！

编译器是将易于编写、阅读和维护的高级计算机语言翻译为计算机语言能解读、运行的低级机器语言的程序。

GCC（GNU Compiler Collection，GNU编译器套件），是由GNU开发的编程语言编译器。gcc原本作为GNU操作系统的官方编译器，现已被大多数Unix操作系统（如Linux、BSD、Mac OS X等）采纳为标准的编译器，gcc同样适用于微软的Windows。

gcc最初用于编译c语言，随着项目的发展gcc已经成为了能够编译C、C++、Java、ADa、fortran、Object C、Object C++、Go语言的编译器大家族。

编译命令格式：

gcc [options] file

g++ [options] file

* 命令、选项和源文件之间使用空格分隔
* 一行命令中共可以有零个、一个或多个选项
* 文件名可以包含文件的绝对路径，也可以使用相对路径
* 如果命令中不包含输出可执行文件的文件名，可执行文件的文件名会自动生成一个默认名称，Linux平台为a.out，Windows平台为a.exe

## 5.1 GCC、gcc、g++三者的关系

gcc（GUN C Compiler）是GCC中的c编译器，而g++（GUN C++ Compiler）是GCC中的c++编译器。
  gcc和g++两者都可以编译c和cpp文件，但存在差异。gcc在编译cpp时语法按照c来编译但默认不能链接到c++的库（gcc默认链接c库，g++默认链接c++库）。g++编译.c和.cpp文件都统一按cpp的语法规则来编译。所以一般编译c用gcc，编译c++用g++。后文有时间会继续深入探讨区别。

# 6 GCC工作流程和常用选项

gcc编译器从拿到一个c源文件到生成一个可执行程序，中间一共经历了四个步骤：

![](https://cdn.jsdelivr.net/gh/moshang1314/myBlog@main/image/layout-demo-2-600x88.png)

四个步骤并不是gcc独立完成的，而是内部调用了其他的工具，从而完成了整个工作流程：

![](https://cdn.jsdelivr.net/gh/moshang1314/myBlog@main/image/clip_image002-1527509458588.png)

gcc工作流程

> ls 1hello.c
>
> 第一步: 进行预处理(**不指定目标文件会直接输出到标准输出)**
>
> gcc -E 1hello.c -o 1hello.i
>
> 第二步: 生成汇编文件
>
>  gcc -S 1hello.i -o 1hello.s
>
> 第三步: 生成目标代码（**不指定目标文件名则默认以原文件名 + .o命名**）
>
> gcc -c 1hello.s -o 1hello.o
>
> 第四步: 生成可以执行文件
>
> gcc 1hello.o -o 1hello 第五步: 执行 deng@itcast:~/share/3rd/1gcc$ ./1hello hello itcast

直接将源文件生成一个可以执行文件

> gcc 1hello.c -o 1hello 
>
>  ./1hello hello itcast

如果不指定输出文件名字, gcc编译器会生成一个默认的可以执行a.out

>  gcc 1hello.c
> ls 1hello 1hello.c 1hello.i 1hello.o 1hello.s a.out deng@itcast:~/share/3rd/1gcc$ ./a.out
> hello itcast



| 选项          | 作用                                                         |
| :------------ | :----------------------------------------------------------- |
| -o file       | 指定生成的输出文件名为file                                   |
| -E            | 只进行预处理                                                 |
| -S            | 只进行预处理和编译                                           |
| -c            | 只进行预处理、编译和汇编                                     |
| -v / –version | 查看gcc版本号                                                |
| -g            | 包含调试信息                                                 |
| -On n=0~3     | 编译优化，n越大优化得越多                                    |
| -Wall         | 提示更多警告信息                                             |
| -D            | 编译时定义宏                                                 |
| -M            | 自动寻找源文件中包含的头文件，并生成一个依赖关系             |
| -MM           | 同上，但依赖文件中不包含标准库的头文件                       |
| -std=c11      | 设置编译标准                                                 |
| -I（大写i）   | 指定头文件查找目录                                           |
| -l（小写L）   | 指定库文件或库文件目录，在/lib和/usr/lib和/usr/local里的库直接用-l参数就能链接 |
| -L            | 如果库文件没有放在上面的三个目录里，需要用-L指定库文件所在目录 |

显示所有警告信息

> gcc -Wall test.c

将警告信息当做错误处理

> gcc -Wall -Werror test.c

测试程序（-D选项）：

```
#include <stdio.h>

int main(void)
{
	printf("SIZE:%d\n", SIZE);
	
	return 0;
}
```

> deng@itcast:~/test$ gcc test.c -DSIZE=10
>
> deng@itcast:~/test$ ./a.out
>
> SIZE: 10

# 10 静态链接和动态链接

链接分为两种：静态链接、动态链接

## 10.1 静态链接

静态链接：由链接器在链接时将库的内容加入到可执行程序中。

优点：

* 对运行环境的依赖性较小，具有较好的兼容性

缺点：

* 生成的程序比较大，需要更多的系统资源，在装入内存是会消耗更多的时间
* 库函数有了更新，必须重新编译应用程序

## 10.2 动态链接

动态链接：链接器在链接时仅仅建立与所需库函数之间的链接关系，在程序运行时才将所需资源调入可执行程序

优点：

* 在需要的时候才会调入对应的的资源函数
* 简化程序的升级；有着较小的程序体积
* 实现进程之间的资源共享（避免重复拷贝）

缺点：

* 依赖动态库，不能独立运行
* 动态库依赖版本问题严重

## 10.3 静态、动态编译对比

前面我们编写的应用程序大量用到了标准库函数，**系统默认采用动态链接的方式进行编译程序，若想采用静态编译，加入-static参数**。

以下是分别采用动态编译、静态编译时文件对比：

测试程序(test.c)如下：

```C++
#include <stdio.h>

int main(void)
{
	printf("hello world\n");
	
	return 0;
}
```

编译：

> gcc test.c -o test_share
>
> gcc -static test.c -o test_static

结果对比：

![](https://cdn.jsdelivr.net/gh/moshang1314/myBlog@main/image/1527510332615.png)

参考博客：[静态链接库与动态链接库](https://blog.csdn.net/freestyle4568world/article/details/49817799)

# 11 静态库和动态库简介

所谓“程序库”，简单说，就是包含了数据和执行代码的文件。其不能单独执行，可以作为其它执行程序的一部分来完成某些功能。

库的存在可以使得程序模块化，可以加快程序的再编译，可以实现代码重用，可以使得程序便于升级。

程序可分为静态库(static library)和共享库(shared library)。

## 11.1 静态库的制作和使用

静态库可以认为是一些目标代码的集合，是在可执行程序运行前就已经加入到可执行文件中，成为可执行文件的一部分。按照习惯，一般以**“.a”**作为文件后缀名。静态库的命名一般分为三个部分：

* 前缀：lib
* 库名称：自己定义即可
* 后缀：.a

所以最终的静态库的名字应该为：libxxx.a

**1）静态库制作**

![](https://cdn.jsdelivr.net/gh/moshang1314/myBlog@main/image/clip_image002-1527510689190.jpg)

步骤1：将c源文件生成对应的.o文件

> gcc -c add.c -o add.o
> gcc -c sub.c -o sub.o deng@itcast:~/test/3static_lib$ gcc -c mul.c -o mul.o gcc -c div.c -o div.o

步骤2：使用打包工具ar将准备好的.o文件打包为.a文件libtest.a

> ar -rcs libtest.a add.o sub.o mul.o div.o

**在使用ar工具的时候需要添加参数：rcs**

* r 更新
* c 创建
* s 建立索引

**2）静态库的使用**

静态库制作完成后，需要将.a文件和头文件一起发布给用户。

假设测试文件为main.c，静态库文件为libtest.a头文件为head.h

编译命令：

> gcc test.c -L./ -I./ -ltest -o test

参数说明：

* -L：表示要链接的库所在目录
* -I./：（大写i）表示指定头文件的目录为当前目录
* -l(小写L)：指定链接时需要的库，去掉前缀和后缀

![](https://cdn.jsdelivr.net/gh/moshang1314/myBlog@main/image/clip_image002-1527510920275.jpg)

# 12 动态库的制作和使用

共享库在程序编译时并不会被链接到目标代码中，而是在程序运行时才被载入。不同的应用程序如果调用相同的库，那么内存里只需要有一份该共享库的实例，规避了空间浪费问题。

动态库在程序运行时才被载入，也解决了静态库对程序的更新、部署和发布带来的麻烦。用户只需要更新动态库即可，增量更新。

按照习惯，一般以“.so"作为文件名后缀。共享库的命名一般分为三个部分：

* 前缀：lib
* 库名称：自己定义即可
* 后缀：.so

所以最终动态库的名字应该为：libxxx.so

![](https://cdn.jsdelivr.net/gh/moshang1314/myBlog@main/image/clip_image002-1527511145606.jpg)

1）动态库的制作

步骤一：生成目标文件，此时要加编译选项：-fPIC （fpic）

>  gcc -fPIC -c add.c
>
> gcc -fPIC -c sub.c 
>
> gcc -fPIC -c mul.c
>
>  gcc -fPIC -c div.c

参数：-fPIC创建与地址无关的编译程序（pic，position independent code），是为了能够在多个应用程序间共享。

步骤二：生成共享库，此时要加链接器选项：-shared（指定生成动态链接库）

> gcc -shared add.o sub.o mul.o div.o -o libtest.so
>
> 合并为：
>
> **gcc -shared -fpic -o libtest.so add.cpp sub.c mul.c div.c**

步骤三：通过nm命令查看对应的函数

nm是names的缩写，nm命令主要是用来列出某些文件（目标文件和可执行文件）中的符号（说白了就是一些函数和全局变量等）。

> nm libtest.so | grep “add” 
>
> nm libtest.so | “grep sub”

ldd查看可执行文件的依赖的动态库

> ldd test
>
>  linux-vdso.so.1 => (0x00007ffcf89d4000) libtest.so => /lib/libtest.so (0x00007f81b5612000) libc.so.6 => /lib/x86_64-linux-gnu/libc.so.6 (0x00007f81b5248000) /lib64/ld-linux-x86-64.so.2 (0x00005562d0cff000)

**2）动态库测试**

引用动态库编译成可执行文件（与动态库方式一样）

> gcc test.c -L. -I. -ltest

参数说明：

* -L：表示要链接的库所在目录
* -I./：（大写i）表示指定头文件的目录为当前目录
* -l(小写L)：指定链接时需要的库，去掉前缀和后缀

然后运行：./a.out，发现竟然报错了！

> ./a.out：error while loading shared libraies：libtest.so：cannot open shared file：NO such file or directory

* 当系统加载可执行代码时候，能够知道的其所依赖的库的名字，但还不知道绝对路径。此时需要系统动态载入器（dynamic linker/loader）

* 对于elf格式的可执行程序，是由ld-linux.so*来完成的，它先后搜索elf文件的DT_RPATH段——环境变量, LD_LIBRARY_PATH——/etc/ld.so.cache文件列表——/lib/，/usr/lib目录找到库文件后将其载入内存。

**3）如何让系统找到动态库**

* 拷贝自己制作的共享库到/lib或者/usr/lib（不能是/lib64目录）

* 临时设置LD_LIBRARY_PATH（本次启动有效，关机后将重置）

  * > export LD_LIBRARY_PATH=$LD_LIBRARY_PATH:库路径

* 永久设置，把export LD_LIBRARY_PATH=$LD_LIBRARY_PATH=$LD_LIBRARY_PATH：库路径，设置到**~/.bashrc**或者**/etc/profile**文件中

  * > vim ~/.bashrc
    >
    > 最后一行添加如下内容：
    >
    > export LD_LIBRARY_PATH=$LD_LIBRARY_PATH:库路径
    >
    > 重新执行配置文件，使之立即生效
    >
    > source ./bashrc

* 将其添加到/etc/ld.so.conf文件中

  * 编辑/etc/ld.so.conf文件，加入库文件所在目录的路径

  * 运行sudo ldconfig -v，该命令会重建/etc/ld.so.cache文件

  * > ```shell
    > sudo vim /etc/ld.so.conf
    > 
    > 文件最后添加动态库路径（绝对路径）
    > 
    > include /etc/ld.so.conf.d/*.conf
    > 
    > /home/deng/share/3rd/2share_test
    > 
    > 使生效
    > 
    > sudo ldconfig -v
    > ```
    >
    > 

* 使用符号链接，但是一定要使用绝对路径，在/lib文件下创建库的符号链接

  * > ln -s /home/deng/test/6share_test/libtest.so /lib/libtest.so

# 13 GDB调试器

## 13.1 GDB简介

GNU根据集中的调试器是GDB(GNU Debugger)，该程序时一个交互工具，工作在字符模式。

除gdb外，linux下比较有名的调试器还有xxgdb，ddd，kgdb，ups。

GDB主要帮你完成一下四个方面的功能：

* 启动程序，可以按照你的自定义的要求随心所欲的运行程序
* 可让被调试的程序在你所指定的断点出停住。（断点可以是条件表达式）
* 当程序被停住时，可以检查此时你的程序中所发生的的事

* 动态的改变你程序的执行环境

## 13.2 生成调试信息

一般来说GDB主要调试的是C/C++的程序。要调试C/C++的程序，首先在编译时，我们必须把调试信息加到可执行文件中。使用编译器（cc/gcc/g++）的-g参数可以做到这一点。如：

> gcc -g hello.c -o hello
>
> g++ -g hello.cpp -o hello

如果没有-g，你将看不见程序的函数名、变量名，所代替的全是运行时的内存地址

## 13.3 启动GDB

* 启动gdb：gdb program
  * program 也就是你的执行文件，一般在当前目录下

* 设置运行参数
  * set args 可指定运行时参数。（如：set args 10 20 30 40 50）
  * show args 命令可以查看设置好的运行参数
* 启动程序
* run：程序开始执行，如果有断点，停在第一个断点处
* start：单步执行
* 退出gdb：quit

## 13.4 显示源代码

用list命令来打印程序的源代码。默认打印10行。(list可简写为l)

* list linenum：打印linenum行的上下文内容
* list n1 n2：列出指定区域的代码
* list function：显示函数名为function的函数的源程序
* list：显示当前行后面的源程序
* list -：显示当前行前面的源程序

一般是打印当前行的上5行和下5行，如果显示函数是上2行下8行，默认是10行，当然你也可以定制显示的范围，使用下面命令可以设置一次显示源程序的行数。

* set listsize count：设置一次显示源程序的行数
* show listsize：查看当前listsize的设置

## 13.5 断点操作

### 13.5.1 简单断点

* break 设置断点，可以简写为b
* b 10 设置断点，在源程序第10行
* b func 设置断点，在func函数入口处

### 13.5.2 多文件设置断点

* C++中可以使用class::function或function(type, type)格式来指定函数名

* 如果有命名空间，可以使用namespace::class::function或者function(type, type)格式来指定函数名。

* break filename：linenum ——在源文件filename的linenum行处停住
* break filename：function ——在源文件filename的function函数入口处停住
* break class::function或function(type, type) ——在类class的function函数的入口处停住
* break namespace::class::function ——在命名空间namespace的类class的function函数的入口处停住

### 13.5.3 查询所有断点

* info b
* info break
* i break
* i b

### 13.5.4 条件断点

一般来说，为断点设置一个条件，我们使用if关键词，后面跟其断点条件。

设置一个条件断点：

**b test.c：8 if Value == 5**

## 13. 6 维护断点

* delete [range……] 删除指定的断点，其简写命令为d。
  * 如果不指定断点号，则表示删除所有的断点。range表示断点号的范围（如3-7）
  * 比删除更好的一种方法是disable停止点，disable了的停止点，GDB不会删除，当你还需要时，enable即可
* disable [range……]：使指定断点无效，简写命令是dis
  * 如果声明都不指定，表示disable所有的停止点
* enable [rang……]：使无效断点生效，简写命令是ena。
  * 如果什么都不指定，表示enable所有的停止点

## 13.7 调试代码

* run 运行程序，可简写为r
* next 单步跟踪，函数调用当作一条简单语句执行，可简写为n
* step单步跟踪，函数调用进入被调函数体内，可简写为s
* finish 退出进入的函数
* until 在一个循环体内单步跟踪时，这个命令可以运行程序直到退出循环体，可简写为u。
* continue 继续执行程序，停在下一个断点的位置，可简写为c
* quit退出gdb，可简写为q

## 13.8 数据查看

* 查看运行时数据
  * print 打印变量、字符串、表达式等的值，可简写为p
  * p count 打印count的值

## 13.9 自动显示

你可以设置一些自动显示的变量，当程序停住时，或是当在你单步跟踪时，这些变量会自动显示。相关的GDB命令是display。

* display 变量名
* info display ——查看display设置的自动显示的信息
* undisplay num (info display时显示的编号)
* delete display dnums …… ——删除自动显示，dnums意为所设置好了自动显示的编号。如果要同时删除多个，编号可以用空格分隔，如果要删除一个范围内的编号，可以用减号表示（如：2-5）
* disable display dnums……
* enable display dnums……
* disable和enable不删除自动显示的设置，而只是让其失效和恢复。

## 13.10 查看修改变量的值

* ptype width ——查看变量width的类型
  * type = double
* p width ——打印变量width的值
* 你可以使用set var命令来告诉GDB，width不是你GDB的参数，而是程序的变量名，如：
  * set var width=47 //将变量var值设置为47
  * 在你改变程序变量取值时，最好都使用set var格式的GDB命令

##13.11 其它命令

* kill 杀死当前正在调试的程序
* return 强制返回当前函数，这样，将会忽略当前函数还没有执行完毕的语句，强制返回。return后面可以接一个表达式，表达式的返回值就是函数的返回值。
* call 强制调用函数
* gdb -tui -q b.out tui模式
  * 在默认设置下方向键和PageUP和PageDn都是用来控制gdbtui的src窗口的，所以，我们常用的上下键来显示前一条命令和后有一条命令的功能就没有了，不过这个时候可以通过Ctrl + n/ Ctrl + p来获取这个功能。
  * （src)source 源代码窗口：显示当前行,断点等信息

