# 1 shell 编程

## 1.1 shell 历史

Shell的作用是解释执行用户的命令，用户输入一条命令，Shell就解释执行一条，这种方式称为交互式（Interactive），Shell还有一种执行命令的方式称为批处理(Batch)，用户事先写一个Shell脚本（Script），其中有很多条命令，让Shell一次把这些命令执行完，而不必一条一条地敲命令。Shell脚本和编程语言很相似，也有变量和流程控制语句，但Shell脚本是解释执行的，不需要编译，Shell程序从脚本中一行一行读取并执行这些命令，相当于一个用户把脚本中的命令一行一行敲到Shell提示符下执行。

由于历史原因，UNIX系统上有很多种Shell：

1) sh(Bourne Shell)：由Steve Bourne开发，各种UNIX系统都配有sh。
2) csh(C Shell)：由Bill Joy开发，随BSD UNIX发布，它的流程控制语句很像C语言，支持很多Bourne Shell所不支持的功能：作业控制，命令历史，命令编辑。
3) ksh(Korn Shell)：由David Korn开发，向后兼容sh的功能，并且添加了csh引入的新功能，是目前很多UNIX系统标准配置的Shell，在这些系统上/bin/sh往往是指向/bin/ksh的符号链接。
4) tcsh(TENEX C Shell)：是csh的增强版本，引入了命令补全等功能，在FreeBSD、MacOS X等系统上替代了csh。
5) bash(Bourne Again Shell)：由GNU开发的Shell，主要目标是与POSIX标准保持一致，同时兼顾对sh的兼容，bash从csh和ksh借鉴了很多功能，是各种Linux发行版标准配置的Shell，**在Linux系统上/bin/sh往往是指向/bin/bash的符号链接**。虽然如此，bash和sh还是有很多不同的，一方面，bash扩展了一些命令和参数，另一方面，bash并不完全兼容sh，有些行为并不一致，所以bash需要模拟sh的行为：当我们通过sh这个程序名启动bash时，bash可以假装自己是sh，不认扩展的命令，并且行为与sh保持一致。

使用$SHELL环境变量查看当前用户使用哪种shell：

/etc/passwd文件中每个用户最后一列信息为用户登录后执行的第一条命令，即启动shell

```shell
echo $SHELL
```

使用**chsh命令**修改当前用户的shell：

```shell
chsh
```



```shell
itcast$ vim /etc/passwd
	其中最后一列显示了用户对应的shell类型
	root:x:0:0:root:/root:/bin/bash
	nobody:x:65534:65534:nobody:/nonexistent:/bin/sh
	syslog:x:101:103::/home/syslog:/bin/false
	itcast:x:1000:1000:itcast,,,:/home/itcast:/bin/bash
	ftp:x:115:125:ftp daemon,,,:/srv/ftp:/bin/false
```

用户在命令行输入命令后，一般情况下Shell会**fork**一个子进程并在子进程中**exec**该命令，**但是Shell的内建命令例外，执行内建命令相当于调用Shell进程中的一个函数，并不创建新的进程。**如cd、alias、umask、exit等命令即是内建命令，凡是用which命令查不到程序文件所在位置的命令都是内建命令，内建命令没有单独的man手册，要在man手册中查看内建命令，应该执行：

```shell
itcast$ man bash-builtins
```

如export、shift、if、eval、[、for、while等等。内建命令虽然不创建新的进程，但也会有Exit Status，通常也用0表示成功非零表示失败，虽然内建命令不创建新的进程，但执行结束后也会有一个状态码，也可以用特殊变量**$?**读出。

## 1.2 执行脚本

编写一个简单的脚本test.sh：

```shell
#!/bin/sh
echo HelloWorld
cd ..
ls
```

Shell脚本中用#表示注释，相当于C语言的//注释。但如果#位于第一行开头，并且使#!（称为Shebang）则例外，它表示该脚本使用后面指定的解释器/bin/sh解释执行。如果把这个脚本文件加上可执行权限然后执行：

```shell
itcast$ chmod a+x test.sh
itcast$ ./test.sh
```

Shell会fork一个子进程并调用exec执行./test.sh这个程序，exec系统调用应该把子进程的代码段替换成./test.sh程序的代码段，并从它的_start开始执行。然而test.sh是个文本文件，根本没有代码段和_start函数，怎么办呢？其实exec还有另外一种机制，如果要执行的是一个文本文件，并且第一行用Shebang指定了解释器，**则用解释器程序的代码段替换当前进程**，并且从解释器的_start开始执行，**而这个文本文件被当做命令行参数传给解释器**。

因此，执行上述脚本相当于执行程序：

```shell
itcast$ /bin/sh ./test.sh
```

**以这种方式执行不需要test.sh文件具有可执行权限。**

**如果将命令行下输入的命令用()括号括起来，那么也会fork出一个子Shell执行小括号中的命令**，一行中可以输入由分号;隔开的多个命令，比如：

```shell
itcast$ (cd ..;ls -l)
```

和上面两种方法执行Shell脚本的效果是相同的，**cd .. 命令改变的是子Shell的PWD，而不会影响到交互式Shell。**然而命令

```shell
itcast$ cd ..; ls -l
```

则有不同的效果，cd .. 命令是直接在交互式Shell下执行的，改变的是交互式Shell的PWD，**然而这种方式相当于这样执行Shell脚本**：

```shell
itcast$ source ./test.sh
```

或者

```shell
itcast$ . ./test.sh
```

**source或者.命令是Shell的内建命令，这种方式也不会创建子Shell，而是直接在交互式Shell下逐行执行脚本中的命令。常用于加载配置文件。**

## 1.3 基本语法

### 1.3.1 变量

按照惯例，Shell变量通常由字母加下划线开头，由任意长度的字母、数字、下划线组成。

在Shell中定义或赋值一个变量：

```shell
VARNAME=value
```

**注意等号两边都不能有空格，否则会被Shell解释成命令和命令行参数**

变量的使用，用$符号跟上变量名表示对某个变量取值，变量名可以加上花括号来表示变量名的范围：

```shell
echo $VARNAME
echo ${VARNAME}_suffix #使用花括号来分离VARNAME和_suffix，不至于把VARNAME_suffix当做变量名
```

#### 1.3.1.1 变量的分类

根据变量的作用域不同，shell分为两种变量

1) 环境变量

   > inux的一个重要概念是环境变量，环境变量需要进行定义。有些是由系统设置的，有些是由用户自定义的，还有一些是由shell或加载执行程序时，由程序设置的。**环境变量是一个赋值给它的字符串**。分配的值可以是数字、文本、文件名、设备或任何其他类型的数据。
   >
   > * Linux 作为一个多用户多任务的操作系统，能够为每个用户提供独立的、合适的工作运行环境，因此，一个相同的环境变量会因为用户身份的不同而具有不同的值。
   > * 子进程修改继承自父进程的环境变量进行修改，不会影响父进程的环境变量。

   **Linux系统中重要的10个环境变量：**

   | 环境变量名称 | 作用                                   |
   | ------------ | -------------------------------------- |
   | HOME         | 用户的主目录（也称家目录）             |
   | SHELL        | 用户使用的 Shell 解释器名称            |
   | PATH         | 定义命令行解释器搜索用户执行命令的路径 |
   | EDITOR       | 用户默认的文本解释器                   |
   | RANDOM       | 生成一个随机数字                       |
   | LANG         | 系统语言、语系名称                     |
   | HISTSIZE     | 输出的历史命令记录条数                 |
   | HISTFILESIZE | 保存的历史命令记录条数                 |
   | PS1          | Bash解释器的提示符                     |
   | MAIL         | 邮件保存路径                           |

   > 环境变量可以从父进程传给子进程，因此Shell进程的环境变量可以从的前Shell进程传给fork出来的子进程。
   >
   > 用**printenv**命令可以**显示当前Shell进程的环境变量**。注意：**环境变量只能从父进程传递给子进程而子进程不能够传给父进程。**

2) 本地变量

   > 只存在于当前Shell进程，**用set命令可以显示当前Shell进程中定义的所有变量（包括本地变量和环境变量）和函数**。
   >
   > 环境变量是任何进程都有的概念，而本地变量是Shell特有的概念。在Shell中，环境变量和本地变量的定义和用法相似。
   >
   > 一个变量定义后仅存在于当前Shell进程，它是本地变量，**用export命令可以把本地变量导出为环境变量，定义和导出环境变量通常可以一步完成**：

   ```shell
   itcast$ export VARNAME=value
   ```

   也可以分两步完成：

   ```shell
   itcast$ VARNAME=value
   itcast$ export VARNAME
   ```

   本地变量根据作用域，也分为全局的本地变量以及局部的本地变量。

   **全局的本地变量在整个shell文件中都可以访问**，比如直接声明和定义一个变量var=value其实就是一个全局的本地变量，但是如果在shell定义的函数里边可以使用**local**来声明一个局部变量，**这种变量作用域仅限于函数内**。

```shell
function test
{
	local var="this is a local variable"
}
```

#### 1.3.1.2 删除变量

用**unset**命令可以删除已定义的环境变量或本地变量。

```shell
itcast$ unset VARNAME
```

如果一个一个变量叫做VARNAME，用'VARNAME'可以表示它的值，在不引起歧义的情况下也可以用VARNAME表示它的值。通过以下例子比较这两种表示法的不同：

```shell
itcast$ echo $SHELL
```

注意，在定义变量时不用""取变量值时要用$。和C语言不同的是，Shell变量不需要明确定义类型，是一种弱类型的语言，事实际上**Shell变量的值都是字符串**，比如我们定义VAR=45，其实VAR的值是字符串45而非整数。Shell变量不需要先定义后使用，**如果对一个没有定义的变量取值，则值为空字符串**。

### 1.3.2 环境变量介绍

> Linux的一个重要概念是环境变量，环境变量需要进行定义。有些是由系统设置的，有些是由用户自定义的，还有一些是由shell或加载执行程序时，由程序设置的。环境变量是一个赋值给它的字符串。分配的值可以是数字、文本、文件名、设备或任何其他类型的数据。

#### 1.3.2.1 Linux 环境变量介绍

> Linux中环境变量包括系统级和用户级，系统级的环境变量是每个登录到系统的用户都要读取的系统变量，而用户级的环境变量则是该用户使用系统时加载的环境变量。所以管理环境变量的文件也分为系统级和用户级的.

