# 1 Linux系统简介

## 1 .1 linux操作系统的作用

操作系统在计算机系统中承上启下的地位：向下封装硬件，向上提供操作接口

![image-20220525233341607](C:\Users\moshang\AppData\Roaming\Typora\typora-user-images\image-20220525233341607.png)

## 1.2 linux内核介绍

内核（kernel)是系统的心脏，是运行程序和管理像磁盘和打印机等硬件设备的核心程序，它提供了一个在裸设备与应用程序间的抽象层。

linux内核版本有分为稳定版和开发版。

内核源码网址：http://www.kernel.org 所有来自全世界的对linux源码的修改最终都会汇总到这个网站，由linux领导的开源社区对其进行甄别和修改最终决定是否进入到linux主线内核源码中。

## 1.3 linux发行版本

linux发行版（也被叫做GUN/Linux发行版）通常包含了包括桌面环境、办公套件、媒体播放器、数据库等应用软件。

这些操作系统通常有Linux内核、以及来自GNU计划的大量函式库，和基于X Window的图形界面，在X Window中用户同样可以通过使用鼠标对窗口、菜单等进行操作来完成相应的工作。

X Window系统是一个非常出色的图形窗口系统，是类UNIX系统的图形用户界面的工业标准。X Window系统最重要的特征之一就是它的结构与设备无关。

X Window系统的主要特点：

- X Window系统是客户机/服务器结构的。X Window的实现是与操作系统内核分开的，其主要由X Server和X Client两部分组成。
- X Window系统不是Unix/Linux操作系统的必须的构成部分，而只是一个可选的应用程序组件。

# 2 linux系统目录结构

Unix/Linux采用了树状结构的文件系统，它由目录和目录下的文件一起构成。但Unix/Linux文件系统不使用驱动器这个概念，而是使用单一的根目录结构，所有分区都挂载到单一的“/”目录上，其结构示意图如图所示：

![image-20220526002654878](C:\Users\moshang\AppData\Roaming\Typora\typora-user-images\image-20220526002654878.png)

## 2.1 Linux常见目录

- **/：**根目录，一般根目录下只存放目录，在Linux下只有一个根目录。所有的东西都是从这里开始。在你终端输入“/home”，你其实是在告诉电脑，先从/（根目录）开始，再进入home目录。
- **/bin：/usr/bin：**可执行二进制文件的目录，如常用的命令ls、tar、mv、cat等。
- **/boot：**放置linux系统启动是用到的一些文件，如Linux的内核文件：/boot/vmlinuz，系统引导管理器：/boot/grub。
- **/dev：**存放linux系统下的设备文件，访问该目录下某个文件，相当于访问某个设备，常用的是挂载光驱：mount/dev/cdrom/mnt。
- **/etc：**系统配置文件存放的目录，不建议在此目录下存放可执行文件，重要的配置文件有/etc/inittab、/etc/fstab、/etc/init.d、/etc/X11、/etc/sysconfig、/etc/xinetd.d。
- **/home：**系统默认的用户家目录，新增用户账号是，用户的家目录都存放在此目录下，~表示当前用户的家目录，-edu表示用户edu的家目录。
- **/lib：/usr/lib：/usr/local/lib**：系统使用的函数库的目录，程序在执行过程中，需要调用一些额外的参数是需要函数库的协助。
- **/lost+fount：**系统异常产生错误是，会将一些遗失的片段放置在此目录下。
- **/mnt：/media：**光盘默认挂载点，通常光盘挂载于/mnt/cdrom下，也不一定，可以选择任意位置进行挂载。
- **/opt：**给主机额外安装软件所摆放的目录。
- **/proc：**此目录的数据都在内存中，如系统核心，外部设备，网络状态，由于数据都存放在内存中，所以不占用磁盘空间，比较重要的目录有/proc/cpuinfo、/proc/interrupts、/proc/ioports、/proc/net/*等。
- **/root：**系统管理员的家目录。
- **/sbin：/usr/sbin：/usr/local/sbin：**存放系统管理员使用的可执行命令，如fdisk、shutdown、mount等。与/bin不同的是，这几个目录是给系统管理员root使用的命令，一般用户只能查看而不能设置和使用。
- **/tmp\*\*：**\*\*一般用户或正在执行的程序临时存放文件的目录，任何人都可以访问，重要数据不可放置在此目录下。
- **/srv：**服务启动之后需要访问的数据目录，如www服务需要访问的网页数据存放在/srv/www内。
- **/usr：**应用程序存放目录，/usr/bin存放应用程序，/usr/share存放共享数据，/usr/lib存放不能直接运行的，却是许多程序运行所必须的一些函数库文件。/usr/local：存放软件升级包。/usr/share/doc：系统说明文件存放目录。
- **/var：**放置系统执行过程中经常变化的文件，如随时更改的日志文件/var/log，/var/log/message：所有登录文件存放目录，/var/spool/mail：邮件存放的目录，/var/run：程序或服务启动后，其PID存放在该目录下。

# 3 Linux命令解析器

## 3.1 概述

* Linux刚出世时没有什么图形用户界面，所有的操作全靠命令完成。
* 近几年来尽管Linux发展迅速，图形界面越来越友好，但是在真正的开发过程中，Linux命令行应用还是占有非常重要的席位，而且许多Linux功能在命令行界面要比图形化界面下要运行的快。
* Linux提供了大量的命令，利用它可以有效地完成大量的工作，如磁盘操作，文件存取，目录操作，进程管理，文件权限设定等。Linux发行版最少目录也有200多个。

## 3.2 shell命令解释器

命令解释器作用：交互式地解释、执行用户输入的命令，将用户的操作翻译成机器可以识别的语言，完成相应的功能。

Linux系统中提供了好几种不同的命令解释器，如shell(/bin/sh)、bash(/bin/bash)等，一般默认使用bash作为默认的解释器。

## 3.3 Bash解析器常用快捷键

### 3.3.1 Tab键

* 补齐命令
* 补齐路径
* 显示当前目录下的所有目录

### 3.3.2 清屏

clear作用为清除终端上的显示（类似于DOS的cls清屏功能），也可以使用快捷键：Ctrl + L。

### 3.3.3 中断线程

Ctrl + C的作用是中断终端的操作。

### 3.3.4 遍历输入的历史命令

* 从当前位置向上遍历：Ctrl + p(↑)
* 从当前位置向下遍历：Ctrl  + n(↓)

### 3.3.5 光标相关操作

* 光标左移：Ctrl + b(←)
* 光标右移：Ctrl + f(→)
* 移到头部：Ctrl + a(Home)
* 移到尾部：Ctrl + e(End)

### 3.3.6 字符删除

* 删除光标前边字符：Ctrl + h(Backspace)
* 删除光标后边字符：Ctrl + d
* 光标后边的字符集光标覆盖的字符
* 删除光标前面所有字符：Ctrl + u
* 删除光标后面所有字符：Ctrl + k

## 3.4 Linux系统相关快捷键

### 3.4.1 图形打开终端方式：（三种方式）

1）Alt + F2		打开运行输入 gnome-terminal

2）搜索您的计算机->输入终端->打开

3）点击桌面打开终端

### 3.4.2 终端相关快捷键

（终端窗口处于激活状态时）

Ctrl + Shift + N 新建一个终端

Ctrl + Shift + T 在终端新建一个标签

Ctrl + Shift + W 关闭标签页

Ctrl + shift + Q 关闭窗口



Ctrl + Shift + C 复制

Ctrl + Shift + V 粘贴

Alt + [1- 9] 标签之间切换

Ctrl + Shift + = 放大终端字体

Ctrl + - 缩小终端字体

Ctrl + 0 普通大小

F11 全屏 再次按F11退出全屏

Ctrl + D 关闭当前一个终端（当前终端标签）

Alt + F4 关闭整个终端（当前窗口的所有终端标签）

Ctrl + Shift + F 查找

终端推荐字体使用：DejaVu Sans Mono Book

Alt + Tab 应用程序间切换

Ctrl + Alt + Backspace 注销

Ctrl + Alt + F7 回到图形界面

Ctrl + Alt + F2 到Ctrl + Alt + F6 可以访问6个虚拟控制台

# 4 内建命令和外部命令

## 4.1 什么是内建命令

shell内建命令是指bash(或其他版本)工具中集成的命令。一般都会有一个与之同名的系统命令，比如bash中的echo命令与/bin/echo是两个不同的命令，尽管他们行为大体相仿。当在bash中键入一个命令时系统会先看他是否是一个内建命令，如果不是才会查看是否是系统命令或第三方工具命令。所以在bash中键入echo命令实际上执行bash工具集中的bash命令也就是内建命令，而不是/bin/echo这个系统命令。

