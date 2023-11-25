# 1 构造对象的优化

> ```c++
> //指针指向临时对象不安全，临时对象的生命周期是该语句
> Test *p = &Test(20);
> //引用指向临时对象是安全的，相当于临时对象有了名字
> const Test &ref = Test(20);	
> 
> Test t2(20, 20);
> //(50, 50)，逗号表达式
> t2 = (Test)(50, 50);
> 
> Test *p1 = new Test(70, 70);
> delete p1;
> ```
>
> **构造对象优化（将无用的构造调用优化掉）**：
>
> 1) 函数参数传递过程中，对象优先按引用传递，不要按值传递
>
> 2) 函数返回对象的时候，应该优先返回一个临时对象，而不是返回一个定义过的对象
>
>    1) ```c++
>       Test GetObject(Test &t){
>       	int val = t.getData();
>       	Test tmp(val);
>       	return tmp;
>       }
>                                     
>       Test GetObject2(Test &t){
>       	int val = t.getData();
>       	return Test(val);
>       }
>       int main(){
>       	Test t1;
>       	Test t2;
>       	/* tmp是局部对象，在GetObject栈帧上，所以会调用拷贝构造生成在main函数的栈帧中生成一个
>       		临时对象，然后再调用拷贝赋值运算赋给t2。	
>       	*/
>       	t2 = GetObject(t1);
>           /*
>           	函数中返回临时对象编译器会做优化，不会把临时对象构造在GetObject2栈帧上，而是直接在main函数栈帧中
>           	创建临时对象，再调用赋值拷贝给t2.相比之下，少了tmp的构造与析构过程
>           */
>       	t2 = GetObject2(t1);
>       }
>       ```
>
> 3)  接收返回值是对象的函数调用时候，优先按初始化的方式接收，不按赋值方式接收。会把临时对象优化掉。
>
>    1) ```c++
>       Test GetObject(Test &t){
>       	int val = t.getData();
>       	return Test(val);
>       }
>       int main(){
>       	Test t1;
>       	/* 
>       	返回临时对象会被优化，被优化成 Test t2 = Test(val); 相当于不会构造临时对象了
>       	*/
>       	Test t2 = GetObject(t1);
>       }
>       ```

## 1.1 右值引用与左值引用

> ```c++
> int main(){
> 	int a = 10;
> 	int &b = a;
> 	
> 	//int &&c = a;	//无法将左值绑定到右值引用
> 	
> 	//int &c = 20;	//不能把非const左值引用绑定到一个右值
> 	const int &c = 20;	/* 相当于 int temp = 20; const int &c = temp; */
>     
>     int &&d = 20;	//可以把一个右值绑定到一个右值引用上
>     //一个右值引用变量，本身是左值
>     //int &&f = d; //非法
> }
> ```

## 1.2 引用折叠与完美转发

> ​	引用折叠：`int && &&` => `int &&`，`int && &` => `int &`。
>
> 因此，在模板中，`&&`表示的不是右值引用，而是万能引用，即既可以接收左值也可以接收右值。即当一个函数的形参是模板类型的右值引用时，那么实参的类型会进行自动推断（而不要求必须是右值）。注意在正常情况下如果一个形参是右值引用，那么给函数传递一个左值会有编译错误的，但是使用模板类型的右值引用就不会有这个问题。
>
> ```c++
> void PerfectForward(T&& t){
> 	Fun(std::forward<T>(t));
> }
> ```
>
> ​	此时传入的t既可以是左值，也可以是右值。
>
> ​	运行以下程序，发现最终识别的都是左值引用。
>
> ```c++
> void Func(int&& x){
> 	cout << "rvalue" << endl;
> }
> 
> void Func(int& x){
> 	cout << "lvalue" << endl;
> }
> 
> template<class T>
> void PerfectForward(T&& t){
> 	PerfectForward(10); //左值
> 	int a = 10;
> 	PerfectForward(a); //左值
> 	PerfectForward(std::move(a)); //左值
> }
> ```
>
> ​	这是因为右值引用一旦引用了，就变成了左值，如果我们还希望保持该右值引用的特性，需要使用`std::forward<>`函数来对其进行封装：
>
> ```c++
> void PerfectForward(T&& t){
> 	Func(std::forward<T>(t));
> }
> ```
>
> ​	`forward(t)`来进行封装的意义在于，保持`t`原来的属性，如果它原来是左值那么封装之后还是左值，如果它是右值的引用，则将其还原成右值。该函数的作用被称为完美转发。由于这一性质，STL容器的插入都可以使用右值引用的模板来统一实现左值和右值版本。