1) 系统级

   > **/etc/environment**：系统在登录时读取的第一个文件，**用于为所有进程设置环境变量**。系统使用此文件时并不是执行此文件中的命令，而是根据`KEY=VALUE`模式的代码，对`KEY`赋值以`VALUE`，因此文件中如果要定义`PATH`环境变量，只需加入类似如`PATH=$PATH:/xxx/bin`的代码即可。
   >
   > **/etc/profile**：是系统登录时执行的第二个文件，**可以用于设定针对全系统所有用户的环境变量**。该文件一般是调用`/etc/bash.bashrc`文件。
   >
   > **/etc/bash.bashrc**：系统级的`bashrc`文件，**为每一个运行bash shell的用户执行此文件**。此文件会在用户每次打开shell时执行一次。
   >
   > **注意**：`/etc/environment`是设置整个系统的环境，而`/etc/profile`是设置所有用户的环境，前者与登录用户无关，后者与登录用户有关。 这两个文件修改后一般都要重启系统才能生效。

2) 用户级

   > **~/.profile:** 是对应当前登录用户的`profile`文件，用于定制当前用户的个人工作环境。
   >
   > 每个用户都可使用该文件输入专用于自己使用的shell信息,**当用户登录时,该文件仅仅执行一次**。默认情况下，会设置一些环境变量，执行用户的`.bashrc`文件。
   >
   > **~/.bashrc**: 是对应当前登录用户的bash初始化文件，**当用户每次打开shell时，系统都会执行此文件一次**。通常设置环境变量修改这个文件。
   >
   > 上述配置文件执行先后顺序如下：
   >
   > > **/etc/enviroment `–> `/etc/profile` –> `~/.profile` –> `/etc/bash.bashrc `–> `~/.bashrc**

#### 1.3.2.2 环境变量的作用

> 环境变量相当于给系统或用户应用程序设置的一些参数，具体起什么作用这当然和具体的环境变量相关。比如`PATH`，是告诉系统，当要求系统运行一个程序而没有告诉它程序所在的完整路径时，系统除了在当前目录下面寻找此程序外，还应到哪些目录下去寻找；再如tc或vc++中，`set include=path1;path2`; 是告诉编译程序到哪里去找.h类型的文件；当然不仅仅是指定什么路径，还有其它的作用的，如`set dircmd=/4` 设置一个环境变量的作用是在使用dir命令时会把`/4`作为缺省的参数添加到你的dir命令之后，就像你的每个命令都加了/4参数，它实际上是给命令解释程序`command`设置的一个环境变量，并且是给`dir`这个内部命令。

#### 1.3.2.3 配置环境变量的方法

1) 临时环境变量

   > linux下设定环境变量时，如果只是临时用一下，可以直接在shell下用`set`或`export`命令设定环境变量。但是只能在当前shell环境下可以用，切换或关闭重新进入就会失效。具体配置方法，如下，
   >
   > ```shell
   > #终端输入：
   > export MYSQLPATH=/home/mysql  #MYSQLPATH设置为该路径
   > #终端查看一个特定环境变量包含的内容，比如，MYSQLPATH，PATH
   > echo $PATH
   > echo $MYSQLPATH
   > ```

   2) 永久环境变量

      > 设置的环境变量，需要经常使用的，而不是临时使用，把上面的设置环境变量命令写到上面提到的相应配置文件中即可，则可以每次开机或打开shell时自动设置，
      >
      > **例如，**
      >
      > 只需要当前用户生效的环境变量：
      >
      > 终端中输入：`sudo vi ~/.bashrc`，编辑这个文件，在其末尾添加：
      >
      > ```shell
      > export MYSQLPATH=/home/mysql:$MYSQLPATH
      > # path采用:来分隔,冒号左右不需要空格.
      > # :$MYSQLPATH在后面新添加的优先搜索，$MYSQLPATH:在前面说明新添加的最后搜索，不加代表新路径设置为MYSQLPATH路径。
      > ```
      >
      > **注意**：在终端执行，`source ~/.bashrc` ，使其立即生效，或者重启电脑即可。
      >
      > 设置所有用户生效的环境变更：
      >
      > 终端中输入：`sudo vi /etc/profile`，编辑这个文件，在其末尾添加：
      >
      > ```shell
      > export MYSQLPATH=/home/mysql:$MYSQLPATH
      > # path采用:来分隔,冒号左右不需要空格.
      > # :$MYSQLPATH在后面新添加的优先搜索，$MYSQLPATH:在前面说明新添加的最后搜索，不加代表新路径设置为MYSQLPATH路径。
      > ```
      >
      > **注意**：在终端执行，`source /etc/profile` ，使其立即生效，或者重启电脑即可。

#### 1.3.2.4 PATH 环境变量

> Linux命令其实是一个个命令行程序，这些程序时分布在不同的众多目录中的。当命令行中输入一个命令时，Linux需要到指定的目录下去查找命令对应的程序，而在PATH环境变量就记录这些目录。所以PATH环境变量的作用就是记录命令的查找路径，多个路径之间用英文冒号分割（和Windows系统的PATH变量不同，Windows的PATH变量的路径是用英文分号分割的），有需要是也可以加入自己的路径。具体配置方法，如下，
>
> ```shell
> #比如添加搜索路径/home/cjavapy/python和/home/cjavapy/java 路径到PATH中,采用:来分隔,冒号左右不需要空格
> export PATH=$PATH:/home/cjavapy/python:/home/cjavapy/java
> #若需要将路径放在优先搜索位置，将$PATH放在后面
> export PATH=/home/cjavapy/python:/home/cjavapy/java:$PATH
> ```
>
> **注意：** 配置PATH环境变量可以永久生效，也可以临时生效。

#### 1.3.2.5 常用环境变量

> Linux系统有一些重要常用的环境变量，具体如下，
>
> > **$HOME**：用户家目录。
> >
> > **$SHELL**：用户在使用的Shell解释器名称。
> >
> > **$HISTSIZE**：输出的历史命令记录条数。
> >
> > **$HISTFILESIZE**：保存的历史命令记录条数。
> >
> > **$MAIL**：邮件保存路径。
> >
> > **$LANG**：系统语言、语系名称。
> >
> > **$RANDOM**：生成一个随机数字。
> >
> > **$PS1** ：Bash解释器的提示符。
> >
> > **$PATH**：定义解释器搜索用户执行命令的路径。
> >
> > **$EDITOR**：用户默认的文本编辑器。
> >
> > **$TERM**：表示显示类型。
> >
> > **$RANDOM**：每次引用时生成一个0到32,767之间的随机整数。
> >
> > **$PWD**：指示由cd命令设置的当前工作目录。
> >
> > **$TZ**：指时区。它可以使用GMT、AST等值。
> >
> > **$UID**：显示当前用户的数字用户 ID，在shell 启动时初始化。
>
> 可以使用`echo`查看变量信息，

#### 1.3.2.6 PS环境变量

> PS即是Prompt String，命令提示符的意思。在bash中一共有四个。分别表示为PS1、PS2、PS3、PS4。
>
> 1) PS1
>
>    > PS1是用来控制默认提示符显示格式。
>    >
>    > ```shell
>    > $ echo $PS1
>    > \h:\W \u\$
>    > ```
>    >
>    > PS1的常用参数以及含义：
>    >
>    > > `\d` 代表日期，格式为weekday month date，例如："Mon Aug 1"
>    > >
>    > > `\H` 代表完整的主机名称
>    > >
>    > > `\h` 仅去主机名中的第一个名字
>    > >
>    > > `\t` 显示时间为24小时格式，如HHMMSS
>    > >
>    > > `\T` 显示时间为12小时格式
>    > >
>    > > `\A` 显示时间为24小时格式HHMM
>    > >
>    > > `\@` 显示时间，为12小时格式am/pm
>    > >
>    > > `\u` 当前用户的账号名称
>    > >
>    > > `\v` BASH的版本信息
>    > >
>    > > `\w` 完整的工作目录名称
>    > >
>    > > `\W` 利用basename取得工作目录名称，只显示最后一个目录名
>    > >
>    > > `\#` 下达的第几个命令
>    > >
>    > > `\$` 提示字符，如果是root用户，提示符为`#`，普通用户则为`$`。
>
> 2) PS2
>
>    > 一个非常长的命令可以通过在末尾加`\` 使其分行显示。多行命令的默认提示符是`>` 。我们可以通过修改PS2，将提示符修改为`->`。
>    >
>    > 例如：
>    >
>    > ```shell
>    > $ PS2='->'
>    > $ ls \
>    > ->/etc \
>    > ->/boot
>    > ```
>
> 3) PS3
>
>    > shell脚本中使用select循环时的提示符。
>    >
>    > 例如，
>    >
>    > ```shell
>    > #!/usr/bin/bash
>    > PS3="Select a program to exectue: "
>    > select program in 'ls -F' pwd date
>    > do
>    >   $program
>    > done
>    >   
>    > (The Command Line)
>    > Select a program to exectue: 2
>    > 1) ls -F
>    > 2) pwd
>    > 3) date
>    > # /home/yang
>    > # 在执行脚本的时候，PS3里面的字符串会显示在菜单的底部
>    > ```
>
> 4) PS4
>
>    > PS4是Prompt String 4的缩写，它是Linux/Unix下的一个用于控制脚本调试显示信息的环境变量。
>    >
>    > 用来修改set -x 跟踪输出的前缀。
>    >
>    > 例如，
>    >
>    > ```shell
>    > $ export PS4="+Debug Info: "
>    > $ set -x test_syntax.sh 
>    > +Debug Info: GREETINGS=
>    > ++Debug Info: pwd
>    > +Debug Info: CURRENT_DIR=/Users/liumiao
>    > +Debug Info: '[' _HELLO = _ ']'
>    > ```

### 1.3.3 文件名代换(Globbing)

这些用于匹配的字符称为通配符（Wildcard），如：*?[]具体如下：

```shell
# * 匹配0个或多个任意字符
# ？匹配一个任意字符
# [若干字符] 匹配方括号中任意一个字符的一次出现

itcast$ ls /dev/ttyS*
itcast$ ls ch0?.doc
itcast$ ls ch0[0-2].doc
itcast$ ls ch[012] [0-9].doc
```

注意，Globbing所匹配的文件名**是由Shell展开的，也就是说在参数还没传给程序之前就已经展开了**，比如上述ls ch0[012].doc命令，如果当前目录有ch00.doc和ch02.doc，则传给ls命令的参数实际上是这两个文件名，而不是一个匹配字符串。

### 1.3.4 命令代换

由“**`**”反引号括起来的也是一条命令，Shell先执行该命令，然后将输出结果立刻代换到当前命令行中。例如定义一个变量存放date命令的输出：