## 4.2 外部命令

外部命令时安装外部软件所带的命令。

## 4.3 内建命令与外部命令对比

内建命令要比系统命令有较高的执行效率。外部命令执行时往往需要fork出（产生出）一个子进程，而内部命令一般不用。外部命令是在bash之外额外安装的，通常放在/bin，/usr/bin，/sbin，/usr/sbin......等等。可通过"echo $PATH"命令查看外部命令的存储路径，比如：ls、vi等。

 内部命令实际上是shell程序的一部分，其中包含的是一些比较简单的linux系统命令，这些命令由shell程序识别并在shell程序内部完成运行，通常在linux系统加载运行时shell就被加载并驻留在系统内存中。内部命令是写在bash源码里面的，其执行速度比外部命令快，因为解析内部命令shell不需要创建子进程。比如：exit，history，cd，echo等。

外部命令是linux系统中的实用程序部分，因为实用程序的功能通常都比较强大，所以其包含的程序量也会很大，在系统加载时并不随系统一起被加载到内存中，而是在需要时才将其调用内存。通常外部命令的实体并不包含在shell中，但是其命令执行过程是由shell程序控制的。shell程序管理外部命令执行的路径查找、加载存放，并控制命令的执行。外部命令是在bash之外额外安装的，通常放在/bin，/usr/bin，/sbin，/usr/sbin......等等。可通过“echo
 $PATH”命令查看外部命令的存储路径，比如：ls、vi等。

## 4.4 命令类型查看方法

使用type命令查看：

格式：

type [-a] 名称 [名称......]

显示命令类型的信息。

使用示例：

type -a cd

# 5 Linux命令格式

command [-options] [parameter1] ...

说明：

* command：命令名，相应功能的英文单词或单词的缩写
* [-options]：选项，可用来对命令进行控制，也可以省略
* parameter：传给命令的参数，可以是零个，一个或多个

**命令选项前的 -：**

* 大多数命令前有 - ：ls -a、rm -r dir等，它们是System V风格
* 有些命令参数前有没有横杆都可以：如tar xvf filename和tar -xvf filename、ps aux和ps -aux等，它们是BSD风格
* 一般格式：- + 字母 如：-r|-p|-l    -- + 单词 如：--help

#  6 帮助文档查看方法

## 6.1 man

man是Linux提供的一个手册，包含了绝大部分的命令、函数使用的说明。

该手册分成很多章节（section），使用man时可以指定不同的章节来浏览不同的内容。

### 6.1.1 man中个个section意义如下：

1）Standard commands(可执行程序或shell命令)

2）System calls（系统调用，内核提供的函数，如open,write）

3）Library functions(库函数，如prinf, fopen)

4）Special devices（设备文件的说明，/dev下各种设备）

5）File formats（文件格式，如passwd）

6）Games and toys（游戏和娱乐）

7）Miscellaneous（杂项、惯例与协定等，例如Linux档案系统、网络协议

8）Administrative Commands（管理员命令，如ifconfig)

### 6.1.2 man使用格式

man [选项] 命令名

### 6.1.3 man设置的功能键

* 空格键：显示手册页下一屏
* Enter键：一次滚动手册页的一行
* b：回滚一屏
* f：前滚一屏
* q：退出man命令
* h：列出所有功能键
* /word：搜索word字符串

### 6.1.4 用法示例

man -k passwd 搜索关键词passwd

man -a passwd 浏览passwd所有相关的页

man -f passwd 等价于 what is

man 1 printf 浏览printf第一章介绍

man 2 read 浏览read第2章介绍

man 3 printf 浏览printf第3章介绍

## 6.2 内建命令（help)

格式：

help + 内建命令

如：help pwd

## 6.3 外部命令（--help）

一般是Linux命令自带的帮助信息，并不是所有命令都自带这个选项。

如我们想查看命令ls的用法：ls --help

# 7 目录相关命令

## 7.1 pwd

格式：

pwd [-LP]

打印当前工作目录的名字

## 7.2 cd

cd命令可以帮助用户切换工作目录，对路径名大小写非常敏感。

cd后面可以跟绝对路径，也可以跟相对路径。如果省略目录，则默认回到当前用户的主目录。

**用法示例：**

* cd  :切换到当前用户的主目录（/home/用户目录），用户登录时，默认的目录
* cd ~:切换到当前用户的主目录（/home/用户目录)
* cd . ：切换到当目录
* cd ..：切换到上级目录
* cd - ：可进入上一个进入的目录

## 7.3 mkdir

用法：mkdir [选项] ...目录 ...

通过mkdir命令可以创建一个新的目录（不能新建普通文件）。参数-p可以递归创建目录。

需要注意的是新建目录的名称不能与当前目录中已有的目录或文件同名，并且**目录创建者必须对当前目录具有写权限**。

使用示例：

* mkdir test  在当前目录下创建test目录
* mkdir /tmp/test 在跟目录下tmp目录里创建test目录
* mkdir file{1..100} 在当前目录下创建file1到file100的目录，这里会创建100个目录
* mkdir "file{1..100}" 在当前目录下创建file{1..100}目录，这里只创建1个目录
* mkdir -p a/b/c 在当前目录下创建a/b/c目录树
* mkdir a\ b 创建以“a b”命名的一个目录,\ 代表转义空格
* mkdir a b 创建目录a和b

## 7.4 rmdir

用法：rmdir [选项] ...目录...

删除指定的**空目录**。

可使用rmdir命令删除一个目录。必须离开目录，并且目录必须为空目录，不然提示删除失败。

使用示例：

* rmdir test 删除当前目录下的test目录，test必须是空目录
* rmdir /tmp/test 删除tmp目录下的test目录
* rmdir file{1..100} 删除file1到file100的目录
* rmdir "file{1..100}" 删除file{1..100}目录
* rmdir "a b" 删除a b这个目录
* rmdir a\ b 删除a b这个目录
* rmdir a b c 删除a目录、b目录和c目录
* rmdir file\\{1..100\\} 使用转义方法删除file{1..100}目录
* rmdir {a..c} 删除a目录、b目录和c目录
* rmdir -p a/b/c 递归删除a/b/c目录树

# 8 文件类型

Uinx/Linux对数据文件（.mp3、.bmp），程序文件(.c、.h、*.o)，设备文件（LCD、触摸屏、鼠标），网络文件（socket）等的管理都抽象为文件，使用统一的方式方法管理。Linux下一切皆文件。

在Uinx/Linux操作系统中也必须区分文件类型，通过文件类型可以判断文件属于可执行文件、文本文件还是数据文件。在Unix/Linux系统中文件可以没有扩展名。

## 8.1 文件分类

通常，Uinx/Linux系统中常用的文件类型有7种：普通文件、目录文件、设备文件、管道文件、链接文件和套接字。

![image-20220529232801662](C:\Users\moshang\AppData\Roaming\Typora\typora-user-images\image-20220529232801662.png)

### 8.1.1 根据字体颜色分辨文件类型

1）普通文件：

* 白色 -- 普通文件
* 绿色 -- 可执行文件
* 红色 -- 压缩文件

2）蓝色 -- 目录

3）青色 -- 链接文件

4）黄色 -- 设备文件

5）灰色 -- 其他文件

**1）普通文件**

普通文件时计算机操作系统用于存放数据、程序等信息的文件，一般长期存放在外存储器（磁盘、磁带等）中。普通文件一般包括**文本文件、数据文件、可执行的二进制程序文件**等。

**在Unix/Linux系统中把目录可以通过file命令来查看文件的类型。如果file文件后面携带文件名，则查看指定文件的类型，如果携带通配符“*”，则可以查看当前目录下的所有文件的类型。**

**2）目录文件**

在Uinx/Linux系统中把目录看成是一种特殊的文件，利用它构成的文件系统的树形结构。

**非用户家目录下的目录文件只允许系统管理员对其进行修改，用户进程可以读取目录文件，但不能对它们进行修改**。

每个目录文件至少包括两个条目，“.."表示上一级目录，”."表示该目录本身。

**3）设备文件**

Unix/Linux系统把每个设备都映射成了一个文件，这就是设备文件。它是用于向I/O设备提供连接的一种文件，分为字符设备和块设备文件。

字符设备的存取以一个字符为单位，块设备的存取以字符快为单位。每一种I/O设备对应一个设备文件，存放在/dev目录中，如行式打印机对应/dev/lp，第一个软盘驱动器对应/dev/fd0。