# 2 绑定器和函数对象

> 1、C++ STL中的绑定器
>
> `bind1st`：`operator()`的第一个形参变量绑定成一个确定的值
>
> `bind2nd`：`operator()`的第二个形参绑定成一个确定的值
>
> ```c++
> #include <iostream>
> #include <vector>
> #include <functional>
> #include <algorithm>
> #include <ctime>
> using namespace std;
> 
> template<typename Container>
> void showContainer( Container& con ) {
>     typename Container::iterator it = con.begin();
>     for (; it != con.end(); ++it) {
>         cout << *it << " ";
>     }
>     cout << endl;
> }
> 
> template<typename Compare , typename T>
> class _mybind1st {
> public:
>     _mybind1st( Compare comp , T val ) : _comp( comp ) , _val( val ) {}
>     bool operator()( const T& second ) {
>         return _comp( _val , second );
>     }
> private:
>     Compare _comp;
>     T _val;
> }
> template<typename Compare , typename T>
> _mybind1st<Compare , T> mybind1st( Compare comp , const T& val ) {
>     //直接使用函数模板，好处是，可以进行类型的推演
>     return _mybind1st<Compare , T>( comp , val );
> }
> 
> int main()
> {
>     vector<int> vec;
>     srand( time( nullptr ) );
>     for (int i = 0; i < 20; ++i) {
>         vec.push_back( rand() % 100 + 1 );
>     }
>     showContainer( vec );
>     sort( vec.begin() , vec.end() , greater<int> );
>     showContainer( vec );
> 
>     // 把70按顺序插入到vec容器当中， 找到第一个小于70的数
>     auto it1 = find_if( vec.begin() , vec.end() , bind1st( greater<int>() , 70 ) );
>     //auto it2 = find_if( vec.begin() , vec.end() , bind2nd( less<int>() , 70 ) );
>     if (it1 != vec.end()) {
>         vec.insert( it1 , 70 );
>     }
>     return 0;
> }
> ```
>
> **2、C++11从Boost库中引入了`bind`和`function`机制**
>
> ```c++
> #include <iostream>
> #include <vector>
> #include <functional>
> #include <algorithm>
> #include <ctime>
> using namespace std;
> using namespace placeholders
> /*
> C++11 bind绑定器 => 返回的结果还是一个函数对象
> */
> void hello( string str ) {
>     cout << str << endl;
> }
> int sum( int a , int b ) { return a + b; }
> class Test {
> public:
>     int sub( int a , int b ) { return a - b; }
> };
> int main()
> {
>     /* bind是函数模板，可以自动推导模板类型参数，其返回一个函数对象
>     可以绑定:
>     1. function
>     2. function object
>     3. member function
>     4. data members
>     返回一个function object
>     */
>     cout << bind( sum , 10 , 20 )( ) << endl;
>     cout << bind( &Test::sub , Test() , 20 , 30 )( ) << endl;
>     cout << bind<double>(&Test::sub, Test(), 20, 30)() << endl; // return double(a - b);
>     //参数占位符
>     bind( hello , placeholders::_1 )( "hello bind 2!" );
>     cout << bind( sum , placeholders::_1 , placeholders::_2 )( 200 , 300 ) << endl;
>     function<void( string )> func1 = bind( hello , _1 );
>     func1( "hello china!" );
>     return 0;
> }
> ```
>
> ```c++
> #include <iostream>
> #include <vector>
> #include <functional>
> #include <algorithm>
> #include <ctime>
> #include <thread>
> #include <typeinfo>
> #include <algorithm>
> using namespace std;
> using namespace placeholders;
> void hello( string str ) {
>     cout << str << endl;
> }
> /*
> C++11 bind绑定器 => 返回的结果还是一个函数对象
> */
> template<typename Fty>
> class myfunction {};
> 
> // template <typename R , typename A1>
> // class myfunction<R( A1 )> {
> // public:
> //     using PFUNC = R( * )( A1 );
> //     myfunction( PFUNC pfunc ) : _pfunc( pfunc ) {}
> //     R operator()( A1 arg ) {
> //         return _pfunc( arg );
> //     }
> // private:
> //     PFUNC _pfunc;
> // };
> 
> template<typename R , typename... A>
> class myfunction<R( A... )> {
> public:
>     using PFUNC = R( * )( A... );
>     myfunction( PFUNC pfunc ) : _pfunc( pfunc ) {}
>     R operator()( A... arg ) {
>         return _pfunc( arg... );
>     }
> private:
>     PFUNC _pfunc;
> };
> 
> int main(){
>     myfunction<void( string )> f(hello);
>     f("hello world!");
>     return 0;
> }
> ```
>
> **3、lambda表达式，底层依赖函数对象的机制实现的**
>
> ​	用什么类型来表示lambda表达式？ =》 函数对象类型`function<>`
>
> ```c++
> class Data{
> public:
>     Data(int val1 = 10, int val2 = 10) : ma(val1), mb(val2 ){}
>     int ma;
>     int mb;
> };
> int main(){
>     //优先级队列
>     priority_queue<Data, vector<Data>, function<bool(Data&, Data&)> Maxheap([](Data &d1, Data &d2)->bool
>                                                                           {
>                                                                               return d1.ma > d2.ma;
>                                                                           });
>     //自定义删除器
>     unique_ptr<FILE, function<void(FILE*)> ptr1(fopen("data.txt", "w"), [](FILE *pf){fclose(pf);});
>     
> 	map<int, function<int(int, int)>> caculateMap;
>     caculateMap[1] = [](int a, int b)->int {return a + b;};
>     caculateMap[2] = [](int a, int b)->int {return a - b;};
>     caculateMap[3] = [](int a, int b)->int {return a * b;};
>     caculateMap[4] = [](int a, int b)->int {return a / b;};
> }
> ```