```shell
itcast$ DATE=`date`
itcast$ echo $DATE
```

命令代换也可以用**$()**表示：

```shell
itcast$ DATE=$(date)
```

> $0：就是shell脚本的文件名，第0个参数，相当于argv[0]，如./test.sh
>
> $1：传入的第一个参数
>
> $2：传入的第二个参数
>
> 以此类推……
>
> **dirname和basename 命令**
>
> > 该命令可以获取给定路径的目录部分。basename与此相反，basename用于去掉文件名的目录和后缀
> >
> > ```shell
> > itcast$ dirname /usr/bin/sort
> > /usr/bin
> > 
> > itcast$ dirname stdio.h
> > .
> > 
> > itcast$ dirname dir1/file1 dir2/file2
> > dir1
> > dir2
> > 
> > #获取当前Shell程序的目录
> > SHELL=$(cd $(dirname $0);pwd)
> > ```
> >
> > 

### 1.3.5 算术代换

使用$(())，用于算术计算，(())中的Shell变量取值将转换成整数，同样含义的$[]代价，例如：

```shell
itcast$ VAR=45
itcast$ echo $(($VAR+3))	#代价于 $((VAR+3)) 或 $[VAR+3] 或 $[$VAR+3]
```

**$(())中只能用+-*/和()运算，并且只能做整数运算。**

$[base#n]，其中base表示进制，n按照base机制解释，后面再有运算数，按十进制解释。

```shell
echo $[8#10+11]
19
echo $[8#10+8#10]
16
```

### 1.3.6 转义字符

和C语言类似，`\`在Shell中被用作转义字符，用于去除紧跟其后的单个字符的特殊含义（回车除外），换句话说，紧跟其后的字符取字面值。例如：

```shell
itcast$ echo $SHELL
/bin/bash
itcast$ echo \$SHELL
$SHELL
itcast$ echo \\
\
```

比如创建一个文件名为“$ $”的文件（$间含有空格）可以这样：

```shell
itcast$ touch \$\ \$
```

还有一个字符虽然不有特殊含义，但是要用它做文件名也很麻烦，就是`-`号。如果要创建一个文件名以`-`号开头的文件，这样是不正确的：

```shell
itcast$ touch -hello
touch: invalid option --h
Try `touch --help` for more information
```

即使加上`\`转义也还是报错：

```shell
itcast$ touch \-hello
touch: invalid option -- h
Try `touch --help' for more information.
```

因为UNIX命令都把`-`号开头的命令行参数当作命令的选项，而不会当作文件名。如果非要处理以`-`号开头的文件名，可以有两种方法：

```shell
itcast$ touch ./-hello
# 或者
itcast$ touch -- -hello
```

`\`还有一种用法，在`\`后敲回车表示续行，Shell并不会立刻执行命令，而是把光标移到下一行，给出一个续行提示符`>`，等待用户继续输入，最后把所有的续行接到一起当作一个命令执行。例如：

```shell
itcast$ ls \
> -l
（ls -l命令的输出）
```

### 1.3.6 单引号

和C语言不同，Shell脚本中的单引号和双引号都是字符串的界定符，而不是字符的界定符。**单引号用于保持引号内所有字符的字面值，即使引号内的`\`和回车也不例外**，但是字符串中不能出现单引号。如果引号没有配对就输入回车，Shell会给出续行提示符，要求用户把引号配上对。例如：

```shell
itcast$ echo '$SHELL'
$SHELL
itcast$ echo 'ABC\(回车)
> DE'(再按一次回车结束命令)
ABC\
DE
```

### 1.3.7 双引号

被双引号括住的内容，**被视为单一字串**。**它不但允许通配符扩展（转移符号`\`起作用），而且允许变量扩展。**这点与单引号的处理方式不一样。

```shell
itcast$ DATE=$(date)
itcast$ echo "$DATE"
Wed Oct 12 01:57:17 CST 2022

itcast$ echo '$DATE'
$DATE

itcast$ VAR=200
itcast$ echo "$VAR"
200
itcast$ echo '$VAR'
$VAR
```

### 1.3.8 总结

> 1. 不加引号：**不会将含有空格的字符串视为一个整体输出**, 如果内容中有变量等，会先把变量解析出结果，然后再输出最终内容来，如果字符串中带有空格等特殊字符，则不能完整的输出，需要改加双引号，一般连续的字符串，数字，路径等可以用。
> 2. 单引号：**可以说是所见即所得**：即将单引号内的内容原样输出，或者描述为单引号里面看到的是什么就会输出什么。
> 3. 双引号：把双引号内的内容输出出来；如果内容中有变量，会先把变量解析出结果，然后在输出最终内容来。单引号和双引号之分，二者的主要区别在于，被单引号括起来的字符都是普通字符，就算特殊字符也不再有特殊含义；而被双引号括起来的字符中，“$”、"\“和反引号是拥有特殊含义的，”$"代表引用变量的值，而反引号代表引用命令。
> 4. 反引号：如果需要调用命令的输出，或把命令的输出赋予变量，则命令必须使用反引号包含，这条命令才会执行。反引号的作用和 $(命令) 是一样的，但是反引号非常容易和单引号搞混，所以推荐大家使用 $(命令) 的方式引用命令的输出。
>
> ```shell
> # 1、有空格，必须加双引号。比如 name=zhang san 这样执行就会出现问题，而必须用引号括起来
> [root@localhost ~]# name="zhang san"
> #定义变量name的值是zhang san
> 
> #2、如果输出时使用单引号，则$name原封不动地输出
> [root@localhost ~]# echo '$name'
> $name
> #但是如果反引号括起来的命令又被单引号括起来，那么这条命令不会执行，`date`会被当成普通字符输出
> [root@localhost ~]# echo '`date`'
> `date`
> 
> #3、如果输出时使用双引号，则会输出变量name的值sc
> [root@localhost ~]# echo "$name"
> zhang san
> #如果被双引号括起来，那么这条命令会正常执行
> [root@localhost ~]# echo "`date`"
> 2013年 10月 21 日星期一 18:14:21 CST
> 
> 
> #4、如果命令不用反引号包含，那么命令不会执行，而是直接输出 
> [root@localhost ~]# echo ls
> ls
> #只有用反引号包含命令，这条命令才会执行
> [root@localhost -]# echo `ls`
> anaconda-ks.cfg install.log install.log.syslog sh test testfile
> #使用用$(命令)的方式也是可以的
> [root@localhost ~]# echo $(date)
> 2013年 10月 21 日 星期一 18:25:09 CST
> #反引号的命令会正常执行
> [root@localhost ~]# echo `date`
> 2013 10月 21 日星期一 18:16:33 CST
> ```
>
> 

## 1.4 脚本语法

### 1.4.1 条件测试

> **命令test或 [ 可以测试一个条件是否成立，如果测试结果为真，则该命令的Exit Status 为0，如果测试结果为假，则命令的Exit status为1（注意与C语言的逻辑表示正好相反）**。例如测试两个数的大小关系：
>
> ```shell
> itcast$ var=2
> itcast$ test $var -gt 1
> itcast$ echo $?
> 0
> itcast$ test $var -gt 3
> itcast$ echo $?
> 1
> itcast$ [$var -gt 3]
> itcast$ echo $?
> 1
> ```
>
> 虽然看起来很奇怪，但左方括号 [ 确实是一个命令的名字，传给命令的各参数之间应该用空格隔开，比如：
>
> $var、-gt、3、]是 [ 命令的四个参数，它们之间必须用空格隔开。命令test或 [ 的参数形式是相同的，只不过test不需要 ] 参数。以 [ 命令为例，常见的测试命令如下表示所示：
>
> > [-d DIR]	如果DIR存在并且是一个目录则为真
> >
> > [-f FILE]	如果FILE存在且是一个普通文件则为真
> >
> > [FILE1 -nt FILE2] 如果FILE1比FILE2更新则为真
> >
> > [FILE1 -ot FILE2] 如果FILE1比FILE2更旧则为真
> >
> > [-b FILE] 如果FILE是块设备文件则为真
> >
> > [-e FILE] 如果FILE存在则为真
> >
> > * -p：判断是否为有名管道
> > * -s：判断文件是否存在，且大小大于0字节
> > * -r：判断文件是否具有读权限
> > * -S：判断是否是一个socket文件
> > * -w：判断是否具有写权限
> > * -x：判断是否具有可执行权限
> >
> > [-h或-L FILE] 如果FILE是一个软链接则为真
> >
> > [-k FILE] 如果FILE粘着位被设置则为真
> >
> > [-z STRING]	如果STRING的长度为零则为真
> >
> > [-n STRING]	如果STRING的长度非零则为真
> >
> > [STRING1 = STRING2] 如果两个字符串相同则为真
> >
> > [STRING1 == STRING2] 同上
> >
> > [STRING1 != STRING2] 如果字符串不相同则为真
> >
> > [ARG1 OP ARG2]  **ARG1和ARG2应该是整数或者取值为整数的变量**，OP是-eq(等于)、-ne(不等于)、-lt(小于)、-le(小于等于)、-gt(大于)、-ge(大于等于)之中的一个
>
> 和C语言类似，测试条件之间还可以做与、或、非逻辑运算：
>
> > [! EXPR]	EXPR可以是上述中的任意一种测试条件，**!表示逻辑非**
> >
> > [EXPR1 -a EXPR2]	EXPR1和EXPR2可以是上述任意一种测试条件，**-a表示逻辑与**
> >
> > [EXPR1 -o EXPR2]	EXPR1和EXPR2可以是上述任意一种测试条件，**-o表示逻辑或**
>
> 例如：
>
> ```shell
> itcast$ VAR=abc
> itcast$ [-d Desktop -a $VAR = 'abc']
> itcast$ echo $?
> 0
> ```
>
> 注意，如果上例中的$VAR变量事先没有定义，则被Shell展开为空字符串，会造成测试条件的语法错误（展开为[-d Desktop -a  = 'abc']），**作为一种好的Shell编程习惯，应该总是把变量取值放在双引号之中**（展开为[-d Desktop -a "" = 'abc'）：
>
> ```shell
> itcast$ unset VAR
> itcast$ [-d Desktop -a $VAR = 'abc']
> bash: [: too many arguments
> itcast$ [-d Desktop -a "$VAR" = "abc"]
> itcast$ echo $?
> 1
> ```

### 1.4.2 分支

#### 1.4.2.1 if/then/elif/fi

> 和C语言类似，在Shell中用if、then、elif、else、fi这几条命令实现分支控制。这种流程控制语句本质上也是由若干条Shell命令组成的，例如先前讲过的：
>
> ```shell
> if [-f ~/.bashrc]; then
> . ~/.bashrc
> fi
> ```
>
> 其实是三条命令，if [-f  ~/.bashrc]是第一条，then .~/.bashrc，是第二条，fi是第三条。如果两条命令写在同一行需要用;号隔开，一行只写一条命令就不需要写;号了，另外，then后面有换行，但这条命令没写完，Shell会自动续行，把下一行接在then后面当作一条命令处理。和[命令一样，要注意命令和各参数之间必须用空格隔开。if命令的参数组成一条子命令，如果**该子命令的Exit Status为0（表示真）**，则执行then后面的子命令，如果**Exit Status非0（表示假）**，则执行elif、else或者fi后面的子命令。if后面的子命令通常是测试命令，但也可以是其它命令。
>
> Shell脚本没有{}括号，所以用fi表示if语句的结束。见下例：
>
> ```shell
> #! /bin/bash
> 
> if [-f /bin/bash]
> then
> 	echo "/bin/bash is a file"
> else
> 	echo "/bin/bash is not a fele"
> fi
> if :; then echo "always true"; fi
> ```
>
> **":"是一个特殊的命令，称为空命令，该命令不做任何事，但Exit Status总是真。此外，也可以执行true(/bin/true)或false(/bin/false)得到真或假的Exit Status。**再看一个例子：
>
> ```shell
> #! /bin/sh
> echo "Is it morning? Please answer yes or no."
> read YES_OR_NO
> if ["$YES_OR_NO" = "yes"]; then
> 	echo "Good morning"
> elif ["$YES_OR_NO" = "no"]; then
> 	echo "Good afternoon!"
> else	#else不需要then
> 	echo "Sorry, $YES_OR_NO not recognized. Enter yes or no."
> fi
> ```
>
> 上例中的**read命令的作用是等待用户输入一行字符串，将该字符串存到一个Shell变量中。**
>
> 此外，**Shell还提供了&&和`||`语法，和C语言类似，具有Short-circuit特性**，很多脚本喜欢写成这样：
>
> ```shell
> test "$(whoami)" != 'root' && (echo you are using a non-privileged account;)
> 
> echo xxxx || exit -1 #如果前面的命令执行失败，那么就执行退出
> ```
>
> **&&相当于"if……then……"，而`||`相当于"if not……then……"。&&和`||`用于连接两个命令，而上面讲述的-a和-o仅用于在测试表达式中连接两个测试条件，要注意它们的区别**，例如：
>
> ```shell
> test "$VAR" -gt 1 -a "$VAR" -lt 3
> ```
>
> 和下面写法是等价的：
>
> ```shell
> test "$VAR" -gt 1 && test "VAR" -lt 3
> ```

#### 1.4.2 case/esac

case命令可类比C语言的switch/case语句，esac表示case语句块的结束。C语言的case只能匹配整型或字符型常量表达式，而Shell脚本的**case可以匹配字符串和Wildcard，每个匹配分支可以有若干命令，末尾必须以;;结束，执行时找到第一个匹配的分支并执行相应的命令，然后直接跳到esac之后，不需要像C语言一样用break跳出。**

```shell
#! /bin/sh

echo "Is it morning? Please answer yes or no."
read YES_OR_NO
case "$YES_OR_NO" in
yes|y|yes|YES)
	echo "Good Morning";;
[nN][Oo])
	echo "Good Afternoon";;