echo "hello world" > /dev/pts/22

hello world

**4）管道文件**

管道文件也是Unix/Linux中较特殊的文件类型，这类文件多用于进程间的通信。

**5）链接文件**

类似于windows系统下的快捷方式，链接又分为软链接（符号链接）和硬链接。

# 9 文件相关命令

## 9.1 ls

ls是英文单词list的简写，**其功能为列出给定文件的信息**，是用户最常用的命令之一，它类似于DOS下的dir命令。Linux文件或者目录名称最长可以有256个字符，“."代表当前目录，”..“代表上一级目录，**以”.“开头的文件为隐藏文件，需要用-a参数才能显示**。

用法： ls [选项] .. [文件]...

### 9.1.1 ls常用参数

* -a 显示指定目录下的所有子目录与文件，包括隐藏文件
* -l 以列表方式显示文件的详细信息
* -h 配合-l以人性化的方式显示文件大小

**使用示例：**

* ls -al    显示所有文件包括隐藏文件列表
* ls -lt 显示按如期时间排序后的文件列表 等价于 ls -l -t
* ls -lh 显示计算大小为KB为单位的文件列表
* ls - dl 显示文件夹详细列表

### 9.1.2 详细信息解析

![image-20220530002139848](C:\Users\moshang\AppData\Roaming\Typora\typora-user-images\image-20220530002139848.png)

**srwxr-xr-- 2 moshang moshang 4096 3月 19 2016 Music**

1）文件类型

* \- ：普通文件
* d：目录
* l：链接符号
* b：块设备
* c：字符设备
* s：socket文件
* p：管道文件

2）所有者权限 同组用户权限 其他人权限

* r：读权限，用户可以读取文件内容，如用cat查看
* w：写权限，用户可以编辑文件
* x：该文件具有可被系统执行的权限，对目录来说，该用户具有可进入目录的权限
* -：表示用户不具有该权限

**可以用二进制和八进制表示文件权限，拥有该权限为1，比如rwx权限用二进制表示是111，八进制表示为7。**

3）文件的硬链接数量

4）该文件或目录的所有者

5）该文件或目录所属的组

6）该文件占用的存储空间，如果是目录则是其下目录表项及其他文件所占存储空间

7）文件最后创建或修改的时间

8）文件名

## 9.1.3 通配符

在Unix/Linux系统中，使用特殊的字符来同时引用多个文件名，这些特殊字符被称为通配符。

* \* ：文件代表文件名中所有字符
* ls te*：查找所有以te开头的文件
* ls *html：查看所有以html结尾的文件
* ？:代表文件名中任意一个字符
* ls ?.c：只找第一个字符任意，后缀为.c的文件
* ls a.?：只找只有3个字符，前两个为a.，最后一个字符任意的文件
* []：“[” 和 “]”将字符组括起来，表示可以匹配字符组中的任意一个。“-”用于表示字符范围，”^"表示取反，即不包含。
* [abc]：匹配从a、b 、c中任意一个
* [a-f]：匹配从a到f范围内的任意一个字符
* ls [a-f]*：找到从a到f范围内的任意一个字符开头的所有文件
* ls a-f：查找文件名为a-f的文件，当“-"处于括号之外时失去通配符的作用
* \：如果要使用通配符作为普通字符使用，可以在其前面加上转义字符。“？“和”*“处于括号内时不用使用转义字符就已经失去了通配符的作用。

## 9.3 touch

用法： touch [选项] ...文件...

1）如果文件不存在，则创建新文件（只能是普通文件，不能是文件夹）

2）如果文件存在，更新文件时间

示例：

* **touch file **创建file空文件，如果file存在则更新file的存取和修改时间
* touch -c file 如果file不存在，则不创建文件
* touch -r file1 file 更新file时间和file1相同
* touch -a file 更新file的存取时间
* touch -m file 更新file的修改时间
* **touch file{2,3,4}** 同时创建file2，file3，file4三个空文件
* touch "file{2, 3, 4}" 创建file{2,3,4}一个空文件

# 9.4 cp

cp命令的功能是将给出的文件或目录复制到另一个文件或目录中，相当于DOS下的copy命令。

常用选项说明：

* -a： 该选项通常在复制目录时使用，它保留链接、文件属性，并递归地复制目录，简单而言，保持文件的原有属性。
* -f： 删除已经存在的同名目标文件而不提示。
* -i：交互式复制，在覆盖目标文件之前将给出提示要求用户确认
* -r：若给出的源文件时目录文件，则cp将递归复制该目录下的所有子目录和文件，目标文件必须为一个目录名。
* -v：显示拷贝进度

示例：

