# 1 记录时长的`duration`

> ​	`duration`是一个模板类，表示时间间隔，可以是几秒、几分钟或者几小时的时间间隔。`duration`的原型如下：
>
> ```c++
> template <class Rep,
> class Period = ratio<1>>
> class duration;
> ```
>
> ​	第一个参数`Rep`是一个数值类型，表示时钟数的类型；第二个模板参数是一个默认模板参数`std::ratio`，表示时钟周期，它的原型如下：
>
> ```c++
> template <intmax_t N, intmax_t D=1> class ratio;
> ```
>
> ​	它表示每个时钟周期的**秒数**，其中第一个模板参数`N`代表分子，`D`代表分母，分母默认为1，因此，`ratio`代表的是一个分子除以分母的分数值。
>
> ```c++
> ratio<2>; //2秒
> ratio<60>; //代表1分钟
> ratio<60*60>; //代表1小时
> ratio<60*60*24>; //代表1天
> ratio<1, 1000>; //代表1毫秒
> ratio<1, 1000000>; //代表1微秒
> ratio<1, 1000000000>;  //代表1纳秒
>     
>   typedef ratio<1,       1000000000000000000> atto;
>   typedef ratio<1,          1000000000000000> femto;
>   typedef ratio<1,             1000000000000> pico;
>   typedef ratio<1,                1000000000> nano;
>   typedef ratio<1,                   1000000> micro;
>   typedef ratio<1,                      1000> milli;
>   typedef ratio<1,                       100> centi;
>   typedef ratio<1,                        10> deci;
>   typedef ratio<                       10, 1> deca;
>   typedef ratio<                      100, 1> hecto;
>   typedef ratio<                     1000, 1> kilo;
>   typedef ratio<                  1000000, 1> mega;
>   typedef ratio<               1000000000, 1> giga;
>   typedef ratio<            1000000000000, 1> tera;
>   typedef ratio<         1000000000000000, 1> peta;
>   typedef ratio<      1000000000000000000, 1> exa;  
> ```
>
> ​	为了方便使用，标准库中定义了一些常用的时间间隔，如时、分、秒、毫秒和纳秒。在`chrono`命名空间下，定义如下：
>
> ```c++
> typedef duration<long long, nano> nanoseconds;	//纳秒
> typedef duration<long long, micro> microseconds; //微秒
> typedef duration<long long, milli> milliseconds; //毫秒
> typedef duration<long long> seconds; //秒
> typedef duration<int, ratio<60> > minutes;	//分钟
> typedef duration<int ratio<3600> > hours;	//小时
> ```
>
> ​	通过这些定义的时间间隔类型，我们能很方便的使用它们，比如线程休眠：
>
> ```c++
> //休眠100毫秒
> this_thread::sleep_for(std::chrono::duration<int, ratio<1, 1000>>(100));
> this_thread::sleep_for(std::chrono::microsconds(100));
> 
> //休眠3秒
> this_thread::sleep_for(std::chrono::duration<int>(3));
> this_thread::sleep_for(std::chrono::seconds(3));
> ```
>
> ​	`chrono`还提供了获取时间间隔的时钟周期数的方法`count()`，它们的基本用法如下：
>
> ```c++
> #include <iostream>
> #include <chrono>
> int mian(){
> 	std::chrono::seconds s(3); //3秒
> 	std::chrono::milliseconds ms = 2 * s; //6000毫秒
> 	std::cout << "3 s duration has" << s.count() << "ticks\n" << endl
> 	<< "6000 ms duration has " << ms.count() << "ticks\n" << endl;
> }
> ```
>
> ​	`duration`在某些情况下可以进行转换，例如，当`duration`的`Rep`都为整型，且源`Period`要大于目标`Period`时，或者目标`duration`的`Rep`为浮点数时可以使用传统类型转化或者隐式调用其单值构造函数，不必调用`duration_cast`。
>
> ```c++
> int main(){
> 	//目标duration的Rep为double
> 	std::chrono::miliseconds int_milliseconds(1024); //1024ms
> 	std::chrono::microseconds int_microseconds(1024); //1024us
> 	
> 	std::chrono::duration<double> double_seconds;
> 	double_seconds = int_microseconds; //1024ms = 1.024s
> 	double_seconds = int_microseconds; //1024us = 0.001024s
> 	
> 	//当duration的Rep都为整型，且源Period可被目标Period整除时
> 	int_microsecond = int_milliseconds;	//ms赋值给us可以，但是us赋值给ms不可以
> 	int_millisecond = std::chrono::seconds(1); //s赋值给ms可以，但是ms赋值给s不可以
> 	
> 	//源duration的Rep为double，目标duration的Rep不为double，不能转换
> 	//std::chrono::milliseconds t1 = std::chrono::duration<double>(1024);
> 	
> 	/*
> 	数据会发生截断时的转换
> 	chrono库提供了duration之间相互转换的函数，其定义如下
> 	*/
> 	template<class ToDuration, class Rep, class Period>
> 	constexpr ToDuration duration_cast(const duration<Rep, Period>& d);
> 	
> 	std::chrono::seconds t1 = std::chrono::duration_cast<std::chrono::seconds>(std::chrono::milliseconds(1024)); //1s = 1024ms（精度损失）
> 	std::chrono::seconds t2 = std::chrono::duration_cast<std::chrono::seconds>(std::chrono::duration<double>(1.024)); //1s = 1.024s （精度损失）
> }
> ```
>
> ​	时间间隔之间可以做运算，计算两端时间间隔的差值的实例如下：
>
> ```c++
> int main(){
> 	std::chrono::minutes t1(10);
> 	std::chrono::seconds t2(50);
> 	std::chrono::seconds t3 = t1 - t2;
> 	cout << t3.count() << " seconds" << endl;
> }
> ```
>
> 	其中，t1代表10分钟，t2代表50秒，t3则是t1减t2，也就是600-50=550秒。通过调用t3的count()输出差值550个时钟周期，因为t3的时钟周期为1秒，所以t3表示时间间隔为550秒。
>
> 值得注意的是，duration的加减运算有一定的规则，当两个duration时钟周期不相同的时候，会先统一成一种时钟，然后再作加减运算。统一成一种时钟的规则如下：
>
> > **对于ratio<x1, y1>count1和ratio<x2, y2>count2。如果x1、x2的最大公约数为x，y1、y2的最小公倍数为y，那么统一之后的ratio为ratio<x, y>**
> >
> > 例如：
> >
> > ```c++
> > int main()
> > {
> > 	std::chrono::duration<double, std::ratio<9, 7>> d1(3);
> > 	std::chrono::duration<double, std::ratio<6, 5>> d2(1);
> > 	auto d3 = d1 - d2;
> > 	cout << "d3类型 : "<<typeid(d3).name() << endl;
> > 	cout << d3.count() << endl;
> > }
> > ```
> >
> > ​	根据前面介绍的规则，对于9/7和6/5，分子取最大公约数3，分母取最小公倍数35，所以，统一之后的duration为std::chrono::duration<double,struct std::ratio<3,35>>。然后再将原来的duration转换为统一的duration，最后计算的时钟周期数为：((9/7)/(3/35)*3)-((6/5)/(3/35)*1)，结果为31

