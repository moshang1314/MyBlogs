# 1 函数

## 1.1 字符串函数

> 函数是指一段可以被另一段程序调用的程序或代码。
>
> ![image-20230613144056108](https://pic1.xuehuaimg.com/proxy/https://cdn.jsdelivr.net/gh/moshang1314/myBlog@main/image/image-20230613144056108.png)
>
> ```sql
> # 字符串拼接
> select concat('hello', 'world!');
> 
> # 字符串转为小写
> select lower('Hello');
> 
> select concat(2, 3); # 23
> 
> # 字符串左填充
> select lpad('01', 5, '-');
> 
> # 由于业务需求变更，企业员工的工号，统一为5位数，目前不足5位数的全部在前面补0.比如1号员工的工号应该为00001
> update emp set workno = lpad(workno, 5, '0');
> ```

# 1.2 数值函数

> ![image-20230613145322615](https://pic1.xuehuaimg.com/proxy/https://cdn.jsdelivr.net/gh/moshang1314/myBlog@main/image/image-20230613145322615.png)
>
> ```sql
> select ceil(1.1); # 2
> 
> select floor(1.9); # 1
> 
> select mod(7,4);  # 3
> 
> select rand();
> 
> select round(2.345, 2) # 2.35
> 
> # 生成一个六位数的随机验证码
> select lpad(round(rand() * 100000, 0), 6, '0');
> ```

# 1.3 日期函数

> ![image-20230613150353403](https://pic1.xuehuaimg.com/proxy/https://cdn.jsdelivr.net/gh/moshang1314/myBlog@main/image/image-20230613150353403.png)
>
> ```sql
> # 当前日期
> select curdate();
> 
> # 当前时间
> select curtime();
> 
> # 当前日期时间
> select now();
> 
> # 获取年份
> select year(now());
> select year(curdate());
> 
> # 指定日期往后推70天
> select date_add(now(), interval 70 day);
> # 后推 70 月
> select date_add(now(), interval 70 month);
> # 后推 70 年
> select date_add(now(), interval 70 year);
> 
> # 求取日期差值
> select datediff('2021-12-01', '2021-10-01');
> 
> # 查询所有员工入职天数，并根据入职天数倒序排序
> select name, datediff(curdate(), entrydate) as 'entrydays' from emp order by entrydays desc;
> ```

# 1.4 流程函数

> ![image-20230613151453228](https://pic1.xuehuaimg.com/proxy/https://cdn.jsdelivr.net/gh/moshang1314/myBlog@main/image/image-20230613151453228.png)
>
> ```sql
> select if(true, 'ok', 'error'); # 'ok'
> 
> select ifnull('ok', 'default'); # 'ok'
> select ifnull('', 'default'); # ''
> select ifnull(null, 'default'); # 'default'
> 
> # 查询emp表的员工姓名，和工作地址（北京/上海 -----> 一线城市，其他 -----> 二线城市)
> select 
> 	name, 
> 	(case workaddr when '北京' then '一线城市' when '上海' then '一线城市' else '二线城市' end) as '工作地址'
> from emp;
> 
> # 统计班级各个学员的成绩，展示的规则如下：
> --- > 85，展示优秀
> --- > 60，展示及格
> --- > 否则，展示不及格
> 
> select 
> id,
> name,
> (case when math >= 85 then '优秀' when math >= 60 then '及格' else '不及格' end) 'math',
> (case when english >= 85 then '优秀' when english >= 60 then '及格' else '不及格' end) 'english',
> (case when chinese >= 85 then '优秀' when chinese >= 60 then '及格' else '不及格' end) 'chinese'
> from score
> ```

# 2 约束

> 概念：约束是作用于表中字段上的规则，用于限制存储在表中的数据。
>
> 目的：保证数据库的正确、有效性和完整性。
>
> ![image-20230613153640392](https://pic1.xuehuaimg.com/proxy/https://cdn.jsdelivr.net/gh/moshang1314/myBlog@main/image/image-20230613153640392.png)
>
> ![image-20230613153903448](https://pic1.xuehuaimg.com/proxy/https://cdn.jsdelivr.net/gh/moshang1314/myBlog@main/image/image-20230613153903448.png)
>
> ```sql
> create table user(
> 	id int primary key auto_increment comment '主键',
>     name varchar(10) not null unique comment '姓名',
>     age int check(age > 0 and age <= 120) comment '年龄',
>     status char(1) default '1' comment '状态',
>     gender char(1) comment '性别'
> ) comment '用户表';
> ```

## 2.1 外键约束

> 外键约束
>
> 外键用来让两张表格的数据之间建立连接，从而保证数据的一致性和完整性。
>
> ![image-20230613155905040](https://pic1.xuehuaimg.com/proxy/https://cdn.jsdelivr.net/gh/moshang1314/myBlog@main/image/image-20230613155905040.png)
>
> **语法**
>
> ```sql
> create table 表名(
> 	字段名 数据类型,
> 	……
> 	[constraint][外键名称] foreign key(外键字段名) references 主表(主表列名)
> );
> 
> alter table 表名 add constraint 约束名称 foreign key(外键字段名) references 主表(主表列名);
> 
> /* 添加外键 */
> alter table emp add constraint fk_emp_dept_id foreign key(dept_id) references dept(id);
> 
> /* 删除外键约束 */
> alter table emp drop foreign key fk_emp_dept_id;
> ```
>
> 删除/更新行为
>
> ![image-20230613160834769](https://pic1.xuehuaimg.com/proxy/https://cdn.jsdelivr.net/gh/moshang1314/myBlog@main/image/image-20230613160834769.png)
>
> ```sql
> alter table 表名 add constraint 约束名称 foreign key (外键字段) references 主表名(主表字段名) on update cascade on delete cascade;
> ```