* **cp file dirs/** 将file文件复制到dirs目录下，默认不提示覆盖同名文件

* **cp file file2 dirs/ ** 将file1，file2文件拷贝到dirs目录下
* **cp -i file1 file2 ** 将file1 拷贝到file2中，并且提示覆盖
*  **cp -t dirs file1**  将文件file1拷贝到dirs目录下
* **cp -r dir1/ dir2/**  把dir1目录复制到dir2目录下
* **cp -R dir1 dir2 ** 把dir1目录拷贝到dir2目录下
* **cp -a file dirs/** 把file文件复制到dirs目录下，保留文件的原来属性
* **cp -rf dir1/ dir2/** 强制复制文件夹而不提示。
*  **cp -rvf dir1/ dir2/** 将dir1目录复制到dir2目录下，并且显示复制过程

# 9.5 rm

可通过rm删除文件或目录。使用rm命令要小心。因为文件删除后不能恢复。为了防止文件误删除，可以在rm后使用-i参数逐个确认要删除的文件。

常用选项含义：

* -i： 以交互式方式执行
* -f：强制删除，忽略不存在的文件，无需提示
* -r：递归地删除目录下的内容，删除文件夹是必须加此选项

示例：

rm a：删除文件a，在删除之前用户需确认删除该文件

rm a b c：删除文件a、 b 和c,在删除之前需确认是否删除

rm -i a：删除文件a，删除之前用户需确认是否删除

rm -f a：删除文件a，不需要用户确认是否删除

rm -v a：删除文件a，删除之后显示结果

rm -r dirs：递归删除dirs，并且每一次删除都需要用户确认是否删除

rm -rf dirs：递归删除目录dirs，每一次删除都不需要用户确认

rm file* 删除所有file打头的文件

rm -i file* 提示是否确定删除

rm -f file* 不提示，强行删除

rm -f [ab].txt 相当于 rm -f a.txt b.txt

rm -f [\^ab].txt 删除除a.txt和b.txt以外所有.txt文件



# 9.6 mv

用户可以使用mv目录来移动文件或目录，也可以给文件或目录重命名

常用选项：

-f：禁止交互操作，如有覆盖也不会提示

-i：确认交互式操作，如果mv操作将导致对已存在的目标文件的覆盖，系统会询问是否重写，要求用户回答以回避误覆盖文件

-v：显示移动进度

示例：

mv file file2 把file文件重命名为file2

mv file dirs/ 把file文件移动到dirs目录下

mv file1 file2 dirs/ 把file1和file2文件剪切到dirs目录下

mv dir1 dir2 dir3/ 把dir1和dir2目录剪切到dir3目录下

mv -i file dir1/ 把file1文件剪切到dir1目录下，如果存在

mv -u dir1/* dir2/ 把dir2目录下所有文件移动到dir2目录下，并且根据文件时间来决定要不要移动

# 10 文件内容查看命令

## 10.1 cat

cat将文件内容一次性输出到终端。

缺点：终端显示的内容有限，如果文件内容太长无法全部显示

示例：

cat /etc/passwd 查看/etc/passwd内容

cat -n /etc/passwd 查看/etc/passwd内容，对输出的所有行编号

cat -b /etc/passwd 查看/etc/passwd内容并且给非空行编号

cat -A /etc/passwd  等价于-vET，显示所有信息，包括换行符

## 10.2 more

more命令将文件内容分页显示到终端，但是只能一直向下浏览，不能回退

more /etc/passwd

相关说明：

回车：显示下一行

空格：显示下一页

q（ctrl + c）：退出

h：获取帮助

## 10.3 less

less命令将文件内容分页显示到终端，可以自由上下浏览

相关说明：

回车（ctrl + n)：显示下一行

ctrl + p：显示上一行

空格（PageDown) ：显示下一页

Pageup：显示上一页

q：退出

## 10.4 head

* head命令从文件头部开始查看前n行内容
* 如果没有指定行数，默认显示前10行内容

命令格式：

head -n[行数] 文件名

示例：

head /etc/passwd 查看/etc/passwd前10行内容

head -c  30 /etc/passwd 查看/etc/passwd前30个字符

head -30 /etc/passwd 查看/etc/passwd前30行内容

head -n 30 /etc/passwd 查看/etc/passwd前30行内容

head -q /etc/passwd 查看/etc/passwd的内容，显示头信息

head -v /etc/passwd 查看/etc/passwd的内容，但是不打印头信息

## 10.5 tail

* 从文件尾部向上查看最后n行内容
* 使用方式：tail -n[行数] 文件名
* 如果没有指定行数，默认显示最后10行

示例：

tail /etc/passwd 查看/etc/passwd后10行内容

tail -n 30 /etc/passwd 查看/etc/passwd后30行内容

tail -c 30 /etc/passwd 查看/etc/passwd后30个字符

**tail -f /etc/passwd  实时查看/etc/passwd文件内容**

tail -q /etc/passwd 查看/etc/passwd的内容，显示头信息

tail -v /etc/passwd 查看/etc/passwd的内容，但是不打印头信息

# 11 du和df命令

## 11.1 du

du命令用于查看某个目录的大小

du命令的使用格式： du [选项] 目录或文件名

选项含义：

* -a：递归显示指定目录中各文件和子目录中文件占用的数据块
* -s：显示指定文件或目录占用的**数据块**
* -b：以字节为单位显示磁盘占用情况
* -h：以K，M，G为单位，提高信息的可读性

## 11.2 df

df命令用于检测文件系统的磁盘空间占用情况和空余情况，可以显示所有文件系统对节点和磁盘块的使用情况。

选项含义：

* -a：显示所有文件系统的磁盘使用情况
* -m：以KB为单位显示
* -h：以K，M，G为单位，提高信息的可读性

# 12 查找相关命令

> which：命令查找，从环境变量中查找
>
> find：文件查找，针对文件名
>
> locate：文件查找，依赖数据库，updatedb命令更新数据库
>
> **alias	ll='ls	-l	–color=auto'	#给命令取别名**

## **12.1 find**

find命令功能非常强大，通常用来在特定的目录下搜索符合条件的文件，也可以用来搜索特定用户属主的文件。

### 12.1.1 按文件名查询：使用参数-name

**命令：find + 路径 + -name + "文件名"**

示例：find 	/home 	-name 	"a.txt"

​			find	/home	-iname	"A.txt"	# -iname	表示不区分大小写

### 12.1.2 按文件大小查询：使用参数-size

**命令：find + 路径 + -size + 范围**

范围：(注意k要小写，M必须大写)

+ 大于：+100k表示大于100k
+ 小于：-100k表示小于100k
+ 等于：100k表示等于100k

示例：查询家目录下大于100k的文件

find ~/ -size +100k

### 12.1.3 按文件类型查询：使用参数-type

**命令：find + 路径 + -type + 类型**

类型：

* 普通文件用**f**表示而不是-
* d -> 目录
* l ->符号链接
* b -> 块设备文件
* c->字符设备文件
* s->socket文件，网络套接字
* p->管道

示例：查找指定目录下的普通文件：

find /home -type f

### 12.1.5 按文件目录深度查找

> **find 	/ 	-maxdepth 	3 	-a	-name 	"ifcfg-en*"**	#查找最大目录深度为3，-a表示and

### 12.1.6 查找之后处理的动作

> **默认动作：-print**
>
> find	/etc	-name	"a.txt"	-print	#-print为默认动作不需要显式指定
>
> **找到后删除：-delete**
>
> find	/etc	-name	"a.txt"	-delete
>
> **找到后复制：-ok 	cp	-rvf	{}	/tmp	\;**
>
> find	/etc	-name	ifcfg*	-ok	cp	-rvf	{}	/tmp	\;

### 12.1.7示例大全

* find /etc -uid 500 查找/etc下uid是500的文件，和-user类似
* find /etc -gid 500 查找/etc下gid是50的文件，和-grouds类似
* find /etc -user root 查找/etc下所属用户组是root的文件
* find /etc -groud root 查找/etc下所属用户组是root的文件
* find /etc -nouser 查找没有所属用户的文件
* find /etc -nogroup 查找没有所属用户组的文件
* **find /etc -perm 111 -type d 查找/etc下权限为111的目录**
* find . -size +10M 查找当前目录大于10M的文件，单位可以有k，M，G，b等
* find / -mtime 1 查找根目录下1天以前修改的所有文件
* find / -mtime +2 查找根目录下2个多小时以前修改的所有文件
* find / -mtime -3 查找根目录下最近3小时内修改的所有文件
* find / -atime 1 查找根目录下1天以前访问或读过的所有文件
* find / -atime -1 查找根目录下最近1天以内访问或读过的所有文件
* find / -ctime -3 查找最近3天内状态发生改变的文件
* find / -cmin -3 查找最近3分钟内状态发生改变的文件
* find / -empty 查找根目录下的所有空白文件或空白目录
* find / -false 查找根目录下总是错误的文件
* find / -false -exec ls -l {} \; 查找根目录下总是错误的文件并用ls -l 查看
* find . -name "*.conf" -exec rm -rf {} \;
* find . -name "*.conf" **| xargs** rm -rf 删除当前目录下所有以.conf结尾的文件

## 12.2 grep

Linux系统中grep命令是一种强大的文本搜索工具，grep允许对文本进行模式查找。如果匹配模式，grep打印包含模式的所有行。

grep一般格式为：

grep [选项] "搜索内容串"文件名

在grep命令中输入字符串参数时，最好引号或双引号括起来。例如：grep 'a'1.txt。

**选项说明：**

* -v：显示不包含匹配文本的所有行（相当于取反）
* -n：显示匹配行及行号
* -i：忽略大小写

命令：

grep -r + "查找的关键字" + 路径

搜索目录需要添加参数：-r 表示递归查找

例如：查找/home/moshang下包含“hello world”字符串的文件

grep -r "hello, world" /home/moshang

**示例：**

* grep -a hello/bin/ls 将二进制文件以文本文件的方式搜索hello
* grep -i hello /etc/passwd 在/etc/passwd文件里查找hello且忽略大小写
* grep -n hello /etc/passwd 在/etc/passwd文件里搜索hello，结果显示在文件中出现的行号
* grep -w hello /etc/passwd 在/etc/passwd文件里搜索完全匹配hello的单词的行
* grep -v hello /etc/passwd 在/etc/passwd文件搜索并显示没有hello的行
* grep -r hello /etc/ 在/etc/目录里所有文件里查找hello并显示结果
* grep -i hello /etc/passwd --color=auto 在/etc/passwd文件中查找hello，并忽略大小写，然后高亮显示匹配的关键字

# 13 管道

管道（|）：一个命令的输出可以通过管道作为另一个命令的输入。

管道可以理解为现实生活中的管子，管子的一头塞东西进去，另一头取出来，这里“|”的左右分为两端，左端塞东西（写）右端取东西（读）。

例如：

cat /etc/passwd | less

## 13.1 tee（T）管道

在管道中间接一个通道，形成T字管道，在把输出传送给下一个命令的同时，可以截取一份保存在文件中。

语法：在管道中间添加tee命令，如tee 2.txt

> cat /etc/passwd | grep "root" | **tee 1.txt** | head -1

## 13.2 参数传递 Xargs

[xargs命令_Linux xargs 命令用法详解：给其他命令传递参数的一个过滤器 (ailinux.net)](http://lnmp.ailinux.net/xargs)

**xargs命令**是给其他命令传递参数的一个过滤器，也是组合多个命令的一个工具。它擅长将标准输入数据转换成命令行参数，xargs能够处理管道或者stdin并将其转换成特定命令的命令参数。xargs也可以将单行或多行文本输入转换为其他格式，例如多行变单行，单行变多行。xargs的默认命令是[echo](http://lnmp.ailinux.net/echo)，空格是默认定界符。这意味着通过管道传递给xargs的输入将会包含换行和空白，不过通过xargs的处理，换行和空白将被空格取代。xargs是构建单行命令的重要组件之一。

cp rm 一些特殊命令不能识别通过管道获取的文本输入

例：

> $	touch file{1, 3, 5}.txt
>
> $	vim file.txt
>
> /home/li/test/file1.txt
>
> /home/li/test/file3.txt
>
> /home/li/test/file5.txt
>
> $	cat file.txt	|	rm -rvf 	# 无法删除
>
> $ cat file.txt	**|xargs**	rm -rvf	#可以删除

### 13.2.1 xargs命令其它用法

xargs用作替换工具，读取输入数据重新格式化后输出。

定义一个测试文件，内有多行文本数据：

```shell
cat test.txt

a b c d e f g
h i j k l m n
o p q
r s t
u v w x y z
```

多行输入单行输出：

```shell
cat test.txt | xargs

a b c d e f g h i j k l m n o p q r s t u v w x y z
```

**-n选项**多行输出：

```shell
cat test.txt | xargs -n3

a b c
d e f
g h i
j k l
m n o
p q r
s t u
v w x
y z
```

**-d选项**可以自定义一个定界符：

```shell
echo "nameXnameXnameXname" | xargs -dX

name name name name
```

结合**-n选项**使用：

```
echo "nameXnameXnameXname" | xargs -dX -n2

name name
name name
```

# 14 压缩包管理

## 14.1 tar

计算机中的数据经常需要备份，tar是Unix/Linux中最常用的备份工具，此命令可以把一系列文件归档到一个大文件中，也可以把档案文件解开以恢复数据。

tar使用格式：

**tar [选项] 打包文件名 文件**

常用选项：

* -c：生成档案文件，创建打包文件

* -v：列出归档解档的详细过程，显示进度

* -f：指定档案文件名称，f后面一定是.tar文件，所以必须放选项最后

* -t：列出档案总包含的文件

* -x：解开档案

  除了f需要放在选项最后，其他选项位置任意

  主要使用选项组合：

  * tar -cvf
  * tar -xvf
  * tar -tvf

示例：

tar -cvf test.tar 1.txt 2.txt  ->**归档（多个文件归档为一个文件）**

tar -xvf music.tar ->还原

tar -tvf music.tar ->查看归档文件内容

## 14.2 gzip

| tar与gzip命令结合使用实现文件打包、压缩。

| tar只负责打包文件，但不压缩，用gzip压缩tar打包后的文件，其扩展名为xxx.tar.gz。

gzip使用格式如下：（**只能压缩文件，且原文件会消失，不能压缩目录，目录需要先打包后压缩**）

gzip [选项] 被压缩文件

常用选项：

* -d：解压 -> gunzip命令
* -r：压缩所有子目录中的文件，递归压缩

示例：（解压和压缩都不保留原文件）

gzip a b c d -> a.gz b.gz c.gz d.gz

gzip -d a.gz -> a

和tar结合使用

**tar -czvf test.tar.gz test** ->打包压缩

**tar -xzvf test.tar.gz** ->解压解档

**tar -xzvf test.tar.ga -C  /tmp** -> 解压解档到/tmp目录 ,-C指定解压到目录

**tar zcvf test.tar.gz 1.c 2.c 3.c 4.c** ->把1.c 2.c 3.c 4.c归档压缩成test.tar.gz

## 14.3 bzip2

* tar与bzip2目录结合使用实现文件打包、压缩(用法与gzip一样) bunzip2
* tar只负责打包文件，但不压缩，用bzip2压缩tar打包后的文件，其扩展名为xxx.tar.bz2
* **在tar命令总增加一个选项（-j）可以调用bzip2实现一个压缩的功能，实现一个先打包后压缩的过程。**
* 压缩用法：tar jcvf 压缩包名 文件.... (tar jcvf bk.tar.bz2 *.c)
* 解压用法：tar jxvf 压缩包名 （tar jxvf bk.tar.bz2）

tar -zcvf

tar -zxvf

tar -jcvf

tar -jxvf

**tar -xvf  万能解压（gzip bzip2）**

## 14.4 zip和unzip

通过zip归档压缩文件的目标文件不需要指定扩展名，默认扩展名为zip。

压缩文件： zip -r 目标文件（没有扩展名）源文件

解压文件：unzip -d 解压后目录文件 压缩文件

示例：

1 把b c d打包压缩成myzip.zip

**zip myzip b c d -> myzip.zip**

2 把myzip.zip解压解档至/test目录下：

**unzip -d /test myzip.zip**

unzip myzip.zp -> 解压归档到当前目录

# 15  文件权限管理

## 15.1 文件权限

文件权限就是文件的访问控制权限，即哪些用户和组群可以访问文件以及可以执行什么样的操作。Unix/lLinux系统是一个典型的多用户系统，不同的用户处于不同的地位，对文件和目录有不同的访问权限。为了保护系统的安全性，Unix/Linux系统除了对用户权限作了严格的界定外，还在用户身份证、访问控制、传输安全、文件读写等方面作了周密的控制。

## 15.2 访问用户分类

通过设定权限可以从以下三种访问方式限制访问限制：

1）只允许用户访问自己（所有者）

所有者就是创建文件的用户，用户是所有用户所创建文件的所有者，用户可以允许所在的用户组能访问用户的文件。

2）允许一个预先指定的用户组中的用户访问（用户组）

用户都组合成用户组，例如，某一类或某一项目中的所有用户都能够被系统管理员归为一个用户组，一个用户能够授予所在用户组的其他成员的文件访问权限。

3）允许系统中的然后用户访问（其他用户）

用户也将自己的文件向系统内的所有用户开放，在这种情况下，系统内的所有用户都能够访问用户的目录和文件。在这种意义上，系统内的其他所有用户都是other用户类。

## 15.3 访问权限说明

用户能够控制一个给定的文件或目录的访问程度，一个文件或目录可能有读、写及执行权限：

* 读权限（r）

对文件而言，具有读取文件内容的权限(cat)；**对目录来说，具有浏览目录的权限(ls)。**

* 写权限（w）

对文件而言，具有新增、修改文件内容的权限；**对目录而言，具有改变目录下文件列表的权限**。

写权限只限制了不改变文件夹内部文件树状结构。但是可以修改内部文件的内容（文件名不可以改变，改变了就会改变树状结构）

文件夹，可以理解为，其存储的内容是，文件夹内部的树状结构信息。那么读写权限，就是对树状结构信息的读取和写入权限，可执行权限，即是否，能够访问其树状结构信息所描述的文件。

* 可执行权限（x）

对文件而言，具有执行文件的权限；**对目录而言，该用户具有进入该目录的权限（cd）**。

注意：通常，Unix/Linux系统只允许文件的属主（所有者）或超级用户改变文件的读写权限。

## 15.4 chmod

chmod修改文件权限有两种使用格式：字母法与数字法

字母法：chmod u/g/o/a +/-/= rwx 文件

选项含义：

* u：user，表示该文件的所有者
* g：group，表示与该文件的所有者同属于一个组者，即用户组
* o：other，表示其他以外的用户
* a：all，表示所有用户

符号含义：

* +：增加权限
* -：撤销权限
* =：设定权限

示例：

* **chmod o+w file** 给其他用户增加写权限
* chmod o+rwx file
* chmod u-r file 给文件file的拥有者撤销读权限
* chmod u=rw, g=r, o=r file 把file的权限设定为rw-r--r--

数字法：“rwx"这些权限可以用数字来代替

如：chmod u=rwx,g=rx,o=r file

等同于：chmod u=7,g=5,o=4 file(不可以执行)

例如：chmod 751 file (可以执行)

**chmod -R 777 /test 递归改变目录下所有文件的权限**

**常用：**

**rwx：7**

**rw-：6**

**r--：4**

## 15.5 chown

chown用于修改文件所有者和用户所属组

使用方法：chown 用户名[:用户组] 文件或目录名

示例：

* 将文件所属者修改为root用户

​	**sudo chown root 1.txt**

* 将文件所属者改为deng，所属组改为root

​	**sudo chown deng:root 1.txt**

* 单独只修改文件所属组

​	**sudo chown :root 1.txt**

## 15.6 chgrp

chgrp用于修改文件所属组

使用方法：chgrp 用户组名 文件或目录名

示例：

**sudo chgrp deng 1.txt**

## 15.7 基本权限ACL(access control list)

ACL文件权限管理：设置不同用户，不同的基本权限（r、w、x）。对象数量不限。

UGO设置基本权限：只能设置一个用户、一个组和其他人的权限。

**ACl权限的优先级高于ugo权限。**

### 15.7.1 语法

命令 选项 用户或组:用户名或组名:权限  文件对象

**setfacl -m u:user01:rw /home/test.txt**

**1 查看文件有哪些ACL权限**

**getfacl /home/test.txt**

**2 设置特定用户或组的ACL权限**

setfacl -m u:user02:rw /home/li

setfacl -m u:user03:- /home/li  # 无任何权限

**setfacl -m o::rw /home/li**  # 设置其它组的权限，会与UGO基本权限叠加

**3 删除ACL权限**

setfacl -x u:user01 /home/li	#删除用户user01的acl权限

setfacl -x g:group01 /home/li	#删除用户组group01的acl权限

setfacl -b /home/li	#删除所有acl权限

**watch -n1 'ls -l /tmp/file'	每隔一秒钟执行一些后面的命令**

## 15.8 特殊权限

[详情页](https://www.cnblogs.com/liuyuelinfighting/p/15623163.html#tid-y3bHrD)

>* 小s权限标识在所有者位的时候，特殊权限位SetUID
>* 小s权限标识在所属组的时候，特殊权限位SetGID
>* 小t权限标识在其他位的时候，特殊权限位Sticky BIT
>
>4代表SUID
>
>2代表SGID
>
>1代表SBIT

### 15.8.1 特殊位suid(gid)

SUID (Set UID)是Linux中的一种特殊权限,其功能是用户在运行某个程序时，如果该程序有SUID权限，那么程序运行为进程时，**进程的属主不是发起者，而是程序文件所属的属主**。但是SUID权限的设置只针对二进制可执行文件。

在执行过程中，调用者会暂时获得该文件的所有者权限,且该权限只在程序执行的过程中有效. 通俗的来讲,假设我们现在有一个可执行文件ls,其属主为root,当我们通过非root用户登录时,如果ls设置了SUID权限,我们可在非root用户下运行该二进制可执行文件,在执行文件时,该进程的权限将为root权限.，利用此特性,我们可通过SUID进行提权。

例 1：

**chmod u+s /usr/bin/cat**	#**cat命令被调用时具有属主的权限。**

例 2：

/bin/passwd具有s权限，而/etc/shadow文件没有任何权限，即普通用户不能对这个文件进行任何操作，但普通用户执行passwd命令时，所属主为/bin/passwd的所属主root，这时就可以把修改的密码写入/etc/shadow中了，换言之，如果/bin/passwd没有s权限，则普通用户就不能修改密码了。

### 15.8.2 文件属性chattr

lsattr命令查看文件属性，该属性对超级管理员也其作用，但超级管理员可以去除该属性

**lsattr	file**

* i(immutable)：在文件启用这个属性时，我们不能更改、重命名或者删除这个文件
  * chattr +i file	#给文件添加i属性
  * chattr -i file	#给文件去除i属性
* a(append)：只允许在文件中进行追加操作，不能对原有内容进行修改
  * chattr +a file	#给文件添加a属性

### 15.8.3 进程掩码umask

在shell进程中创建文件，先查看当前用户的umask权限

> $ umask
>
> $ 0022	#最高位位代表特殊权限位，如setuid、setgid、sticky bit
>
> ## 0777 - 0022 = 0755（0777 % ~0022），所以系统创建目录/文件夹的时候，权限为755，对有有用户都有x权限，即进入目录的权限
>
> ## 系统为了保护自己，所以在创建的文件上，去掉了所有的可执行权限，所以0755 - 0111 = 0644，即系统创建文件的默认权限为0644
>
> $ umask 000	#设置掩码为000,0777-0000=0777,0777 - 0111 = 0666，即创建目录权限为0777，文件为0666

# 16 软件安装和卸载

Linux安装软件和Windows安装软件的概念不太一样。

Windows总是把一个软件安装在一个指定的文件夹里面。而Linux不一样，Linux是把要安装的软件发布在整个系统的各个文件夹里面，比如所有软件的配置文件都安装在/etc下面，软件需要的库文件都安装在/lib下面，日志文件都在/var/log下面，/bin下是常用的程序，等等。

## 16.1 RPM包管理

**RPM包：**

> RPM Package Manager(原Red Hat Package Manager，现在是一个递归缩写)，由Red Hat公司提出，被众多Linux发行版所采用。
>
> RPM包也被称为二进制（binary code）包无需编译，可以直接使用，无法设定个人设置，开关功能。
>
> 软件包示例：mysql-community-common-5.7.12-1.el7.x86_64.rpm （**软件名称 版本号(主版本、次版本、修订号) 操作系统（enterprise linux 7） cpu架构**）
>
> 操作系统：el6 el5 fedora suse debin ubuntu
>
> x86_64：表示64位软件
>
> noarch：表示32、64位通用

**源码包：**

> **源代码包(source code)需要编译环境编译才能运行**
>
> 可以修改源码设定个人设置，如安装位置，开关功能，但配置复杂
>
> 软件包示例：nginx-1.8.1.tar.gz
>
> 

## 16.1 在线安装

如果是在ubuntu平台，软件的安装可以通过互联网在线安装，更加方便快捷：

命令：

* sudo apt-get update 更新源
* sudo apt-get install xxx 从源中安装xxx软件
* sudo apt-get remove xxx 删除xxx软件
* sudo apt-get clean 清理下载包的存档
* sudo apt-get upgrade 更新已安装的包
* sudo apt-get source 下载该包的源代码

## 16.2 软件包安装

在ubuntu下安装文件为deb格式

软件安装：

**sudo dpkg -i xxx.deb**（-i表示install）

软件卸载：

**sudo dpkg -r 软件名** （-r表示remove）

# 17  重定向

重定向

标准输入 代码0 默认设备为键盘

标准输出 代码1 默认设备为屏幕

错误输出 代码2 默认设备为屏幕

cat 1.txt > output.txt 标准正确输出重定向到output.txt

cat 2.txt >> output.txt 以追加的方式将标准正确输出重定向到output.txt，即不会覆盖原内容

cart 4.txt 2> error.txt 将标准错误输出重定向到error.txt

cart 4.txt 2>> error.txt 将标准错误输出以追加的方式重定向到error.txt

cart 5.txt 2> /dev/null 将标准错误输出重定向至黑洞

ls tgh /etc/passwd &< /dev/null 将标准正确和标准错误输出都重定向到黑洞

ls tgh /etc/passwd &<< 2.txt 将标准正确输出和标准错误输出都重定向到2.txt

cat < file  标准输入重定向 file作为cat输入的内容

## 17.1 Linux输入重定向

对于输入重定向来说，其需要用到的符号以及作用如表1所示。

|     命令符号格式     |                             作用                             |
| :------------------: | :----------------------------------------------------------: |
|     命令 < 文件      |                 将指定文件作为命令的输入设备                 |
|    命令 << 分界符    | 表示从标准输入设备（键盘）中读入，直到遇到分界符才停止（读入的数据不包括分界符），这里的分界符其实就是自定义的字符串 |
| 命令 < 文件1 > 文件2 |  将文件1作为命令的输入设备，该命令的执行结果输出到文件2中。  |

【例1】

默认情况下，cat 命令会接受标准输入设备（键盘）的输入，并显示到控制台，但如果用文件代替键盘作为输入设备，那么该命令会以指定的文件作为输入设备，并将文件中的内容读取并显示到控制台。

以/etc/passwd文件（存储了系统中所有用户的基本信息）为例，执行如下命令：

```shell
cat /etc/passwd
#这里省略输出结果
cat < /etc/passwd
#输出结果同上面命令相同
```

注意，虽然执行结果相同，但第一行代表是以键盘作为输入设备，而第二行代码是以 /etc/passwd 文件作为输入设备。

【例2】

```shell
cat << 0
>c.biancheng.net
>Linux
>0
c.biancheng.net
Linux
```

可以看到，当指定了 0 作为分界符之后，只要不输入 0，就可以一直输入数据。

【例3】

首先，新建文本文件a.txt，然后执行如下命令

```shell
cat a.txt
cat < /etc/passwd > a.txt
cat a.txt
#输出了和/etc/passwd文件内容相同的数据
```

可以看到，通过重定向 /etc/passwd 作为输入设备，并输出重定向到 a.txt，最终实现了将 /etc/passwd 文件中内容复制到 a.txt 中。

## 17.2 Linux输出重定向

相较于输入重定向，我们使用输出重定向的频率更高。并且，和输入重定向不同的是，输出重定向还可以细分为标准输出重定向和错误输出输出重定向两种技术。

例如，使用 ls 命令分别查看两个文件的属性信息，但其中一个文件是不存在的，如下所示：

```shell
touch demo1.txt
ls -l demo1.txt
-rw-rw-r--. 1 root root 0 Oct 12 15:02 demo1.tx
ls -l demo2.txt # 不存在的文件
ls:cannot access demo2.txt:No such file or directory

cat 2.txt > /dev/null	#输出重定向至空洞
```

上述命令中，demo1.txt 是存在的，因此正确输出了该文件的一些属性信息，这也是该命令执行的标准输出信息；而 demo2.txt 是不存在的，因此执行 ls 命令之后显示的报错信息，是该命令的错误输出信息。

> 再次强调，要想把原本输出到屏幕上的数据转而写入到文件中，这两种输出信息就要区别对待。	

在此基础上，标准输出重定向和错误输出重定向又分别包含清空写入和追加写入两种模式。因此，对于输出重定向来说，其需要用到的符号以及作用如表 2 所示。

|                  命令符号格式                  | 作用                                                         |
| :--------------------------------------------: | ------------------------------------------------------------ |
|                  命令 > 文件                   | 将命令执行的标准输出结果重定向输出到指定的文件中，如果该文件已包含数据，会清空原有数据，再写入新数据。 |
|                  命令 2> 文件                  | 将命令执行的错误结果重定向到指定的文件中，如果该文件已包含数据，会清空原有数据，再写入新数据。 |
|                  命令 >> 文件                  | 将命令执行的标准输出结果重定向输出到指定的文件中，如果该文件已包含数据，新数据将写入到原有内容的后面。 |
|                 命令 2 >> 文件                 | 将命令执行的错误输出结果重定向到指定的文件中，如果该文件中已包含数据，新数据将写入到原有内容的后面。 |
| 命令 >> 文件 2>&1<br />或者<br />命令 &>> 文件 | 将标准输出或者错误输出写入到指定文件，如果该文件中已包含数据，新数据将写入到原有内容的后面。1：表示stdout标准输出 2：表示stderr标准错误 &：表示等同于的意思。这里2>&1表示2的输出重定向等同于1，也就是标准错误输出重定向到标准输出。因为前面标准标准输出已经重定向到了指定文件，所以标准错误也重定向到指定文件。 |

【例4】

新建一个包含有 "Linux" 字符串的文本文件 Linux.txt，以及空文本文件 demo.txt，然后执行如下命令：

```shell
[root@localhost ~]# cat Linux.txt > demo.txt
[root@localhost ~]# cat demo.txt
Linux
[root@localhost ~]# cat Linux.txt > demo.txt
[root@localhost ~]# cat demo.txt
Linux     <--这里的 Linux 是清空原有的 Linux 之后，写入的新的 Linux
[root@localhost ~]# cat Linux.txt >> demo.txt
[root@localhost ~]# cat demo.txt
Linux
Linux     <--以追加的方式，新数据写入到原有数据之后
[root@localhost ~]# cat b.txt > demo.txt
cat: b.txt: No such file or directory  <-- 错误输出信息依然输出到了显示器中
[root@localhost ~]# cat b.txt 2> demo.txt
[root@localhost ~]# cat demo.txt
cat: b.txt: No such file or directory  <--清空文件，再将错误输出信息写入到该文件中
[root@localhost ~]# cat b.txt 2>> demo.txt
[root@localhost ~]# cat demo.txt
cat: b.txt: No such file or directory
cat: b.txt: No such file or directory  <--追加写入错误输出信息
```

# 18 其他命令

## 18.2 tree 

tree以树状形式查看指定目录内容，使用该命令需要安装软件tree：sudo apt install tree

常用选项：

-L n 查看n层目录

示例：

tree -L 2 /home/moshang/test

## 18.3 ln（创建软硬链接）

ln命令主要用于创建链接文件，LInux链接文件类似于Windows下的快捷方式。

链接文件分为软链接和硬链接：

* 软链接（符号链接）：软链接相当于重新创建了一个不一样的属性的链接文件，该文件内容为创建链接时使用的源文件的路径，所以创建软链接时，尽量使用绝对路径。软链接默认权限为777，源文件删除，则链接文件失效。

* 硬链接：相当于创建了一个文件指针，与源文件拥有相同的属性（inode），即拥有相同的权限，大小等，与源文件指向同一个磁盘空间，一个被修改，另一个也会随之改变。一个被删除，另外一个依旧有效。**不能对目录创建硬链接，只能对普通文件创建硬链接**。硬链接只能创建在同一个文件系统（分区，逻辑卷）下。

  可以使用readlink命令读取符号链接文件的内容（存储源文件的路径）

  readlink a_link

  /home/deng/share/a

使用格式：

ln 源文件 链接文件 创建硬链接

ln -s 源文件 链接文件 创建软链接



* nautilus 路径 召唤路径的界面

* which 外部命令 查找命令文件

# 19 source命令

当我修改了/etc/profile文件，我想让它立刻生效，而不用重新登录；这时就想到用source命令，如:source /etc/profile

## 19.1 source命令：

source命令也称为“点命令”，也就是一个点符号（.）,是bash的内部命令。

功能：使Shell读入指定的Shell程序文件并依次执行文件中的所有语句

source命令通常用于重新执行刚修改的初始化文件，使之立即生效，而不必注销并重新登录。

用法：

source filename 或 . filename

source命令（从C Shell而来）是bash shell的内置命令；点命令（.）就是点符号（从Bourne Shell而来）是source的另一名称

## 19.2. source filename 与 sh filename 及./filename执行脚本的区别在那里呢？

1.当shell脚本具有可执行权限时，用sh filename与./filename执行脚本是没有区别得。./filename是因为当前目录没有在PATH中，所有"."是用来表示当前目录的。
2.sh filename 重新建立一个子shell，在子shell中执行脚本里面的语句，该子shell继承父shell的环境变量，但子shell新建的、改变的变量不会被带回父shell，除非使用export。
3.source filename：这个命令其实只是简单地读取脚本里面的语句依次在当前shell里面执行，没有建立新的子shell。那么脚本里面所有新建、改变变量的语句都会保存在当前shell里面。

## 19.3. 举例说明：

1.新建一个test.sh脚本，内容为:A=1
2.然后使其可执行chmod +x test.sh
3.运行sh test.sh后，echo $A，显示为空，因为A=1并未传回给当前shell
4.运行./test.sh后，也是一样的效果
5.运行source test.sh 或者 . test.sh，然后echo $A，则会显示1，说明A=1的变量在当前shell中

## 19.4.登录Linux时要执行文件的过程

在登录Linux时要执行文件的过程如下：
在刚登录Linux时，首先启动 /etc/profile 文件，然后再启动用户目录下的 ~/.bash_profile、 ~/.bash_login或 ~/.profile文件中的其中一个，

执行的顺序为：~/.bash_profile、 ~/.bash_login、 ~/.profile。

如果 ~/.bash_profile文件存在的话，一般还会执行 ~/.bashrc文件。

执行顺序为： **/etc/profile -> (~/.bash_profile | ~/.bash_login | ~/.profile) -> ~/.bashrc -> /etc/bashrc -> ~/.bash_logout**

