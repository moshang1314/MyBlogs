# 1 Makefile简介

一个工程中的源文件不计其数，其按类型、功能、模块分别放在若干个目录中，makefile定义了一系列的规则来指定，那些文件需要先编译，那些文件需要后编译，那些文件需要重新编译，甚至于更复杂的功能操作，因为makefile就像一个shell脚本一样，其中也可以执行操作系统的命令。

Makefile带来的好处就是——“自动化编译”，一旦写好，只需要一个make命令，整个工程完全自动编译，极大的提高了软件开发的效率。make是一个命令工具，是一个解释makefile中指令的命令工具，一般来说，大多数的IDE都有这个命令，比如Delphi的make，Visual C++的nmake，Linux下的GNU的make。可见，makefile都成为了一种在工程方面的编译方法。

make主要解决两个问题：

**1）大量代码的关系维护**

大项目中源代码比较多，手工维护、编译时间长而且编译命令复杂，难以记忆及维护

把代码维护命令及编译命令写在makefile文件中，然后再用make工具解析此文件自动执行相应命令，可实现代码的合理编译。

**2）减少重复编译时间**

在改动其中一个文件的时候，能判断哪些文件被修改过，可以只对该文件进行重新编译，然后重新链接所有的目标文件，节省编译时间。

**Makefile文件命名规则**

makefile和Makefile都可以，推荐使用Makefile。

**make工具的安装**

* sudo apt install make

## 1.1 Makefile语法规则

一条规则：

> 目标：依赖文件列表
>
> <Tab>命令列表

Makefile基本规则三要素：

1）目标：

* 通常是要产生的文件名称，目标可以是可执行文件或其它obj文件，也可以是动作名称

2）依赖文件

* 用来输入从而目标的文件
* 有多个命令时，每个命令占一行

【注】：**target**也就是一个目标文件，可以是**Object File**，也可以是执行文件。还可以是一个标签（Label），对于标签这种特性，在后续的“伪目标”章节中会有叙述。

​    **prerequisites**就是，要生成那个target所需要的文件或是目标。

​    **command**也就是make需要执行的命令。（任意的Shell命令）



​    这是一个文件的依赖关系，也就是说，target这一个或多个的目标文件依赖于prerequisites中的文件，其生成规则定义在command中。说白一点就是说，prerequisites中如果有一个以上的文件比target文件要新的话，command所定义的命令就会被执行。这就是Makefile的规则。也就是Makefile中最核心的内容。

在看别人写的Makefile文件时，你可能会碰到以下三个变量：$@，$^，$<代表的意义分别是： 

他们三个是十分重要的三个变量，所代表的含义分别是：

​        **$@--目标文件，$^--所有的依赖文件，$<--第一个依赖文件。**

举例说明：

测试代码：

```shell
test:test1 
    echo "hello world"
test1:
    echo "test1"
test2:
    echo "test2"
```

## 1.2make命令格式

make是一个命令工具，它解释Makefile中的指令（应该说是规则）

make命令格式：

make \[-f file\]\[options\]\[targets\]

* [-f file]
  * make默认在工作目录中寻找名为GNUmakefile、makefile、Makefile的文件作为makefile输入文件
  * -f可以指定以上名字以外的文件作为makefile输入文件
* [options]
  * -v：显示make工具的版本信息
  * -w：在处理makefile之前和之后显示工作路径
  * -C dir：读取makefile之前改变工作路径至dir目录
  * -n：只打印要执行的命令但不执行
  * -s：执行但不显示执行的命令
* [targets]
  * 若使用make命令时没有指定目标，则make工具默认会实现makefile文件内的第一个目标，然后退出
  * 指定了make工具要实现的目标，目标可以是一个或多个（多个目标间用空格隔开）

## 1.3 Makefile示例

测试程序：test.c add.c sub.c mul.c div.c

### 1.3.2 最简单的Makefile

```Shell
test:test.c add.c sub.c mul.c div.c
	gcc test.c add.c sub.c mul.c div.c -o test
```

缺点：效率低，修改一个文件，所有文件会被全部编译

### 1.3.3 第二个版本Makefile

```shell
test:test.o add.o sub.o mul.o div.o
    gcc test.o add.o sub.o mul.o div.o -o test

test.o:test.c
    gcc -c test.c
add.o:add.c
    gcc -c add.c
sub.o:sub.c
    gcc -c sub.c
mul.o:mul.c
    gcc -c mul.c
div.o:div.c
    gcc -c div.c
```

## 1.4 Makefile中的变量

在Makefile中使用变量有点类似C语言中的宏定义，使用该变量相当于内容替换，使用变量可以使Makefile易于维护，修改内容变得简单，变量定义及使用。

### 1.4.1 自定义变量

1）定义变量的方法：

变量名=变量值

2）引用变量：

$（变量名）或${变量名}

3）makefile的变量名：

* makefile变量名可以以数字开头
* 变量是大小写敏感的
* 变量一般都在makefile的头部定义
* 变量几乎可在makefile的任何地方使用

示例：

```shell
#变量
OBJS=add.o sub.o mul.o div.o test.o
TARGET=test
​
$(TARGET):$(OBJS)
    gcc $(OBJS) -o $(TARGET) 
​
add.o:add.c
    gcc -c add.c -o add.o
​
sub.o:sub.c
    gcc -c sub.c -o sub.o
​
mul.o:mul.c
    gcc -c mul.c -o mul.o
​
div.o:div.c
    gcc -c div.c -o div.o
​
test.o:test.c
    gcc -c test.c -o test.o
​
clean:
    rm -rf $(OBJS) $(TARGET)
​
```

除了使用用户自定义变量，makefile中也提供了一些变量（变量名大写）供用户直接使用，我们可以直接对其进行赋值

* CC = gcc # arm-linux-gcc
* CPPFLAGS：C预处理的选项 如：-l
* CFLAGS：C编译器的选项 -wall -g -c
* LDFLAGS：链接器选项 -L -l

### 1.4.2 自动变量

* $@：表示规则中的目标的集合
* $<：表示规则中的第一个依赖
* $^：表示规则中的所有依赖，组成一个列表，以空格隔开，如果这个列表中有重复的项，则消除重复项

**注意：自动变量只能在规则的命令中使用**

参考示例：

```shell
#变量
OBJS=add.o sub.o mul.o div.o test.o add.o
TARGET=test
CC=gcc
​
#$@: 表示目标
#$<: 表示第一个依赖
#$^: 表示所有的依赖
​
$(TARGET):$(OBJS)
    #$(CC) $(OBJS) -o $(TARGET) 
    $(CC) $^ -o $@
    echo $@
    echo $<
    echo $^
​
add.o:add.c
    $(CC) -c $< -o $@ 
​
sub.o:sub.c
    $(CC) -c $< -o $@ 
​
mul.o:mul.c
    $(CC) -c $< -o $@ 
​
div.o:div.c
    $(CC) -c $< -o $@ 
​
test.o:test.c
    $(CC) -c $< -o $@
​
clean:
    rm -rf $(OBJS) $(TARGET)
```

## 1.5 模式规则

模式规则示例：

> %.o:%.c
>
> gcc -c $< -o $@

Makefile第三个版本：

```shell
OBJS=test.o add.o sub.o mul.o div.o
TARGET=test
$(TARGET):$(OBJS)
    gcc $(OBJS) -o $(TARGET) 
​
%.o:%.c
    gcc -c $< -o $@
```

## 1.6 Makefile中的函数

makefile中的函数有很多，在这里介绍两个最常用的。

> 1. wildcard - 查找指定目录下的指定类型的文件
>
>    src = $(wildcard *.c) //找到当前目录下所有后缀为.c的文件，赋值给src
>
> 2. patsubst - 匹配替换
>
>    obj = $(patsubst %.c, %.o, $(src)) //把src变量里所有后缀为.c的文件替换为.o

在makefile中所有的函数都是有返回值的。

Makefile第四个版本：

```shell
# 获取当前目录下所有的.c文件
SRC=$(wildcard ./*.c)

# 将SRC中所有出现的.c文件名替换成.o，即获取对应的的目标序列名
OBJS=$(patsubst %.c, %.o, $(SRC))

TARGET = test

$(TARGET):$(OBJS)
	gcc $^ -o $@
	
# 模式匹配 所有的.o都依赖对应的.c
# 将所有的.c生成对应的.o
%.o:%.c
	gcc -c $< -o $@
clean:
	rm -rf $(OBJS) $(TARGET)
```

## 1.7 Makefile中的伪目标

clean用途：清除编译生成的中间.o文件和最终目标文件

make clean 如果当前目录下有同名clean，则不执行clean命令。

伪目标声明：**.PHONY:clean**

声明目标为伪目标之后，makefile将不会判断目标是否存在或是否需要更新

clean命令中的特殊符号：

* “-”此条命令出错，make也会继续执行后续的命令。如“-rm main.o”
* "@"不显示命令本身，只显示结果。如：“@echo clean done"。

Makefile第五个版本：

```makefile
SRC=$(wildcard *.c)
OBJS=$(patsubst %.c, %.o, $(SRC))
TARGET=test
$(TARGET):$(OBJS)
    gcc $(OBJS) -o $(TARGET) 
    
%.o:%.c
    gcc -c $< -o $@
.PHONY:clean
clean:
    rm -rf $(OBJS) $(TARGET)
```

## 1.8 Makefile工作原理

1）若想生成目标，检查规则中的依赖条件是否存在，如不存在，则寻找是否有规则用来生成该依赖文件