# 2 表示时间点的`time_point`

> ​	`time_point`表示一个时间点，用来获取从它的`clock`的开始所经过的`duration`（比如，可能是从1970.1.1以来的时间间隔）和当前时间，可以做一些时间的比较和算术运算，可以和`ctime`库结合起来显示时间。`time_point`必须用`clock`来计时。`time_point`有一个函数`time_from_epoch()`用来获取1970年1月1日到`time_point`经过的`duration`。
>
> ​	`time_point`是一个类模板，原型如下：
>
> ```c++
> template <class Clock,
> class Duration = typename Clock::duration>
> class time_point;
> ```
>
> ​	第一个模板类参数`Clock`用来指定要使用的时钟（标准库中有三种时钟，`system_clock`，`steady_clock`和`high_resolution_clock`）第二个模板函数参数用来表示时间的计量单位（特化的`std::chrono::duration<>`）
>
> 计算当前时间距离1970年1月1日有多少天：
>
> ```c++
> #include <iostream>
> #include <chrono>
> #include <ratio>
> using namespace std::chrono;
> int main(){
> 	using days_type = duration<int, std::ratio<60 * 60 * 24>>;
> 	time_point<system_clock, days_type> today = time_point_cast<days_type>(system_clock::now());
> 	std::cout << today.time_since_epoch().count << " days since epoch" << endl;
> }
> ```