## 19.5. 各个文件的解释

（1） /etc/profile：(**系统环境变量**) 此文件为系统的每个用户设置环境信息,当用户第一次登录时,该文件被执行. 并从/etc/profile.d目录的配置文件中搜集shell的设置。）

（2） /etc/bashrc: (**用户变量**)为每一个运行bash shell的用户执行此文件.当bash shell被打开时,该文件被读取（即每次新开一个终端，都会执行bashrc）。

（3） ~/.bash_profile: 每个用户都可使用该文件输入专用于自己使用的shell信息,当用户登录时,该文件仅仅执行一次。默认情况下,设置一些环境变量,执行用户的.bashrc文件。

（4） ~/.bashrc: 该文件包含专用于你的bash shell的bash信息,当登录时以及每次打开新的shell时,该该文件被读取。

（5） ~/.bash_logout: 当每次退出系统(退出bash shell)时,执行该文件. 另外,/etc/profile中设定的变量(全局)的可以作用于任何用户,而~/.bashrc等中设定的变量(局部)只能继承 /etc/profile中的变量,他们是"父子"关系。

（6） ~/.bash_profile: 是交互式、login 方式进入 bash 运行的~/.bashrc 是交互式 non-login 方式进入 bash 运行的通常二者设置大致相同，所以通常前者会调用后者。

