# 1 正则表达式

> * 正则表达式是一套规则和方法
> * 正则表达式以单位进行，一次处理一行
> * 正则表达式在linux命令中仅三剑客（sed、awk和grep）支持，其它命令无法使用正则表达式。
> * 通配符是大部分普通命令都支持的，用于查找文件或目录，而正则表达式是通过三剑客命令在文件（数据流）中过滤内容的

## 1.1 linux三剑客

> 文本处理工具，均支持正则表达式引擎
>
> * grep：文本过滤工具，（模式：pattern）工具
> * sed：stream editor，流编辑器；
> * awk：Linux的文本报告生成器（格式化文本），Linux是gawk

## 1.2 基本正则表达式BRE集合

|   符号   |                             作用                             |
| :------: | :----------------------------------------------------------: |
|    ^     |    用于模式的最左侧，如"^oldboy"，匹配oldboy单词开头的行     |
|    \$    | 用于匹配模式的最右侧，如"oldboy\$"，匹配以oldboy单词结尾的行 |
|    ^$    |          组合符，表示空行，以空字符开头，空字符结尾          |
|    .     |           匹配任意一个且只有一个字符，不能匹配空行           |
|    \     | 转义字符，让特殊含义的字符现出原形，还原本意，如`\.`，代表小数点 |
|    *     | 匹配前一个字符（连续出现）0次或1次以上，重复0次代表空，即匹配所有内容 |
|    .*    |                     组合符，匹配所有内容                     |
|  [abc]   |        匹配[]集合内的任意一个字符，如[1-3]，[01-100]         |
| [`^abc`] |                   匹配除了^后面的任意字符                    |
|          |                                                              |

## 1.3 扩展正则表达式ERE集合

> 扩展正则必须用 grep -E 才能生效

|  字符  |                 作用                 |
| :----: | :----------------------------------: |
|   +    |       匹配前一个字符1次或多次        |
| [:/]+  | 匹配括号内的":"或者"/"字符1次或多次  |
|   ?    |        匹配前一个字符0次或1次        |
|   \|   |     表示或者，同时过滤多个字符串     |
|   ()   | 分组过滤，被括起来的内容表示一个整体 |
| a{n,m} |    匹配前一个字符最少n次，最多m次    |
| a{n,}  |        匹配前一个字符最少n次         |
|  a{n}  |        匹配前一个字符正好n次         |
| a{,m}  |        匹配前一个字符最多m次         |
|        |                                      |

# 2 grep

> 全拼：Global Search REgular expression and Print out the line
>
> 作用：文本搜索工具，根据用户指定的模式（过滤条件）对目标文件逐行进行匹配检查，打印匹配的行
>
> 模式：由正则表达式的元字符及文本字符所编写出的过滤条件；
>
> ```tex
> 语法：
> grep [option] [pattern] file
> 命令  参数		匹配模式 文件数据
> 		-i：ignorecase，忽略字符的大小写
> 		-o：仅显示匹配到的字符串本身
> 		-v, --invert-match：显示不能被模式匹配到的行
> 		-E：支持使用扩展的正则表达式元字符
> 		-q,--quiet,--silent：静默模式，即不输出任何信息；
> ```
>
> grep命令是Linux系统中最重要的命令之一，功能是从文本文件 或 管道数据流中筛选匹配的行和数据，如果再配合正则表达式，功能十分强大，是Linux运维人员必备的命令。
>
> grep命令里的匹配模式就是你想要找的内容，可以是普通的文字符号，也可以是正则表达式
>
> |  参数选项   |                    解释说明                     |
> | :---------: | :---------------------------------------------: |
> |     -v      |         排除匹配结果，输出不匹配的内容          |
> |     -n      |                显示匹配行与行号                 |
> |     -i      |                  不区分大小写                   |
> |     -c      |                只统计匹配的行数                 |
> |     -E      | 使用egrep命令，可以使用扩展的正则表达式的元字符 |
> | –color=auto |             为grep过滤结果添加颜色              |
> |     -w      |                只匹配过滤的单词                 |
> |     -o      |            只输出匹配到的字符串本身             |
> |             |                                                 |
>
> 例如：
>
> ```shell
> # 查找空行所在行的行
> grep -n "^$" test.txt
> 
> # 查找非注释行
> grep -v "^#" test.sh
> 
> # 查找以.结尾的行
> grep "\.$" test.txt
> 
> # 找出所有允许登录的用户，解释器是/bin/bash
> grep -n "/bin/bash$" /etc/passwd
> 
> grep -E "g(oo|la)d" test.txt
> ```
>
> 注意：
>
> > 在Linux平台下，所有文件的每行结尾都有一个$符
> >
> > 可以用cat -An 或cat -En查看
>
> **()小括号**：
>
> 将一个或多个字符捆在一起，当作一个整体处理；
>
> * 小括号功能之一是分组过滤被括起来的内容，括号内的内容表示一个整体
> * 括号()内的内容可以被后面的"`\n`"正则引用，n为数字，表示引用第几个括号的内容
>   * `\1`：表示从左侧起，第一个括号中的模式所匹配到的字符
>   * `\2`：表示从左侧起，第二个括号中的模式所匹配到的字符
>
> 例如：
>
> ```shell
> # 分组过滤
> $ grep -E "g(oo|la)d" test.txt
> 
> $ cat lovers.txt
> I like my lover.
> I love my lover.
> He likes his lovers
> He love his lovers
> 
> $ grep -E "(l..e).*\1" lovers.txt
> 
> I love my lover.
> He love his lovers
> ```