# 3 模板的完全特例化和非完全（部分）特例化

> ​	注意区分函数类型和函数指针类型。
>
> `int (*)(int, int)` => 函数指针类型
>
> `int(int, int)` => 函数类型
>
> ```c++
> int fun(int a, int b){
>     return a + b;
> }
> int main(){
>     /* 函数指针类型 */
> 	typedef int(*Pfunc1)(int, int); 
>     Pfunc1 pf1 = fun;
>     /* 函数类型 */
>     typedef int Pfunc2(int, int);
>     using Pfunc3 = int(int, int);
>     Pfunc2 *pf2 = fun;
>     cout &lt;&lt; (*pf2)(10, 20) &lt;&lt; endl;
> }#
> ```

## 3.1 模板实参推演

> ```c++
> #include <iostream>
> #include <typeinfo>
> using namespace std;
> class Test{
> public:
>     int sum(int a, int b){ return a + b;}
> };
> 
> template<typename T>
> void func(T a){
> 	cout << typeid(T).name() << endl;
> }
> 
> //函数重载，将返回值类型，参数类型分解出来
> template<typename R, typename A1, typename A2>
> void func2(R (*a)(A1, A2)){
>     cout << typeid(R).name() << endl;
>     cout << typeid(A1).name() << endl;
>     cout << typeid(A2).name() << endl;
> }
> //函数重载，进一步将函数所属类的类型分解出来
> template<typename R, typename T, typename A1, typename A2>
> void func3(R (T::*a) (A1, A2)){
>     cout << typeid(R).name() << endl;
>     cout << typeid(T).name() << endl;
>     cout << typeid(A1).name() << endl;
>     cout << typeid(A2).name() << endl;
> }
> int sum(int a, int b){
> 	return a + b;
> }
> 
> int main(){
> 	func(10);
> 	func("aaa");
> 	func(sum);	//T int (*)(int, int)
>     func3(&Test::sum); 
> }
> ```

# 4 可变参数

> ​	c++11中新增加了一项内容，叫做变参数模板，所谓变参数模板，顾名思义就是参数个数和类型都可能发生变化的模板，要实现这一点，那就必须要使用模板形参包。模板形参包是可以接受0个或者n个模板实参的模板形参，至少有一个模板形参包的模板就可以称作变参数模板，所以说白了，搞懂了模板形参包就明白变参数模板了，因为变参数模板就是基于模板形参包来实现的，接下来我们就来看看到底啥是模板形参包。

## 4.1 模板形参包

> ​	模板形参包主要出现在函数模板中，目前来讲，模板形参包主要有三种，即：非类型模板形参包、类型模板形参包、模板模板形参包。

### 4.1.1 非类型模板形参包