## 19.6.shell脚本中常用

1. **修改当前用户信息后，让其立刻生效**
   *~/.bash_profile用户当前用户，只在登录时读取一次，输入专用于自己使用的shell信息*
   *~/.bashrc:该文件包含专用于你的bash shell的bash信息,当登录时以及每次打开新的shell时,该文件被读取*

```shell
source ~/.bash_profile  
source ~/.bashrc 
```

1. ~/是进入当前用户的主目录
2. .bashrc 是进入.bashrc文件夹，就是用户目录下的名字是.bashrc的目录

​	3..bashrc 的确是在 /root 目录下。. 开头的文件指的是隐藏文件

2. **修改/etc/profile文件后，让其立刻生效**
   */etc/profile:此文件为系统的每个用户设置环境信息,当用户第一次登录时,该文件被执行*

# 20 wc

wc [选项] [文件]

输出每个指定文件的行数、单词计数和字节数，如果指定了多于一个文件，继续给出所有相关数据的总计。如果没有指定文件，或者文件为“-”，则从标准输入读取数据。

**选项说明：**

-c，–bytes：输出字节数统计

-m，–chars：输出字符数统计

-l，–lines：输出行数统计

-L，–max-line-length：显示最长行的长度

-w，–words：显示单词计数

–help：显示此帮助信息并退出

