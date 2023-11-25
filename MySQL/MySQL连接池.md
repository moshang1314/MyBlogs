# 1 MySQL C API

## 1.1 初始化连接环境

> ```c++
> /*
> 参数 mysql -> null
> 返回值：该函数将分配、初始化、并返回新对象
> 通过返回的新对象去连接MySQL的服务器
> */
> MYSQL *mysql_init(MYSQL *mysql);
> ```

## 1.2 连接mysql服务器

> ```c++
> /*
> 返回值：
> 	成功：返回MYSQL*连接句柄，对于成功的连接，返回值与第1个参数的值相同。
> 	失败：返回NULL
> */
> MYSQL *mysql_real_connect(
> 	MYSQL *mysql,	//mysql_init()函数的返回值
> 	const char *host,	//mysql服务器的主机地址，写IP地址即可，(localhost, null)->代表本地连接
> 	const char *user,	//连接mysql服务器的用户名，默认：root
> 	const char *passwd,	//连接mysql服务器对应用户的密码
>     
> 	const char *db,	//要使用的数据库的名称
> 	unsigned int port,	//连接的mysql服务器监听的端口，如果为0，使用mysql默认的端口3306，不为0，则使用指定端口
> 	const char* unix_socket,	//本地套接字，不使用指定为null
> 	unsigned long client_flag	//通常指定为0
> );
> ```

## 1.3 执行sql语句

> ```c++
> //执行一个sql语句，添删查改的sql语句都可以
> int mysql_query(MYSQL *mysql, const char *query);
> /*
> 参数：
> 	- mysql：mysql_real_connect()的返回值
> 	- query：一个可以执行的sql语句，结尾的位置不需要加';'
> 返回值：
> 	- 查询成功，返回0.如果是查询，结果集在mysql对象中
> 	- 如果查询出现错误，返回非0值
> */
> ```

## 1.4 获取结果集

> ```c++
> // 将结果集从mysql（参数）对象中取出
> // MYSQL_RES 对应一块内存，里边存着这个查询之后得到的结果集
> // 如何将行和列的数据从结果集中取出，需要使用其他接口
> //返回值：具有多个结果的MYSQL_RES结果集合。如果出现错误，返回NULL
> MYSQL_RES *mysql_store_result(MYSQL *mysql);
> ```

## 1.5 得到结果集中的列数

> ```c++
> //从结果集中获得列的个数
> // 参数：调用mysql_store_result()得到的返回值
> //返回值： 结果集中的列数
> unsigned int mysql_num_fields(MYSQL_RES *result);
> ```

## 1.6 获取表头->列名（字段名）

> ```c++
> //参数： 调用mysql_store_result()得到的返回值
> //返回值：MYSQL_FIELD*指向一个结构体数组
> //通过该函数获得结果集中所有列的名字
> MYSQL_FIELD *mysql_fetch_fields(MYSQL_RES *result);
> ```
>
> **返回值`MYSQL_FIELD`对应的是一个结构体，在mysql.h中定义如下：**
>
> ```c++
> // mysql.h
> //结果集中的每—个列对应一个MYSQL_FIELD
> typedef struct st_mysql_field {
> 	char *name;				/* 列名→字段的名字 */
> 	char *org_name;			/* Original column name, if an alias */
> 	char *table;			/* Table of column if column was a field */
> 	char korg_table;		/* 0rg table name,if table was an alias */
> 	char *db;				/* Database for table */
> 	char *catalog;			/* catalog for table */
> 	char *def;				/* Default value (set by mysql_list_fields) */
> 	unsigned long length;	/* Width of column (create length) */
> 	unsigned long max_length;/*Max width for selected set */
> 	unsigned int name_Length;
> 	unsigned int org_name_length;
>     unsigned int table_length ;
>     unsigned int org_table_length;
>     unsigned int db_length;
> 	unsigned int catalog_Length;
>     unsigned int def_length;
>     unsigned int flags;			/* Div flags */
> 	unsigned int decimals;		/*Number of decimals in field */
> 	unsigned int charsetnr;		/*character set */
> 	enum enum_field_types type;/* Type of field. See mysql_com.h for types */
>     void *extension;
> } MYSQL_FIELD;
> 
> //eg:
> //得到存储头信息的数组的地址
> MYSQL_FIELD* fields = mysql_fetch_fields(res);
> //得到列数
> int num = mysql_num_fields(res);
> //遍历得到的每一列的列名
> for(int i = 0; i < num; ++i){
>     printf("当前列的名字：%s\n", fields[i].name);
> }
> ```