> ​	非类型模板形参包语法是这样的：
>
> ```c++
> template<类型 ... args>
> ```
>
> ​	初看会很疑惑，说是非类型模板形参包，怎么语法里面一开始就是一个类型的，其实这里的非类型是针对typename和class关键字来的，都知道模板使用typename或者class关键字表示它们后面跟着的名称是类型名称，而这里的形参包里面类型其实表示一个固定的类型，所以这里其实不如叫做固定类型模板形参包。
> 对于上述非类型模板形参包而言，类型选择一个固定的类型，args其实是一个可修改的参数名，如下：
>
> ```
> template<int ... data> xxxx;
> ```
>
> ​	**注意：这个固定的类型是有限制的，标准C++规定，只能为整型、指针和引用。**
>
> ​	但是这个形参包该怎么用呢，有这样一个例子，比如我想统计这个幼儿园的小朋友们的年龄总和，但是目前并不知道总共有多少个小朋友，那么此时就可以用这个非类型模板形参包，代码如下：
>
> ```c++
> #include <iostream>
> using namespace std;
> //这里加一个空模板函数是为了编译可以通过，否则编译期间调用printAmt<int>(int&)就会找不到可匹配的函数
> //模板参数第一个类型实际上是用不到的，但是这里必须要加上，否则就是调用printAmt<>(int&)，模板实参为空，但是模板形参列表是不能为空的
> template<class type>
> void printAmt(int &isSumAge){
> 	return;
> }
> 
> template<class type, int age0, int ... age>
> void printAmt(int &iSumAge){
> 	iSumage += age0;
> 	//这里sizeof ...(age)是计算形参包里的形参个数，返回类型是std::size_t
> 	if((sizeof ...(age)) > 0){
> 		//这里的age...，其实就是语法中的一种包展开
> 		printAmt<type, age...>(iSumAge);
> 	}
> }
> int main(){
>     int sumAge = 0;
>     printAmt<int, 1, 2, 3, 4, 5, 7, 8>(sumAge);
>     cout << "the sum of age is " << sumAge << endl;
>     return 0;
> }
> ```
>
> **非类型模板形参包总结**
>
> * 非类型模板形参包类型是固定的，但参数名跟普通函数参数一样，是可以修改的。
> * 传递给非类型模板形参包的实参不是类型，而是实际值。

## 可变参数展开

> **逗号表达式展开**
>
> ```c++
> #include <iostream>
> 
> template <class ...Args>
> void FormatPrint(Args... args){
> 	(void)std::initializer_list<int> {(std::cout << args << endl, 0)...};
> 	int arr[] = {(std::cout << args << " ", 0)...};
> }
> 
> int main(){
>     FormatPrint(1, 2, 3, 4);
>     return 0;
> }
> ```
>
> ​	在C++中，展开运算符`...`只在特定的语境下有效，即在支持变长参数的上下文中，例如函数参数列表，模板参数列表或初始化列表。在其他地方展开运算符是不允许的。
>
> 比如：
>
> * 在函数调用传参中展开：
>
>   * ```c++
>     using namespace std;
>                                         
>     int add( int a ) {
>         return a + 1;
>     }
>                                         
>     int fun( int a , int b , int c , int d ) {
>         cout << a << " "
>             << b << " "
>             << c << " "
>             << d << " "
>             << endl;
>     }
>                                         
>     template <typename... Args>
>     void expand( Args... args ) {
>         fun( add( args )... );
>     }
>     
>     
>     int main() {
>         expand(2, 3, 4, 5);
>         return 0;
>     }
>     ```
>
> * 在初始化列表中
>
>   * ```c++
>     using namespace std;
>                                         
>     template <typename... Args>
>     int expand( Args... args ) {
>         initializer_list<int> l { ( cout << args << endl , 0 )... };
>     }
>     
>     
>     int main() {
>         expand(2, 3, 4, 5);
>         return 0;
>     }
>     ```

# 5 `emplace_back方法原理`