*)
	echo "Sorry, $YES_OR_NO not recognized. Enter yes or no"
	return 1;;
esac
```

使用case语句的例子可以在系统服务的脚本目录/etc/init.d中找到。这个目录下的脚本大多数具有这种形式（以/etc/init.d/nfs-kernel-server为例）：

```shell
case "$1" in
start)
	……
	;;
stop)
	……
	;;
reload | force-reload)
	……
	;;
restart)
	……
	;;
*)
	log_success_msg"Usage: nfs-kernel-server {start|stop|status|reload|force-reload|restart}"
	;;
esac
```

启动nfs-kernel-server服务的命令是：

```shell
itcast$ sudo /etc/init.d/nfs-kernel-server start
```

$1是一个特殊变量，在执行脚本是自动取值为第一个命令行参数，也就是start，所以进入start)分支执行相关命令。同理，命令行参数指定为stop、reload或restart可以进入其它分支执行停止服务、重新加载配置文件或重新启动服务的相关命令。

### 1.4.3 循环

#### 1.4.3.1 for/do/done

> Shell脚本的for循环结构和C语言很不一样，它类似于某些编程语言的foreach循环。例如：
>
> ```shell
> #! /bin/sh
> 
> for FRUIT in apple banana pear; do
> 	echo "I like $FRUIT"
> done
> ```
>
> FRUIT是一个循环变量，第一次循环$FRUIT的取值是apple，第二次取值是banana，第三次取值是pear。再比如，要将当前目录下的chap0、chap1、chap2等文件改为chap0~、chap1、chap2~等（按惯例，末尾有~字符的文件名表示临时文件），这个命令可以这样写：
>
> ```shell
> for FILENAME in chap?;do mv $FILENAME $FILENAME~; done
> ```
>
> 也可以这样写：
>
> ```shell
> for FILENAME in $(ls chap?); do mv $FILENAME $FILENAME~; done
> ```

#### 1.4.3.2 while/do/done

while的用法和C语言类似。比如一个验证密码的脚本：

```shell
#! /bin/sh

echo "Enter password:"
read TRY
while [ "$TRY" != "secret" ]; do
	echo "Sorry, try again"
	read TRY
done
```

下面的例子通过算术运算控制循环的次数：

```shell
#! /bin/sh

COUNTER=1
while [ "$COUNTER" -lt 10 ]; do
	echo "Here we go again"
	COUNTER=$((COUNTER+1))
done
```

另外，Shell还有until循环，类似于C语言的do……while。

#### 1.4.3.3 break和continue

break[n]可以指定跳出几层循环；continue条过本次循环，但不会跳出循环。

即break跳出，continue跳过。

如：验证密码，用户输错五次便报错退出。

```shell
#! /bin/bash

echo "please input your password:"
read var
count=4
while [ "$var" != "li123" ]; do
	if [ "$count" -le 0 ]; then
		echo "You have aready input secret wrongly for five times!"
		break
	fi
	echo "err, please input again!"
	read var
	count=$((count-1))
done
```

### 1.4.4 位置参数和特殊变量

**有很多特殊变量是被Shell自动赋值的，**我们已经遇到了$?和$1。其它常用的位置参数和特殊变量在这里总结一下：

> $0	相当于C语言main函数的argv[0]
>
> $1、$2…… 这些称为位置参数（Positional Parameter），相当于C语言main函数的argv[1]，argv[2]……
>
> $#	相当于main函数的argc - 1，注意这里的#后面不表示注释
>
> $@	表示参数列表"$1" "$2" ……，例如可以用在for循环中的in后面
>
> $*	表示参数列表"$1" "$2"……，同上
>
> $?	表示上一条命令的Exit Status
>
> $$	当前进程号

**位置参数可以用shift命令左移。**比如shift3表示原来的$4变成$1，原来的$5现在变成了$2等等，原来的$1、$2和$3被丢弃，$0不移动。不带参数的shift命令相当于shift1。例如：

```shell
#! /bin/sh

echo "The program $0 is now running"
echo "The first parameter is $1"
echo "The second parameter is $2"
echo "The parameter list is $@"
shift
echo "The first parameter is $1"
echo "The second parameter is $2"
echo "The parameter list is $@"
```

shift一般可以用来检测参数数量，如：

```shell
#! /bin/sh
sum=0

while [-n "$1"]
do
	su=$[$sum+$1]
	shift
done

echo $sum
```

**注意：另外，$1……$9只能表示9个参数，当参数超过9个之后，必须使用shift才能取到后面的参数，同时$@是支持n个的，没有9个的限制。**

### 1.4.5 输入输出

#### 1.4.5.1 echo，printf

> 显示文本行或变量，或者把字符串输入到文件。
>
> ```shell
> $ echo [option] string
> # -e 解析转义字符
> # -n 不回车换行。默认情况echo回显的内容后面跟一个回车换行。
> $ echo "hello\n\n"
> hello\n\n
> $ echo -e "hello\n\n"
> hello
> 
> 
> $ echo "hello"
> hello
> $ echo -n "hello"
> hello$ echo file{1..10..2}	# 2表示步长
> file1 file3 file5 file7 file
> $ printf "%d\t%s\n" 123 hello
> 123	hello
> ```

#### 1.4.5.2 管道

可以通过`|`把一个命令的输出传递给另一个命令做输入。

实际上是将前面进程的标准输出重定向到后边进程的标准输入。

```shell
cat myfile | more
ls -l | grep  "myfile"
df -k | awk `{print $1}` | grep -v "文件系统" #df -k 查看磁盘空间，找到第一列，去除“文件系统”，并输出
```

#### 1.4.5.3 tee

在管道中间接一个通道，形成T字管道，在输出到标准输出的同时，可以截取一份保存在文件中。

```shell
df -k | awk `{print $1} | grep -v "文件系统" | tee a.txt
```

tee -a a.txt 表示追加操作

```shell
df -k | awk `{print $1} | grep -v "文件系统" | tee a.txt
```

#### 1.4.5.4 文件重定向

```shell
cmd > file		# 把标准输出重定向到新文件中
cmd >> file 	# 追加
cmd > file 2>&1 	# 标准出错也重定向到1所指向的file里
cmd >> file 2>&1
cmd < file1 > file2 	# 输入输出都重定向到文件里
cmd < &fd 	# 把文件描述符fd作为标准输入
cmd > &fd 	# 把文件描述符fd作为标准输出
cmd < &- 	# 关闭标准输入
```

### 1.4.6 函数

和C语言类似，Shell中也有函数的概念，但是函数定义中没有返回值也没有参数列表。例如：

```shell
#! /bin/sh

