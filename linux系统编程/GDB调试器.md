# 1 GDB

## 1.1 常用调试命令

> 调试开始：执行`gdb [filename]`，进入gdb调试程序，其中exefilename为要调试的可执行文件名
>
> ```bash
> ## 以下命令后括号内为命令的简化使用，比如run(r)，直接输入命令r就代表命令run
> 
> $(gdb) help(h)	# 查看命令帮助，具体命令查询在gdb中输入help + 命令
> 
> $(gdb) run(r)	# 重新开始运行文件（run-text: 加载文本文件，run-bin：加载二进制文件）
> 
> $(gdb) start	# 单步执行，运行程序，停在第一行执行语句
> 
> $(gdb) list(l)	# 查看原代码（list-n，从第n行开始查看代码。list + 函数名：查看具体函数）
> 
> $
> ```
>
> 