> ```c++
> #include <iostream>
> #include <functional>
> #include <algorithm>
> #include <ctime>
> #include <thread>
> #include <typeinfo>
> #include <algorithm>
> #include <unistd.h>
> using namespace std;
> 
> class Test {
> public:
>     Test() {
>         cout << "无参构造" << endl;
>     }
>     Test( int a ) : m_a( a ) {
>         cout << "有参构造" << endl;
>     }
>     Test( const Test& t ) {
>         cout << "拷贝构造" << endl;
>         m_a = t.m_a;
>     }
>     Test( Test&& t ) {
>         cout << "移动构造" << endl;
>         m_a = m_a;
>     }
>     ~Test() {
>         cout << "析构函数" << endl;
>     }
> private:
>     int m_a;
> };
> 
> template<typename T>
> struct MyAllocator {
>     // allocate deallocate
>     //分配空间
>     T* allocate( size_t size ) {
>         return (T*)malloc( size * sizeof( T ) );
>     }
> 
>     // construct destory
>     template<typename... Types>
>     void construct( T* ptr , Types&&... args ) {
>         //args
>         new ( ptr ) T( std::forward<Types>( args )... );
>     }
> 
>     void deallocate( T* ptr ) {
>         free( ptr );
>     }
>     
>     void destroy( T* ptr , int size ) {
>         T* tmp = ptr;
>         for (int i = 0; i < size; i++) {
>             ptr->~T();
>             ptr += sizeof( T );
>         }
>         deallocate( tmp );
>     }
> 
> };
> 
> template<typename T, typename Alloc = MyAllocator<T>>
> class vector {
> public:
>     vector() : vec_( nullptr ) , size_( 0 ) , idx_( 0 ) {}
>     ~vector() {
>         allocator_.destroy( vec_ , idx_ );
>     }
>     // 预留内存空间，不生成对象
>     void reserve(size_t size) {
>         vec_ = allocator_.allocate( size );
>         size_ = size;
>     }
>     // push_back
>     // void push_back( const T& val ) {
>     //     allocator_.construct( vec_ + idx_ , val );
>     //     idx_++;
>     // }
> 
>     // void push_back( T&& val ) {
>     //     allocator_.construct( vec_ + idx_ , std::move( val ) );
>     //     idx_++;
>     // }
>     template<typename Type>
>     void push_back( Type&& val ) {
>         allocator_.construct( vec_ + idx_ , std::forward<Type>( val ) );
>         idx_++;
>     }
>     // 1，引用折叠 
>     template<typename... Types>
>     void emplace_back( Types&&... args ) {
>         //不管是左值引用，还是右值引用，它本身是个左值。传递过程中
>         //要保持args的引用类型（左值的？右值的？）类型的完美转发forward
>         allocator_.construct( vec_ + idx_ , std::forward<Types>( args )... );
>         idx_++;
>     }
> private:
>     T* vec_;
>     int size_;
>     int idx_;
>     Alloc allocator_;
> };
> int main() {
>     //Test t1( 10 );
>     vector<Test> v;
>     v.reserve( 100 );
>     v.push_back( Test( 30 ) );
>     v.emplace_back( 20 );
>     return 0;
> }
> ```

# 6 Any类型的底层原理

> ```c++
> class Any {
> public:
>     Any() = default;
>     ~Any() = default;
>     //成员变量unique_ptr都不可拷贝
>     Any( const Any& ) = delete;
>     Any& operator=( const Any& ) = delete;
>     Any( Any&& ) = default;
> 
>     //这个构造函数可以让Any类型接收任意其它类型的数据
>     template<typename T>
>     Any( T data ) : base_( std::make_unique<Derive<T>>( data ) ) {}
> 
>     //这个方法能把Any对象里面存储的data数据提取出来
>     template<typename T>
>     T cast_() {
>         // 基类指针 -> 派生类指针
>         Derive<T>* pd = dynamic_static<Derive<T>*>( base_.get() );
>         if (pd == nullptr) {
>             throw "type is unmatch!";
>         }
>         return pd->data_;
>     }
> private:
>     //基类类型
>     class Base {
>     public:
>         virtual ~Base() = default;
>     };
> 
>     //派生类类型
>     template<typename T>
>     class Derive : public Base {
>     public:
>         Derive( T data ) : data_( data ) {}
>         T data_;    //保存了其他的任意类型 
>     };
> private:
>     //定义一个基类的指针
>     std::unique_ptr<Base> base_;
> };
> ```

# 7 `__thread`关键字的用法