function foo(){ echo "Function foo is called";}
echo "-=start=-"
echo $(foo)	#输出函数返回状态
echo "-=end=-"
```

​	注意函数体的**左花括号 { 和后面的命令之间必须有空格或换行**，如果将**最后一条命令和右花括号写在同一行，命令末尾必须有分号；。**但，不建议将函数定义写至一行上，不利于脚本阅读。另外，function关键字可以省略，小括号也可以省略，但是两个必须要保留一个，不然解析器不知道要定义一个函数。

​	在定义foo()函数时并不执行函数体中的命令，就像定义变量一样，只是给foo这个名一个定义，到后面调用foo函数的时候（注意Shell中的函数调用不写括号）才执行函数中的命令。Shell脚本中的函数必须先定义后调用，一般把函数定义语句写在脚本的前面，把函数调用和其它命令写在脚本的最后（类似于C语言中的main函数，这才是整个脚本实际开始开始执行命令的地方）。

​	Shell函数没有参数列表并不表示不能传参数，事实上，函数就像是迷你脚本，调用函数时可以传任意个参数，在函数内同样是用$0、$1、$2当变量来提取参数，**函数中的位置参数相当于函数的局部变量，**改变这些变量并不会影响函数外面的$0、$1、$2等变量。函数中可以用return命令返回，**return后面只能跟一个整数，不能返回其它类型，表示函数的Exit Status**。如果没有return语句，就以函数里边最后执行的一条命令的返回状态作为整个函数的退出状态。

​	下面这个脚本可以一次创建多个目录，各目录名通过命令行参数传入，脚本逐个测试各目录是否存在，如果目录不存在，首先打印信息然后试着创建该目录。

```shell
#! /bin/sh

is_directory()
{
	DIR_NAME=$1
	if [ ! -d $DIR_NAME ]; then
		return 1
	else
		return 0
	fi
}

for DIR in "$@"; do
	if is_directory "$DIR"
	then:
	else
		echo "$DIR doesn't exist. Creating it now..."
		mkdir $DIR > /dev/null 2>&1
		if [ $? -ne 0 ]; then
			echo "Cannot create directory $DIR"
			exit 1
		fi
	fi
done
```

注意：is_directory()返回0表示真返回1表示假

> 函数支持递归：
>
> 作业：遍历目录，包括子目录，如果是文件就输出xxx是文件，如果是目录就输出xxx是目录。
>
> ```shell
> #! /bin/sh
> 
> function visit
> {
> 	local dir="$1"
> 	fo f in $(ls $1)
> 	do
> 		if [ -f "$dir/$f" ]
> 		then
> 			echo "$dir/$f is a file"
> 		elif [ -d "$dir/$f" ]
> 		then
> 			echo "$dir/$f is a dir"
> 			visit "$dir/$f"
> 		else
> 			echo "$dir/$f is not recognized"
> 		fi
> }
> 
> visit .
> ```

## 1.5 Shell 脚本调试方法

> Shell提供了一些用于调试脚本的选项，如：
>
> -n	读一遍脚本中的命令但不执行，用于检查脚本中的语法错误
>
> -v	一边执行脚本，一边将执行过的脚本命令打印到标准错误输出。
>
> -x	提供跟踪执行信息，**将执行的每一条命令和结果依次打印出来。**
>
> 这些选项有三种常见的使用方法：
>
> 1) 在命令行提供参数。如：
>
>    ```shell
>    $ sh -x .script.sh
>    ```
>
> 2) 在脚本开头提供参数。如
>
>    ```shell
>    #! /bin/sh -x
>    ```
>
> 3) 在脚本中用set命令启用或禁用参数。如：
>
>    ```shell
>    #! /bin/sh
>                
>    if [ -z "$1" ]; then
>    	set -x
>    	echo "ERROR: Insufficient Args."
>    	exit 1
>    	set +x
>    fi
>    ```
>
>    **set -x和set +x分别表示启用和禁用-x参数，这样可以只对脚本中的某一段进行跟踪调试。**

## 1.6 正则表达式

> ​	以前我们用grep在一个文件中找出包含某些字符串的行，比如在头文件中找出一个宏定义。其实grep还可以找出符合某个模式(Pattern)的一类字符串。例如找出所有符合xxxxx@xxxx.xxx模式的字符串（也就是email地址），要求x字符可以是字母、数字、下划线、小数点或减号，email地址的每一部分可以有一个或多个x字符，例如abc.d@ef.com、1_2@987-6.54，当然符合这个模式的不全是合法的email地址，但至少可以做一次初步筛选，筛掉a.b、c@b等肯定不是email地址的字符串。再比如，找出所有符合yyy.yyy.yyy.yyy模式的字符串（也就是ip地址），要求y是0-9的数字，IP地址的每一部分可以1-3个y字符。
>
> ​	如果要用grep查找一个模式，如何表示这个模式，这一类字符串，而不是一个特定的字符串呢？从这两个简单的例子可以看出，要表示一个模式至少应该包含以下信息：
>
> ​	**字符类(Character Class)**：如上例的x和y，它们在模式中表示一个字符，但是取值范围时一类字符中的任意一个。
>
> ​	**数量限定符（Quantifier）**：邮件地址的每一部分可以有一个或多个x字符，IP地址的每一部分可以有1-3个y字符。
>
> ​	各种字符类以及普通字符之间的位置关系：例如邮件地址分三部分，用普通字符@和.隔开，IP地址分四部分，用.隔开，每一部分都可以用字符类和数量限定符描述。为了表示位置关系，还有**位置限定符（Anchor）**的概念，将在下面介绍：
>
> 规定一些特殊语法表示字符类、数量限定符合位置关系，然后用这些特殊语法和普通字符一起表示一个模式，
>
> 这就是正则表达式（Regular Expression）。例如email地址的正则表达式可以写成[a-zA-Z0-9.-]+@[a-zA-Z0-9.-]+.[a-zA-Z0-9_.-]+，IP地址的正则表达式可以写成[0-9]{1,3}.[0-9]{1,3}.[0-9]{1,3}.[0-9]{1,3}。下面介绍正则表达式的语法，我们先看看正则表达式在grep中怎么用。例如有这样一个文本文件testfile：
>
> ```shell
> $ cat testfile
> 192.168.1.1
> 1234.234.04.5678
> 123.4234.045.678
> abcde
> $ egrep '[0-9]{1,3}\.[0-9]{1,3}\.[0-9]{1,3}\.[0-9]{1,3}' testfile
> ```
>
> ​	**egrep相当于grep -E**，表示采用**Extende正则表达式语法。**grep的正则表达式有Basic和Extended两种规范，它们之间的区别下一节解释。另外还有**fgrep命令，相当于grep -F**，表示只搜索固定字符串而不搜索正则表达式模式，**不会按正则表达式的语法解释后面的参数**。
>
> ​	**注意：正则表达式参数用单引号括起来，因为正则表达式中用到很多特殊字符，在Shell中也有特殊含义，只有用单引号括起来才能保证这些字符原封不动地传给grep命令，而不会被Shell解释掉。**
>
> ​	192.168.1.1符合上述模式，由三个.隔开的四段组成，每段都是1到3个数字，所以这一行被找出来了，可为什么1234.234.04.5478也被找出来了呢？**这是因为grep找的是包含某一模式的行，**这一行包含一个符合模式的字符串234.234.04.567。相反，123.4234.045.678这一行不包含符合模式的字符串，所以不会被找出来。
>
> ​	grep是一种查找过滤工具，正则表达式在grep中用来查找符合模式的字符串。其实正则表达式还有一个重要的应用是验证用户输入是否合法，例如用户通过网页表单提交自己的email地址，就需要程序验证一下是不是合法的email地址，这个工作可以在网页的Javascript中做，也可以在网站后台的程序中做，例如PHP、Perl、Python、Ruby、Java或C，所有这些语言都支持正则表达式，可以说，目前不支持正则表达式的编程语言实在很少见了。除了编程语言之外，很多UNIX命令和根据也都支持正则表达式，例如grep、vi、awk、emacs等等。“正则表达式”就像“变量”一样，它是一个广泛的概念，而不是某一种工具或编程语言的特性。

### 1.6.1 基本语法

> ​	我们知道C的变量和Shell脚本变量的定义和使用方法很不相同，表达能力也不相同，C的变量有各种类型，而Shell脚本变量都是字符串。同样道理，各种工具和编程语言所使用的正则表达式规范的语法并不相同，表达能力也各不相同，有的正则表达式规范引入很多扩展，能表达更复杂的模式，但各种正则表达式规范的基本概念都是相通的。本节介绍egrep(1)所使用的正则表达式，它大致上符合POSIX正则表达式规范，详见regex(7)。

#### 1.6.1.1 字符类

| 字符      | 含义                                                     | 举例                                             |
| --------- | -------------------------------------------------------- | ------------------------------------------------ |
| .         | 匹配任意一个字符                                         | abc.可以匹配abcd、abc9                           |
| []        | 匹配括号中的任意一个字符                                 | [abc]d可以匹配ad、bd和cd                         |
| -         | 在[]括号内表示字符范围                                   | [0-9a-fA-F]可以匹配一位十六进制数字              |
| ^         | 位于[]括号内的开头，匹配除括号中的字符之外的任意一个字符 | [^xy]匹配xy之外的任一字符                        |
| [[:xxx:]] | grep工具预定义的一些命名字符类                           | [[:alpha:]]匹配一个字母，[[:digit:]]匹配一个数字 |

#### 1.6.1.2 数量限定符

| 字符  | 含义                                     | 举例                                                         |
| ----- | ---------------------------------------- | ------------------------------------------------------------ |
| ?     | 紧跟在它前面的单元应匹配零次或一次       | [0-9]?\.[0-9]匹配0.0、2.3、.5等，由于.在正则表达式中是一个特殊字符，所以需要用`\`转义一下，取字面值 |
| +     | 紧跟在它前面的单元应匹配一次或多次       | `[a-zA-Z0-9_.-]+@[a-zA-Z0-9_.-]+\.[a-zA-Z0-9_.-]+`匹配email地址 |
| *     | 紧跟在它前面的单元应匹配零次或多次       | `[0-9][0-9]*`匹配至少一位数字，等价于[0-9]+，`[a-zA-Z_]+[a-zA-Z_0-9]*`匹配C语言的标识符 |
| {N}   | 紧跟在它前面的单元应精确匹配N次          | `[1-9][0-9]{2}`匹配从100到999的整数                          |
| {,N}  | 紧跟在它前面的单元应匹配至少N次          | `[1-9][0-9]{2,}`匹配三位以上的整数                           |
| {,M}  | 紧跟在它前面的单元应匹配最多M次          | [0-9]{,1}相当于[0-9]?                                        |
| {N,M} | 紧跟在它前面的单元应匹配最少N次，最多M次 | [0-9]{1,3}\.[0-9]{1,3}\.[0-9]{1,3}\.[0-9]{1,3}匹配IP地址     |

**注意grep找的是包含某一模式的行，而不是完全匹配某一模式的行。**

#### 1.6.1.3 位置限定符

| 字符 | 含义                       | 举例                                          |
| ---- | -------------------------- | --------------------------------------------- |
| ^    | 匹配行首的位置             | ^Content匹配位于一行开头的Content             |
| $    | 匹配行末的位置             | ;$匹配位于一行结尾的;号，^$匹配空行           |
| `\<` | 匹配单词开头的位置         | `\<th`匹配… this，但不匹配ethernet、tenth     |
| `\>` | 匹配单词结尾的位置         | `p\>`匹配leap ….，但不匹配parent、sleepy      |
| `\b` | 匹配单词开头或结尾的位置   | `\bat\b`匹配… at …,但不匹配cat、atexit、batch |
| `\B` | 匹配非单词开头和结尾的位置 | `\Bat\B`匹配battery，但不匹配… attend、hat …  |

位置限定符可以帮助grep更准确地查找。

例如上一节我们使用^[0-9]{1,3}\.[0-9]{1,3}\.[0-9]{1,3}\.[0-9]{1,3}$查找，就可以把1234.234.04.5678这一行过滤掉了。

#### 1.6.1.4 其它特殊字符

| 字符 | 含义                                                         | 举例                                                         |
| ---- | ------------------------------------------------------------ | ------------------------------------------------------------ |
| `\`  | 转义字符，普通字符转义为特殊字符，特殊字符转义为普通字符     | 普通字符`<`写成`\<`表示单词开头的位置，特殊字符`.`写成`\.`就当做普通字符来匹配 |
| ()   | 将正则表达式的一部分括起来组成一个单元，可以对整个单元使用数量限定符 | `([0-9]{1,3}\.){3}[0-9]{1,3}`匹配IP地址                      |
| `|`  | 连接两个子表达式，表示或的关系                               | `n(o|either)`匹配no或neither                                 |

### 1.6.2 Basic正则和Extended正则区别

> 以上介绍的grep正则表达式的Extended规范，**Basic规范也有这些语法**，只是`?+{}|()`应解释为普通字符，**要表示上述特殊含义则需要加`\`转义**。如果用grep而不是egrep，并且不加-E参数，则应遵照Basic规范来写正则表达式。

## 1.7 grep 

### 1.7.1 作用

> Linux系统中grep命令是一种强大的文本搜索工具，它能使用正则表达式搜索文本，并把匹配的行打印出来。grep全称是Global Regualr Expression Print，表示全局正则表达式版本，它的使用权限是所有用户。
>
> ​	grep家族包括grep、egrep和fgrep。egrep和fgrep的命令只跟grep有很小的不同。egrep是grep的扩展，支持更多的re元字符，fgrep就是fixed grep或fast grep，它把所有的字符都看做单词，也就是说，正则表达式中的元字符表示回其自身的字面意义，不再特殊。linux使用GNU版本的grep。它功能更强，可以通过-G、-E、-F命令行选项来使用egrep和fgrep功能。

### 1.7.2 格式及主要参数

> ```tex
> grep [options]
> 
> 主要参数：grep –help 可查看
> 	-c：只输出匹配行的计数
> 	-i：不区分大小写
> 	-h：查询多文件是不显示文件名
> 	-l：查询多文件是只输出包含匹配字符的文件名
> 	-n：显示匹配行及行号
> 	-s：不显示不存在或无匹配文本的错误信息
> 	-v：显示不包含匹配文本的所有行
> 	--color=auto：可以将找到的关键词部分加上颜色的显示
> ```

### 1.7.3 grep使用案例

```shell
itcast$ grep 'test' d*
显示所以以d开头的文件中包含test的行