> `chrono`库中用一个`time_point模板类`，表示一个时间点。通过一个相对`epoch`的时间间隔`duration`来实现，`epoch`可以是1970-01-01T00:00:00时刻，对于同一个时钟来说，所有的`time_point`的`epoch`都是固定的。这个类可以与标准库`ctime`结合起来显示时间，`ctime`内部的`time_t`类型就是代表这个秒数。
>
> ```c++
> #include <iostream>
> #include <chrono>
> #include <ctime>
> 
> int mian(){
> 	//std::chrono::system_clock::time_point tp; //创建已沟通time_point类的时间点对象
> 	std::chrono::time_point<std::chrono::system_clock> tp; //创建一个time_point类的时间点对象
> 	//system_clock 采用的时钟
> 	
> 	tp = std::chrono::system_clock::now(); //获取系统现在的时间点
> 	std::time_t cur_time = std::chrono::system_clock::to_time_t(tp);
> 	char stime[30];
> 	errno_t err = ctime_s(stime, sizeof(stime), &cur_time);
> 	std::cout << stime << std::endl;
> 	
> 	return 0;
> }
> ```
>
> ```c++
> #include <iostream>
> #include <chrono>
> #include <ctime>
> 
> int main(){
> 	std::chrono::time_point<std::chrono::system_clock> tp;
> 	tp = std::chrono::system_clock::now();
> 	
> 	typedef std::chrono::duration<int, std::ratio<60 * 60 * 24>> days_type; 
> 	std::chrono::time_point<std::chrono::system_clock, daystype> today; //创建time_point时间点
> 	//std::chrono::system_clock 采用的时钟
> 	//days_type 以这个时间段为单位
> 	
> 	today = std::chrono::time_point_cast<days_type>(tp); //单位转换
> 	//把tp转换为以days_type为单位的时间点
> 	
> 	int x = today.time_since_epoch().count();
> 	//函数time_from_epoch()用来获取从epoch到当前时间点time_point经过的duration。如果time_point的duration为天
> 	//则返回的单位为天
> 	std::cout << x << std::endl;
> 	return 0;
> }
> ```
>
> ```c++
> #include <iostream>
> #include <chrono>
> #include <ctime>
> 
> int main(){
> 	std::chrono::time_point<std::chrono::system_clock> tp;
> 	tp = std::chrono::system_clock::now();
> 	
> 	std::chrono::duration<int, std::radio<60 * 60 * 24>> one_day(1); //创建duration时间段
> 	std::chrono::system_clock::time_point today = std::chrono::system_clock::now();
> 	std::chrono::system_clock::time_point tomorrow = today + one_day; //时间点加时间段
> 	
> 	std::time_t tt;
> 	char stime[30];
> 	tt = std::chrono::system_clock::to_time_t(today); //把time_point转换成time_t
> 	errno_t err = ctime_s(stime, sizeof(stime), &tt);
> 	std::cout << "today is: " << stime << std::endl;
> 	
> 	tt = std::chrono::system_clock::to_time_t(tomorrow);
> 	err = ctime_s(stime, sizeof(stime), &tt);
> 	std::cout << "tomorrow will be: " << stime << std::endl;
> 	
> 	tomorrow = std::chrono::system_clock::from_time_t(tt); //从time_t转换成time_point
> 	return 0;
> }
> 
> ```
>
> 

# 3 时钟`clock`

> C++11为我们提供了三种时钟类型：`system_clock`、`steady_clock`、`high_resolution_clock`
>
> 这三个时间类都提供了`rep`【周期】、`period`【单位比率】、`duration`成员类型
>
> 这三个时钟类都提供了一个静态成员函数`now()`用于获取当前时间，该函数的返回值是一个`time_point`类型
>
> 注意：虽然这三个时钟都很多相同的成员类型和成员函数，但它们是没有亲缘关系的。这三个时钟类型都是类，并非模板类
>
> 这三个时钟有什么区别呢？
>
> `system_clock`：就类似Windows系统右下角那个时钟，是系统时间。明显这个时钟是可以自己设置的。
>
> `system_clock`除了`now()`函数外，还提供了`to_time_t()`静态成员函数。用于将系统时间转换成熟悉的`std::time_t`类型，得到了`std::time_t`类型的值，就可以很方便地打印当前时间了
>
> `steady_clock`：是单调的时钟，相当于教练手中的秒表；只会增长，适合用于记录程序耗时，他表示时钟是不能设置的
>
> `high_resolution_clock`：是当前系统能够提供的最高精度的时钟；它也是不可以修改的。相当于 `steady_clock` 的高精度版本
>
> **`system_clock`例子**
>
> ```c++
> #include <iostream>
> #include <ratio>   //表示时间单位的库
> #include <chrono>
> #include<ctime>
> 
> int main() {
> 
>    auto tp = std::chrono::system_clock::now(); //获取系统当前时间点
>    std::time_t cur_time = std::chrono::system_clock::to_time_t(tp);  //将时间转换为ctime的time_t格式
> 
>    char stime[30];
>    errno_t err=ctime_s(stime, sizeof(stime), &cur_time);  //转换为字符串
> 
>    std::cout << stime << std::endl;
> 
>     return 0;
> }
> ```
>
> ​	不要将steady_clock、high_resolution_clock时钟的now()返回值作为to_time_t的参数，这会导致编译通不过。因为类型不匹配
>
> auto自动为此变量选择匹配的类型
>
> **`steady_clock`例子**
>
> ​	`steady_clock`的实现是使用`monotonic`时间，而`monotonic`时间一般是从boot启动后开始计数的。明显这不能获取日历时间(年月日时分秒)。那么`steady_clock`有什么用途呢？时间比较！并且是不受用户调整系统时钟影响的时间比较。
>
> ```c++
> #include <iostream>
> #include <string>
> #include <functional>
> #include <chrono>
> #include <ratio>
> #include <ctime>
> #include <thread>
> using namespace std;
> using namespace chrono;
> 
> int main() {
> 
>     auto begin = std::chrono::steady_clock::now();  //获取系统启动后到现在的时间点
>     for (int i = 0; i < 10; ++i) {
>         std::this_thread::sleep_for( std::chrono::seconds( 1 ) );
>         std::cout << i << std::endl;
>     }
>     auto end = std::chrono::steady_clock::now();
>     auto diff = ( end - begin ).count();    //返回时间差，单位纳秒
>     std::cout << "diff = " << diff << std::endl;
>     return 0;
> }
> ```