## 1.7 得到结果集中字段的长度

> ```c++
> /*
> 返回结果集内当前行的列的长度
> 	1.如果打算复制字段值，使用该函数能避免调用strlen()
> 	2.如果结果集包含二进制，必须使用该函数来确定数据的大小，原因在于，对于包含NULL字符的任何字段，strlen()函数将停止计数
> */
> unsigned long *mysql_fetch_length(MYSQL_RES *result);
> /*
> 参数：
> 	-result：通过查询得到的结果集
> 返回值：
> 	- 无符号长整数的数组表示各列的大小。如果出现错误，返回NULL
> */
> 
> //eg:
> MYSQL_ROW row;
> unsigned long *lengths;
> unsigned int num_fields;
> 
> row = mysql_fetch_row(result);
> if(row){
>     num_fields = mysql_num_fields(result);
>     lengths = mysql_fetch_length(result);
>     for(int i = 0; i < num_fields; i++){
>         printf("Column %u is %lu bytes in length.\n", i, lengths[i]);
>     }
> }
> ```

## 1.8 遍历结果集

> ```c++
> typedef char** MYSQL_ROW;
> // 遍历结果集的下一行
> // 如果想遍历整个结果集，需要对该函数进行循环调用
> // 返回值是二级指针，char**
> //	-- 指向一个指针数组，类型是数组，里边的每一个元素都是指针，char*类型
> //	-- char* [];数组中的字符串对应的一列数据
> 
> //需要对MYSQL_ROW遍历就可以得到每一列的值
> // 如果要遍历整个结果集，需要循环调用这个函数
> MYSQL_ROW mysql_fetch_row(MYSQL_RES *result);
> /* 
> 参数：
> 	- result: 通过查询得到的结果集
> 返回值：
> 	- 成功：得到当前记录中每个字段的值
> 	- 失败：NULL，说明数据已经读完了
> */
> ```

## 1.9 资源释放

> ```c++
> //释放结果集
> void mysql_free_result(MYSQL_RES *result);
> 
> //关闭mysql实例
> void mysql_close(MYSQL *mysql);
> ```

## 1.10 字符编码

> ```c++
> // 获取api默认使用的字符编码
> // 为当前连接返回的默认的字符集
> const char *mysql_character_set_name(MYSQL *mysql);
> //返回值：默认字符集
> 
> //设置api使用的字符集
> // 第二个参数 csname 就是要设置的字符集 -> 支持中文：utf8
> int mysql_set_character_set(MYSQL *mysql, char *csname);
> ```

## 1.11 事务操作

> ```c++
> // mysql中默认会进行事务的提交
> 
> //因为自动提交事务，会对我们的操作造成影响
> 
> //如果我们操作的步骤比较多，集合的开始和结束需要用户自己去设置，需要改为手动方式提交事务
> my_bool mysql_autocommit(MYSQL *mysql, my_bool mode);
> /*
> 参数：
> 	如果模式为“1”，启用autocommit模式；如果模式为“0”，禁止autocommit模式。
> 返回值：
> 	如果成功，返回0，如果出现错误，返回非0值。
> */
> 
> //事务提交
> my_bool mysql_commit(MYSQL *mysql);
> /*
> 返回值：
> 	成功：0
> 	失败：非0
> */
> 
> //数据回滚
> my_bool mysql_roolback(MYSQL *mysql);
> /*
> 返回值：
> 	成功：0
> 	失败：非0
> */
> ```

## 1.12 打印错误信息

> ```c++
> //返回错误的描述
> const char *mysql_error(MYSQL *mysql);
> //返回错误的编码
> unsigned int mysql_errno(MYSQL *mysql);
> ```

# 2 MySQL API的封装

> 
>