> [C/C++编程：线程局部变量 __thread 关键字_c++ __thread_OceanStar的学习笔记的博客-CSDN博客](https://blog.csdn.net/zhizhengguan/article/details/121764165)

# 8 编译器内建函数`builtin_expect()`

> [__builtin_expect的作用_freshman94的博客-CSDN博客](https://blog.csdn.net/qq_22660775/article/details/89028258)

# 9 `long syscall(long number, ...);`

> [syscall()_For Nine的博客-CSDN博客](https://blog.csdn.net/m0_51551385/article/details/125116618)
>
> ​	当调用C库中没有包装函数的系统调用时，使用syscall()非常有用。（如有一个函数gettid()可以得到线程的真正PID，但glibc并没有实现该函数，只能通过Linux的系统调用syscall来获取）

# 10 `operator new()`

> [C++中运算符new的深入讲解_new运算符__Santiago的博客-CSDN博客](https://blog.csdn.net/mary288267/article/details/129811443)

# 11 `std::enable_shared_from_this`类的原理分析

> [关于boost中enable_shared_from_this类的原理分析 - 阿玛尼迪迪 - 博客园 (cnblogs.com)](https://www.cnblogs.com/codingmengmeng/p/9123874.html)

# 12 `unordered_map`

> [std::unordered_map - cppreference.com](https://en.cppreference.com/w/cpp/container/unordered_map)
>
> [C++中的unordered_map常见用法详解_花无凋零之时的博客-CSDN博客](https://blog.csdn.net/weixin_55267022/article/details/122689446)
>
> [c++映射map、multimap详解 - 知乎 (zhihu.com)](https://zhuanlan.zhihu.com/p/591391194?utm_id=0)
>
> ```c++
> template < class Key,					// uordered_map::key_type
> 		   class T,						// unordered_map::mapped_type
> 		   class Hast = hash<Key>,		// unordered_map::hasher
> 		   class Pred = equal_to<Key>,	// unordered_map::key_equal
> 		   class Alloc = allocator< pair<const Key, T>>	// unordered_map::allocator_type
>            > class unordered_map;
> ```
>
> ![image-20230818130402296](https://pic1.xuehuaimg.com/proxy/https://cdn.jsdelivr.net/gh/moshang1314/myBlog@main/image/image-20230818130402296.png)
>
>   ![image-20230818130709527](https://pic1.xuehuaimg.com/proxy/https://cdn.jsdelivr.net/gh/moshang1314/myBlog@main/image/image-20230818130709527.png)
>
> ![image-20230818130742489](https://pic1.xuehuaimg.com/proxy/https://cdn.jsdelivr.net/gh/moshang1314/myBlog@main/image/image-20230818130742489.png)
>
> ![image-20230818130857106](https://pic1.xuehuaimg.com/proxy/https://cdn.jsdelivr.net/gh/moshang1314/myBlog@main/image/image-20230818130857106.png)
>
> ![image-20230818131237824](https://pic1.xuehuaimg.com/proxy/https://cdn.jsdelivr.net/gh/moshang1314/myBlog@main/image/image-20230818131237824.png)
>
> ![image-20230818131432979](https://pic1.xuehuaimg.com/proxy/https://cdn.jsdelivr.net/gh/moshang1314/myBlog@main/image/image-20230818131432979.png)
>
> ![image-20230818131759757](https://pic1.xuehuaimg.com/proxy/https://cdn.jsdelivr.net/gh/moshang1314/myBlog@main/image/image-20230818131759757.png)
>
> ![image-20230818132153008](https://pic1.xuehuaimg.com/proxy/https://cdn.jsdelivr.net/gh/moshang1314/myBlog@main/image/image-20230818132153008.png)
>
> ![image-20230818132238949](https://pic1.xuehuaimg.com/proxy/https://cdn.jsdelivr.net/gh/moshang1314/myBlog@main/image/image-20230818132238949.png)
>
> ![image-20230818132313501](https://pic1.xuehuaimg.com/proxy/https://cdn.jsdelivr.net/gh/moshang1314/myBlog@main/image/image-20230818132313501.png)

# 13  `#pragma pack()`

> [#pragma pack(push,1)与#pragma pack(1)的区别_随风而去飘飘飘的博客-CSDN博客](https://blog.csdn.net/vbLittleBoy/article/details/6935165?spm=1001.2101.3001.6650.9&utm_medium=distribute.pc_relevant.none-task-blog-2~default~BlogCommendFromBaidu~Rate-9-6935165-blog-116009943.235^v38^pc_relevant_anti_vip&depth_1-utm_source=distribute.pc_relevant.none-task-blog-2~default~BlogCommendFromBaidu~Rate-9-6935165-blog-116009943.235^v38^pc_relevant_anti_vip&utm_relevant_index=15)

# 14  `MutexLock`、`MutexLockGuard`和`Condition`

> 如果基类中的默认构造函数、拷贝构造函数、拷贝赋值运算符或析构函数是被删除的函数或者不可访问，则派生类中对应的成员将是被删除的，原因是编译器不能使用基类成员来执行派生类对象基类部分的构造、赋值或销毁操作。
>
> ```c++
> class noncopyable{
> public:
> 	noncopyable(const noncopyable &) = delete;
> 	noncopyable& operator=(const noncopyable &) = delete;
> protected:
> 	noncopyable() = default;
> 	~noncopyable() = default;
> };
> 
> namespace boost{
> 	class noncopyable{
> 	protected:
> 		noncopyable() {}
> 		noncopyable() {}
> 	private:
> 		noncopyable(const noncopyable &);
> 		noncopyable &operator=(const noncopyable &);
> 	};
> }
> ```
>
> 

> `MutexLock`:
>
> ```c++
> #include <pthread.h>
> #include <unistd.h>
> #include <sys/syscall.h>
> #include <cassert>
> 
> namespace CurrentThread {
>  __thread int t_cachedTid;
>  int cacheTid() {
>      if (t_cachedTid == 0) {
>          t_cachedTid = static_cast<pid_t>( ::syscall( SYS_gettid ) );
>      }
>  }
>  inline int tid() {
>      if (__builtin_expect( t_cachedTid == 0 , 0 )) {
>          cacheTid();
>      }
>      return t_cachedTid;
>  }
> }
> class noncopyable {
> protected:
>  noncopyable() {}
>  ~noncopyable() {}
> private:
>  noncopyable( const noncopyable & );
>  noncopyable &operator=( const noncopyable & );
> };
> 
> class MutexLock : noncopyable {
> public:
>  MutexLock()
>      : holder_( 0 ) {
>      pthread_mutex_init( &mutex_ , NULL );
>  }
>  ~MutexLock() {
>      assert( holder_ == 0 );
>      pthread_mutex_destroy( &mutex_ );
>  }
>  bool isLockedByThisThread() {
>      return holder_ == CurrentThread::tid();
>  }
>  void assertLocked() {
>      assert( isLockedByThisThread() );
>  }
>  void lock() {
>      pthread_mutex_lock( &mutex_ );  // 这两行顺序不能反
>      holder_ = CurrentThread::tid();
>  }
>  void unlock() {
>      holder_ = 0;
>      pthread_mutex_unlock( &mutex_ );
>  }
> 
>  pthread_mutex_t *getPthreadMutex() {
>      return &mutex_;
>  }
> private:
>  pthread_mutex_t mutex_;
>  pid_t holder_;
> };
> ```
>
> `MutexLockGuard`：
>
> ```c++
> class MutexLockGuard : noncopyable {
> public:
>  explicit MutexLockGuard( MutexLock &mutex )
>      : mutex_( mutex ) {
>      mutex_.lock();
>  }
>  ~MutexLockGuard() {
>      mutex_.unlock();
>  }
> private:
>  MutexLock &mutex_;
> };
> 
> #define MutexLockGuard(x) static_assert(false, "missing mutex guard var name")
> ```
>
> `Condition`:
>
> ```c++
> class Condition : noncopyable {
> public:
>     explicit Condition( MutexLock &mutex )
>         : mutex_( mutex ) {
>         pthread_cond_init( &pcond_ , NULL );
>     }
>     ~Condition() {
>         pthread_cond_destroy( &pcond_ );
>     }
> 
>     void wait() {
>         pthread_cond_wait( &pcond_ , mutex_.getPthreadMutex() );
>     }
>     void notify() {
>         pthread_cond_signal( &pcond_ );
>     }
>     void notifyAll() {
>         pthread_cond_broadcast( &pcond_ );
>     }
> private:
>     MutexLock &mutex_;
>     pthread_cond_t pcond_;
> };
> ```

# 15 使用`pthread_once()`来实现线程安全的单例模式

```c++
template <typename T>
class Sigleton : noncopyable{
public:
	static T& instance(){
        pthread_once(&ponce_, &Single::init);
        return *value_;
    }
private:
    Singleton();
    ~Singleton();
    static void init(){
        value_ = new T();
    }
private:
    static pthread_once_t ponce_;
    static T* value_;
};

template<typename T>
pthread_once_t Singleton<T>::ponce_ = PTHREAD_ONCE_INIT;
T* Singleton<T>::value_
```

# 16 借`shared_ptr`实现`copy-on-write`（读写锁）

> 《Linux 多线程服务端编程 使用muduo C++网络库》P52

# 17 Linux下C/C++输入输出

> [Linux 下 C/C++ 输入输出 - 知乎 (zhihu.com)](https://zhuanlan.zhihu.com/p/376099115?utm_id=0)

## 18 `lower_bound()`和`upper_bound()`

> **`lower_bound()`和`upper_bound( )`都是利用二分查找的方法在一个排好序的数组中进行查找的。**
>
> **在从小到大的排序数组中，`lower_bound( begin,end,num)`：从数组的`begin`位置到`end-1`位置二分查找第一个大于或等于`num`的数字，找到返回该数字的地址(`iter`)，不存在则返回`end`。通过返回的地址减去起始地址`begin`,得到找到数字在数组中的下标。**
>
> **`upper_bound( begin,end,num)`：从数组的`begin`位置到`end-1`位置二分查找第一个大于`num`的数字，找到返回该数字的地址(iter)，不存在则返回`end`。通过返回的地址减去起始地址`begin`,得到找到数字在数组中的下标。**
>
> **在从大到小`greater()`的排序数组中，重载`lower_bound()`和`upper_bound()`**
>
> **`lower_bound( begin,end,num,greater() )`:从数组的`begin`位置到`end-1`位置二分查找第一个小于或等于`num`的数字，找到返回该数字的地址，不存在则返回`end`。通过返回的地址减去起始地址`begin`,得到找到数字在数组中的下标。**
>
> **`upper_bound( begin,end,num,greater() )`:从数组的`begin`位置到`end-1`位置二分查找第一个小于`num`的数字，找到返回该数字的地址，不存在则返回`end`。通过返回的地址减去起始地址`begin`,得到找到数字在数组中的下标。**
>
> **函数定义：**
>
> ```c++
> template<typename _ForwardIterator, typename _Tp, typename _Compare>
>     inline _ForwardIterator
>     lower_bound(_ForwardIterator __first, _ForwardIterator __last,
> 		const _Tp& __val, _Compare __comp);
> 
> template<typename _ForwardIterator, typename _Tp>
>     inline _ForwardIterator
>     lower_bound(_ForwardIterator __first, _ForwardIterator __last,
> 		const _Tp& __val);
> 
> /*--------------------------------------------------*/
> /*--------------------------------------------------*/
> 
> template<typename _ForwardIterator, typename _Tp, typename _Compare>
>     inline _ForwardIterator
>     upper_bound(_ForwardIterator __first, _ForwardIterator __last,
> 		const _Tp& __val, _Compare __comp);
> 
> template<typename _ForwardIterator, typename _Tp>
>     inline _ForwardIterator
>     upper_bound(_ForwardIterator __first, _ForwardIterator __last,
> 		const _Tp& __val);
> ```
>
> **示例：**
>
> ```c++
> void Func2() {
> 	std::vector<int> vec = { 2,4,6,8,10 };
> 	auto iter1 = std::lower_bound(vec.begin(), vec.end(), 4, std::less<int>());
> 	std::cout << iter1 - vec.begin() << " " << *iter1 << std::endl;
> 	auto iter2 = std::upper_bound(vec.begin(), vec.end(), 4, std::less<int>());
> 	std::cout << iter2 - vec.begin() << " " << *iter2 << std::endl;
> 	auto iter3 = std::lower_bound(vec.begin(), vec.end(), 4, std::greater<int>());
> 	if (iter3==vec.end()) {
> 		/*return;*/这里找不到
> 	}
> 	//std::cout << iter3 - vec.begin() << " " << *iter3 << std::endl;
> 	
>     //大到小排序
> 	std::vector<int> vec2 = { 5,4,3,2,1 };
> 	auto iter4 = std::lower_bound(vec2.begin(), vec2.end(), 2, std::greater<int>());
> 	std::cout << iter4 - vec2.begin() << " " << *iter4 << std::endl; //iter4-vec2.begin()得到的是该地址的下标
> 
> }
> ```
>
> ```c++
> 1 4
> 2 6
> 3 2
> ```