# 3 sed

> 注意：**sed和awk使用单引号，双引号有特殊解释**
>
> sed是Stream Editor（字符流编辑器）的缩写，简称流编辑器。
>
> sed是操作、过滤和转换文本内容的强大工具。
>
> 常用功能包括结合正则表达式对文件实现快速增删改查，其中查询的功能中最常用的两大功能是过滤（过滤指定字符串）、取行（取出指定行）。
>
> ![image-20221017110531643](https://cdn.jsdelivr.net/gh/moshang1314/myBlog@main/image/image-20221017110531643.png)

> **语法：**
>
> ```shell
> sed [选项] [sed内置命令字符] [输入文件]
> ```
>
> **选项：**
>
> | 参数选项 |                        解释                         |
> | :------: | :-------------------------------------------------: |
> |    -n    |      取消默认sed的输出，常与sed内置命令p一起用      |
> |    -i    | 直接将修改结果写入文件，不用-i，sed修改的是内存数据 |
> |    -e    |               多次编辑，不需要管道符                |
> |    -r    |                 支持正则表达式扩展                  |
>
> sed的内置命令字符用于对文件进行不同的操作功能，如对文件增删改查
>
> **sed常用的内置命令字符：**
>
> |  sed的内置命令字符  |                           解释                            |
> | :-----------------: | :-------------------------------------------------------: |
> |          a          |    append，对文本追加，在指定行后面添加一行或多行文本     |
> |          d          |                    delete，删除匹配行                     |
> |          i          |    insert，表示插入文本，在指定行前添加一行或多行文本     |
> |          p          |         print，打印匹配行的内容，通常p与-n一起用          |
> | `s/正则/替换内容/g` | 匹配正则内容，然后替换内容（支持正则），结尾g代表全局匹配 |
>
> **sed匹配范围：**
>
> |    范围     |                             解释                             |
> | :---------: | :----------------------------------------------------------: |
> |   空地址    |                     全文处理，如"a `\n`"                     |
> |   单地址    |                 指定文件某一行，如 "3a `\n`"                 |
> | `/pattern/` |                     被模式匹配到的每一行                     |
> |  范围区间   | 10,20 十到二十行，10,+5第10行向下5行，`/pattern1/, /pattern2/`多个匹配模式 |
> |    步长     |  1~2，表示1、3、5、7、9行，2~2两个步长，表示2、4、6、8、10   |

## 3.1 sed案例

> 准备测试数据
>
> ```shell
> $ cat -n luffycity.txt
> My name is chaoge.
> I teach linux.
> I like play computer game.
> My qq is 1992256738
> My website is http://pythonav.cn.
> ```
>
> **输出文件指定内容**
>
> ```shell
> # 输出文件的2-3行
> $ sed "2,3p" luffycity.txt
> My name is chaoge.
> I teach linux.
> I like play computer game.
> My qq is 1992256738
> My website is http://pythonav.cn.
> 
> # 使用-n参数取消输出不匹配的行
> $ sed -n "2,3p" luffycity.txt
> I teach linux.
> I like play computer game.
> 
> # 输出第2行起始的后三行总共四行
> $ sed -n "2,+3p" luffycity.txt
> I teach linux.
> I like play computer game.
> My qq is 1992256738
> My website is http://pythonav.cn.
> ```
>
> **过滤**
>
> ```shell
> # 过滤出含linux的行
> $ sed "/linux/p" luffycity.txt -n
> I teach linux.
> 
> # 删除含game的行，但不更新至文件，加上-i可删除文件中的内容，即更新至文件
> $ sed "/game/d" luffycity.txt
> My name is chaoge.
> I teach linux.
> My qq is 1992256738
> My website is http://pythonav.cn.
> 
> # 删除第2行至结尾，只是删除读取到内存中的数据，没有更新到文件
> $ sed "2,$d" luffycity.txt
> My name is chaoge.
> ```
>
> **替换**
>
> ```shell
> # 将文件中的My全部替换为His
> $ sed 's/My/His/g' luffycity.txt
> His name is chaoge.
> I teach linux.
> I like play computer game.
> His qq is 1992256738
> His website is http://pythonav.cn.
> 
> # 多次编辑
> $ sed -e "s/My/His/g" -e "s/1992256738/100/g" luffycity.txt
> His name is chaoge.
> I teach linux.
> I like play computer game.
> His qq is 100
> His website is http://pythonav.cn.
> ```
>
> **追加**
>
> ```shell
> # 在第2行后追加一行信息
> $ sed "2a My linux is good." luffycity.txt
> My name is chaoge.
> I teach linux.
> My linux is good
> I like play computer game.
> My qq is 1992256738
> My website is http://pythonav.cn.
> 
> #在第4行上面插入一行信息
> $ sed "4i My telphone is 134." luffycity.txt
> My name is chaoge.
> I teach linux.
> I like play computer game.
> My telphone is 134.
> My qq is 1992256738
> My website is http://pythonav.cn.
> 
> # 追加多行数据
> $ sed "3a I like girl.\nI like pretty girl." luffycity.txt
> My name is chaoge.
> I teach linux.
> I like play computer game.
> I like girl.
> I like pretty girl.
> My qq is 1992256738
> My website is http://pythonav.cn.
> 
> # 对每一行后面都追加
> $ sed "a ---------------" luffycity.txt
> My name is chaoge.
> ---------------
> I teach linux.
> ---------------
> I like play computer game.
> ---------------
> My qq is 1992256738
> ---------------
> 
> My website is http://pythonav.cn.
> ---------------
> ```

## 3.2 sed 配合正则表达式企业案例

> ```shell
> # 取出Linux的IP地址
> $ ifconfig ens33
> ens33: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
>         inet 10.0.0.204  netmask 255.255.255.0  broadcast 10.0.0.255
>         inet6 fe80::c2d9:2de0:7fa6:36ff  prefixlen 64  scopeid 0x20<link>
>         ether 00:0c:29:1f:88:5b  txqueuelen 1000  (Ethernet)
>         RX packets 675660  bytes 722736348 (689.2 MiB)
>         RX errors 0  dropped 0  overruns 0  frame 0
>         TX packets 338248  bytes 20533153 (19.5 MiB)
>         TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0
> 
> $ ifconfig ens33 | sde "2p" -n
> inet 10.0.0.204  netmask 255.255.255.0  broadcast 10.0.0.255
> 
> $ ifconfig ens33 | sde "2p" -n | sed "s/^.*inet//"
> 10.0.0.204  netmask 255.255.255.0  broadcast 10.0.0.255
> 
> $ ifconfig ens33 | sde "2p" -n | sed "s/^.*inet//" | sed "s/net.*$//"
> 10.0.0.204  
> # 多次编辑
> $ ifconfig ens33 | sed -e "2s/^.*inet//" -e "2s/net.*$//p" -n
> ```

# 4 awk

> awk是一个强大的linux命令，有强大的文本格式化的能力，好比将一些文本数据格式化成专业的excel表的样式
>
> awk早期在Unix上实现，我们用的awk是gawk，是GUN awk的意思
>
> awk更是一门编程语言，支持条件判断、数组、循环等功能

> **awk语法：**
>
> ```shell
> awk [option] 'pattern[{action}]' file ...
> 
> awk 参数	'条件动作'	文件
> ```
>
> Action指的是动作，awk擅长文本格式化，且输出格式化后的结果，因此最常用的动作就是`print`和`printf`
>
> 默认模式是所有行都执行动作，默认动作是print{$0}，打印完整行
>
> **awk参数：**
>
> | 参数 |            解释             |
> | :--: | :-------------------------: |
> |  -F  |       指定分隔符字段        |
> |  -v  | 定义或修改一个awk内部的变量 |
> |  -f  |   从脚本文件中读取awk命令   |
>
> 

## 4.1 awk使用场景

> 准备测试文本：
>
> ```shell
> $ cat alex.txt
> alex1 alex2 alex3 alex4 alex5
> alex6 alex7 alex8 alex9 alex10
> alex11 alex12 alex13 alex14 alex15
> alex16 alex17 alex18 alex19 alex20
> alex21 alex22 alex23 alex24 alex25
> alex26 alex27 alex28 alex29 alex30
> alex31 alex32 alex33 alex34 alex35
> alex36 alex37 alex38 alex39 alex40
> alex41 alex42 alex43 alex44 alex45
> alex46 alex47 alex48 alex49 alex50
> ```
>
> 打印数据：
>
> > awk默认以空格为分隔符，且多个空格也识别为一个空格，作为分隔符
> >
> > awk是按行处理文件，一行处理完毕，处理下一行，工具用户指定的分隔符去工作，没有指定则默认空格
> >
> > **awk指定了分隔符后，awk把每一行切割后的数据赋给对应的内置变量。**
> >
> > 所以，$0代表完整数据，$1代表第1列数据，$2代表第2列数据，以此类推……
> >
> > 倒数第二列可以写成$(NF-1)
>
> **awk内置变量：**
>
> |       内置变量        |               解释                |
> | :-------------------: | :-------------------------------: |
> |          $n           | 指定分隔符后，当前记录的第n个字段 |
> |          $0           |          完整的输入记录           |
> |          FS           |      字段分隔符，默认是空格       |
> |          OFS          |            输出分隔符             |
> | NF(Number of fields)  |  分隔后，当前行一共有多少个字段   |
> | NR(Number of records) |         当前记录数，行数          |

## 4.2 自定义输出内容

> awk，必须外层单引号，内层双引号。
>
> 内置变量$1、$2都不得添加双引号，否则会识别成文本。
>
> ```shell
> $ awk '{print "第一列："$1,"第二列："$3}' 1.txt
> 第一列：lex1 第二列：alex3
> 第一列：alex6 第二列：alex8
> 第一列：alex11 第二列：alex13
> 第一列：alex16 第二列：alex18
> 第一列：alex21 第二列：alex23
> 第一列：alex26 第二列：alex28
> 第一列：alex31 第二列：alex33
> 第一列：alex36 第二列：alex38
> 第一列：alex41 第二列：alex43
> 第一列：alex46 第二列：alex48
> ```
>
> **按行输出内容：**
>
> ```shell
> # 注意NR在awk中表示当前处理的行号，NR==5表示行号是5的那一行
> # 注意一个等于号，是修改变量的值的意思，两个等于号是关系运算符，是“等于”的意思
> # 输出文件第5行信息
> $ awk 'NR==5' /etc/passwd
> lp:x:4:7:lp:/var/spool/lpd:/sbin/nologin
> 
> # 输出第5行第2列信息
> $ awk 'NR==5{print $2}' alex.txt
> alex22
> 
> # 输出第5行完整信息
> $ awk 'NR==5{print $0}' alex.txt
> alex21 alex22 alex23 alex24 alex25
> # 输出2-5行完整信息
> $ awk 'NR==2,NR==5{print $0}'
> alex21 alex22 alex23 alex24 alex25
> alex26 alex27 alex28 alex29 alex30
> alex31 alex32 alex33 alex34 alex35
> 
> # 给每行添加行号
> $ awk '{print NR,$0}' alex.txt
> 1 lex1 alex2 alex3 alex4 alex5
> 2 alex6 alex7 alex8 alex9 alex10
> 3 alex11 alex12 alex13 alex14 alex15
> 4 alex16 alex17 alex18 alex19 alex20
> 5 alex21 alex22 alex23 alex24 alex25
> 6 alex26 alex27 alex28 alex29 alex30
> 7 alex31 alex32 alex33 alex34 alex35
> 8 alex36 alex37 alex38 alex39 alex40
> 9 alex41 alex42 alex43 alex44 alex45
> 10 alex46 alex47 alex48 alex49 alex50
> 
> # 打印倒数第二列
> awk '{print $(NF-1)}'
> alex4
> alex9
> alex14
> alex19
> alex24
> alex29
> alex34
> alex39
> alex44
> alex49
> 
> # 抓取ip
> ifconfig ens33 | awk 'NR==2{print $2}'
> 10.0.0.204
> ```

## 4.3 分隔符

> awk分隔符有两种
>
> * 输入分隔符，awk默认是空格，空白字符，英文是field separator，变量名是FS
> * 输出分隔符，output field separator，简称OFS
>
> **FS输入分隔符**
>
> awk逐行处理文本的时候，以输入分隔符为准，把文本切成多个片段，默认符号是空格
>
> 当我们处理特殊文件，没有空格的时候，可以自由指定分隔符特点
>
> ```shell
> # 取/etc/passwd文件第一列信息
> $ awk -F ":" '{print $1}' /etc/passwd
> # 或
> $ awk -v FS=":" '{print $1}'
> 
> # 设置输出分隔符, -v修改awk变量
> $ awk -F ":" -v OFS="===" '{print $1, $NF}' /etc/passwd
> 
> $ awk -F ":" -v OFS="\t" '{print $1, $NF}' /etc/passwd
> ```

## 4.4 awk 变量

> 对于awk而言，变量分为：
>
> * 内置变量
> * 自定义变量
>
> | 内置变量 | 解释                                                         |
> | :------- | ------------------------------------------------------------ |
> | FS       | 输入字段分隔符，默认为空白字符                               |
> | OFS      | 输出字段分隔符，默认为空白字符                               |
> | RS       | 输入记录分隔符（输入换行符），指定输入时的换行符             |
> | ORS      | 输出记录分隔符（输出换行符），输出是用指定符号代替换行符     |
> | NF       | NF：number of Field，当前行的字段的个数（即当前行被分割成了几列），字段数量 |
> | NR       | NR：行号，当前处理的文本行的行号                             |
> | FNR      | FNR：各文件分别计数的行号                                    |
> | FILENAME | FILENAME：当前文件名                                         |
> | ARGC     | ARGC：命令行参数个数                                         |
> | ARGV     | ARGV：数组，保存的是命令行所给定的各参数                     |
>
> **内置变量**
>
> > * awk的内置变量NR、NF和FNR等各变量不用添加$符号
> > * 而$0、$1、$2……是需要添加$符号的
>
> **处理多个文件显示行号：**
>
> ```shell
> # 普通的NR的变量，会将多个文件按照顺序排序,相当于组合成一个文件
> $ awk '{print NR, $0}' alex.txt pwd.txt
> 
> # 使用FNR变量，可以分别对文件行数计数
> $ awk '{print FNR, $0}' alex.txt pwd.txt
> ```
>
> **修改记录分隔符：**
>
> ```shell
> # 修改以空白符为记录分隔符
> $ awk -v RS=' ' '{print NR, $0}' alex.txt
> 
> # 修改输出记录分隔符为-------
> $ awk -v ORS='---------' '{print NR, $0}' alex.txt
> 
> # 每个记录前显示文件名
> $ awk '{print FILENAME, $0}' alex.txt
> ```
>
> **变量ARGC、ARGV：**
>
> > ARGV表示的是一个数组，数组中保存的是命令行所给的参数
> >
> > 数组是一种数据类型
> >
> > ```shell
> > # BEGIN 表示不操作文件，直接输出，即在处理文件前输出
> > $ awk 'BEGIN{print "666"} {print $0}' alex.txt
> > 666
> > lex1 alex2 alex3 alex4 alex5
> > alex6 alex7 alex8 alex9 alex10
> > alex11 alex12 alex13 alex14 alex15
> > alex16 alex17 alex18 alex19 alex20
> > alex21 alex22 alex23 alex24 alex25
> > alex26 alex27 alex28 alex29 alex30
> > alex31 alex32 alex33 alex34 alex35
> > alex36 alex37 alex38 alex39 alex40
> > alex41 alex42 alex43 alex44 alex45
> > alex46 alex47 alex48 alex49 alex50
> > 
> > $ awk 'BEGIN{print "666"} {print ARGV[0],ARGV[1],$0}' 1.txt
> > 666
> > awk 1.txt lex1 alex2 alex3 alex4 alex5
> > awk 1.txt alex6 alex7 alex8 alex9 alex10
> > awk 1.txt alex11 alex12 alex13 alex14 alex15
> > awk 1.txt alex16 alex17 alex18 alex19 alex20
> > awk 1.txt alex21 alex22 alex23 alex24 alex25
> > awk 1.txt alex26 alex27 alex28 alex29 alex30
> > awk 1.txt alex31 alex32 alex33 alex34 alex35
> > awk 1.txt alex36 alex37 alex38 alex39 alex40
> > awk 1.txt alex41 alex42 alex43 alex44 alex45
> > awk 1.txt alex46 alex47 alex48 alex49 alex50
> > 
> > $ awk '{print "666"} {print ARGV[0],ARGV[1],$0}' 1.txt 
> > 666
> > awk 1.txt lex1 alex2 alex3 alex4 alex5
> > 666
> > awk 1.txt alex6 alex7 alex8 alex9 alex10
> > 666
> > awk 1.txt alex11 alex12 alex13 alex14 alex15
> > 666
> > awk 1.txt alex16 alex17 alex18 alex19 alex20
> > 666
> > awk 1.txt alex21 alex22 alex23 alex24 alex25
> > 666
> > awk 1.txt alex26 alex27 alex28 alex29 alex30
> > 666
> > awk 1.txt alex31 alex32 alex33 alex34 alex35
> > 666
> > awk 1.txt alex36 alex37 alex38 alex39 alex40
> > 666
> > awk 1.txt alex41 alex42 alex43 alex44 alex45
> > 666
> > awk 1.txt alex46 alex47 alex48 alex49 alex50
> > ```
> >
> > **自定义变量：**
> >
> > ```shell
> > $ awk -v myname="lqs" 'BEGIN{print "我的名字是?", myname}'
> > 我的名字是? lqs
> > ```

## 4.5 awk格式化

> print只能对文本简单的输出，并不能美化或者修改格式
>
> **printf和print的区别：**
>
> > 1) 其与print最大的不同是，printf需要指定format
> > 2) format用于指定后面每个item的输出格式
> > 3) printf语句不会自动打印换行符
> >
> > format格式的指示符都以%开头，后跟一个字符：如下
> >
> > %c：显示字符的ASCII码
> >
> > %d，%i：十进制整数
> >
> > %e，%E：科学计数法显示数值
> >
> > %f：显示浮点数
> >
> > %g，%G：以科学计数法的格式或浮点数格式显示数值
> >
> > %s：显示字符串
> >
> > %u：无符号整数
> >
> > %%：显示%自身
> >
> > printf修饰符：
> >
> > -：左对齐，默认右对齐
> >
> > +：显示数值符号，printf "%+d"
> >
> > ```shell
> > $ awk '{printf "%s\n", $0}' alex.txt
> > 
> > # linux中的printf命令
> > $ printf "%s--\n" a b c
> > a--
> > b--
> > c--
> > # 不匹配的项不会输出，如3
> > awk 'BEGIN{printf "%d\n%d\n", 1, 2, 3}'
> > 1
> > 2
> > 
> > $  awk 'NR==1,NR==3{printf "第一列：%s  第二列：%s  第三列: %s\n", $1, $2, $3}' 1.txt
> > 第一列：lex1  第二列：alex2  第三列: alex3
> > 第一列：alex6  第二列：alex7  第三列: alex8
> > 第一列：alex11  第二列：alex12  第三列: alex13
> > 
> > ```