itcast$ grep '[a-z]\{5\}' aa  # == grep -E '[a-z]{5}' aa
显示所有包含连续5个小写字符的字符串的行

itcast$ grep 'w(es)t.*\1' aa
# 如果west诶匹配，则es就被存储到内存中，并标记为1，然后搜索任意个字符(.*)，这些字符后面紧跟着另外一个es(\1)，找到就显示该行。如果用egrep或grep -E，就不用"\"号进行转义，直接写成'w(es)t.*\1'就可以了。
```

> 明确要求搜索子目录：
>
> ```shell
> grep -r
> ```
>
> 或忽略子目录：
>
> ```shell
> grep -d skip
> ```
>
> 还有些用于搜索的特殊符号：\< 和 \> 分别标注单词的开始与结尾。
>
> ```tex
> 例如：
> grep man * 会匹配 ‘Batman’、’manic’、’man’等，
> grep ‘\<man’ * 匹配’manic’和’man’，但不是’Batman’，
> grep ‘\<man\>’ 只匹配’man’，而不是’Batman’或’manic’等其他的字符串。
> ‘^’： 指匹配的字符串在行首，
> ‘$’： 指匹配的字符串在行尾，
> ```

## 1.8 find

> ​	由于find具有强大的功能，所以它的选项也很多。即使系统中含有网络文件系统NFS，find命令在该文件系统中同样有效，只要你具有相应的权限。
>
> ​	在运行一个非常消耗资源的find命令时，很多时候都倾向于把它放在后台执行，因为遍历一个大的文件系统可能会花费很长的时间（这里是指30G字节以上的文件系统）。
>
> 一、find命令格式
>
> 1、find命令的一般形式为：
>
> ```shell
> find pathname -options [-print -exec -ok ...]
> ```
>
> 2、find命令选项
>
> ```shell
> pathname：find命令所查找的目录路径。例如用.来表示当前目录，用/表示系统根目录，递归查找。
> -print：find命令将匹配的文件输出到标准输出
> -exec：find命令将匹配的文件执行该参数所给出的shell命令。相应的的形式为'command' {} \;注意{}内部无空格，和\;之间含有一个空格分隔符。
> -ok：和-exec作用相同，只不过以一种更为安全的模式来执行参数所给出的shell命令，在执行每一个命令之前，都会给出提示，让用户来确定是否执行。只有用户明确输入y才会执行后边的语句。
> ```
>
> 3、find 命令选项
>
> ```shell
> -name 按照文件名查找文件。
> -perm 按照文件权限来查找文件
> -prune 按照这一选项可以使find命令不在当前指定的目录中查找，如果同时使用-depth选项，那么-prune将被find命令忽略。
> -user 按照文件属主来查找文件
> -group 按照文件所属组来查找文件
> -mtime -n +n 按照文件的更改时间来查找文件，-n表示文件更改时间距现在n天内，+n表示文件更改时间距现在n天以前。find命令还有-atime和ctime选项。
> -nogroup 查找无有效所属组的文件，即该文件的所属的组在/etc/groups中不存在
> -nouser 查找无有效所属主的文件，即该文件的属主在/etc/passwd中不存在。
> -newer file1 ! file2 查找更改时间比文件file新但比文件file2旧的文件。
> -type 查找某一类型的文件，诸如：
> b - 块设备
> d - 目录
> c - 字符设备文件
> p - 管道文件
> l - 符号链接文件
> f - 普通文件
> -size n：[c] 查找文件长度为n块的文件，带有c时表示文件长度以字节计。
> -depth 在查找文件时，首先查找当前目录中的文件，然后再在其子目录中查找
> -fstype 查找位于某一类型文件系统中的文件，这些文件系统类型通常可以在配置文件/etc/fstab中找到，该配置文件中包含了本系统中有关文件系统的信息。
> -mount 在查找文件时不跨越文件系统mount点
> -follow 如果find命令遇到符号链接文件，就跟踪至所指向的文件。
> ```
>
> 另外，下面三个的区别：
>
> ```shell
> -amin n：查找系统中最后N分钟访问的文件
> -atime n：查找系统中最后n*24小时访问的文件
> -cmin n：查找系统中最后N分钟被改变文件状态的文件
> -ctime n：查找系统中最后n*24小时被改变文件状态的文件
> -mmin n：查找系统中最后n分钟被改变文件数据的文件
> -mtime n：查找系统中最后n*24小时被改变文件数据的文件
> ```

### 1.8.1 使用exec或ok来执行shell命令

> ​	使用find时，只要把想要的操作写在一个文件里，就可以用exec来配合find查找，很方便的。
>
> ​	在有些操作系统中只允许-exec选项执行诸如ls或ls -l这样的命令。大多数用户使用这一选项是为了查找旧文件并删除它们。建议在真正执行rm命令删除文件之前，最好先用ls命令查看一下，确认它们是所要删除的文件。
>
> ​	exec选项后面跟随着所要执行的命令或脚本，然后是一对儿{}，一个空格和一个`\`，最后是一个分号。为了使用exec选项，必须要同时使用print选项。如果验证一下find命令，会发现该命令只输出当前路径起的相对路径及文件名。
>
> ​	例如：为了用ls -l命令列出所匹配的文件，可以把ls -l命令放在find命令的-exec选项中
>
> ```shell
> $ find . -type f -exec ls -l {} \;
> # 在/logs目录中查找更改时间在5日以前的文件并删除它们
> find /logs -type f -mtime +5 -exec rm {} \;
> ```
>
> ​	在下面的例子中，find命令在当前目录中查找所有文件名以.conf结尾、更改时间在5日以前的文件，并删除它们，只不过在删除之前先给出提示。
>
> ```shell
> $ find . -name "*.conf" -mtimes +5 -ok rm {} \;
> < rm ... ./conf/httpd.conf > ? n	# 按y键删除文件，按n键不删除
> ```
>
> ​	任何形式的命令都可以在-exec选项中使用。
>
> ​	在下面的例子中我们使用grep命令。find命令首先匹配所有文件名为“ passwd*”的文件，例如passwd、passwd.old、passwd.bak，然后执行grep命令看看在这些文件中是否存在一个itcast用户。
>
> ```shell
> $ find /etc -name "passwd*" -exec grep "itcast" {} \;
> itcast:x:1000:1000::/home/itcast:/bin/bash
> ```

### 1.8.2 find命令案例

1) 查找当前用户主目录下的所有文件：

   下面两种方法都可以使用

   ```shell
   $ find $HOME -print
   $ find ~ -print
   ```

2) 让当前目录中文件属主具有读、写权限，并且文件所属组的用户和其它用户具有权限的文件：

   ```shell
   $ find . -type f -perm 644 -exec ls -l {} \;
   ```

3) 为了查找系统中所有文件长度为0的普通文件，并列出它们的完整路径；

   ```shell
   $ find / -type f -size 0 -exec ls -l {} \;
   ```

4) 查找/var/logs目录中更改时间在7日以前的普通文件，并在删除之前询问；

   ```shell
   $ find /var/logs -type f -mtime +7 -ok rm {} \;
   ```

5) 为了查找系统中所有属于root组的文件

   ```shell
   $ find . -group root -exec ls -l {} \;
   ```

6) find命令将删除当前目录中访问时间在7日以内、含有数字后缀的admin.log文件。该命令只检查三位数字，所以相应的文件的后缀不要超过999.

   ```shell
   $ find . -name "admin.log[0-9][0-9][0-9]" -atime -7 -ok rm {} \;
   ```

7) 为了查找当前文件系统中的所有目录并排序

   ```shell
   $ find . -type d | sort
   ```


### 1.8.3 xargs

> ​	在使用find命令的-exec选项处理匹配到的文件时，find命令将所有匹配到的文件一起传递给exec执行。但有写系统对能够传递给exec的命令长度有限制，这样在find命令运行几分钟之后，就会出现溢出错误。错误信息通常是“参数列太长”或“参数列溢出”。这就是xargs命令的用处所在，特别是与find命令一起使用。
>
> ​	find命令把匹配到的文件传递给xargs命令，而xargs命令每次只获取一部分而不是全部，不像-exec选项那样。这样它可以先处理最先获取的一部分文件，然后是下一批，并如此继续下去。
>
> ​		在有些系统中，使用-exec选项会为处理每一个匹配到的文件而发起一个相应的进程，并非将匹配到的文件全部作为参数一次执行；这样在有些情况下就会出现进程过多，系统性能下降的问题，因而效率不高；
>
> ​	而使用xargs命令则只有一个进程。另外，在使用xargs命令时，究竟是一次获取所有的参数，还是分批取得参数，以及每一次获取的参数数目都会根据该命令的选项及内核中相应的可调参数来确定。

> **参数与标准输入**
>
> 这两个词在Linux命令中是很常见的。但是参数和标准输入其实是有区别的。我们日常使用的很多命令，例如：ls -lah .中。l、 a、 h 、.都是命令ls的参数。至于标准输入，可以说它是某种流数据。而通常来讲标准输入的流数据来源就是我们的终端输入。在Linux命令中，有些命令可以接收标准输入，有些是不能的。像上面的ls，和echo、kill和rm等就是只能接收参数，不能接收标准输入。像cat命令或grep命令，这些是可以的。
>
> 怎么分辨一个命令可不可以接收标准输入？很简单，当你敲完命令回车后，终端会等待接收你的输入，例如当你在终端输入cat后，终端会等待你输入字符，当你输入一些字符后，然后按Ctrl-C发送终止符号。这时cat命令接收标准输入完毕，执行命令，也就是将刚才键入的内容输出的标准输出上(屏幕)。
>
> **管道**
>
> 管道的作用是将前面命令的标准输出作为后面命令的标准输入。这里要注意，后面的命令接收的是标准输入，所以如果命令不支持接收标准输入，那么就不能直接使用管道，例如常用的ls命令，只能使用参数，而不能使用标准输入，所以[command] | ls是不能使用的。而命令如echo或cat就可以。那么肯定有方法来实现这些不能使用标准输入的命令与管道结合，这时候xargs便出场了。
>
> **xargs命令**
>
> xargs命令通俗来讲就是将标准输入转成各种格式化的参数，所以命令[command 1] | xargs [command 2]就是将command 1的标准输出结果，通过管道|变成xargs的标准输入，然后xargs再将此标准输入变成参数，传给[command 2]。这样一来，通过xargs命令，我们便可以在管道后面使用那些不接收标准输入的命令了。例如[command 1]|xargs ls，是不是很熟悉？

> 1) xargs用于产生某个命令的参数，xargs可以读入stdin的数据，并且以空格符或回车符将stdin的数据分隔成参数
> 2) 许多命令不能接受过多参数，命令执行可能会失败，xargs可以解决。
>
> ![](https://img2018.cnblogs.com/common/1580705/201912/1580705-20191203155513147-2117121480.png)
>
> ```shell
> # 查找当前目录下的每一个普通文件，然后使用xargs命令来测试它们分别属于哪类文件
> $ find . -type -print | xargs file
> 
> # 在当前目录下查找所有用户具有读、写和执行权限的文件，并回收相应的权限：
> $ find . -perm -7 -print | xargs chmod o-w
> 
> # 用grep命令在所有的普通文件中搜索hello这个词：
> $ find . -type f -print | xargs grep "hello"
> 
> # 用grep命令在当前目录下的所有普通文件中搜索hello这个词
> $ find -name \* -type f -print | xargs grep "hello"	# \用来取消*号在shell中的特殊含义
> ```

#### 1.8.3.1 xargs选项

1) -0(数字0)

   > xargs默认以空格、Tab制表符、回车符为分隔符和结束符，当有的文件本身包含空格时，就会出现问题。例如：
   >
   > ```shell
   > $ touch 'hello world.txt'
   > $ touch 'ni hao.txt'
   > $ find ./ -name '*.txt'
   > ./hello world.txt
   > ./ni hao.txt
   > # xargs会把'hello world.txt'识别为两个文件
   > $ find ./ -name '*.txt' | xargs rm 
   > rm: 无法删除"./hello": 没有那个文件或目录
   > rm: 无法删除"world.txt": 没有那个文件或目录
   > rm: 无法删除"./ni": 没有那个文件或目录
   > rm: 无法删除"hao.txt": 没有那个文件或目录
   > ```
   >
   > 针对该问题，find提供了一个`print0`选项，设置find在输出文件名后自动添加一个NULL来替代换行符符，而xargs也可以通过-0选项，指定使用NULL而不是空格、Tab制表符、回车符为分隔符和结束符。
   >
   > ```shell
   > find ./ -name '*.txt' -print0 | xargs -0 rm
   > ```

2) -n

   > 指定一次读取几个参数
   >
   > ```shell
   > $ seq 5
   > 1
   > 2
   > 3
   > 4
   > 5
   > $ seq 5 | xargs 
   > 1 2 3 4 5
   > $ seq 5 | xargs -n 2 # 设置一次调用2个参数
   > 1 2
   > 3 4
   > 5
   > ```

3) -I

   > 指定一个替换字符串
   >
   > ```shell
   > $ touch {a,b,c}.txt
   > $ ls *.txt | xargs -I[] cp[] /tmp/ []的值依次为a.txt、b.txt、c.txt
   > ```

4) -i

   > 与-I类似，如果不指定替换符，默认为{}
   >
   > ```shell
   > $ ls *.txt | xargs -i cp {} /tmp/
   > $ ls *.tar | xargs -i docker load -i {} # 逐个导入镜像
   > ```

5) -r

   > no-run-if-empty 当xargs的输入为空时则停止xargs，不再取执行了。

### 1.8.4 find命令的参数使用

1) 使用name选项

   > 文件名选项是find命令最常用的选项，要么单独使用该选项，要么和其它选项一起使用。
   >
   > 可以使用某种文件名模式来匹配文件，记住要用引号将文件名模式引起来。
   >
   > 不管当前路径是什么，如果想要在自己的根目录HOME中查找文件名符合*.txt的文件，使用‘~’作为‘pathname’的参数，波浪号代表了你的HOME目录。
   >
   > ```shell
   > $ find ~ -name "*.txt" -print
   > # 在当前目录查找文件名以两个小写字母开头，跟着两个数字，最后是.txt的文件，下面的命令就能够返回例如名为ax37.txt的文件
   > $ find . -name "[a-z][a-z][0-9][0-9].txt" -pritn
   > ```

2) 使用perm选项

   > 按照文件权限模式用-perm选项，按文件权限模式来查找文件的话。最好使用八进制的权限表示法。
   >
   > ```shell
   > # 在当前目录下查找文件权限为755的文件，即文件属主可以读、写、执行，其它用户可以读、执行的文件
   > $ find . -perm 755 -print
   > ```
   >
   > 还有一种表达方法：在八进制数字前面要加一个横杠-，表示都匹配，如-007就相当于777，-006相当于666
   >
   > \# ls -l
   >
   > \# find . -perm 006
   >
   > \# find . -perm -006
   >
   > -perm mode:文件许可正好符合mode
   >
   > -perm +mode:文件许可部分符合mode
   >
   > -perm -mode: 文件许可完全符合mode

3) 忽略某个目录

   > ​	如果在查找文件时希望忽略某个目录，因为你知道那个目录没有你所要查找的文件，那么可以使用-prune选项来指出需要忽略的目录。在使用-prune选项时要当心，因为如果你同时使用了-depth选项，那么-prune选项就会被find命令忽略。
   >
   > ```shell
   > # 如果希望在/apps目录下查找文件，但不希望在/apps/bin目录下查找，可以用：
   > $ find /apps -path "/apps/bin" -prune -o -print
   > 
   > # 忽略多个目录
   > $ find /home \(-path /home/itcast/f1 -o -path /home/itcast/f2 \) -prune -o -print
   > ```

4) 使用user和nouser选项

   > ```shell
   > # 按文件属主查找文件，如再$HOME目录中查找文件属主为itcast的文件，可以用
   > $ find ~ -user itcast -print
   > ```
   >
   > ​	为了查找属主账户已经本删除账户的文件，可以使用-nouser。这样就能找到那些属主在/etc/passwd文件中没有账户的文件。在使用-nouser选项时，不必给出用户名；
   >
   > ```shell
   > # 在/home目录下查找所有这类文件
   > $ find /home -nouser -print
   > ```

5) 按照更改时间或访问时间等查找文件

   > ​	如果希望按照更改时间来查找文件，可以使用mtime，atime或ctime选项。如果系统突然没有可用空间了，很有可能某一个文件的长度在此期间增长迅速，这时就可以用mtime选项来查找这样的文件。
   >
   > ​	用减号-来限定更改时间在距今n日以内的文件，而用加号+来限定更改时间在距今n日以前的文件。
   >
   > ```shell
   > # 在系统根目录下查找更改时间在5日以内的文件，可以用
   > $ find / -mtime -5 -print
   > 
   > # 在/var/adm目录下查找更改时间在3日以前的文件，可以用：
   > $ find /var/adm -mtime +3 -print
   > ```

6) 查找比某个文件新或旧的文件

   > ​	如果希望查找更改时间比某个文件新但比另一个文件旧的所有文件，可以使用-newer选项。它的一般形式为：
   >
   > ```
   > - newer newest_file_name ! oldest_file_name
   > ```

7) 使用size选项

   > 可以按照文件长度来查找文件，这里所指的文件长度既可以用块（block）来计量，也可以用字节来计量。以字节计量文件长度的表达形式为N c；以块计量文件长度只用数字表示即可。
   >
   > * c 字节
   > * w 双字
   > * k KB
   > * G GB
   > * M MB
   >
   > 在按照文件长度查找文件时，一般使用这种以字节表示的文件长度，在查看文件系统的大小，因为这时使用块来计量更容易转换。 
   >
   > ```shell
   > # 在当前目录下查找文件长度大于1 M字节的文件：
   > $ find . -size +1000000c -print
   > 
   > # 在/home/apache目录下查找文件长度恰好为100字节的文件：
   > $ find /home/apache -size 100c -print
   > 
   > # 在当前目录下查找长度超过10块的文件（一块等于512字节）：
   > $ find . -size +10 -print
   > ```

8) 使用depth选项

   > ​	在使用find命令时，可能希望先匹配所有的文件，再在子目录中查找。使用depth选项就可以使find命令这样做。
   >
   > ```shell
   > # 先查找所有文件，再进入子目录中查找
   > $ find / -name "CON.FILE" -depth -print
   > ```

9) 使用mount选项

   > 在当前的文件系统中查找文件（不进入其他文件系统），可以使用find命令的mount选项。
   >
   > 从当前目录开始查找位于本文件系统中文件名以XC结尾的文件：
   >
   > ```shell
   > $ find . -name "*.XC" -mount -print
   > ```

10) 逻辑运算符

    > 选项：
    >
    > * -a：and逻辑与
    > * -o：or逻辑或
    > * -not：not逻辑非
    >
    > ```shell
    > # 查找当前目录下大于2KB，并且文件类型是普通文件的文件
    > $ find -size +2k -a type f
    > 
    > # 在当前目录下搜索文件名要么是cangls的文件，要么是bols的文件
    > find . -name cangls -o -name bols
    > 
    > # 查找当前目录下文件名不是cangls的文件
    > $ find . -not -name cangls
    > ```

# 2 C程序中使用正则表达式

> ​	POSIX规定了正则表达式的C语言库函数，详见regex(3)。
>
> ​	C语言处理正则表达式常用的函数有regcomp()、regexec()、regfree()和regerror()，一般分为三个步骤，如下所示：
>
> 1) 编译正则表达式 regcomp()
>
> 2) 匹配正则表达式regexec()
>
> 3) 释放正则表达式regfree()
>
>    regcomp()函数把指定的正则表达式pattern编译成一种特定的数据格式，这样可以使匹配更有效。函数regexec()会使用这个数据在目标文本串中进行模式匹配。执行成功返回0.
>
>    ```c
>    int regcomp(regex_t* compiled, const char *pattern, int cflags);
>       
>    /*
>    regex_t	是一个结构体数据类型，用来存放编译后的正则表达式，它的成员re_nsub用来存储正则表达式中的子正则表达式的个数，正则表达式就是用圆括号包起来的部分表达式。
>    pattern  是指向我们写好的正则表达式的指针
>    cflags	有如下4个值或者是它们或运算(1)后的值：
>    	REG_EXTENDED	以功能更加强大的扩展正则表达式的方式进行匹配。
>    	REG_ICASE	匹配字母时忽略大小写
>    	REG_NOSUB	不用存储匹配后的结果，只返回是否成功匹配。如果设置该标志位，那么regexec将忽略nmatch和pmatch两个参数。
>    	REG_NEWLINE	识别换行符，这样'$'就可以从行尾开始匹配，’^‘就可以从行的开头开始匹配。
>    */
>    ```
>
>    ​	当我们编译好正则表达式后，就可以用regexec()匹配我们的目标文本串了，如果在编译正则表达式的时候没有指定cflags的参数为REG_NEWLINE，则默认情况下是忽略换行符的，也就是把整个文本串当作一个字符串处理。
>
>    执行成功时返回0.
>
>    regmatch_t是一个结构体数据类型，在regex.h中定义：
>
>    ```c
>    typedef struct
>    {
>    	regoff_t rm_so;
>    	regoff_t rm_eo;
>    } regmatch_t;
>    /*
>    	成员rm_so存放匹配文本串在目标串中开始的位置，rm_eo存放结束位置。通常我们以数组的形式定义一组这样的结构。因为往往我们的正则表达式中还包含子正则表达式。数组0单元主正则表达式位置，后边的单元依次存放子正则表达式的位置。
>    */
>    ```
>
>    ```c
>    int regexec(const regex_t *compiled, const char* string, size_t nmatch, regmatch_t matchptr[], int eflags)
>    /*
>    compiled	是已经用regcomp函数编译好的正则表达式
>    string	是目标文本串
>    nmatch	是regmatch_t结构体数组的长度
>    matchptr	rematch_t类型的结构体数组，存放匹配文本串的位置信息。
>    eflags	有两个值：
>    	REG_NOTBOL：让特殊字符^无用
>    	REG_NOTEOL：让特殊字符$无作用
>    */
>    ```
>
>    当我们使用完编译好的正则表达式后，或者要重新编译其他正则表达式的时候，我们可以用这个函数清空compiled指向的regex_t结构体的内容，请记住，如果是重新编译的话，一定要先清空regex_t结构体。
>
>    ```c
>    void regfree(regex_t *compiled);
>    ```
>
>    当执行regcomp 或者regexec 产生错误的时候，就可以调用这个函数而返回一个包含错误信息的字符串。
>
>    ```c
>    size_t regerror (int errcode, regex_t *compiled, char *buffer, size_t length)
>       
>    /*
>    errcode 	是由regcomp 和 regexec 函数返回的错误代号。
>    compiled 	是已经用regcomp函数编译好的正则表达式，这个值可以为NULL。
>    buffer 		指向用来存放错误信息的字符串的内存空间。
>    length 		指明buffer的长度，如果这个错误信息的长度大于这个值，则regerror 函数会自动截断超出的字符串，但他仍然会返回完整的字符串的长度。所以我们可以用如下的方法先得到错误字符串的长度。
>    */
>    ```

## 2.1 测试案例

> ```c
> #include <sys/types.h>
> #include <regex.h>
> #include <stdio.h>
> 
> int main(int argc, char *argv[])
> {
> 	if(argc != 3)
>     {
>         printf("Usage: %s RegexString Text\n", argv[0]);
>         return 1;
>     }
>     const char *pregexstr = argv[1];
>     const char *ptext = argv[2];
>     regex_t oregex;
>     int nerrcode = 0;
>     char szerrmsg[1024] = {0};
>     size_t unerrmsglen = 0;
>     if((nerrcode = regcomp(&oregex, pregexstr, REG_EXTENDED|REG_NOSUB)) == 0)
>     {
>         if((nerrcode = regcomp(&oregex, ptex, 0, NULL, 0)) == 0)
>         {
>             printf("%s matchs %s\n", ptext, pregexstr);
>             regfree(&oregex);
>             return 0;
>         }
>     }
>     unerrmsglen = regerror(nerrcode, &oregex, szerrmsg, sizeof(szerrmsg));
>     unerrmsglen = unerrmsglen < sizeof(szerrmsg) ? unerrmsglen : sizeof(szerrmsg) - 1;
>     szerrmsg[unerrmsglen] = '\0';
>     printf("ErrMsg: %s\n", szerrmsg);
>     regfree(&oregex);
>     
> 	return 0;
> }
> ```
>
> 匹配网址：
>
> ```shell
> ./a.out "http:\/\/www\..*\.com" "http://www.taobao.com"
> ```
>
> 匹配邮箱：
>
> ```shell
> ./a.out "^[a-zA-Z0-9]+@[a-zA-Z0-9]+.[a-zA-Z0-9]+" "itcast123@itcast.com"
> ./a.out "\w+([-+.]\w+)*@\w+([-.]\w+)*\.\w+([-.]\w+)*" "itcast@qq.com"
> 注：\w匹配一个字符，包含下划线
> ```
>
> 除了gnu提供的函数外，还常用PCRE处理正则，全称是Perl Compatible Regular Ex-pressions。从名字我们可以看出PCRE库是与Perl中正则表达式相兼容的一个正则表达式库。PCRE是免费开源的库，它是由C语言实现的，这里是它的官方主页：http://www.pcre.org/，感兴趣的朋友可以在这里了解更多的内容。 要得到PCRE库，可以从这里下载：http://sourceforge.net/projects/pcre/files/
>
> PCRE++是一个对PCRE库的C++封装，它提供了更加方便、易用的C++接口。这里是它的官方主页：http://www.daemon.de/PCRE，感兴趣的朋友可以在这里了解更多的内容。 要得到PCRE++库，可以从这里下载：http://www.daemon.de/PcreDownload
>
> 另外c++中常用 boost regex。

# 3 sort命令

> 命令从标准输入中读取数据然后按照字符串内容进行排序，或对文本以行为单位进行排序
>
> * -f	忽略大小写
> * -n   比较数值大小
> * -t    指定分割符，默认是以空白为分割符
> * -k    指定分割后进行比较的字段，如: -k1 (按第1个字段排序)   -k 1.2, 3.4 (按从第1个字段的第2个字符起，到第3个字段第4个字符止，进行排序)
> * -u    重复的行只显示一次
> * -r    反向排序
> * -R   打乱顺序，同样的两行无法洗乱
> * -d    排序是只按字母、数字或空格，忽略其它字符
>
> ```shell
> #!/usr/bin/env bash
>  
> file="tmp/s.txt"
> r1="tmp/r1.txt"
>  
> #正常排序
> sort ${file}
>  
> #按数值大小排序
> sort -n ${file}
>  
> #按数值大小,以第二个字段排序
> sort -n -k2 ${file}
>  
> #按数值大小,以空格为分隔符,以第二个字段排序
> sort -n -t ' ' -k2 ${file}
>  
> #按数值大小,以空格为分隔符,以第二个字段, 倒序排序
> sort -n -t ' ' -k2 -r ${file}
>  
> #按数值大小,以空格为分隔符,以第二个字段, 倒序排序,输出到文件
> sort -n -t ' ' -k2 -r -m ${file} -o ${r1}
> 
> # 对/etc/passwd按用户ID进行排序
> $ cat /etc/passwd | sort -t : -k3 -n
> ```

# 4 uniq命令

> 去除重复的行，前提是重复的行连续
>
> * -c   显示每行重复的次数
> * -d   仅显示重复过的行
> * -u   仅显示不曾重复的行