–version：显示版本信息并退出

# 21 set

* set -e：表示后续所有的bash命令的返回code如果不是0，那么脚本立即退出，后续的脚本将不会得到执行的机会;

# 22 umask

**umask [-p] [-S] [mode]**

**umask命令用来指定建立文件时预设的权限掩码。当没有给定mode时，表示查询预设权限**

使用mode描述的格式设置文件模式掩码，如果未指定mode，展示当前掩码。mode可以是八进制数或者类似chmod中的符号字符串。

选项：

* -p：以八进制数字表示掩码
* -S：以符号格式表示掩码

【例1】

```shell
umask -p 0022
umask
0022

umask -S u=rwx,g=rx,o=rx
umask -S
u=rwx,g=rx,o=rx
```

【例2】

```shell
# 使用指令umask查看当前权限掩码
$ umake
# 执行上面命令后输出：
0022
# 接下来，使用指令"mkdir"创建一个目录，并使用指令"ls"获取该目录的详细信息，输入命令如下：
$ mkdir test
$ ls -d -l test/
# 执行上面的命令后，将显示新创建目录的详细信息，如下所示：
drwxr-xr-x 2 rootlocal rootlocal 4096 2011-9-19 21:46 test1/
# 注意：在上面的输出信息中，"drwxr-xr-x"="777-022=755"
```