### 4.5.1测试案例

```shell
# 制作测试文件
$ awk 'NR==1,NR==6{print $0}' /etc/passwd >> pwd.txt
$ cat pwd.txt
root:x:0:0:root:/root:/bin/bash
bin:x:1:1:bin:/bin:/sbin/nologin
daemon:x:2:2:daemon:/sbin:/sbin/nologin
adm:x:3:4:adm:/var/adm:/sbin/nologin
lp:x:4:7:lp:/var/spool/lpd:/sbin/nologin
sync:x:5:0:sync:/sbin:/bin/sync

$ awk -F ":" 'BEGIN{printf "%-20s\t %-20s\t %-20s\t %-20s\t %-20s\t %-20s\t %-20s\n", "用户名", "密码", "UID", "GID", "用户注释", "用户家目录", "用户使用的解释器"} {printf "%-20s\t %-20s\t %-20s\t %-20s\t %-20s\t %-20s\t %-20s\n", $1, $2, $3, $4, $5, $6, $7}' pwd.txt
```

## 4.6 awk模式

> awk是按行处理文本，如果不指定任何模式（条件），awk默认一行行处理，如果指定了模式，只有符合模式的才会被处理。
>
> 特殊模式，BEGIN和END模式:
>
> * BEGIN模式是处理文本之前需要执行的操作
> * END模式是处理完所有行之后执行的操作
>
> ```shell
> $ awk 'BEGIN{print "处理文本之前"}{print $0}END{print "处理文本之后"}' akex.txt
> 处理文本之前
> lex1 alex2 alex3 alex4 alex5
> alex6 alex7 alex8 alex9 alex10
> alex11 alex12 alex13 alex14 alex15
> alex16 alex17 alex18 alex19 alex20
> alex21 alex22 alex23 alex24 alex25
> alex26 alex27 alex28 alex29 alex30
> alex31 alex32 alex33 alex34 alex35
> alex36 alex37 alex38 alex39 alex40
> alex41 alex42 alex43 alex44 alex45
> alex46 alex47 alex48 alex49 alex50
> 处理文本之后
> 
> ```
>
> awk的模式
>
> | 关系运算符 | 解释             | 示例              |
> | ---------- | ---------------- | ----------------- |
> | <          | 小于             | x<y               |
> | <=         | 小于等于         | x<=y              |
> | ==         | 等于             | x==y              |
> | !=         | 不等于           | x!=y              |
> | >=         | 大于等于         | x>=y              |
> | >          | 大于             | x>y               |
> | ~          | 匹配正则表达式   | `x~/正则表达式/`  |
> | !~         | 不匹配正则表达式 | `x!~/正则表达式/` |
>
> > * 空模式，没有指定任何的模式（条件），因此每一行都执行了对应的动作，空模式会匹配文档的每一行，每一行都满足了（空模式）
> >
> > * 关系运算符匹配模式
> >
> >   * ```shell
> >     $ awk 'NSR==2, NR==4{print $0}' alex.txt
> >     ```
> >
> > * 正则表达式匹配模式，awk '`/正则表达式/动作`' /etc/passwd
> >
> >   * ```shell
> >     # 打印以games开头的行的第一列和最后一列
> >     $ awk -F ":" '/^games/{print $1, $NF}' pwd.txt
> >             
> >     # 多个模式匹配用逗号隔开，表示从匹配前一个模式的后到匹配后一个模式的行的所有行
> >     $ awk -F ":" '/^games/,/^music/{print $1, $NF}' pwd.txt
> >     ```

