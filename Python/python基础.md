# 1 数据类型

> ![image-20231003163930418](python基础.assets/image-20231003163930418.png)

## 1.1 数值类型

> ![image-20231003173936807](python基础.assets/image-20231003173936807.png)
>
> ![image-20231003174209115](python基础.assets/image-20231003174209115.png)
>
> ```python
> print(round(0.1 + 0.2), 2) # 保留两位小数
> ```
>
> ![image-20231006165351040](python基础.assets/image-20231006165351040.png)
>
> ```python
> x = 3 + 4j
> print('实数部分', x.real)
> print('虚数部分', x.imag)
> ```

## 1.2 字符串类型

> ![image-20231006165629286](python基础.assets/image-20231006165629286.png)
>
> ![image-20231006165751721](python基础.assets/image-20231006165751721.png)
>
> ![image-20231006165940857](python基础.assets/image-20231006165940857.png)
>
> ![image-20231006170245280](python基础.assets/image-20231006170245280.png)

## 1.3 布尔类型

> ![image-20231006170427610](python基础.assets/image-20231006170427610.png)

## 1.4 数据类型转换

> ![image-20231006172331604](python基础.assets/image-20231006172331604.png)
>
> ```python
> x = 10
> y = 3
> z = x / y	# 在执行除法运算结果为float类型
> print(z, type(z)) # 3.3333333333333 <class 'float'>
> print(bin(y)) # 0b11 转成二进制字符串
> 
> # float类型转int类型，只保留整数部分
> print(int(3.14)) # 3
> print(int(-3.14)) # -3
> print(int(-0.5)) # 0
> 
> # 将字符串str转成int或float类型报错的情况
> print(int('18a')) # ValueError
> print(int('3.14')) # ValueError
> 
> print(ord('杨')) # 26472
> print(chr(26472)) # '杨'
> ```

# 1.5 `eval()`函数

> ![image-20231006172524751](python基础.assets/image-20231006172524751.png)
>
> ```python
> age = eval(input('请输入您的年龄：')) # 相当于执行了int(age)操作
> eval('print("hello")')
> ```

## 1.6 算术运算符

> ![image-20231006201457070](python基础.assets/image-20231006201457070.png)
>
> ```python
> print(10 / 2) # 5.0 结果为float类型
> ```
>
> ![image-20231006201758517](python基础.assets/image-20231006201758517.png)
>
> ```python
> # python支持系列解包赋值
> a, b = 10, 20
> # 交换两个变量的值
> a, b = b, a
> ```

## 1.6 比较运算符

> ![image-20231006202319636](python基础.assets/image-20231006202319636.png)

# 1.7 逻辑运算符

> ![image-20231006202436247](python基础.assets/image-20231006202436247.png)
>
> ```python
> # 会发生短路现象，当第一个表达式为false，就不会去计算第二个表达式了，or同理
> print(8 < 7 and 10/0)
> ```

## 1.8 位运算

> ![image-20231006202827463](python基础.assets/image-20231006202827463.png)
>
> ![image-20231006202854083](python基础.assets/image-20231006202854083.png)
>
> ![image-20231006202915584](python基础.assets/image-20231006202915584.png)
>
> ![image-20231006202944763](python基础.assets/image-20231006202944763.png)
>
> ![image-20231006203014227](python基础.assets/image-20231006203014227.png)
>
> ![image-20231006203107681](python基础.assets/image-20231006203107681.png)
>
> **运算符优先级**
>
> ![image-20231006203153006](python基础.assets/image-20231006203153006.png)

#  2 python保留字

> ![image-20231003171537222](python基础.assets/image-20231003171537222.png)
>
> ```python
> import keyword
> print(keyword.kwlist)
> ```

# 3 python 标识符命名规范

> ![image-20231003171915217](python基础.assets/image-20231003171915217.png)
>
> ![image-20231003173129052](python基础.assets/image-20231003173129052.png)
>
> ![image-20231003173749354](python基础.assets/image-20231003173749354.png)

# 4 分支结构

## 4.1  `if`语句

> ```python
> number = eval(input("输入您的6位中奖号码: "))
> # if-else赋值
> result = '中奖了' if number == 987654 else '未中奖'
> ```

## 4.2 循环结构

> ![image-20231008174941629](python基础.assets/image-20231008174941629.png)
>
> 

# python中的引用

> [python中的引用_python 引用_愈努力俞幸运的博客-CSDN博客](https://blog.csdn.net/qq_37891604/article/details/124528827)

 