# 23 stat 命令

**stat file**

查看文件详细信息

# 24 ps

众所周知，Linux是一个多任务和多用户系统。因此，它允许多个过程同时运行而不会相互干扰。过程是Linux操作系统的重要基本概念之一。进程是程序的执行实例，并在操作系统中执行不同的任务。

Linux为我们提供了一个名为**ps**的实用程序，用于查看与系统上的进程相关的信息，该实用程序代表**“进程状态”**的缩写。ps命令用于列出当前正在运行的进程及其PID以及其他一些信息，具体取决于不同的选项。它从 **/proc** 文件系统中的虚拟文件中读取进程信息。/proc 包含虚拟文件，这就是它被称为虚拟文件系统的原因。

ps提供了许多选项，用于根据我们的需要操作输出。

语法

```shell
ps	[options]
```

选项

* a：显示所有进程
* -a：显示同一终端下的所有进程
* -A：显示所有进程
* c：显示进程的真实名称
* -N：
* -e：等于“-A"
* e：显示环境变量
* f：显示程序间的关系
* -H：显示树状结构
* r：显示当前终端的进程
* T：显示当前终端的所有进程
* -au：显示比较详细的资讯
* -aux：显示所有包含其它使用者的行程

#    25 用户管理

## 25.1 用户信息存储文件

### 25.1.1 用户基本信息文件

**/etc/passwd（冒号分割为7列字段）**

```tex
用户名:x:uid:gid:描述:HOME:shell (x为密码占位符，密码信息不在该文件)
例如：
root:x:0:0:root:/root:/bin/bash

uid：用户的身份证
系统约定：RHEL7
uid：0 -> 特权用户
uid：1~499 -> 系统用户
uid：1000+ -> 普通用户

GID：GROUP组号

描述：一般为用户名，用于描述

HOME：家目录，登录系统时，所在目录，如/root，/home/li

/bin/bash：登录shell，命令解释器
/bin/nologin或/sbin/nologin表示该用户不允许登录系统
```

### 25.1.2 用户密码信息文件

**/etc/shadow（8列，密码加密）**

```tex
例如：
root:$1$MYG2N:15636:0:99999:7: : :
1）“登录名”是与/etc/passwd文件中的登录名相一致的用户账号
2）密码加密值（口令），“口令”字段存放的是加密的用户口令字，如果为空，则对应的用户没有口令，登录时不需要口令；
星号代表账号被锁定；双感叹号表示这个密码已经过期了。
$6$开头的，表明是用SHA-512加密的；
$1$表明是MD5加密的
$2$表明是Blowfish加密的
$5$表明是SHA-256加密的。
3）"最后一次修改时间"表示从某一时刻起，到用户最后一次修改口令时的天数。时间起点对不同的系统可能不一样。
4）“最小时间间隔”指的是两次修改口令之间所需的最小天数
5）“最大时间间隔”指的是口令保持有效的最大天数，99999代表永久
6）“警告时间”字段表示的是从系统开始警告用户到用户密码正式失效之间的天数。
7）“不活动时间”表示的是用户没有登录活动但账号仍能保持有效的最大天数。（软限制，到期后多少天就不能使用账号了，禁用）
8）“失效时间”字段给出的是一个绝对天数，如果使用了这个字段，那么就给出相应的的账号生存期。期满后，该账号就不再是一个合法的账号，也就不能再用来登录了。（硬限制。）
9）保留
```

### 25.1.3 用户组信息文件

**/etc/group(4列)**

```tex
例如：
root:x:0:
组名:组密码:组ID:组成员
```

## 25.2 用户管理

### 25.2.1 创建用户 未指定选项

```tex
> useradd user01
# 查询用户信息
> id user01
uid=507(user01) gid=512(user01) groups=512(user01)

>ls /var/spool/mail/user01
/var/spool/mail/user01

# 设置密码
> passwd user01

小结：
如果创建一个用户时，未指定任何选项，系统会创建一个和用户名相同的组作为用户的Primary Group##
```

### 25.2.2 创建用户 指定选项

```tex
# 指定uid
> useradd user02 -u 1503
# 指定家目录
> useradd user03 -d /user04
```

### 25.2.3 删除用户

```tex
userdel -r user04   # -r 表示删除用户家目录
```

### 25.2.4 修改密码

```tex
# root修改其它用户密码
> passwd user03

# 普通用户修改密码(只能修改自己的密码)
> passwd
```

### 25.2.5 修改用户属性

```tex
usermod -s /sbin/nologin user03 # -s 表示-shell修改登录shell

# 修改用户的基本组
usermod user03 -g group02

# 修改用户的附加组
usermod user03 -G group03

# 修改用户id
usermod -u 1314 li
```

## 25.3 用户组管理

### 25.3.1 用户组的创建

```tex
> groupadd group01
# 创建组group02，并指定gid 2000
> groupadd group02 -g 2000

# 查看/etc/group中组group02的信息
grep 'group02' /etc/group

# 删除组group02
groupdel group02

# 将用户从组中移除
gpasswd -d user01 group04

# 修改组id
groupmod -g 1512 group02
```

### 25.3.2 用户组的分类（相对于用户来说）

每个用户必属于一个组。

基本组：随用户而创建的组，组名与用户名相同，id不一定相同，一个用户只有一个基本组，且可被修改。

每个用户必属于一个基本组。

附加组：用户加入的其它组

> /etc/passwd 可以查看用户的基本组
>
> /etc/group可以查看用户的附加组。id命令也可以看

## 25.4 用户提权

> 1、切换到管理员身份
>
> ```shell
> # 切换到root用户，但环境变量依旧是当前用户的
> $ su root
> # 切换到root用户，同时变更工作目录以及环境变量
> $ su - root
> 
> # 变更为root并在执行ls命令后退出变回原使用者
> $ su -c ls root
> ```
>
> 
>
> 2、**sudo使普通用户具有完成部分特权指令的权力**
>
> sudo配置文件语法
>
> 用户 登录的主机 = （可以变换的身份）可以执行的命令
>
> user MACHINE=COMMANDS
>
> > vim /etc/sudoers
> >
> > %wheel ALL=(ALL)NOPASSWD:ALL #允许wheel用户组中的用户在不输入该用户的密码的情况下使用所有命令



# 26 alias

> alias rm='rm -i'	#给字符串取别名，rm相当于rm -i
>
> unalias rm	#取消别名