![](https://cdn.jsdelivr.net/gh/moshang1314/myBlog@main/image/clip_image002.jpg)

2）检查规则中的目标是否需要更新，必须先检查它的所有依赖，依赖中有任一个被更新，则目标必须更新。

![](https://cdn.jsdelivr.net/gh/moshang1314/myBlog@main/image/clip_image002-1527647235911.jpg)

**总结**

* 分析各个目标和依赖之间的关系
* 根据依赖关系自底向上执行命令
* 根据修改时间比目标新，确定更新
* 如果目标不依赖任何条件，则执行对应的命令，以示更新

# 2 Makefile

## 2.1 一个示例

正如前面所说的，如果一个工程有3个头文件，和8C文件，我们为了完成前面所述的三个规则，我们的Makefile应该是下面的这个样子的。

```shell
edit : main.o kbd.o command.o display.o \

          insert.o search.o files.o utils.o

           cc -o edit main.o kbd.o command.o display.o \

                      insert.o search.o files.o utils.o

 

   main.o : main.c defs.h

           cc -c main.c

   kbd.o : kbd.c defs.h command.h

           cc -c kbd.c

   command.o : command.c defs.h command.h

           cc -c command.c

   display.o : display.c defs.h buffer.h

           cc -c display.c

   insert.o : insert.c defs.h buffer.h

           cc -c insert.c

   search.o : search.c defs.h buffer.h

           cc -c search.c

   files.o : files.c defs.h buffer.h command.h

           cc -c files.c

   utils.o : utils.c defs.h

           cc -c utils.c

   clean :

           rm edit main.o kbd.o command.o display.o \

              insert.o search.o files.o utils.o
```

 **反斜杠（\）**是换行符的意思。这样比较便于Makefile的易读。我们可以把这个内容保存在文件为“Makefile”或“makefile”的文件中，然后在该目录下直接输入命令“make”就可以生成执行文件edit。如果要删除执行文件和所有的中间目标文件，那么，只要简单地执行一下“make clean”就可以了。

​    在这个makefile中，目标文件（target）包含：执行文件edit和中间目标文件（*.o），依赖文件（prerequisites）就是冒号后面的那些 .c 文件和 .h文件。每一个 .o 文件都有一组依赖文件，而这些 .o 文件又是执行文件 edit 的依赖文件。依赖关系的实质上就是说明了目标文件是由哪些文件生成的，换言之，目标文件是哪些文件更新的。

在定义好依赖关系后，后续的那一行定义了如何生成目标文件的**操作系统命令，一定要以一个Tab键作为开头。记住，make并不管命令是怎么工作的，他只管执行所定义的命令**。make会比较targets文件和prerequisites文件的修改日期，如果prerequisites文件的日期要比targets文件的日期要新，或者target不存在的话，那么，make就会执行后续定义的命令。

这里要说明一点的是，clean不是一个文件，它只不过是一个动作名字，有点像C语言中的lable一样，其冒号后什么也没有，那么，make就不会自动去找文件的依赖性，也就不会自动执行其后所定义的命令。要执行其后的命令，就要在make命令后明显得指出这个lable的名字。这样的方法非常有用，我们可以在一个makefile中定义不用的编译或是和编译无关的命令，比如程序的打包，程序的备份，等等。

## 2.2 make是如何工作的

在默认的方式下，也就是我们只输入make命令。那么，

1.  make会在当前目录下找名字叫“Makefile”或“makefile”的文件。
2.  如果找到，它会找文件中的第一个目标文件（target），在上面的例子中，他会找到“edit”这个文件，并把这个文件作为最终的目标文件。
3.  如果edit文件不存在，或是edit所依赖的后面的 .o 文件的文件修改时间要比edit这个文件新，那么，他就会执行后面所定义的命令来生成edit这个文件。
4.  如果edit所依赖的.o文件也存在，那么make会在当前文件中找目标为.o文件的依赖性，如果找到则再根据那一个规则生成.o文件。（这有点像一个堆栈的过程）
5.  当然，你的C文件和H文件是存在的啦，于是make会生成 .o 文件，然后再用 .o 文件声明make的终极任务，也就是执行文件edit了。

这就是整个make的依赖性，make会一层又一层地去找文件的依赖关系，直到最终编译出第一个目标文件。在找寻的过程中，如果出现错误，比如最后被依赖的文件找不到，那么make就会直接退出，并报错，而对于所定义的命令的错误，或是编译不成功，make根本不理。make只管文件的依赖性，即，如果在我找了依赖关系之后，冒号后面的文件还是不在，那么对不起，我就不工作啦。

 通过上述分析，我们知道，像clean这种，没有被第一个目标文件直接或间接关联，那么它后面所定义的命令将不会被自动执行，不过，我们可以显示要make执行。即命令——***\*“make clean”，以此来清除所有的目标文件，以便重编译\****。

   于是在我们编程中，如果这个工程已被编译过了，当我们修改了其中一个源文件，比如file.c，那么根据我们的依赖性，我们的目标file.o会被重编译（也就是在这个依性关系后面所定义的命令），于是file.o的文件也是最新的啦，于是file.o的文件修改时间要比edit要新，所以edit也会被重新链接了（详见edit目标文件后定义的命令）。

而如果我们改变了“command.h”，那么，kdb.o、command.o和files.o都会被重编译，并且，edit会被重链接。

## 2.3 makefile中使用变量

在上面的例子中，先让我们看看edit的规则：

```shell
edit : main.o kbd.o command.o display.o \

​         insert.o search.o files.o utils.o

​      cc -o edit main.o kbd.o command.o display.o \

​           insert.o search.o files.o utils.o
```

 我们可以看到[.o]文件的字符串被重复了两次，如果我们的工程需要加入一个新的[.o]文件，那么我们需要在两个地方加（应该是三个地方，还有一个地方在clean中）。当然，我们的makefile并不复杂，所以在两个地方加也不累，但如果makefile变得复杂，那么我们就有可能会忘掉一个需要加入的地方，而导致编译失败。所以，为了makefile的易维护，在makefile中我们可以使用变量。makefile的变量也就是一个字符串，理解成C语言中的宏可能会更好。

比如，我们声明一个变量，叫objects, OBJECTS, objs, OBJS, obj, 或是 OBJ，反正不管什么啦，只要能够表示obj文件就行了。我们在makefile一开始就这样定义：

  objects = main.o kbd.o command.o display.o \

​       insert.o search.o files.o utils.o

于是，我们就可以很方便地在我们的makefile中以***\*“$(objects)”\****的方式来使用这个变量了，于是我们的改良版makefile就变成下面这个样子：

```shell
objects = main.o kbd.o command.o display.o \
             insert.osearch.o files.o utils.o 
   edit : $(objects)
           cc -o edit $(objects)
   main.o : main.c defs.h
           cc -c main.c
   kbd.o : kbd.c defs.h command.h
           cc -c kbd.c
   command.o : command.c defs.h command.h
           cc -c command.c
   display.o : display.c defs.h buffer.h
           cc -c display.c
   insert.o : insert.c defs.h buffer.h
           cc -c insert.c
   search.o : search.c defs.h buffer.h
           cc -c search.c
   files.o : files.c defs.h buffer.h command.h
           cc -c files.c
   utils.o : utils.c defs.h
           cc -c utils.c
   clean :
           rm edit $(objects)
```

于是如果有新的 .o 文件加入，我们只需简单地修改一下 objects 变量就可以了。

关于变量更多的话题，我会在后续给你一一道来。

## 2.4 让make自动推导

GNU的make很强大，它可以自动推导文件以及文件依赖关系后面的命令，于是我们就没必要去在每一个[.o]文件后都写上类似的命令，因为，我们的make会自动识别，并自己推导命令。

  只要make看到一个[.o]文件，它就会自动的把[.c]文件加在依赖关系中，如果make找到一个whatever.o，那么whatever.c，就会是whatever.o的依赖文件。并且 cc -c whatever.c 也会被推导出来，于是，我们的makefile再也不用写得这么复杂。我们的是新的makefile又出炉了。

```shell
objects = main.o kbd.o command.o display.o \
             insert.o search.o files.o utils.o
 
   edit : $(objects)
           cc -o edit $(objects)
 
   main.o : defs.h
   kbd.o : defs.h command.h
   command.o : defs.h command.h
   display.o : defs.h buffer.h
   insert.o : defs.h buffer.h
   search.o : defs.h buffer.h
   files.o : defs.h buffer.h command.h
   utils.o : defs.h
 
   .PHONY : clean
   clean :
           rm edit $(objects)
```

这种方法，也就是make的“隐晦规则”。上面文件内容中，“.PHONY”表示，clean是个伪目标文件。

关于更为详细的“隐晦规则”和“伪目标文件”，我会在后续给你一一道来。

## 2.5 清空目标文件的规则

每个Makefile中都应该写一个清空目标文件（.o和执行文件）的规则，这不仅便于重编译，也很利于保持文件的清洁。这是一个“修养”（呵呵，还记得我的《编程修养》吗）。一般的风格都是：

​    clean:

​      rm edit $(objects)

更为稳健的做法是：

​    .PHONY : clean

​    clean :

​        -rm edit $(objects)

前面说过，.PHONY意思表示clean是一个“伪目标”，。而在rm命令前面加了一个小减号的意思就是，也许某些文件出现问题，但不要管，继续做后面的事。当然，clean的规则不要放在文件的开头，不然，这就会变成make的默认目标，相信谁也不愿意这样。不成文的规矩是——“clean从来都是放在文件的最后”。

上面就是一个makefile的概貌，也是makefile的基础，下面还有很多makefile的相关细节，准备好了吗？准备好了就来。

# 3 makefile综述

## 3.1 Makefile里有什么？

[Makefile](https://so.csdn.net/so/search?q=Makefile&spm=1001.2101.3001.7020)**里主要包含了五个东西：显式规则、隐晦规则、变量定义、文件指示和注释。**

1. 显式规则。显式规则说明了，如何生成一个或多的的目标文件。这是由Makefile的书写者明显指出，要生成的文件，文件的依赖文件，生成的命令。
2. 隐晦规则。由于我们的make有自动推导的功能，所以隐晦的规则可以让我们比较粗糙地简略地书写Makefile，这是由make所支持的。
3. 变量的定义。在Makefile中我们要定义一系列的变量，变量一般都是字符串，这个有点你C语言中的宏，当Makefile被执行时，其中的变量都会被扩展到相应的引用位置上。
4. 文件指示。其包括了三个部分，一个是在一个Makefile中引用另一个Makefile，就像C语言中的include一样；另一个是指根据某些情况指定Makefile中的有效部分，就像C语言中的预编译#if一样；还有就是定义一个多行的命令。有关这一部分的内容，我会在后续的部分中讲述。
5.  注释。Makefile中只有行注释，和UNIX的Shell脚本一样，其注释是用“#”字符，这个就像C/C++中的“//”一样。如果你要在你的Makefile中使用“#”字符，可以用反斜框进行转义，如：“\#”。

最后，还值得一提的是，在Makefile中的命令，必须要以[Tab]键开始。

## 3.2 Makefile的文件名

默认的情况下，make命令会在当前目录下按顺序找寻文件名为**“GNUmakefile”、“makefile”、“Makefile”**的文件，找到了解释这个文件。在这三个文件名中，最好使用“Makefile”这个文件名，因为，这个文件名第一个字符为大写，这样有一种显目的感觉。最好不要用“GNUmakefile”，这个文件是GNU的make识别的。有另外一些make只对全小写的“makefile”文件名敏感，但是基本上来说，大多数的make都支持“**makefile”和“Makefile”**这两种默认文件名。

   当然，你可以使用别的文件名来书写Makefile，比如：“Make.Linux”，“Make.Solaris”，“Make.AIX”等，**如果要指定特定的Makefile，你可以使用make的“-f”和“--file”参数**，如：make -f Make.Linux或make --file Make.AIX。

## 3.3 引用其它的Makefile

在Makefile使用include关键字可以把别的Makefile包含进来，这很像C语言的#include，被包含的文件会原模原样的放在当前文件的包含位置。include的语法是：

> ```
> include<filename>filename可以是当前操作系统Shell的文件模式（可以保含路径和通配符）
> ```

**在include前面可以有一些空字符，但是绝不能是[Tab]键开始。include和可以用一个或多个空格隔开。**举个例子，你有这样几个Makefile：a.mk、b.mk、c.mk，还有一个文件叫foo.make，以及一个变量$(bar)，其包含了e.mk和f.mk，那么，下面的语句：

  include foo.make *.mk $(bar)

等价于：

 include foo.make a.mk b.mk c.mk e.mk f.mk

make命令开始时，会把找寻include所指出的其它Makefile，并把其内容安置在当前的位置。就好像C/C++的#include指令一样。如果文件都没有指定绝对路径或是相对路径的话，make会在当前目录下首先寻找，如果当前目录下没有找到，那么，make还会在下面的几个目录下找：

>```
>1.如果make执行时，有“-I”或“--include-dir”参数，那么make就会在这个参数所指定的目录下去寻找。
>```
>
>```
>2.如果目录/include（一般是：/usr/local/bin或/usr/include）存在的话，make也会去找。
>```

如果有文件没有找到的话，make会生成一条警告信息，但不会马上出现致命错误。它会继续载入其它的文件，一旦完成makefile的读取，make会再重试这些没有找到，或是不能读取的文件，如果还是不行，make才会出现一条致命信息。如果你想让make不理那些无法读取的文件，而继续执行，你可以在include前加一个减号“-”。如：

-include\<filename>

其表示，无论include过程中出现什么错误，都不要报错继续执行。和其它版本make兼容的相关命令是sinclude，其作用和这一个是一样的。

## 3.4 环境变量 MAKEFILES

如果你的当前环境中定义了环境变量MAKEFILES，那么，make会把这个变量中的值做一个类似于include的动作。这个变量中的值是其它的Makefile，用空格分隔。只是，它和include不同的是，从这个环境变中引入的Makefile的“目标”不会起作用，如果环境变量中定义的文件发现错误，make也会不理。

但是在这里我还是建议不要使用这个环境变量，因为只要这个变量一被定义，那么当你使用make时，所有的Makefile都会受到它的影响，这绝不是你想看到的。在这里提这个事，只是为了告诉大家，也许有时候你的Makefile出现了怪事，那么你可以看看当前环境中有没有定义这个变量。

## 3.5 make的工作方式

**GNU的make工作时的执行步骤入下：（想来其它的make也是类似）**

1. 读入所有的Makefile。

2. 读入被include的其它Makefile。

3. 初始化文件中的变量。

4. 推导隐晦规则，并分析所有规则。

5. 为所有的目标文件创建依赖关系链。

6. 根据依赖关系，决定哪些目标要重新生成。

7. 执行生成命令。

1-5步为第一个阶段，6-7为第二个阶段。第一个阶段中，如果定义的变量被使用了，那么，make会把其展开在使用的位置。但make并不会完全马上展开，make使用的是拖延战术，如果变量出现在依赖关系的规则中，那么仅当这条依赖被决定要使用了，变量才会在其内部展开。

当然，这个工作方式你不一定要清楚，但是知道这个方式你也会对make更为熟悉。有了这个基础，后续部分也就容易看懂了。

## 3.6 makefile书写规则

规则包含两个部分，一个是***依赖关系***，一个是***生成目标的方法***。

***在Makefile中，规则的顺序是很重要的***，因为，***Makefile中只应该有一个最终目标***，其它的目标都是被这个目标所连带出来的，所以一定要让make知道你的最终目标是什么。一般来说，定义在Makefile中的目标可能会有很多，但是第一条规则中的目标将被确立为最终的目标。如果第一条规则中的目标有很多个，那么，第一个目标会成为最终的目标。make所完成的也就是这个目标。

好了，还是让我们来看一看如何书写规则。

### 3.6.1 规则举例

```shell
 foo.o: foo.c defs.h       # foo模块

           cc -c -g foo.c
```

看到这个例子，各位应该不是很陌生了，前面也已说过，foo.o是我们的目标，foo.c和defs.h是目标所依赖的源文件，而只有一个命令“cc -c -g foo.c”（以Tab键开头）。这个规则告诉我们两件事：

> 1. 文件的依赖关系，foo.o依赖于foo.c和defs.h的文件，如果foo.c和defs.h的文件日期要比foo.o文件日期要新，或是foo.o不存在，那么依赖关系发生。
>
> 2. 如果生成（或更新）foo.o文件。也就是那个cc命令，其说明了，如何生成foo.o这个文件。（当然foo.c文件include了defs.h文件）

## 3.7 语法规则

>  targets : prerequisites
>
> ​    command
>
> ​    ...
>
> 或是这样：
>
>    targets : prerequisites ; command
>
> ​      command
>
> ​      ...

targets是文件名，以空格分开，可以使用通配符。一般来说，我们的目标基本上是一个文件，但也有可能是多个文件。

command是命令行，如果其不与“target:prerequisites”在一行，那么，必须以[Tab键]开头，如果和prerequisites在一行，那么可以用分号做为分隔。（见上）

prerequisites也就是目标所依赖的文件（或依赖目标）。如果其中的某个文件要比目标文件要新，那么，目标就被认为是“过时的”，被认为是需要重生成的。这个在前面已经讲过了。

如果命令太长，你可以使用反斜框（‘\’）作为换行符。make对一行上有多少个字符没有限制。规则告诉make两件事，文件的依赖关系和如何成成目标文件。

一般来说，make会以UNIX的标准Shell，也就是/bin/sh来执行命令。

### 3.7.8 在规则中使用通配符

如果我们想定义一系列比较类似的文件，我们很自然地就想起使用通配符。make支持三各通配符：“*”，“?”和“[...]”。这是和Unix的B-Shell是相同的。

**“~”**

波浪号（“~”）字符在文件名中也有比较特殊的用途。如果是“~/test”，这就表示当前用户的$HOME目录下的test目录。而“~hchen/test”则表示用户hchen的宿主目录下的test目录。（这些都是Unix下的小知识了，make也支持）而在Windows或是MS-DOS下，用户没有宿主目录，那么波浪号所指的目录则根据环境变量“HOME”而定。

**"*"**

通配符代替了你一系列的文件，如“*.c”表示所以后缀为c的文件。一个需要我们注意的是，如果我们的文件名中有通配符，如：“*”，那么可以用转义字符“\”，如“\*”来表示真实的“*”字符，而不是任意长度的字符串。

好吧，还是先来看几个例子吧：

  clean:

​    rm -f *.o

上面这个例子我不不多说了，这是操作系统Shell所支持的通配符。这是在命令中的通配符。

  print: *.c

​    lpr -p $?

​    touch print

上面这个例子说明了通配符也可以在我们的规则中，目标print依赖于所有的[.c]文件。其中的“$?”是一个自动化变量，我会在后面给你讲述。

  objects = *.o

上面这个例子，表示了，通符同样可以用在变量中。并不是说[*.o]会展开，不！objects的值就是“*.o”。Makefile中的变量其实就是C/C++中的宏。如果你要让通配符在变量中展开，也就是让objects的值是所有[.o]的文件名的集合，那么，你可以这样：

  objects := $(wildcard *.o)

这种用法由关键字“wildcard”指出，关于Makefile的关键字，我们将在后面讨论。

## **3.8 文件搜寻**

在一些大的工程中，有大量的源文件，我们通常的做法是把这许多的源文件分类，并存放在不同的目录中。所以，当make需要去找寻文件的依赖关系时，你可以在文件前加上路径，但最好的方法是把一个路径告诉make，让make在自动去找。

Makefile文件中的特殊变量“VPATH”就是完成这个功能的，如果没有指明这个变量，make只会在当前的目录中去找寻依赖文件和目标文件。如果定义了这个变量，那么，make就会在当当前目录找不到的情况下，到所指定的目录中去找寻文件了。

>   VPATH = src:../headers

上面的的定义指定两个目录，“src”和“../headers”，make会按照这个顺序进行搜索。目录由“冒号”分隔。（当然，当前目录永远是最高优先搜索的地方）

另一个设置文件搜索路径的方法是使用make的“vpath”关键字（注意，它是全小写的），这不是变量，这是一个make的关键字，这和上面提到的那个VPATH变量很类似，但是它更为灵活。它可以指定不同的文件在不同的搜索目录中。这是一个很灵活的功能。它的使用方法有三种：

> 1. vpath \< pattern> \< directories>   为符合模式< pattern>的文件指定搜索目录\<directories>。
> 2.  vpath < pattern>                清除符合模式< pattern>的文件的搜索目录。
> 3.  vpath                         清除所有已被设置好了的文件搜索目录。

vapth使用方法中的< pattern>需要包含“%”字符。“%”的意思是匹配零或若干字符，例如，“%.h”表示所有以“.h”结尾的文件。< pattern>指定了要搜索的文件集，而< directories>则指定了的文件集的搜索的目录。例如：

> vpath %.h ../headers

该语句表示，要求make在“../headers”目录下搜索所有以“.h”结尾的文件。（如果某文件在当前目录没有找到的话）

我们可以连续地使用vpath语句，以指定不同搜索策略。如果连续的vpath语句中出现了相同的< pattern>，或是被重复了的< pattern>，那么，make会按照vpath语句的先后顺序来执行搜索。如：

> vpath %.c foo
>
>   vpath %  blish
>
>   vpath %.c bar

其表示“.c”结尾的文件，先在“foo”目录，然后是“blish”，最后是“bar”目录。

>  vpath %.c foo:bar
>
>   vpath %  blish

而上面的语句则表示“.c”结尾的文件，先在“foo”目录，然后是“bar”目录，最后才是“blish”目录。

## 3.9 伪目标

最早先的一个例子中，我们提到过一个“clean”的目标，这是一个“伪目标”，

  clean:

​      rm *.o temp

正像我们前面例子中的“clean”一样，即然我们生成了许多文件编译文件，我们也应该提供一个清除它们的“目标”以备完整地重编译而用。 （以“make clean”来使用该目标）

因为，我们并不生成“clean”这个文件。“伪目标”并不是一个文件，只是一个标签，由于“伪目标”不是文件，所以make无法生成它的依赖关系和决定它是否要执行。我们只有通过显示地指明这个“目标”才能让其生效。当然，“伪目标”的取名不能和文件名重名，不然其就失去了“伪目标”的意义了。

当然，为了避免和文件重名的这种情况，我们可以使用一个特殊的标记“.PHONY”来显示地指明一个目标是“伪目标”，向make说明，不管是否有这个文件，这个目标就是“伪目标”。

> .PHONY : clean

只要有这个声明，不管是否有“clean”文件，要运行“clean”这个目标，只有“make clean”这样。于是整个过程可以这样写：

 .PHONY: clean

  clean:

​      rm *.o temp

伪目标一般没有依赖的文件。但是，我们也可以为伪目标指定所依赖的文件。伪目标同样可以作为“默认目标”，只要将其放在第一个。一个示例就是，如果你的Makefile需要一口气生成若干个可执行文件，但你只想简单地敲一个make完事，并且，所有的目标文件都写在一个Makefile中，那么你可以使用“伪目标”这个特性：

> all : prog1 prog2 prog3
>
>   .PHONY : all
>
>  
>
>   prog1 : prog1.o utils.o
>
> ​      cc -o prog1 prog1.o utils.o
>
>  
>
>   prog2 : prog2.o
>
> ​      cc -o prog2 prog2.o
>
>  
>
>   prog3 : prog3.o sort.o utils.o
>
> ​      cc -o prog3 prog3.o sort.o utils.o

我们知道，Makefile中的第一个目标会被作为其默认目标。我们声明了一个“all”的伪目标，其依赖于其它三个目标。由于伪目标的特性是，总是被执行的，所以其依赖的那三个目标就总是不如“all”这个目标新。所以，其它三个目标的规则总是会被决议。也就达到了我们一口气生成多个目标的目的。“.PHONY : all”声明了“all”这个目标为“伪目标”。

随便提一句，从上面的例子我们可以看出，目标也可以成为依赖。所以，伪目标同样也可成为依赖。看下面的例子：

>  .PHONY: cleanall cleanobj cleandiff
>
>  
>
>   cleanall : cleanobj cleandiff
>
> ​      rm program
>
>  
>
>   cleanobj :
>
> ​      rm *.o
>
>  
>
>   cleandiff :
>
> ​      rm *.diff

“make clean”将清除所有要被清除的文件。“clean obj”和“clean diff”这两个伪目标有点像“子程序”的意思。我们可以输入“make cleanall”和“make clean obj”和“make cleandiff”命令来达到清除不同种类文件的目的。



## 3.10 多目标

Makefile的规则中的目标可以不止一个，其支持多目标，有可能我们的多个目标同时依赖于一个文件，并且其生成的命令大体类似。于是我们就能把其合并起来。当然，多个目标的生成规则的执行命令是同一个，这可能会可我们带来麻烦，不过好在我们的可以使用一个自动化变量“$@”（关于自动化变量，将在后面讲述），这个变量表示着目前规则中所有的目标的集合，这样说可能很抽象，还是看一个例子吧。

bigoutput littleoutput : text.g

​      generate text.g -$(subst output,,$@) > $@

  上述规则等价于：

 

  bigoutput : text.g

​      generate text.g -big > bigoutput

  littleoutput : text.g

​      generate text.g -little > littleoutput

其中，-$(subst output,,$@)中的“$”表示执行一个Makefile的函数，函数名为subst，后面的为参数。关于函数，将在后面讲述。这里的这个函数是截取字符串的意思，“$@”表示目标的集合，就像一个数组，“$@”依次取出目标，并执于命令。

## 3.11 静态模式

静态模式可以更加容易地定义多目标的规则，可以让我们的规则变得更加的有弹性和灵活。我们还是先来看一下语法：

<targets...>: \<target-pattern>: <prereq-patterns ...>

　　　\<commands>

...

argets定义了一系列的目标文件，可以有通配符。是目标的一个集合。

target-parrtern是指明了targets的模式，也就是的目标集模式。

prereq-parrterns是目标的依赖模式，它对target-parrtern形成的模式再进行一次依赖目标的定义。

这样描述这三个东西，可能还是没有说清楚，还是举个例子来说明一下吧。如果我们的\<target-parrtern>定义成“%.o”，意思是我们的集合中都是以“.o”结尾的，而如果我们的\<prereq-parrterns>定义成“%.c”，意思是对\<target-parrtern>所形成的目标集进行二次定义，其计算方法是，取\<target-parrtern>模式中的“%”（也就是去掉了[.o]这个结尾），并为其加上[.c]这个结尾，形成的新集合。

所以，我们的“目标模式”或是“依赖模式”中都应该有“%”这个字符，如果你的文件名中有“%”那么你可以使用反斜杠“\”进行转义，来标明真实的“%”字符。

看一个例子：

  objects = foo.o bar.o

 

  all: $(objects)

 

  $(objects): %.o: %.c

​      $(CC) -c $(CFLAGS) $< -o $@

 

上面的例子中，指明了我们的目标从$object中获取，“%.o”表明要所有以“.o”结尾的目标，也就是“foo.o bar.o”，也就是变量$object集合的模式，而依赖模式“%.c”则取模式“%.o”的“%”，也就是“foobar”，并为其加下“.c”的后缀，于是，我们的依赖目标就是“foo.cbar.c”。而命令中的“$<”和“$@”则是自动化变量，“$<”表示所有的依赖目标集（也就是“foo.c bar.c”），“$@”表示目标集（也褪恰癴oo.o bar.o”）。于是，上面的规则展开后等价于下面的规则：

 foo.o : foo.c

​      $(CC) -c $(CFLAGS) foo.c -o foo.o

  bar.o : bar.c

​      $(CC) -c $(CFLAGS) bar.c -o bar.o

试想，如果我们的“%.o”有几百个，那种我们只要用这种很简单的“静态模式规则”就可以写完一堆规则，实在是太有效率了。“静态模式规则”的用法很灵活，如果用得好，那会一个很强大的功能。再看一个例子：

 

  files = foo.elc bar.o lose.o

 

  $(filter %.o,$(files)): %.o: %.c

​      $(CC) -c $(CFLAGS) $< -o $@

  $(filter %.elc,$(files)): %.elc: %.el

​      emacs -f batch-byte-compile $<

$(filter%.o,$(files))表示调用Makefile的filter函数，过滤“$filter”集，只要其中模式为“%.o”的内容。其的它内容，我就不用多说了吧。这个例字展示了Makefile中更大的弹性。

## 3.12 自动生成依赖

我们在编译时，希望改了某个c文件，就只编译那一个c文件，同时，改了一个头文件时，也希望包含这个头文件的c文件也编译。对于前者，很简单，make默认就是这样做的，而对于后者，却需要我们配合gcc的一些选项来实现这个目的。

要想实现头文件改动，包含了此头文件的c文件也编译，需要在Makefile中提现出c文件与头文件的依赖关系，比如func.c中包含了func.h，那么Makefile中如果有如下依赖关系，就可以实现改了func.h后，func.c也被编译：

```
func.o:func.c func.h
```

这样，改了func.h就会自动编译func.c生成func.o。
gcc提供了一个-M选项，来实现自动生成这种依赖的功能。我们在编译时，加上-MD选项，就能在生成.o的位置自动生成对应c文件的.d依赖文件。当然，生成了.d依赖文件，我们还需要将.d依赖文件包含到makefile中才行，这样make时才会使用这种依赖关系。

比如有如下工程：
main.c

```C
#include<stdio.h>
#include<def.h>
#include<func.h>

void main()
{
	printf("value is %d\n" , GG_VALUE);
	func(1);
}
```

func.c

```C
#include<stdio.h>
#include<func.h>

void func(int a)
{
	printf("func called, a=%d \n", a);
}
```

比如有如下工程：

func.h

```
void func(int);
```

def.h

```
#define GG_VALUE   10
```

makefile如下：

```shell
EXE:=a.out
CC:=arm-linux-gnueabihf-gcc

obj_dir:=./objs_dir

srcs:=main.c func.c
#$(info $(srcs))

objs:=$(patsubst %.c,$(obj_dir)/%.o,$(srcs))
all_depends:=$(patsubst %.c,$(obj_dir)/%.d,$(srcs))
#$(info $(objs))
$(EXE):   $(objs) build

#$(obj_dir)/main.o:main.c
#	$(CC) -c $< -o $@ -I.

$(obj_dir)/%.o: %.c
	@ $(CC) -c -MD   $< -o $@ -I.
	@ echo "CC $<"

-include $(all_depends)

build:
	@$(CC) $(objs) -o $(EXE)
	@echo "create $(EXE)"

clean:
	rm $(obj_dir)/*.o
	rm $(EXE)
```

# 4 Makefile 书写命令

 每条规则中的命令和操作系统Shell的命令行是一致的。make会一按顺序一条一条的执行命令，每条命令的开头必须以[Tab]键开头，除非，命令是紧跟在依赖规则后面的分号后的。在命令行之间中的空格或是空行会被忽略，但是如果该空格或空行是以Tab键开头的，那么make会认为其是一个空命令。

我们在UNIX下可能会使用不同的Shell，但是make的命令默认是被“/bin/sh”——UNIX的标准Shell解释执行的。除非你特别指定一个其它的Shell。Makefile中，“#”是注释符，很像C/C++中的“//”，其后的本行字符都被注释。

## 4.1 显示命令

通常，make会把其要执行的命令行在命令执行前输出到屏幕上。当我们用“@”字符在命令行前，那么，这个命令将不被make显示出来，最具代表性的例子是，我们用这个功能来像屏幕显示一些信息。如：

  @echo 正在编译XXX模块......

当make执行时，会输出“正在编译XXX模块......”字串，但不会输出命令，如果没有“@”，那么，make将输出：

  echo 正在编译XXX模块......

  正在编译XXX模块......

如果make执行时，带入make参数“-n”或“--just-print”，那么其只是显示命令，但不会执行命令，这个功能很有利于我们调试我们的Makefile，看看我们书写的命令是执行起来是什么样子的或是什么顺序的。

而make参数“-s”或“--slient”则是全面禁止命令的显示。

## 4.2 命令执行

当依赖目标新于目标时，也就是当规则的目标需要被更新时，make会一条一条的执行其后的命令。需要注意的是，如果你要让上一条命令的结果应用在下一条命令时，你应该使用分号分隔这两条命令。比如你的第一条命令是cd命令，你希望第二条命令得在cd之后的基础上运行，那么你就不能把这两条命令写在两行上，而应该把这两条命令写在一行上，用分号分隔。如：

  示例一：

​    exec:

​        cd /home/hchen

​        pwd

 

  示例二：

​    exec:

​        cd /home/hchen; pwd

当我们执行“make exec”时，第一个例子中的cd没有作用，pwd会打印出当前的Makefile目录，而第二个例子中，cd就起作用了，pwd会打印出“/home/hchen”。

make一般是使用环境变量SHELL中所定义的系统Shell来执行命令，默认情况下使用UNIX的标准Shell——/bin/sh来执行命令。但在MS-DOS下有点特殊，因为MS-DOS下没有SHELL环境变量，当然你也可以指定。如果你指定了UNIX风格的目录形式，首先，make会在SHELL所指定的路径中找寻命令解释器，如果找不到，其会在当前盘符中的当前目录中寻找，如果再找不到，其会在PATH环境变量中所定义的所有路径中寻找。MS-DOS中，如果你定义的命令解释器没有找到，其会给你的命令解释器加上诸如“.exe”、“.com”、“.bat”、“.sh”等后缀。

## 4.3 命令出错

每当命令运行完后，make会检测每个命令的返回码，如果命令返回成功，那么make会执行下一条命令，当规则中所有的命令成功返回后，这个规则就算是成功完成了。如果一个规则中的某个命令出错了（命令退出码非零），那么make就会终止执行当前规则，这将有可能终止所有规则的执行。

有些时候，命令的出错并不表示就是错误的。例如mkdir命令，我们一定需要建立一个目录，如果目录不存在，那么mkdir就成功执行，万事大吉，如果目录存在，那么就出错了。我们之所以使用mkdir的意思就是一定要有这样的一个目录，于是我们就不希望mkdir出错而终止规则的运行。

为了做到这一点，忽略命令的出错，我们可以在Makefile的命令行前加一个减号“-”（在Tab键之后），标记为不管命令出不出错都认为是成功的。如：

 clean:

​      -rm -f *.o

还有一个全局的办法是，给make加上**“-i”或是“--ignore-errors”**参数，那么，Makefile中所有命令都会忽略错误。而如果一个规则是以**“.IGNORE”**作为目标的，那么这个规则中的所有命令将会忽略错误。这些是不同级别的防止命令出错的方法，你可以根据你的不同喜欢设置。

还有一个要提一下的make的参数的是**“-k”或是“--keep-going”**，这个参数的意思是，如果某规则中的命令出错了，那么就终目该规则的执行，但继续执行其它规则。

## 4.4 嵌套执行make

在一些大的工程中，我们会把我们不同模块或是不同功能的源文件放在不同的目录中，我们可以在每个目录中都书写一个该目录的Makefile，这有利于让我们的Makefile变得更加地简洁，而不至于把所有的东西全部写在一个Makefile中，这样会很难维护我们的Makefile，这个技术对于我们模块编译和分段编译有着非常大的好处。

 例如，我们有一个子目录叫subdir，这个目录下有个Makefile文件，来指明了这个目录下文件的编译规则。那么我们总控的Makefile可以这样书写：

 subsystem:

​      cd subdir && $(MAKE)

其等价于：

   subsystem:

​      $(MAKE) -C subdir

定义$(MAKE)宏变量的意思是，也许我们的make需要一些参数，所以定义成一个变量比较利于维护。这两个例子的意思都是先进入“subdir”目录，然后执行make命令。

我们把这个Makefile叫做“总控Makefile”，总控Makefile的变量可以传递到下级的Makefile中（如果你显示的声明），但是不会覆盖下层的Makefile中所定义的变量，除非指定了“-e”参数。

如果你要传递变量到下级Makefile中，那么你可以使用这样的声明：

export<variable ...>

如果你不想让某些变量传递到下级Makefile中，那么你可以这样声明：

unexport<variable ...>

如：

​    示例一：

​     export variable = value

​    
 其等价于：

​     variable = value

​    export variable

 

​    其等价于：

 export variable := value

​     其等价于：

 

​    variable := value

​    export variable

示例二：

 

​    export variable += value

 

​    其等价于：

 

​    variable += value

​    export variable

   如果你要传递所有的变量，那么，只要一个export就行了。后面什么也不用跟，表示传递所有的变量。

需要注意的是，有两个变量，一个是SHELL，一个是MAKEFLAGS，这两个变量不管你是否export，其总是要传递到下层Makefile中，特别是MAKEFILES变量，其中包含了make的参数信息，如果我们执行“总控Makefile”时有make参数或是在上层Makefile中定义了这个变量，那么MAKEFILES变量将会是这些参数，并会传递到下层Makefile中，这是一个系统级的环境变量。

但是make命令中的有几个参数并不往下传递，它们是“-C”,“-f”,“-h”“-o”和“-W”（有关Makefile参数的细节将在后面说明），如果你不想往下层传递参数，那么，你可以这样来：

 subsystem:

​      cd subdir && $(MAKE) MAKEFLAGS=

如果你定义了环境变量MAKEFLAGS，那么你得确信其中的选项是大家都会用到的，如果其中有“-t”,“-n”,和“-q”参数，那么将会有让你意想不到的结果，或许会让你异常地恐慌。

还有一个在“嵌套执行”中比较有用的参数，“-w”或是“--print-directory”会在make的过程中输出一些信息，让你看到目前的工作目录。比如，如果我们的下级make目录是“/home/hchen/gnu/make”，如果我们使用“make -w”来执行，那么当进入该目录时，我们会看到：

 make: Entering directory `/home/hchen/gnu/make'.

而在完成下层make后离开目录时，我们会看到：  

  make: Leaving directory `/home/hchen/gnu/make'



当你使用“-C”参数来指定make下层Makefile时，“-w”会被自动打开的。如果参数中有“-s”（“--slient”）或是“--no-print-directory”，那么，“-w”总是失效的。

## 4.5 定义命令包

如果Makefile中出现一些相同命令序列，那么我们可以为这些相同的命令序列定义一个变量。定义这种命令序列的语法以“define”开始，以“endef”结束，如：

  define run-yacc

  yacc $(firstword $^)

  mv y.tab.c $@

  endef

这里，“run-yacc”是这个命令包的名字，其不要和Makefile中的变量重名。在“define”和“endef”中的两行就是命令序列。这个命令包中的第一个命令是运行Yacc程序，因为Yacc程序总是生成“y.tab.c”的文件，所以第二行的命令就是把这个文件改改名字。还是把这个命令包放到一个示例中来看看吧。

  foo.c : foo.y

​      $(run-yacc)

我们可以看见，要使用这个命令包，我们就好像使用变量一样。在这个命令包的使用中，命令包“run-yacc”中的“$^”就是“foo.y”，“$@”就是“foo.c”（有关这种以“$”开头的特殊变量，我们会在后面介绍），make在执行命令包时，命令包中的每个命令会被依次独立执行。

# 5 使用变量

## 5.1 变量的基础

变量在声明时需要给予初值，而在使用时，需要给在变量名前加上“$"符号，但最好用小括号”（）“或是大括号”{}“把变量给包起来。如果你要使用真实的”$“字符，那么你需要用”$$“来表示。变量可以使用在许多地方，如规则中的”目标“、”依赖“、“命令”以及新的变量中。

如：

objects = program.o foo.o utils.o

program:$(objects)

cc -o program $(objects)



$(objects):defs.h

**变量会在使用它的地方精确地展开，就像C/C++中的宏一样，例如：**

foo = c
prog.o : prog.$(foo)
$(foo)$(foo) -$(foo) prog.$(foo)



展开后得到：

prog.o : prog.c
cc -c prog.c

然，千万不要在你的Makefile中这样干，这里只是举个例子来表明Makefile中的变量在使用处展开的真实样子。可见其就是一个“替代”的原理。另外，给变量加上括号完全是为了更加安全地使用这个变量，在上面的例子中，如果你不想给变量加上括号，那也可以，但我还是强烈建议你给变量加上括号。

## 5.2 变量中的变量

在定义变量的值时，我们可以使用其它变量来构造变量的值，在Makefile中有两种方式来在用变量定义变量的值。


先看第一种方式，也就是简单的使用“=”号，在“=”左侧是变量，右侧是变量的值，右侧变量的值可以定义在文件的任何一处，也就是说，右侧中的变量不一定非要是已定义好
的值，其也可以使用后面定义的值。如：

foo = $(bar)
bar = $(ugh)
ugh = Huh?


all:
echo $(foo)

我们执行“make all”将会打出变量$(foo)的值是“Huh?”（ $(foo)的值是$(bar)，$(bar)的值是$(ugh)，$(ugh)的值是“Huh?”）可见，变量是可以使用后面的变量来定义的。
这个功能有好的地方，也有不好的地方，好的地方是，我们可以把变量的真实值推到后面来定义，如：

CFLAGS = $(include_dirs) -O
include_dirs = -Ifoo -Ibar


当“CFLAGS”在命令中被展开时，会是“-Ifoo -Ibar -O”。但这种形式也有不好的地方
，那就是递归定义，如：


CFLAGS = $(CFLAGS) -O


或：


A = $(B)
B = $(A)


这会让make陷入无限的变量展开过程中去，当然，我们的make是有能力检测这样的定义，并会报错。还有就是如果在变量中使用函数，那么，这种方式会让我们的make运行时非常慢，更糟糕的是，他会使用得两个make的函数“wildcard”和“shell”发生不可预知的错误。因为你不会知道这两个函数会被调用多少次。

为了避免上面的这种方法，我们可以使用make中的另一种用变量来定义变量的方法。这种方法使用的是“:=”操作符，如：

x := foo
y := $(x) bar
x := later

其等价于：

y := foo bar
x := later


值得一提的是，这种方法，前面的变量不能使用后面的变量，只能使用前面已定义好了的变量。如果是这样：


y := $(x) bar
x := foo


那么，y的值是“bar”，而不是“foo bar”。

上面都是一些比较简单的变量使用了，让我们来看一个复杂的例子，其中包括了make的函数、条件表达式和一个系统变量“MAKELEVEL”的使用：


ifeq (0,${MAKELEVEL})
cur-dir := $(shell pwd)
whoami := $(shell whoami)
host-type := $(shell arch)
MAKE := ${MAKE} host-type=${host-type} whoami=${whoami}
endif


关于条件表达式和函数，我们在后面再说，对于系统变量“MAKELEVEL”，其意思是，如果我们的make有一个嵌套执行的动作（参见前面的“嵌套使用make”），那么，这个变量会记录了我们的当前Makefile的调用层数。


下面再介绍两个定义变量时我们需要知道的，请先看一个例子，如果我们要定义一个变量，其值是一个空格，那么我们可以这样来：

nullstring :=
space := $(nullstring) # end of the line


nullstring 是一个Empty变量，其中什么也没有，而我们的space的值是一个空格。因为在操作符的右边是很难描述一个空格的，这里采用的技术很管用，先用一个 Empty变量来标明变量的值开始了，而后面采用“#”注释符来表示变量定义的终止，这样，我们可以定义出其值是一个空格的变量。请注意这里关于“#”的使用，注释符“#”的这种特性值得我们注意，如果我们这样定义一个变量：

dir := /foo/bar # directory to put the frobs in

dir这个变量的值是“/foo/bar”，后面还跟了4个空格，如果我们这样使用这样变量来指定别的目录——“$(dir)/file”那么就完蛋了。


还有一个比较有用的操作符是“?=”，先看示例：


FOO ?= bar


其含义是，如果FOO没有被定义过，那么变量FOO的值就是“bar”，如果FOO先前被定义过，那么这条语将什么也不做，其等价于：

ifeq ($(origin FOO), undefined)
FOO = bar
endif

## 5.3 变量的高级用法

### 5.3.1 变量值的替换

我们可以替换变量中的共有的部分，其格式是**“$(var:a=b)”**或是**“${var:a=b}”**，其意思是，把变量“var”中所有以“a”字串“结尾”的“a”替换成“b”字串。这里的“结尾”意思是“空格”或是“结束符”。

还是看一个示例吧：

```shell
foo := a.o b.o c.o
bar := $(foo:.o=.c)
```

这个示例中，我们先定义了一个“$(foo)”变量，而第二行的意思是把“$(foo)”中所有以“.o”字串“结尾”全部替换成“.c”，所以我们的“$(bar)”的值就是“a.c b.c c.c”。

另外一种变量替换的技术是以“静态模式”（参见前面章节）定义的，如：


foo := a.o b.o c.o
bar := $(foo:%.o=%.c)


这依赖于被替换字串中的有相同的模式，模式中必须包含一个“%”字符，这个例子同样让$(bar)变量的值为“a.c b.c c.c”。

### 5.3.2 变量的值再当成变量

第二种高级用法是——“把变量的值再当成变量”。先看一个例子：


x = y
y = z
a := $($(x))


在这个例子中，$(x)的值是“y”，所以$($(x))就是$(y)，于是$(a)的值就是“z”。（注意，是“x=y”，而不是“x=$(y)”）


我们还可以使用更多的层次：


x = y
y = z
z = u
a := $($($(x)))


这里的$(a)的值是“u”，相关的推导留给读者自己去做吧。

让我们再复杂一点，使用上“在变量定义中使用变量”的第一个方式，来看一个例子：


x = $(y)
y = z
z = Hello
a := $($(x))


这里的$($(x))被替换成了$($(y))，因为$(y)值是“z”，所以，最终结果是：a:=$(z)，也就是“Hello”。

再复杂一点，我们再加上函数：


x = variable1
variable2 := Hello
y = $(subst 1,2,$(x))
z = y
a := $($($(z)))


这个例子中，“$($($(z)))”扩展为“$($(y))”，而其再次被扩展为“$($(subst 1,2,$(x)))”。$(x)的值是“variable1”，subst函数把“variable1”中的所有“1”字串替换成“2”字串，于是，“variable1”变成“variable2”，再取其值，所以，最终，$(a)的值就是$(variable2)的值—— “Hello”。（喔，好不容易）

在这种方式中，或要可以使用多个变量来组成一个变量的名字，然后再取其值：

first_second = Hello
a = first
b = second
all = $($a_$b)


这里的“$a_$b”组成了“first_second”，于是，$(all)的值就是“Hello”。


再来看看结合第一种技术的例子：


a_objects := a.o b.o c.o
1_objects := 1.o 2.o 3.o


sources := $($(a1)_objects:.o=.c)


这个例子中，如果$(a1)的值是“a”的话，那么，$(sources)的值就是“a.c b.c c.c”；如果$(a1)的值是“1”，那么$(sources)的值是“1.c 2.c 3.c”。

再来看一个这种技术和“函数”与“条件语句”一同使用的例子：


ifdef do_sort
func := sort
else
func := strip
endif

bar := a d b g q c

foo := $($(func) $(bar))


这个示例中，如果定义了“do_sort”，那么：foo := $(sort a d b g q c)，于是$(foo)的值就是“a b c d g q”，而如果没有定义“do_sort”，那么：foo := $(sort a d bg q c)，调用的就是strip函数。

当然，“把变量的值再当成变量”这种技术，同样可以用在操作符的左边：

dir = foo
$(dir)_sources := $(wildcard $(dir)/*.c)
define $(dir)_print
lpr $($(dir)_sources)
endef

这个例子中定义了三个变量：“dir”，“foo_sources”和“foo_print”。

### 5.3.3 追加变量值

我们可以使用“+=”操作符给变量追加值，如：

objects = main.o foo.o bar.o utils.o
objects += another.o

于是，我们的$(objects)值变成：“main.o foo.o bar.o utils.o another.o”（another.o被追加进去了）

使用“+=”操作符，可以模拟为下面的这种例子：

objects = main.o foo.o bar.o utils.o
objects := $(objects) another.o

如果变量之前没有定义过，那么，“+=”会自动变成“=”，如果前面有变量定义，那么“+=”会继承于前次操作的赋值符。如果前一次的是“:=”，那么“+=”会以“:=”作为其赋值符，如：

variable := value
variable += more


等价于：

variable := value
variable := $(variable) more

但如果是这种情况：


variable = value
variable += more


由于前次的赋值符是“=”，所以“+=”也会以“=”来做为赋值，那么岂不会发生变量的递补归定义，这是很不好的，所以make会自动为我们解决这个问题，我们不必担心这个问题。

### 5.3.4 override 指示符

如果有变量是通常make的命令行参数设置的，那么Makefile中对这个变量的赋值会被忽略。如果你想在Makefile中设置这类参数的值，那么，你可以使用“override”指示符。其语法是：

override \<variable> = \<value>
override \<variable> := \<value>


当然，你还可以追加：


override \<variable> += \<more text>


对于多行的变量定义，我们用define指示符，在define指示符前，也同样可以使用ovveride指示符，如：


override define foo
bar
endef

## 5.4 多行变量

还有一种设置变量值的方法是使用define关键字。使用define关键字设置变量的值可以有换行，这有利于定义一系列的命令（前面我们讲过“命令包”的技术就是利用这个关键字）。

define 指示符后面跟的是变量的名字，而重起一行定义变量的值，定义是以endef关键字结束。其工作方式和“=”操作符一样。变量的值可以包含函数、命令、文字，或是其它变量。因为命令需要以[Tab]键开头，所以如果你用define定义的命令变量中没有以[Tab]键开头，那么make就不会把其认为是命令。

下面的这个示例展示了define的用法：

define two-lines
echo foo
echo $(bar)
endef

## 5.5 环境变量

make 运行时的系统环境变量可以在make开始运行时被载入到Makefile文件中，但是如果Makefile中已定义了这个变量，或是这个变量由make命令行带入，那么系统的环境变量的值将被覆盖。（如果make指定了“-e”参数，那么，系统环境变量将覆盖Makefile中定义的变量）

因此，如果我们在环境变量中设置了“CFLAGS”环境变量，那么我们就可以在所有的Makefile中使用这个变量了。这对于我们使用统一的编译参数有比较大的好处。如果Makefile中定义了CFLAGS，那么则会使用Makefile中的这个变量，如果没有定义则使用系统环境变量的值，一个共性和个性的统一，很像“全局变量”和“局部变量”的特性。   当make嵌套调用时（参见前面的“嵌套调用”章节），上层Makefile中定义的变量会以系统环境变量的方式传递到下层的Makefile中。当然，默认情况下，只有通过命令行设置的变量会被传递。而定义在文件中的变量，如果要向下层 Makefile传递，则需要使用exprot关键字来声明。（参见前面章节）

  当然，我并不推荐把许多的变量都定义在系统环境中，这样，在我们执行不用的Makefile时，拥有的是同一套系统变量，这可能会带来更多的麻烦。

## 5.6 目标变量


前面我们所讲的在Makefile中定义的变量都是“全局变量”，在整个文件，我们都可以访问这些变量。当然，“自动化变量”除外，如“$<”等这种类量的自动化变量就属于“规则型变量”，这种变量的值依赖于规则的目标和依赖目标的定义。

当然，我样同样可以为某个目标设置局部变量，这种变量被称为“Target-specific Variable”，它可以和“全局变量”同名，因为它的作用范围只在这条规则以及连带规则中，所以其值也只在作用范围内有效。而不会影响规则链以外的全局变量的值。

其语法是：


<target ...> : \<variable-assignment>



<target ...> : overide \<variable-assignment>


\<variable-assignment>可以是前面讲过的各种赋值表达式，如“=”、“:=”、“+=”或是“？=”。第二个语法是针对于make命令行带入的变量，或是系统环境变量。

这个特性非常的有用，当我们设置了这样一个变量，这个变量会作用到由这个目标所引发的所有的规则中去。如：

prog : CFLAGS = -g
prog : prog.o foo.o bar.o
$(CC) $(CFLAGS) prog.o foo.o bar.o


prog.o : prog.c
$(CC) $(CFLAGS) prog.c


foo.o : foo.c
$(CC) $(CFLAGS) foo.c


bar.o : bar.c
$(CC) $(CFLAGS) bar.c


在这个示例中，不管全局的$(CFLAGS)的值是什么，在prog目标，以及其所引发的所有规则中（prog.o foo.o bar.o的规则），$(CFLAGS)的值都是“-g”

## 5.7 模式变量

在GNU的make中，还支持模式变量（Pattern-specific Variable），通过上面的目标变量中，我们知道，变量可以定义在某个目标上。模式变量的好处就是，我们可以给定一种“模式”，可以把变量定义在符合这种模式的所有目标上。

我们知道，make的“模式”一般是至少含有一个“%”的，所以，我们可以以如下方式给所有以[.o]结尾的目标定义目标变量：


%.o : CFLAGS = -O


同样，模式变量的语法和“目标变量”一样：


<pattern ...> : \<variable-assignment>


<pattern ...> : override \<variable-assignment>


override同样是针对于系统环境传入的变量，或是make命令行指定的变量。

# 6 使用条件判断

使用条件判断，可以让make根据运行时不同情况选择不同的分支。条件表达式可以是比较变量的值，或是比较变量和常量的值。

## 6.1 示例

下面的例子，判断$(CC)变量是否是“gcc”，如果是的话，则使用GNU函数编译目标。

```
libs_for_gcc = -lgnu
normal_libs = 

foo:$(objects)
	ifeq($(CC), gcc)
	$(CC) -o foo $(objects) $(libs_for_gcc)
	else
	$(CC) -o foo $(objects) $(normal_libs)
	endif
```


可见，在上面示例的这个规则中，目标“foo”可以根据变量“$(CC)”值来选取不同的函数库来编译程序。

我们可以从上面的示例中看到三个关键字：ifeq、else和endif。ifeq的意思表示条件语句的开始，并指定一个条件表达式，表达式包含两个参数，以逗号分隔，表达式以圆括号括起。else表示条件表达式为假的情况。endif表示一个条件语句的结束，任何一个条件表达式都应该以endif结束。

当我们的变量$(CC)值是“gcc”时，目标foo的规则是：


foo: $(objects)
$(CC) -o foo $(objects) $(libs_for_gcc)


而当我们的变量$(CC)值不是“gcc”时（比如“cc”），目标foo的规则是：

foo: $(objects)
$(CC) -o foo $(objects) $(normal_libs)


当然，我们还可以把上面的那个例子写得更简洁一些：

```makefile

libs_for_gcc = -lgnu
normal_libs =


ifeq ($(CC),gcc)
libs=$(libs_for_gcc)
else
libs=$(normal_libs)
endif


foo: $(objects)
	$(CC) -o foo $(objects) $(libs)
```

## 6.2 语法

条件表达式的语法为：

\<conditional-directive>
\<text-if-true>
endif


以及：

\<conditional-directive>
\<text-if-true>
else
\<text-if-false>
endif


其中\<conditional-directive>表示条件关键字，如“ifeq”。这个关键字有四个。

第一个是我们前面所见过的“ifeq”

ifeq (\<arg1>, \<arg2> )
ifeq '\<arg1>' '\<arg2>'
ifeq "\<arg1>" "\<arg2>"
ifeq "\<arg1>" '\<arg2>'
ifeq '\<arg1>' "\<arg2>"


比较参数“arg1”和“arg2”的值是否相同。当然，参数中我们还可以使用make的函数。如：

ifeq ($(strip $(foo)),)
\<text-if-empty>
endif


这个示例中使用了“strip”函数，如果这个函数的返回值是空（Empty），那么<text-if-empty>就生效。

第二个条件关键字是“ifneq”。语法是：

ifneq (\<arg1>, \<arg2> )
ifneq '\<arg1>' '\<arg2>'
ifneq "\<arg1>" "\<arg2>"
ifneq "\<arg1>" '\<arg2>'
ifneq '\<arg1>' "\<arg2>"


其比较参数“arg1”和“arg2”的值是否相同，如果不同，则为真。和“ifeq”类似。




第三个条件关键字是“ifdef”。语法是：


ifdef \<variable-name>


如果变量\<variable-name>的值非空，那到表达式为真。否则，表达式为假。当然，<variable-name>同样可以是一个函数的返回值。注意，ifdef只是测试一个变量是否有值，其并不会把变量扩展到当前位置。还是来看两个例子：


示例一：
bar =
foo = $(bar)
ifdef foo
frobozz = yes
else
frobozz = no
endif


示例二：
foo =
ifdef foo
frobozz = yes
else
frobozz = no
endif


第一个例子中，“$(frobozz)”值是“yes”，第二个则是“no”。


第四个条件关键字是“ifndef”。其语法是：


ifndef \<variable-name>


这个我就不多说了，和“ifdef”是相反的意思。

在\<conditional-directive>这一行上，多余的空格是被允许的，但是不能以[Tab]键做为开始（不然就被认为是命令）。而注释符“#”同样也是安全的。“else”和“endif”也
一样，只要不是以[Tab]键开始就行了。

特别注意的是，make是在读取Makefile时就计算条件表达式的值，并根据条件表达式的值来选择语句，所以，你最好不要把自动化变量（如“$@”等）放入条件表达式中，因为自动化变量是在运行时才有的。

而且，为了避免混乱，make不允许把整个条件语句分成两部分放在不同的文件中。

# 7 函数调用

## 7.1 函数调用语法

函数调用，很像变量的使用，也是以"$"来标识的，其语法如下：

$(\<function> \<arguments>)

或是

${\<function> \<argument>}

这里，\<function>就是函数名，make支持的函数不多。\<arguments>是函数的参数，参数间以逗号“,"分隔，而函数名和参数之间以空格分隔。函数调用以“$”开头，以圆括号或花括号把函数名和参数括起来。感觉很像一个变量，是不是？函数中的参数可以使用变量，为了风格的统一，函数和变量的括号最好一样，如使用"$(subst a,b,$(x))"这样的形式，而不是“$(subst a,b,${x})”的形式。因为统一会更清楚，也会减少一些不必要的麻烦。

还是来看一个示例：

```makefile
comma :=,
empty :=
space := $(empty)$(empty)
foo := a b c
bar := $(subst $(space),$(comma),$(foo))
```

在这个示例中，$(comma)的值是一个逗号。$(space)使用了$(empty)定义了一个空格，$(foo)的值是“a b c”，$(bar)的定义调用了函数“subst”，这是一个替换函数，这个函数有三个参数，第一个参数是被替换字串，第二个参数是替换字串，第三个参数是替换操作作用的字串。这个函数也就是把$(foo)中的空格替换成逗号，所以$(bar)的值是“a,b,c”。

## 7.2 字符串处理函数

### 7.2.1 subst

$(subst \<from>, \<to>, \<text>)

名称：字符串替换函数——subst

功能：把字串\<text>中的\<from>字符串替换成\<to>。

返回：函数返回被替换过后的字符串

示例：

$(subst ee,  EE,feet on the street),

把“feet on the street"中的”ee"替换成“EE"，返回结果是”fEEt on the strEEt"。

### 7.2.2 patsubst

$(patsubst \<pattern>,\<replacement>, \<text>)

名称：模式字符串替换函数——patsubst

功能：查找\<text>中的单词（单词以“空格”、“Tab"或”回车“”换行“分隔）是否符合模式\<pattern>,如果匹配的话，则以\<replacement>替换。这里，\<pattern>可以包括通配符”%“，表示任意长度的子串。如果\<replacement>中也包含”%“，那么，\<replacement>中的这个”%“将是\<pattern>中的那个”%“所代表的子串。（可以用”\“来转义，以”\%“来表示真实含义的”%"字符）

返回：函数返回被替换过后的字符串

示例：

$(patsubst %.c, %.o,x.c.c bar.c)

把子串“x.c.c bar.c”符合模式[%.c]的单词替换成[%.0]，返回结果是“x.c.o bar.o”

这和我们前面“变量章节”说过的相关知识有点相似。如：

"$(var:\<pattern>=\<replacement>)"

相当于

"$(patsubst \<pattern>, \<replacement>, $(var))"，

而“$(var:\<suffix>=\<replacement>)"

则相当于

“$(patsubst %\<suffix>, %\<replacement>,$(var))"。

例如有：objects = foo.o bar.o baz.o，
那么，“$(objects:.o=.c)”和“$(patsubst %.o,%.c,$(objects))”是一样的。

### 7.2.3 strip

$(strip \<string>)

名称：去空格函数——strip

功能：去掉\<string>字串中开头和结尾的空字符

返回：返回被去掉空格的字符串值

$(strip a b c )

把字符串“a b c "去掉开头和结尾的空格，结果是”a b c“。

### 7.2.4 findstring

$(findstring \<find>, \<in>)

名称：查找字符串函数——findstring

功能：在字符串\<in>中查找\<find>字符串

返回：如果找到返回\<find>，否则返回空字符串。

示例：

$(findstring a,a b c)

$(findstring a,b c)

第一个函数返回“a”字符串，第二个返回“”字符串（空字符串）

### 7.2.5 filter

$(filter \<pattern….>, \<text>)

名称：过滤函数——filter

功能：以\<pattern>模式过滤\<text>字符串中的单词，保留符合模式\<pattern>的单词。可以有很多个模式。

返回：返回符合模式\<pattern>的子串

示例：

sources :=foo.c bar.c baz.s ugh.h

foo: $(sources)

cc $(filter %.c %.s, $(sources)) -o foo

$(filter %.c %.s,$(sources))返回的值是“foo.c bar.c baz.s"。

### 7.2.6 filter-out

$(filter-out \<pattern..>,\<text>)

名称：反过滤函数——filter-out

功能：以\<pattern>模式过滤\<text>字符串中的单词，去除符合模式\<pattern>的单词。可以有多个模式。

返回：返回不符合模式\<pattern>的字符串

示例：

objects=main1.o foo.o main2.o bar.o

mains=main1.o main2.o

$(filter-out $(main),$(objects))返回值是“foo.o bar.o”。

### 7.2.7 sort

$(sort \<list>)

名称：排序函数——sort

功能：给字符串\<list>中的单词排序（升序）。

返回：返回排序后的字符串

示例：$(sort foo bar lose)返回“bar foo  lose"。

备注：sort函数会去掉\<lise>中相同的单词。

### 7.2.8 word

$(word \<n>, \<text>)

名称：取单词函数——word

功能：取字符串\<text>中第\<n>个单词

返回：返回字符串\<text>中第\<n>个单词。如果\<n>比\<text>中的单词数要大，那么返回空字符串

示例：$(word,2,foo bar baz)返回值是“bar"。

### 7.2.9 wordlist

$(wordlist \<s>, \<e>,\<text>)

名称：取单词串函数——wordlist

功能：从字符串\<text>中取从\<s>开始到\<e>的单词串。\<s>和\<e>是一个数字

返回：返回字符串\<text>中从\<s>到\<e>的单词串。如果\<s>比\<text>中的单词数要大，那么返回空字符串。如果\<e>大于\<text>的单词数，那么返回从\<s>开始，到\<text>结束的单词串。

示例：$(wordlist 2,3, foo bar baz)返回值是“bar baz”

### 7.2.10 words

名称：单词个数统计函数——words。

功能：统计\<text>中字符串中的单词个数。

返回：返回\<text>中的单词数。

示例：$(words, foo bar baz)返回值是“3"。

### 7.2.11 firstword 

$(firstword \<text>)

名称：首单词函数——firstword

功能：取字符串\<text>中的第一个单词

返回：返回字符串\<text>的第一个单词

示例：$(firstword foo bar)返回值是“foo”

以上，是所有的字符串操作函数，如果搭配混合使用，可以完成比较复杂的功能。这里，
举一个现实中应用的例子。我们知道，make使用“VPATH”变量来指定“依赖文件”的搜索
路径。于是，我们可以利用这个搜索路径来指定编译器对头文件的搜索路径参数CFLAGS，
如：

override CFLAGS += $(patsubst %,-I%,$(subst :, ,$(VPATH)))


如果我们的“$(VPATH)”值是“src:../headers”，那么“$(patsubst %,-I%,$(subst :
, ,$(VPATH)))”将返回“-Isrc -I../headers”，这正是cc或gcc搜索头文件路径的参数
。

## 7.3 文件名操作函数

下面我们要介绍的函数主要是处理文件名的。每个函数的参数字符串都会被当做一个或是
一系列的文件名来对待。

### 7.3.1 dir

$(dir \<names…>)

名称：取目录函数——dir。

功能：从文件名序列\<names>中取出目录部分。目录部分是指最后一个反斜杠（“/"）之前的部分。如果没有反斜杠，那么返回“./”。

返回：返回文件名序列\<names>的目录部分

示例：$(dir src/foo.c hacks)返回值是“src/ ./”。

### 7.3.2 notdir

$(notdir \<names…>)

名称：取文件函数——notdir。

功能：从文件名序列\<names>中取出非目录部分。非目录部分是指最后一个反斜杠（"/"）之后的部分。

返回：返回文件名序列\<names>的非目录部分。

示例：$(notdir src/foo.c hacks)返回值是“foo.c hacks"。

### 7.3.3 suffix

名称：取后缀函数——suffix

功能：从文件名序列\<names>中取出各个文件名的后缀。

返回：返回文件名序列\<names>的后缀序列，如果文件没有后缀，则返回空字符串。

示例：$(suffix src/foo.c src-1.0/bar.c hacks)返回值是“.c .c"。

### 7.3.4 basename

名称：取前缀函数——basename。

功能：从文件名序列\<names>中取出各个文件名的前缀部分

返回：返回文件名序列\<names>的前缀序列，如果文件没有前缀，则返回空字符串。

示例：$(basename src/foo.c src-1.o/bar.c hacks)返回值是“src/foo src-1.0/bar hacks”。

### 7.3.5 addsuffix

名称：加后缀函数——addsuffix

功能：把后缀\<suffix>加到\<names>中的每个单词后面。

返回：返回加过后缀的文件名序列。

示例：$(addsuffix .c, foo bar)返回值是“foo.c bar.c"

### 7.3.6 addprefix

$(addprefix \<prifix>,\<names…>)

名称：加后缀函数——addprefix

功能：把前缀\<prefix>加到\<names>中的每个单词后面。

返回：返回加过前缀的文件名序列

示例：$(addprefix src/, foo bar)返回值是“src/foo src/bar"。

### 7.3.7 join

$(join \<list1>, \<list2>)

名称：连接函数——join

功能：把\<list2>中的单词对应的地加到\<list1>的单词后面。如果\<list1>的单词个数要比\<list2>的多，那么，\<list1>中多出来的单词将保持原样。如果\<list2>的单词个数要比\<list1>多，那么，\<list2>多出来的单词将被复制到\<list2>中。

返回：返回连接过后的字符串

示例：$(join aaa bbb, 111 222 333)返回值是"aaa111 bbb222 333"

# 8 foreach函数

foreach函数和别的函数非常的不一样。因为这个函数是用来做循环用的，Makefile中的foreach函数几乎是仿照与Unix标准Shell（/bin/sh）中的for语句，或是C-Shell（/bin/csh）中的foreach语句而构建的。它的语法是：

$(foreach \<var>, \<list>,\<text>)

这个函数的意思是，把参数\<list>中单词逐一取出来放到参数\<var>所指定的变量中，然后再执行\<text>所包含的表达式。每一次\<text>会返回一个字符串，循环过程中，\<text>的所返回的每个字符串会以空格分隔，最后当整个循环结束时，\<text>所返回的每个字符串所组成的整个字符串（以空格分隔）将会是foreach函数的返回值。

所以，\<var>最好是一个变量名，\<list>可以是一个表达式，而\<text>中一般会使用\<var>这个参数来依次枚举\<list>中的单词。举个例子：

```makefile
names := a b c d
files := $(foreach n,$(names), $(n).o)
```

上面的例子中，$(name)中的单词会被挨个取出，并存到变量“n”中，“$(n).o”每次根据“$(n)”计算出一个值，这些值以空格分隔，最后作为foreach函数的返回，所以，$(f
iles)的值是“a.o b.o c.o d.o”。


注意，foreach中的<var>参数是一个临时的局部变量，foreach函数执行完后，参数<var>的变量将不在作用，其作用域只在foreach函数当中。

# 9 if函数

if函数很像GNU的make所支持的条件语句——ifeq，if函数的语法是：

$(if \<condition>,\<then-part>)

或是

$(if\<condition>,\<then-part>,\<else-part>)

可见，if函数可以包含“else”部分，或是不包含。即if函数的参数可以是两个，也可以是三个。\<condition>参数是if的表达式，如果其返回的为非空字符串，那么这个表达式就相当于返回真，于是，\<then-part>会被计算，否则\<else-part>会被计算。

而if函数的返回值是，如果\<condition>为真（非空字符串），那个\<then-part>会是整个函数的返回值，如果\<condition>为假（空字符串），那么\<else-part>会是整个函数的返回值，此时如果\<else-part>没有被定义，那么整个函数返回空字符串。

所以，\<then-part>和\<else-part>只会有一个被计算。

# 10 call函数

call函数是唯一一个可以用来创建新的参数化的函数。你可以写一个非常复杂的表达式，这个表达式中，你可以定义许多参数，然后你可以用call函数来向这个表达式传递参数。其语法是：

$(call \<expression>, \<parm1>,\<parm2>,\<parm3>……)

当make执行这个函数时，\<expression>参数中的变量，如$(1)，$(2)，$(3)等，会被参数\<parm1>，\<parm2>，\<parm3>依次取代，而\<expression>的返回值就是call函数的返回值。例如：

```makefile
reverse = $(1)$(2)

foo = $(call reverse, a, b)
```

那么foo的值就是“a b”。当然，参数的次序是可以自定义的，不一定是顺序的，如：

```makefile
reverse = $(2)$(1)
foo = $(call reverse, a, b)
```

此时的foo的值就是“b a”。

# 11 origin函数

origin函数不像其它函数，它并不操作变量的值，它只是告诉你你的这个变量是哪里来的？其语法是：

$(origin \<variable>)

注意，\<variable>是变量的名字，不应该是引用。所以最好不要在\<variable>中使用$字符。Origin函数会以其返回值来告诉你这个变量的“出生情况”，下面，是origin函数的返回值：

“undefined”

如果\<variable>从来没有定义过，origin函数返回这个值“undefined”

“default"

如果\<variable>是一个默认的定义，比如“CC"这个变量，这种变量我们将在后面讲述。

“environment”

如果\<variable>是一个环境变量，并且当Makefile被执行时，“-e”参数没有被打开。

“file"

如果\<variable>这个变量被定义在Makefile中。

“command line"

如果这个变量是被命令行定义的。

“override"

如果\<variable>是被override指示符重新定义的。

"automatic"

如果\<variable>是一个命令运行中的自动化变量。关于自动化变量将在后面讲述。

这些信息对于我们编写Makefile是非常有用的，例如，假设我们有一个Makefile其包了一个定义文件Make.def，在Make.def中定义了一个变量“bletch”，而我们的环境中也有一
个环境变量“bletch”，此时，我们想判断一下，如果变量来源于环境，那么我们就把之重定义了，如果来源于Make.def或是命令行等非环境的，那么我们就不重新定义它。于是
，在我们的Makefile中，我们可以这样写：


ifdef bletch


ifeq "$(origin bletch)" "environment"


bletch = barf, gag, etc.

endif

endif

当然，你也许会说，使用override关键字不就可以重新定义环境中的变量了吗？为什么需要使用这样的步骤？是的，我们用override是可以达到这样的效果，可是override过于粗暴，它同时会把从命令行定义的变量也覆盖了，而我们只想重新定义环境传来的，而不想重新定义命令行传来的。

# 12 shell函数

shell函数也不像其它的函数。顾名思义，它的参数应该是操作系统Shell的命令。它和反引号“`”是相同的功能。这就是说，shell函数把执行操作系统命令后的输出作为函数返回。于是，我们可以用操作系统命令以及字符串处理命令awk，sed等等命令来生成一个变量，如：

```makefile
content := $(shell cat foo)
files := $(shell echo *.c)
```

注意，这个函数会新生成一个shell程序来执行命令，所以需要注意其运行性能，如果您的makefile中有一些比较复杂的规则，并大量使用了这个函数，那么对于你的系统性能是有害的。特别是makefile的隐晦规则可能让你的shell函数执行的次数比你想象的多得多。

# 13 控制make的函数

make提供了一些函数来控制make的运行。通常，你需要检测一些运行Makefile时的运行时信息，并且根据这些信息来决定，你是让make继续执行，还是停止。

$(error \<text….>)

产生一个致命的错误，\<text…>是错误信息。注意，error函数不会在一被使用就会产生错误信息，所以如果你把其定义在某个变量中，并在后续的脚本中使用这个变量，那么也是可以的。例如：

示例一、

```makefile
ifdef ERROR_001
$(error error is $(ERROR_001))
endif
```

示例二、

```makefile
ERR = $(error found an error)

.PHONY:err
err:$(ERR)
```

示例一会在变量ERROR_001定义了后执行时产生error调用，而示例二则在目录err被执行时才发生error调用。

$(warning <text ...> )


这个函数很像error函数，只是它并不会让make退出，只是输出一段警告信息，而make继续执行。

# 14 make的运行

## 14.1 make的退出码

> 0——表示成功执行。
>
> 1——如果make运行时出现任何错误，其返回1。
>
> 2——如果你使用了make的“-q”选项，并且make使得一些目标不需要更新，那么返回2

## 14.2 指定Makefile

前面我们说过，GNU make找寻默认的Makefile的规则是在当前目录下依次找三个文件——“GNUmakefile"、”makefile“和”Makefile"。其按顺序找这三个文件，一旦找到，就开始读取这个文件并执行。

当前，我们也可以给make命令指定一个特殊名字的Makefile。要达到这个功能，我们要使用make的“-f”或是“–file”参数（“–makeflie”参数也行）。例如，我们有个makefile的名字是“hchen.mk”，那么我们可以这样让make来执行这个文件：

```shell
make -f hchen.mk
```

如果在make的命令行时，你不止一次地使用了“-f"参数，那么，所有指定的makefile将会被连在一起传递给make执行。

## 14.3 指定目标

一般来说，make的最终目标是makefile中的第一个目标，而其它目标一般是由这个目标连带出来的。这是make的默认行为。当然，一般来说，你的makefile中的第一个目标是由许多个目标组成，你可以指示make，让其完成你所指定的目标。要达到这一目的很简单，需在make命令后直接跟目标的名字就可以完成（如前面提到的“make clean"形式）任何在makefile中的目标都可以被指定成终极目标，但是除了以”-“打头，或是包含了”=“的目标，因为这些字符的目标，会被解析成命令行参数或是变量。甚至没有被我们明确写出来的目标也可以成为make的终极目标，也就是说，只要make可以找到其隐含规则推导规则，那么这个隐含目标同样可以被指定成终极目标。

有一个make的环境变量叫“MAKECMDGOALS"，这个变量中会存放你所指定的终极目标列表，如果在命令行上，你没有指定目标，那么，这个变量是空值。这个变量可以让你使用在一些比较特殊情况下。比如下面的例子：

```makefile
sources = foo.c bar.c
ifneq($(MAKECMEGOALS), clean)
include $(sources:.c=.d)
endif
```

基于上面的这个例子，只要我们输入的命令不是“make clean”，那么makefile会自动包含“foo.d”和“bar.d”这两个makefile。

使用指定终极目标的方法可以很方便地让我们编译我们的程序，例如下面这个例子：

```makefile
.PHONY:all
all:prog1 prog2 prog3 prog4
```

从这个例子中，我们可以看到，这个makefile中有四个需要编译的程序——“prog1”， “prog2”， “prog3”和 “prog4”，我们可以使用“make all”命令来编译所有的目标
（如果把all置成第一个目标，那么只需执行“make”），我们也可以使用“make prog2”来单独编译目标“prog2”。

即然make可以指定所有makefile中的目标，那么也包括“伪目标”，于是我们可以根据这种性质来让我们的makefile根据指定的不同的目标来完成不同的事。在Unix世界中，软件
发布时，特别是GNU这种开源软件的发布时，其 makefile都包含了编译、安装、打包等功能。我们可以参照这种规则来书写我们的makefile中的目标。

```tex
"all"		这个伪目标是所有目标的目标，其功能一般是编译所有的目标
"clean"		这个伪目标功能是删除所有被make创建的文件
"install"	这个伪目标功能是安装已编译好的程序，其实就是把目标执行文件拷贝到指定的目标中去。
"print"		这个伪目标的功能是列出改变过的源文件。
"tar"		这个伪目标功能是把源程序打包备份。也就是tar文件。
"dise"		这个伪目标功能是创建一个压缩文件，一般是tar文件压缩成z文件。或是gz文件。
"TAGS"		这个伪目标功能是更新所有的目标，已备完整的重编译使用。
"check"和"test"	这两个伪目标一般用来测试makefile的流程。
```

当然一个项目的makefile中也不一定要书写这样的目标，这些东西都是GNU的东西，但是我想，GNU搞出这些东西一定有其可取之处（等你的UNIX下的程序文件一多时你就会发现这些功能很有用了），这里只不过是说明了，如果你要书写这种功能，最好使用这种名字命名你的目标，这样规范一些，规范的好处就是——不用解释，大家都明白。而且如果你的makefile中有这些功能，一是很实用，二是可以显得你的makefile很专业（不是那种初学者的作品）。

## 14.4 检查规则

有时候，我们不想让我们的makefile中的规则执行起来，我们只是想检查一下我们的命令，或是执行的序列。于是我们可以使用make命令的下述参数：

* “-n”
* “–just-print”
* “–dry-run”
* “–recon”

不执行参数，这些参数只是打印命令，不管目标是否更新，把规则和连带规则下的命令打印出来，但不执行，这些参数对于我们调试makefile很有用处。

* “-t”
* “–touch”

这个参数的意思就是把目标文件的时间更新，但不更改目标文件。也就是说，make假装编译目标，但不是真正的编译目标，只是把目标变成已编译过的状态。

* “-q”

* “–question”

这个参数的行为是找目标的意思，也就是说，如果目标存在，那么其什么也不会输出，当然也不会执行编译，如果目标不存在，其会打印出一条出错信息。

* “-W \<file>”
* “–what-if=\<file>”
* “–assume-new=\<file>"
* "–new-file=\<file>"

这个参数需要指定一个文件。一般是源文件（或依赖文件），Make会根据规则推导来运行依赖于这个文件的命令，一般来说，可以和“-n”参数一同使用，来查看这个依赖文件所发生的依赖规则命令。

另外一个很有意思的用法是结合“-p”和“-v”来输出makefile被执行时的信息（这个将在后面讲述）。

## 14.5 make的参数

下面列举了所有GNU make 3.80版的参数定义。其它版本和产商的make大同小异，不过其它产商的make的具体参数还是请参考各自的产品文档。

* -b、-m：这两个参数的作用是忽略和其它版本make的兼容。
* -B、-always-make：认为所有的目标都需要更新（重编译）。
* -C\<cir>、–directory=\<dir>：指定读取makefile的目录。如果有多个-C参数，make的解释是后面的路径以前面的作为相对路径，并以最后的目录作为被指定目录。如：make -C ~hchen/test -C prog 等价于make -C ~hchen/test/prog。
* –debug[=\<options]：输出make的调试信息。它由几种不同的级别可供选择，如果没有参数，那就是输出最简单的调试信息。下面是\<options>的取值：
  * a：也就是all，输出所有的调试信息。（会非常的多）
  * b：也就是basic，只输出简单的调试信息。即输出不需要编译的目标。
  * v：也就是verbose，在b选项的级别之上。输出的信息包括哪个makefile被解析，不需要编译的依赖文件（或是依赖目标）等。
  * i：也就是implicit：输出所以的隐含规则。
  * j：也就是jobs，输出执行规则中命令的详细信息，如命令的PID、返回码等。
  * m：也就是makefile，输出make读取makefile，更新makefile，执行makefile的信息。
* -d：相当于–debug=a
* -e、-environment-overrides：指明环境变量的值覆盖makefile中定义的变量的值。
* -f=\<file>、–file=\<file>、–makefile=\<file>：指定需要执行的makefile。
* -h、–help：显示帮助信息
* -i、–ignore-error：在执行时忽略所有的错误。
* -I \<dir>、–include-dir=\<dir>：指定一个被包含makefile的搜索目标。可以使用多个-I参数来指定多个目录。
* -j[\<jobsnum]、-jobs[=\<jobsnum>]：指同时运行命令的个数。如果没有这个参数，make运行命令时能运行多少就运行多少。如果有一个以上的-j参数，那么仅最后一个有效。[这个参数在MS-DOS中是无用的]
* -k、–keep-going：出错也不停止运行。如果生成一个目标失败了，那么依赖于其上的目标就不会被执行了。
* -I \<load>、–load-average[=\<load>]、–max-load[=\<load>]：指定make运行命令时的负载。
* -n、–just-print、–dry-run、–recon：仅输出执行过程中的命令序列，但并不执行
* -o \<file>、-old-file=\<file>、–assume-old=\<file>：不重新生成指定的\<file>，即使这个目标的依赖文件新于它。
* -p、-print-data-base：输出makefile中的所有数据，包括所有的规则和变量。这个参数会让一个简单的makefile都会输出一堆信息。如果你只是想输出信息而不想执行makefile，你可以使用make -qp 命令。如果您想查看执行makefile前的预设变量和规则，你可以使用make -p -f /dev/null。这个参数输出的信息会包含着你的makefile文件的文件名和行号，所以，用这个参数来调试你的makefile会是很有用的，特别是当你的环境变量很复杂的时候。
* -q、–question：不运行命令，也不输出。仅仅检查所指定的目标是否需要更新。如果是0则说明要更新，如果是2则说明有错误发生。
* -r、–no-builtin-rules：禁止make使用任何隐含规则。
* -R、–no-builtin-variabes：禁止make使用任何作用于变量上的隐含规则。
* -s、–silent、–quiet在命令运行时不输出命令的输出。
* -S、–no-keep-going、–stop：取消-k选项的作用。因为有些时候，make的选项是从环境变量MAKEFLAGS中继承下来的。所以你可以在命令行中使用这个参数来然环境变量中的-k选项失效。
* -t、–touch：相当于UNIX的touch命令，只是把目标的修改日期变成最新的，也就是阻止生成目标的命令运行。
* -w、–print-directory：输出运行makefile之前和之后的信息。这个参数对于跟踪嵌套式调用make时很有用。
* –no-print-directory：禁止-w选项。
* -v、–version；输出make程序的版本、版权等关于make的信息。
* -W \<file>、–what-if=\<file>、–new-file=\<file>、–assume-file=\<file>：假定目标\<file>需要更新，如果-n选项使用，那么这个参数会输出该目标更新时的运行动作。如果没有“-n”那么就像运行UNIX的“touch”命令一样，使得\<file>的修改时间为当前时间。
* –warn-undefined-varibales：只要make发现有未定义的变量，那么就输出警告信息。

# 15 隐含规则

在我们使用makefile时，有一些我们会经常使用，而且使用频率非常高的东西，比如，我们编译C/C++的源程序为中间目标文件（Unix下是[.o]文件，Windows下是[.obj]文件）。本章讲述的就是一些在Makefile中的隐含的，早先约定了的，不需要我们再写出来的规则。

“隐含规则”也就是一种惯例，make会按照这种惯例心照不宣地来运行，哪怕我们的makefile没有书写这样的规则。例如，把[.c]文件编译成[.o]文件这一规则，你根本就不用写出来，make会自动推导出这种规则，并生成我们需要的[.o]文件。

“隐含规则”会使用一些我们系统变量，我们可以改变系统变量的值来定制隐含规则的运行时的参数。如系统变量“CFLAGS”可以控制编译时编译器参数。我们还可以通过“后缀规则”可以用来保证我们Makefile的兼容性。

我们了解了“隐含规则”，可以让其为我们更好的服务，也会让我们知道一些“约定俗成”了的东西，而不至于使得我们在运行Makefile时出现一些我们觉得莫名其妙的东西。当然，任何事物都是矛盾的，水能载舟，亦可覆舟，所以，有时候“隐含规则”也会给我们造成不小的麻烦。只有了解了它，我们才能更好地使用它。

## 15.1 使用隐含规则

如果要使用隐含规则生成你需要的目标，你所需要做的就是不要写出这个目标的规则。那么，make会试图去自动推导产生这个目标的规则和命令，如果make可以自动推导生成这个目标的规则和命令，那么这个行为就是隐含规则的自动推导。当然，隐含规则是make事先约定好的一些东西。例如，我们下面的一个Makefile：

foo：foo.o bar.o

cc -o foo foo.o bar.c $(CFLAGS) $(LDFLAGS)

我们可以注意到，这个Makefile中并没有写下如何生成foo.o和bar.o这两个目标的规则和命令。因为make的“隐含规则”功能会自动为我们自动去推导这两个目标的依赖目标和生成命令。

make会在自己的“隐含规则”库中寻找可以用的规则，如果找到，那么就会使用。如果找不到，那么就会报错。在上面的那个例子中，make调用的隐含规则是，把[.o]的目标的依赖文件置成[.c]，并使用C的编译命令“cc -c $(CFLAGS)[.c]”来生成[.o]的目标。也就是说，我们完全没有必要写下面的两条规则：

foo.o:foo.c

cc -c $(CFLAGS)

bar.o : bar.c

cc -c bar.c $(CFLAGS)

因为，这已经是“约定”好了的事了，make和我们约定好了用C编译器“cc”生成[.o]文件的规则，这就是隐含规则。

当然，如果我们为[.o]文件书写了自己的规则，那么make就不会自动推导并调用隐含规则，它会按照我们写好的规则忠实地执行。

还有，在make的隐含规则库中，每一条隐含规则都在库中有其顺序，越靠前的越是经常被使用的，所以，这会导致我们有些时候即使我们显式地指定了目标依赖，make也不会管。如下面这条规则（没有命令）：

foo.o:foo.p

依赖文件“foo.p”（Pascal程序的源文件）有可能变得没有意义。如果目录下存在了“foo.c”文件，那么我们的隐含规则一样会生效，并会通过 “foo.c”调用C的编译器生成foo.o文件。因为，在隐含规则中，Pascal的规则出现在C的规则之后，所以，make找到可以生成foo.o的 C的规则就不再寻找下一条规则了。如果你确实不希望任何隐含规则推导，那么，你就不要只写出“依赖规则”，而不写命令。

## **15.2 隐含规则一览**



这里我们将讲述所有预先设置（也就是make内建）的隐含规则，如果我们不明确地写下规则，那么，make就会在这些规则中寻找所需要规则和命令。当然，我们也可以使用make的参数“-r”或“--no-builtin-rules”选项来取消所有的预设置的隐含规则。


当然，即使是我们指定了“-r”参数，某些隐含规则还是会生效，因为有许多的隐含规则都是使用了“后缀规则”来定义的，所以，只要隐含规则中有“后缀列表 ”（也就一系统
定义在目标.SUFFIXES的依赖目标），那么隐含规则就会生效。默认的后缀列表是：.out,.a, .ln, .o, .c, .cc, .C, .p, .f, .F, .r, .y, .l, .s, .S, .mod, .sym, .def, .
h, .info, .dvi, .tex, .texinfo, .texi, .txinfo, .w, .ch .web, .sh, .elc, .el。具体的细节，我们会在后面讲述。


还是先来看一看常用的隐含规则吧。


1、编译C程序的隐含规则。
“<n>.o”的目标的依赖目标会自动推导为“<n>.c”，并且其生成命令是“$(CC) –c $(CPPFLAGS) $(CFLAGS)”


2、编译C++程序的隐含规则。
“<n>.o” 的目标的依赖目标会自动推导为“<n>.cc”或是“<n>.C”，并且其生成命令是“$(CXX) –c $(CPPFLAGS) $(CFLAGS)”。（建议使用“.cc”作为C++源文件的后缀，而
不是“.C”）


3、编译Pascal程序的隐含规则。
“<n>.o”的目标的依赖目标会自动推导为“<n>.p”，并且其生成命令是“$(PC) –c $(PFLAGS)”。


4、编译Fortran/Ratfor程序的隐含规则。
“<n>.o”的目标的依赖目标会自动推导为“<n>.r”或“<n>.F”或“<n>.f”，并且其生成命令是:
“.f” “$(FC) –c $(FFLAGS)”
“.F” “$(FC) –c $(FFLAGS) $(CPPFLAGS)”
“.f” “$(FC) –c $(FFLAGS) $(RFLAGS)”


5、预处理Fortran/Ratfor程序的隐含规则。
“<n>.f”的目标的依赖目标会自动推导为“<n>.r”或“<n>.F”。这个规则只是转换Ratfor或有预处理的Fortran程序到一个标准的Fortran程序。其使用的命令是：
“.F” “$(FC) –F $(CPPFLAGS) $(FFLAGS)”
“.r” “$(FC) –F $(FFLAGS) $(RFLAGS)”


6、编译Modula-2程序的隐含规则。
“<n>.sym” 的目标的依赖目标会自动推导为“<n>.def”，并且其生成命令是：“$(M2C) $(M2FLAGS) $(DEFFLAGS)”。“<n.o>” 的目标的依赖目标会自动推导为“<n>.mod”，
并且其生成命令是：“$(M2C) $(M2FLAGS) $(MODFLAGS)”。


7、汇编和汇编预处理的隐含规则。
“<n>.o” 的目标的依赖目标会自动推导为“<n>.s”，默认使用编译品“as”，并且其生成命令是：“$(AS) $(ASFLAGS)”。“<n>.s” 的目标的依赖目标会自动推导为“<n>.S”
，默认使用C预编译器“cpp”，并且其生成命令是：“$(AS) $(ASFLAGS)”。


8、链接Object文件的隐含规则。
“<n>” 目标依赖于“<n>.o”，通过运行C的编译器来运行链接程序生成（一般是“ld”），其生成命令是：“$(CC) $(LDFLAGS) <n>.o $(LOADLIBES) $(LDLIBS)”。这个规则对
于只有一个源文件的工程有效，同时也对多个Object文件（由不同的源文件生成）的也有效。例如如下规则：

x : y.o z.o


并且“x.c”、“y.c”和“z.c”都存在时，隐含规则将执行如下命令：


cc -c x.c -o x.o
cc -c y.c -o y.o
cc -c z.c -o z.o
cc x.o y.o z.o -o x
rm -f x.o
rm -f y.o
rm -f z.o


如果没有一个源文件（如上例中的x.c）和你的目标名字（如上例中的x）相关联，那么，你最好写出自己的生成规则，不然，隐含规则会报错的。


9、Yacc C程序时的隐含规则。

“<n>.c”的依赖文件被自动推导为“n.y”（Yacc生成的文件），其生成命令是：“$(YACC) $(YFALGS)”。（“Yacc”是一个语法分析器，关于其细节请查看相关资料）


10、Lex C程序时的隐含规则。
“<n>.c”的依赖文件被自动推导为“n.l”（Lex生成的文件），其生成命令是：“$(LEX) $(LFALGS)”。（关于“Lex”的细节请查看相关资料）


11、Lex Ratfor程序时的隐含规则。
“<n>.r”的依赖文件被自动推导为“n.l”（Lex生成的文件），其生成命令是：“$(LEX
) $(LFALGS)”。


12、从C程序、Yacc文件或Lex文件创建Lint库的隐含规则。
“<n>.ln” （lint生成的文件）的依赖文件被自动推导为“n.c”，其生成命令是：“$(LINT) $(LINTFALGS) $(CPPFLAGS) -i”。对于“<n>.y”和“<n>.l”也是同样的规则。
