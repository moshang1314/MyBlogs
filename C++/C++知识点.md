# 1 C++ 析构函数私有化

* 可以防止对象在堆上被删除：私有析构函数可以防止通过`delete`运算符在堆上删除对象。由于只有类内部的成员函数可以访问私有成员，因此无法从类外部直接删除对象。这可以用于**实现单例模式等特殊的对象创建和销毁。**

  * ```c++
    #include <iostream>
    
    using namespace std;
    
    class base {
    public:
        base() {
            cout << "无参构造" << endl;
        }
    private:
        ~base() {
            cout << "析构函数" << endl;
        }
    };
    //base c;
    int main() {
        base *p = new base;
        delete p;
    }
    /* 报错 */
    test.cpp: In function ‘int main()’:
    test.cpp:18:12: error: ‘base::~base()’ is private within this context
         delete p;
                ^
    test.cpp:11:5: note: declared private here
         ~base() {
    ```

* 禁止对象的自动存储期：**私有构造函数还可以防止在栈上创建对象**，因为只有类内部的成员函数可以调用私有析构函数。**这可以用于强制对象只能通过特定的创建函数或静态成员函数进行实例化**。

  * ```c++
    #include <iostream>
    
    using namespace std;
    
    class base {
    public:
        base() {
            cout << "无参构造" << endl;
        }
    private:
        ~base() {
            cout << "析构函数" << endl;
        }
    };
    //base c;
    int main() {
        base p = base();
        delete p;
    }
    
    /// 报错
    test.cpp: In function ‘int main()’:
    test.cpp:17:19: error: ‘base::~base()’ is private within this context
         base p = base();
                       ^
    test.cpp:11:5: note: declared private here
         ~base() {
         ^
    test.cpp:17:19: error: ‘base::~base()’ is private within this context
         base p = base();
                       ^
    test.cpp:11:5: note: declared private here
         ~base() {
         ^
    test.cpp: At global scope:
    test.cpp:19:13: error: expected unqualified-id before ‘^’ token
                 ^
                 ^
    ```

* 阻止派生类的析构函数调用：如果基类的析构函数声明为私有，则派生类无法直接调用基类的析构函数。这可以用于强制使用

# 3 C++智能指针`shared_ptr`

> `shared_ptr`类对象默认初始化为一个空指针。
>
> ```
> shared_ptr<int>p1; //nullptr
> shared_ptr<string> p2 = make_shared<string>(); // 非nullptr,调用string默认构造
> ```

## 3.1 `shared_ptr`类的操作

![image-20230612113355424](https://pic1.xuehuaimg.com/proxy/https://cdn.jsdelivr.net/gh/moshang1314/myBlog@main/image/image-20230612113355424.png)

![image-20230612145538488](https://pic1.xuehuaimg.com/proxy/https://cdn.jsdelivr.net/gh/moshang1314/myBlog@main/image/image-20230612145538488.png)

## 3.1 `make_shared`函数

> 最安全的分配和使用动态内存的方法就是调用`make_shared`函数，此函数在内存中动态分配对象并初始化它，返回指向此对象的`shared_ptr`。`make_shared` 也定义在头文件 `memory` 中。
>
> ```c++
> //指向一个值为42的int的shared_ptr
> shared_ptr<int> p3 = make_shared<int>(42);
> //p2指向一个值为'999999999'的string
> shared_ptr<string> p4=make_shared<string>(10, '9');
> //p3指向一个值初始化为0的int数
> shared_ptr<int> p5 = make_shared<int>();
> ```
>
> 类似顺序容器的 `emplace` 成员，`make_shared`用其参数来构造给定类型的对象。例如，调用 `make_shared`时传递的参数必须与 `string` 的某个构造函数相匹配。
>
> 当然，我们通常用`auto`定义一个对象来保存`make_shared`的结果：
>
> ```c++
> auto p4 = make_shared<vector<string>>();
> ```

## 3.2 `shared_ptr`的拷贝赋值

> 当进行拷贝或赋值操作时，每个shared_ptr都会记录有多少个其他shared_ptr指向相同的对象：
>
> ```c++
> auto p = make_shared<int>(42); // p指向的对象只有 p 一个引用者
> auto q(p); // p和q指向相同对象，此对象有两个引用者
> ```
>
> ​	每个`shared_ptr`都有一个关联的计数器，通常称其为引用计数。
>
> > 对`shared_ptr`类进行拷贝时，计数器就会增加。例如：当用一个`shared_ptr`初始化另一个`shared_ptr`、或者它作为参数传递给一个函数以及作为函数的返回值，它所关联的计数器就会增加当我们给让`shared_ptr`指向另一个对象或者`shared_ptr`销毁时，原对象的计数器就会递减一旦一个`shared_ptr`的计数器为0，就会自动释放该对象的内存。
>
> ```c++
> auto r = make_shared<int>(42); // r指向的 int 只有一个引用者
> r = q; // 给r赋值，那么： r原来所指的对象引用计数变为0，然后自动释放内存，q所指的对象的引用计数+1
> ```

## 3.3 `shared_ptr`与`new`结合使用

> 如果我们不初始化一个智能指针，它就会被初始化为一个空指针，我们也能用new返回的指针来初始化一个智能指针，因为，new申请的动态内存的使用、释放等规则仍然符合`shared_ptr`类的使用规则：
>
> ```c++
> shared_ptr<double> p1;	//shared_ptr可以指向一个double
> shared_ptr<int> p2(new int(42));	//p2指向一个值为42的int
> ```
>
> ​	因为智能指针的构造函数是`explicit`的。因此：我们不能将一个内置指针隐式地转换为一个智能指针，必须使用**直接初始化**形式来初始化一个智能指针。
>
> ```c++
> shared_ptr<int> p = new int(1024);   //错误 
> shared_ptr<int> p2(new int(1024)); //正确：使用直接初始化
> ```
>
> ​	动态内存作为返回值时的使用手法：限于上面的使用语法，一个返回`shared_ptr`的函数不能在其返回语句中隐式转换为一个普通指针
>
> ```c++
> shared_ptr<int> clone(int p)
> {
>     return new int(p); //错误
> }
>  
> shared_ptr<int> clone(int p)
> {
>     return shared_ptr<int>(new int(p)); //正确
> }
> ```
>
> ​	默认情况下，一个用来初始化智能指针的普通指针必须指向动态内存，因为智能指针默认使用delete释放它所关联的对象。
>
> **定义`shared_ptr`的其他方法：**
>
> ![image-20230612151358283](https://pic1.xuehuaimg.com/proxy/https://cdn.jsdelivr.net/gh/moshang1314/myBlog@main/image/image-20230612151358283.png)

## 3.4 不要混合使用普通/智能指针

> 当一个函数的参数是`shared_ptr`类时，有以下规则：
>
> - 函数的调用是传值调用
> - 调用函数时，该`shared_ptr`类所指向的对象引用计数加1。但是函数调用完成之后，`shared_ptr`类自动释放，对象的引用计数又减1
>
> ```c++
> void process(shared_ptr<int> ptr){ 
> 	... // 使用ptr
> } // ptr 离开作用域，被销毁
> 
> shared_ptr<int> p(new int(42)); //初始化一个智能指针对象p，引用计数+1
> process(p);  //p所指的对象引用计数=1，引用计数总数为2
> //process函数调用之后，p所指的引用计数减1
> int i=*p; //正确
> ```
>
> ​	因为`shared_ptr`类会在生存周期结束之后，将引用计数减1，当引用计数为0时，会释放内存空间下面是一个特殊的应用场景，需要注意：
>
> ```c++
> void process(shared_ptr<int> ptr){ ... }
>  
> int *x(new int(1024));
> process(x);  //错误，不能将int*转换为一个shared_ptr<int>
> process(shared_ptr<int>(x)); //合法的，但是process函数返回之后内存会被释放
> int j=*x; //错误，x所指的内存已经被释放了
> ```
>
> > 注意：使用一个内置指针来访问一个智能指针所负责的对象是很危险的，因为我们无法知道对象何时被销毁。

## 3.5 不要使用`get()`初始化另一个智能指针或为智能指针赋值

> `shared_ptr`类的`get()`函数返回一个内置指针，指向智能指针所管理的内存空间。
>
> 此函数的设计情况：我们需要向不能使用智能指针的代码传递一个内置指针
>
> * `get()`函数价格呢内存的访问权限传递给一个普通指针，但是之后代码未`delete`该内存的情况下，通过普通指针去访问该空间才是安全的。
> * 永远不要用`get()`返回的普通指针初始化另一个智能指针或者为另一个智能指针赋值。
>
> ```c++
> shared_ptr<int> p(new int(42));  //引用计数变为1
> int *q = p.get();  //正确：使用q需要注意，不要让它管理的指针被释放
>  
> {//新语句块
>     shared_ptr<int>(q); //用q初始化一个智能指针对象
> } //语句块结束之后，智能指针对象释放它所指的内存空间
>  
> int foo = *p;//错误的，p所指的内存已经被释放了
> ```

## 3.6 `reset`、`unique`函数的使用

> ​	`reset`函数会将`shared_ptr`类原先所指向的内存对象引用计数减1，并指向一块新的内存。
>
> ```c++
> shared_ptr<int> p;
> 
> p = new int(1024);	//错误，不能将一个指针赋予shared_ptr
> p = reset(new int(1024)); //正确，p指向一个新对象。
> ```
>
> ​	`reset`函数与`unique`函数配合使用：在改变对象之前，检查自己是否为当前对象的唯一用户
>
> ```c++
> shared_ptr<string> p = make_shared<string>("hello");
> 
> if(!q.unique())	//p所指向的内存空间还有其它智能指针管理
> 	p.reset(new string(*p)); //现在可以放心的改变p了
> 
> *p += newVal; //p所指向的对象只有自己一个智能指针，现在可以放心的改变对象的值了
> ```

## 3.7 异常处理

> 当程序发生异常时，我们可以捕获异常来将资源被正确的释放。但是如果没有对异常进行处理，则有以下规则：
>
>   `shared_ptr`的异常处理：如果程序发生异常，并且过早的结束了，那么智能指针也能确保在内存不再需要时将其释放
>
>   `new`的异常处理：如果释放内存在异常终止之后，那么就造成内存浪费
>
> ```c++
> voif f()
> {
>     shared_ptr<int> sp(new int(42));
>     ...//此时抛出异常，未捕获，函数终止
> }//shared_ptr仍然会自动释放内存
> ```
>
> ```c++
> voif f()
> {
>     int *ip = new int(42);
>     ...//此时抛出异常，未捕获
>     delete ip; //在退出之前释放内存，此语句没有执行到，导致内存浪费
> }
> ```

## 3.8 智能指针陷阱

> ![image-20230612160044692](https://pic1.xuehuaimg.com/proxy/https://cdn.jsdelivr.net/gh/moshang1314/myBlog@main/image/image-20230612160044692.png)

## 3.9 `shared_ptr`与动态数组的使用

> 与`unique_ptr`不同，shared_ptr不直接支持管理动态数组。如果希望使用`shared_ptr`管理动态数组，必须提供自己定义的删除器。如果未提供删除器，`shared_ptr`默认使用`delete`删除动态数组，此时`delete`少一个“[]”，因为会产生错误。
>
> ```c++
> //本例中，传递给shared_ptr一个lambda作为删除器
> shared_ptr<int> sp(new int[10], [](int *p){delete[] p;});
> shared_ptr<int> sp2(new int[3]{1, 2, 3}, [](int *p) {delete[] p;});
> 
> sp2.reset(); //使用自定义的删除器释放数组。
> ```
>
> ​	动态数组的访问：`shared_ptr`不支持点和箭头成员运算符访问数组，并且不提供下标运算符访问数组，只能通过`get()`函数来获取一个内置指针，然后再访问数组元素。
>
> ```c++
> shared_ptr<int> sp(new int[3]{1, 2, 3}, [](int *p){delete[] p;});
> for (size_t i = 0; i != 3; ++i){
> 	*(sp.get() + i) = i;
> }
> ```

# 4 类的静态成员

[(254条消息) C++静态成员（static）_Verdure的博客-CSDN博客](https://blog.csdn.net/weixin_59179454/article/details/127759353)

# 5 C++默认构造函数提供机制

[(254条消息) C++默认构造函数提供机制_系统可以提供默认的析构函数_马斯尔果的博客-CSDN博客](https://blog.csdn.net/qq_42108501/article/details/114747408)

> * 如果自定义了任意的构造函数（包括拷贝构造，移动构造等），编译器将不再提供默认的无参构造。
>
> * **如果自定义了一个构造函数，编译器还会提供默认的拷贝构造。**
>
> * **如果自定义了一个拷贝构造，编译器将不再提供默认拷贝构造和默认的移动拷贝构造以及移动赋值运算符，但依旧会提供默认的拷贝赋值。**
>
> * **如果自定义了一个移动拷贝构造，编译器将不再提供默认的移动拷贝构造和默认的拷贝构造函数，以及不再提供默认的拷贝赋值和默认的移动拷贝赋值。**
>
>   * ```c++
>     class Test{
>     public:
>     	/* 显式删除该函数，则编译器将不再提供默认的任何拷贝函数 */
>     	Test(Test&&) = delete;
>     }
>     ```
>
>
> * **如果自定义了一个拷贝赋值运算符，则编译器将不再提供默认的拷贝赋值运算符和默认的移动拷贝赋值运算符。但依旧会提供默认的拷贝构造和移动拷贝构造。**
>
> * **如果自定义了一个移动拷贝赋值运算符，则编译器将不再提供默认的拷贝构造和移动拷贝构造，以及拷贝赋值运算符。**
>
>   * ```c++
>     class Test{
>     public:
>     	/* 显式删除该函数，则编译器将不再提供默认的任何拷贝函数 */
>     	Test& operator=(Test&&) = delete;
>     }
>     ```
>
> * **如果拷贝构造被显式删除，相当于显式地定义了拷贝构造，因此编译器不会提供默认的移动拷贝构造和移动赋值运算符，但默认的赋值运算符依旧会提供。**
>
>   * ```c++
>     class Test{
>     public:
>     	/* 这意味着删除以下两个函数后，编译器将不再提供任何的默认拷贝工作 */
>     	Test(const Test&) = delete;
>     	Test& operator=(const Test&) = delete;
>     }
>     ```

# 6 抛出异常与栈展开

原文地址： http://www.cnblogs.com/zhuyf87/archive/2012/12/23/2829725.html

> 抛出异常时，将暂停当前函数的执行，开始查找匹配的catch子句。首先检查throw本身是否在try块内部，如果是，检查与该try相关的catch子句，看是否可以处理该异常。如果不能处理，就退出当前函数，并且释放当前函数的内存并销毁局部对象，继续到上层的调用函数中查找，直到找到一个可以处理该异常的catch。这个过程称为栈展开（stack unwinding）。当处理该异常的catch结束之后，紧接着该catch之后的点继续执行。
>
> **1. 为局部对象调用析构函数**
>
> 如上所述，在栈展开的过程中，会释放局部对象所占用的内存并运行类类型局部对象的析构函数。但需要注意的是，如果一个块通过new动态分配内存，并且在释放该资源之前发生异常，该块因异常而退出，那么在栈展开期间不会释放该资源，编译器不会删除该指针，这样就会造成内存泄露。
>
> **2. 析构函数应该从不抛出异常**
>
> 在为某个异常进行栈展开的时候，析构函数如果又抛出自己的未经处理的另一个异常，将会导致调用标准库terminate()函数。通常terminate()函数将调用abort()函数，导致程序的非正常退出。所以析构函数应该从不抛出异常。
>
> **3. 异常与构造函数**
>
> 如果在构造函数对象时发生异常，此时该对象可能只是被部分构造，要保证能够适当的撤销这些已构造的成员。
>
> **4. 未捕获的异常将会终止程序**
>
> 不能不处理异常。如果找不到匹配的catch，程序就会调用库函数terminate().

# 7 `std::mem_fn`

> `std::mem_fn`，**用以把一个成员函数转换成一个可调用的函数对象。函数对象的功能是借助成员函数来实现的。**
>
> 这个函数对象具备以下几个特点：
>
> * 函数对象类型重载的可调用运算符函数的第一个参数是成员函数对应的类类型的变量，或指针或引用。如果成员函数有参数，可调用运算符函数的第二个参数起，每个参数与成员函数的参数一一匹配。
> * 函数对象类型有一个成员变量（result_type）， 即是成员函数的返回值类型。
> * 如果成员函数没有参数，函数对象类型有一个成员变量（argument_type），即是成员函数对应的类类型。
> * 如果成员函数有参数，函数对象类型有一个成员变量（first_argument_type），即是成员函数对应的类类型；还有一个成员变量（second_argument_type)，即是成员函数第一个参数的类型
> * 函数对象类型的几个操作是异常安全的，包括移动构造函数，拷贝构造函数，拷贝赋值运算符，他们都是不跑出任何异常的。
>
> ```c++
> // mem_fn example
> #include <iostream>     // std::cout
> #include <functional>   // std::mem_fn
> #include <typeinfo>
> 
> struct char_holder {
>   char value;
>   char triple() {   //无参成员函数
>       std::cout<<"call char_holder"<<std::endl;
>       return value;
>       }
> };
> 
> 
> struct int_holder {
>   int value;
>   int triple(char a) { //一个参数的成员函数
>       std::cout<<"call int_holder with one paramter a is "<<a<<std::endl;
>       return value * 3;
>       }
> };
> 
> 
> struct char_int_holder {
>   char cr;
>   int value;
>   int triple(char a, char b) { //两个或多个参数的成员函数
>       std::cout<<"call char_int_holder with two paramter a is "<<a<<" b is "<<b<<std::endl;
>       return value*3;
>       }
> };
> 
> 
> 
> int main () {
>   char_holder aaa{'a'};
>   int_holder five {5};
>   char_int_holder afive {'a', 5};
> 
>   std::cout<<"call member directly==========="<<std::endl;
>   std::cout<<aaa.triple()<<'\n';
>   std::cout << five.triple('a') << '\n';
>   std::cout<<afive.triple('a', 'b')<<'\n';
> 
>   std::cout<<"same as above using a mem_fn=========="<<std::endl;
>   auto charTriple = std::mem_fn(&char_holder::triple);
>   auto intTriple = std::mem_fn (&int_holder::triple);
>   auto charIntTriple = std::mem_fn(&char_int_holder::triple);
> 
>   std::cout<<charTriple(aaa)<<std::endl;
>   std::cout<<intTriple(five, 'a')<<std::endl;
>   std::cout <<charIntTriple(afive, 'a', 'b') << '\n';
> 
> 
>   return 0;
> }
> ```

## 7.1 适用场景

> **不支持的场景**
>
> * 不支持全局函数
>
> * 不支持类protected访问权限的成员（函数或数据）
> * 不支持类private访问权限的成员（函数或数据）
>
> **支持的场景**
>
> 1) 传入类对象
> 2) 传入引用对象
> 3) 传入右值
> 4) 传入对象指针
> 5) 传入智能指针std::shared_ptr
> 6) 传入智能指针std::unique_ptr
> 7) 传入派生类对象
> 8) 带参数的成员函数
>
> 场景示例：
>
> ```c++
> #include <functional>
> #include <iostream>
>  
> int Add(int a, int b)
> {
>     return a + b;
> }
>  
> class Age
> {
> public:
>     Age(int default = 12) : m_age(default)
>     {}
>  
>     bool compare(const Age& t) const
>     {
>         return m_age < t.m_age;
>     }
>  
>     void print() const
>     {
>         std::cout << m_age << ' ';
>     }
>  
>     int m_age;
>  
> protected:
>     void add(int n)
>     {
>         m_age += n;
>     }
>  
> private:
>     void sub(int m)
>     {
>         m_age -= m;
>     }
> };
>  
> class DerivedAge : public Age
> {
> public:
>     DerivedAge(int default = 22) : Age(default)
>     {}
> };
>  
> int main(int argc, char* argv[])
> {
>     // 0.不支持的示例
>     {
>         // 1.不支持全局函数
>         // auto globalFunc = std::mem_fn(Add); // ERROR: 语法无法通过
>         // 2.不支持类protected访问权限的函数
>         // auto addFunc = std::mem_fn(&Age::add); // ERROR: 语法无法通过
>         // 3.不支持类private访问权限的函数
>         // auto subFunc = std::mem_fn(&Age::sub); // ERROR: 语法无法通过
>     }
>     // 1.成员函数示例
>     {
>         auto memFunc = std::mem_fn(&Age::print);
>  
>         // 方式一：传入类对象
>         Age obja{ 18 };
>         memFunc(obja);
>  
>         Age& refObj = obja;
>         refObj.m_age = 28;
>         // 方式二：传入引用对象
>         memFunc(refObj);
>  
>         // 方式三：传入右值
>         Age objb{ 38 };
>         memFunc(std::move(objb));
>  
>         // 方式四：传入对象指针
>         Age objc{ 48 };
>         memFunc(&objc);
>  
>         // 方式五：传入智能指针std::shared_ptr
>         std::shared_ptr<Age> pAge1 = std::make_shared<Age>(58);
>         memFunc(pAge1);
>  
>         // 方式六：传入智能指针std::unique_ptr
>         std::unique_ptr<Age> pAge2 = std::make_unique<Age>(68);
>         memFunc(pAge2);
>  
>         // 方式七：传入派生类对象
>         DerivedAge aged{ 78 };
>         memFunc(aged);
>         
>         // 方式八：带参数成员函数
>         auto memFuncWithParams = std::mem_fn(&Age::compare);
>         std::cout << memFuncWithParams(Age{ 25 }, Age{ 35 }) << std::endl;
>     }
>  
>     std::cout << std::endl;
>  
>     // 2.成员变量示例
>     {
>         auto memData = std::mem_fn(&Age::m_age);
>  
>         // 方式一：传入类对象
>         Age obja{ 19 };
>         std::cout << memData(obja) << ' ';
>  
>         Age& refObj = obja;
>         refObj.m_age = 29;
>         // 方式二：传入引用对象
>         std::cout << memData(refObj) << ' ';
>  
>         // 方式三：传入右值
>         Age objb{ 39 };
>         std::cout << memData(std::move(objb)) << ' ';
>  
>         // 方式四：传入对象指针
>         Age objc{ 49 };
>         std::cout << memData(&objc) << ' ';
>  
>         // 方式五：传入智能指针std::shared_ptr
>         std::shared_ptr<Age> pAge1 = std::make_shared<Age>(59);
>         std::cout << memData(pAge1) << ' ';
>  
>         // 方式六：传入智能指针std::unique_ptr
>         std::unique_ptr<Age> pAge2 = std::make_unique<Age>(69);
>         std::cout << memData(pAge2) << ' ';
>  
>         // 方式七：传入派生类对象
>         DerivedAge aged{ 79 };
>         std::cout << memData(aged) << ' ';
>     }
>  
>     return 0;
> }
>  
> /* run output:
> 18 28 38 48 58 68 78 1
> 19 29 39 49 59 69 79
> */
> ```
>
> 应用示例：
>
> ```c++
> #include <functional>
> #include <iostream>
> #include <algorithm>
> #include <vector>
>  
> class Age
> {
> public:
>     Age(int v) : m_age(v)
>     {}
>  
>     bool compare(const Age& t) const
>     {
>         return m_age < t.m_age;
>     }
>  
>     void print() const
>     {
>         std::cout << m_age << ' ';
>     }
>  
>     int m_age;
> };
>  
> bool compare(const Age& t1, const Age& t2)
> {
>     return t1.compare(t2);
> }
>  
> int main(int argc, char* argv[])
> {
>     // 以前写法
>     {
>         std::vector<Age> ages{ 1, 7, 19, 27, 39, 16, 13, 18 };
>         std::sort(ages.begin(), ages.end(), [&](const Age& objA, const Age& objB) {
>             return compare(objA, objB);
>             });
>         for (auto item : ages)
>         {
>             item.print();
>         }
>         std::cout << std::endl;
>     }
>     // 利用std::mem_fn写法
>     {
>         std::vector<Age> ages{ 100, 70, 290, 170, 390, 160, 300, 180 };
>         std::sort(ages.begin(), ages.end(), std::mem_fn(&Age::compare));
>         std::for_each(ages.begin(), ages.end(), std::mem_fn(&Age::print));
>         std::cout << std::endl;
>     }
>  
>     return 0;
> }
>  
> /* run output:
> 1 7 13 16 18 19 27 39
> 70 100 160 170 180 290 300 390
> */
> ```

# 8 `weak_ptr`

> C++ Prime P420 

# 9 `std::distance()`

> [C++ STL distance()函数用法详解（一看就懂） (biancheng.net)](http://c.biancheng.net/view/7372.html)

# 10 `std::advance()`函数

> [C++ STL advance()函数用法详解 (biancheng.net)](http://c.biancheng.net/view/7370.html)

# 11 C++中的`new`、`operator new`与`placement new`

> [C++中的new、operator new与placement new - 阿凡卢 - 博客园 (cnblogs.com)](https://www.cnblogs.com/luxiaoxun/archive/2012/08/10/2631812.html)
>
> **C++中的new/delete与operator new/operator delete**
>
> new operator/delete operator就是new和delete操作符，而operator new/operator delete是函数。
>
> new operator
> （1）调用operator new分配足够的空间，并调用相关对象的构造函数
> （2）不可以被重载
>
> operator new
> （1）只分配所要求的空间，不调用相关对象的构造函数。当无法满足所要求分配的空间时，则
>     ->如果有`new_handler`，则调用new_handler，否则
>     ->如果没要求不抛出异常（以nothrow参数表达），则执行bad_alloc异常，否则
>     ->返回0
> （2）可以被重载
> （3）重载时，返回类型必须声明为void*
> （4）重载时，第一个参数类型必须为表达要求分配空间的大小（字节），类型为size_t
> （5）重载时，可以带其它参数
>
> delete 与 delete operator类似
>
> ```c++
> #include <iostream>
> #include <string>
> using namespace std;
> 
> class X
> {
> public:
>     X() { cout<<"constructor of X"<<endl; }
>     ~X() { cout<<"destructor of X"<<endl;}
> 
>     void* operator new(size_t size, string str)
>     {
>         cout<<"operator new size "<<size<<" with string "<<str<<endl;
>         return ::operator new(size);
>     }
> 
>     void operator delete(void* pointee)
>     {
>         cout<<"operator delete"<<endl;
>         ::operator delete(pointee);
>     }
> private:
>     int num;
> };
> 
> int main()
> {
>     X *px = new("A new class") X;
>     delete px;
> 
>     return 0;
> }
> ```
>
> X* px = new X; //该行代码中的new为new operator，它将调用类X中的operator new，为该类的对象分配空间，然后调用当前实例的构造函数。
> delete px; //该行代码中的delete为delete operator，它将调用该实例的析构函数，然后调用类X中的operator delete，以释放该实例占用的空间。
>
> new operator与delete operator的行为是不能够也不应该被改变，这是C++标准作出的承诺。而operator new与operator delete和C语言中的malloc与free对应，只负责分配及释放空间。但使用operator new分配的空间必须使用operator delete来释放，而不能使用free，因为它们对内存使用的登记方式不同。反过来亦是一样。你可以重载operator new和operator delete以实现对内存管理的不同要求，但你不能重载new operator或delete operator以改变它们的行为。
>
> 为什么有必要写自己的operator new和operator delete？
> 答案通常是：为了效率。缺省的operator new和operator delete具有非常好的通用性，它的这种灵活性也使得在某些特定的场合下，可以进一步改善它的性能。尤其在那些需要动态分配大量的但很小的对象的应用程序里，情况更是如此。具体可参考《Effective C++》中的第二章内存管理。
>
> **Placement new的含义**
>
> placement new 是重载operator new 的一个标准、全局的版本，它不能够被自定义的版本代替（不像普通版本的operator new和operator delete能够被替换）。
>
> void *operator new( size_t, void * p ) throw() { return p; }
>
> placement new的执行忽略了size_t参数，只返还第二个参数。其结果是允许用户把一个对象放到一个特定的地方，达到调用构造函数的效果。和其他普通的new不同的是，它在括号里多了另外一个参数。比如：
>
> Widget * p = new Widget;          //ordinary new
>
> pi = new (ptr) int; pi = new (ptr) int;   //placement new
>
> 括号里的参数ptr是一个指针，它指向一个内存缓冲器，placement new将在这个缓冲器上分配一个对象。Placement new的返回值是这个被构造对象的地址(比如括号中的传递参数)。placement new主要适用于：在对时间要求非常高的应用程序中，因为这些程序分配的时间是确定的；长时间运行而不被打断的程序；以及执行一个垃圾收集器 (garbage collector)。
>
> **new 、operator new 和 placement new 区别**
>
> （1）new ：不能被重载，其行为总是一致的。它先调用operator new分配内存，然后调用构造函数初始化那段内存。
>
> new 操作符的执行过程：
> \1. 调用operator new分配内存 ；
> \2. 调用构造函数生成类对象；
> \3. 返回相应指针。
>
> （2）operator new：要实现不同的内存分配行为，应该重载operator new，而不是new。
>
> operator new就像operator + 一样，是可以重载的。如果类中没有重载operator new，那么调用的就是全局的::operator new来完成堆的分配。同理，operator new[]、operator delete、operator delete[]也是可以重载的。
>
> （3）placement new：只是operator new重载的一个版本。它并不分配内存，只是返回指向已经分配好的某段内存的一个指针。因此不能删除它，但需要调用对象的析构函数。
>
> 如果你想在已经分配的内存中创建一个对象，使用new时行不通的。也就是说placement new允许你在一个已经分配好的内存中（栈或者堆中）构造一个新的对象。原型中void* p实际上就是指向一个已经分配好的内存缓冲区的的首地址。
>
> **Placement new 存在的理由**
>
> 1.用placement new 解决buffer的问题
>
> 问题描述：用new分配的数组缓冲时，由于调用了默认构造函数，因此执行效率上不佳。若没有默认构造函数则会发生编译时错误。如果你想在预分配的内存上创建对象，用缺省的new操作符是行不通的。要解决这个问题，你可以用placement new构造。它允许你构造一个新对象到预分配的内存上。
>
> 2.增大时空效率的问题
>
> 使用new操作符分配内存需要在堆中查找足够大的剩余空间，显然这个操作速度是很慢的，而且有可能出现无法分配内存的异常（空间不够）。placement new就可以解决这个问题。我们构造对象都是在一个预先准备好了的内存缓冲区中进行，不需要查找内存，内存分配的时间是常数；而且不会出现在程序运行中途出现内存不足的异常。所以，placement new非常适合那些对时间要求比较高，长时间运行不希望被打断的应用程序。
>
> **Placement new使用步骤**
>
> 在很多情况下，placement new的使用方法和其他普通的new有所不同。这里提供了它的使用步骤。
>
> 第一步 缓存提前分配
>
> 有三种方式：
>
> 1.为了保证通过placement new使用的缓存区的memory alignment(内存队列)正确准备，使用普通的new来分配它：在堆上进行分配
> class Task ;
> char * buff = new [sizeof(Task)]; //分配内存
> (请注意auto或者static内存并非都正确地为每一个对象类型排列，所以，你将不能以placement new使用它们。)
>
> 2.在栈上进行分配
> class Task ;
> char buf[N*sizeof(Task)]; //分配内存
>
> 3.还有一种方式，就是直接通过地址来使用。(必须是有意义的地址)
> void* buf = reinterpret_cast<void*> (0xF00F);
>
> 第二步：对象的分配
>
> 在刚才已分配的缓存区调用placement new来构造一个对象。
> Task *ptask = new (buf) Task
>
> 第三步：使用
>
> 按照普通方式使用分配的对象：
>
> ptask->memberfunction();
>
> ptask-> member;
>
> //...
>
> 第四步：对象的析构
>
> 一旦你使用完这个对象，你必须调用它的析构函数来毁灭它。按照下面的方式调用析构函数：
> ptask->~Task(); //调用外在的析构函数
>
> 第五步：释放
>
> 你可以反复利用缓存并给它分配一个新的对象（重复步骤2，3，4）如果你不打算再次使用这个缓存，你可以象这样释放它：delete [] buf;
>
> 跳过任何步骤就可能导致运行时间的崩溃，内存泄露，以及其它的意想不到的情况。如果你确实需要使用placement new，请认真遵循以上的步骤。
>
> ```c++
> #include <iostream>
> using namespace std;
> 
> class X
> {
> public:
>     X() { cout<<"constructor of X"<<endl; }
>     ~X() { cout<<"destructor of X"<<endl;}
> 
>     void SetNum(int n)
>     {
>         num = n;
>     }
> 
>     int GetNum()
>     {
>         return num;
>     }
> 
> private:
>     int num;
> };
> 
> int main()
> {
>     char* buf = new char[sizeof(X)];
>     X *px = new(buf) X;
>     px->SetNum(10);
>     cout<<px->GetNum()<<endl;
>     px->~X();
>     delete []buf;
> 
>     return 0;
> }
> ```

# 12 位操作

> * `x & (-x)`可以获得`x`的二进制表示中最低位的1的位置。这是因为`(-x)`在计算机中以补码的形式存储，它等于`~b + 1`。如果和`~b`进行按位与运算，那么会得到0，但是当`~b`增加1之后，最低位的连续的1都变为了0，而最低位的0变为1，对应到`b`中即为最低位的1，因此当`b`和`~b + 1`进行按位与运算时，只有最低的1会被保留。
>
> * `x & (x - 1)`可以将`x`的二进制表示中的最低位的1置为0.·
>
> * `(1 << n) - 1`可以获得一个长度为$n$的，且每位上都是$1$的二进制数。
>
> * 除了位或（`|`）操作，可以利用亦或（`^`）操作来置位（只能对原本该位确定未置位的情况下使用）。
>
>   * ```c++
>     int main(){
>     	// a: 0001 0101
>     	char a = 13;
>     	// b: 0010 0000
>     	char b = 16;
>     	//对第6位置位
>     	a ^= b;
>     	//对第6位取消置位
>     	a ^= b;
>     }
>     ```
>
>   * `a ^ a = 0`，`a ^ 0 = a`，`a ^ b ^ a = b`，`a ^ (b ^ c) = (a ^ b) ^ c`

## 12.1 `__builtin_*位运算函数`

> 以`__builtin`开头的函数，是一种相当神奇的位运算函数，所有带`ll`的名称，均为`unsigned long long`类型，否则当作`unsigned int`类型。
>
> *  `__biltin_ctz(unsigned int)/__bilt_ctzll(unsigned long long)`
>
>   * 用法：返回括号内数的二进制表示形式中末尾0的个数
>
>   * ```c++
>     #include <bits/stdc++.h>
>     using namespace std;
>       
>     int main(){
>     	ios::sync_with_stdio(0);
>     	cin.tie(0);
>     	cout << __builtin_ctz(8) << endl; // 3 8 = 1000
>     	return 0;
>     }
>     ```
>
> * `__builtin_clz(unsigned int) / __builtin_clzll(unsigned long long)`
>
>   * 用法：返回括号内数的二进制表示的前导0的个数
>
>   * ```c++
>     #include <bits/stdc++.h>
>     using namespace std;
>     int main(){
>     	/*
>     	输出为28
>     	8 = 0000 0000 0000 0000 0000 0000 0000 1000，整型(int)为32位
>     	*/
>     	cout << __builtin_clz(8) << endl;
>     	return 0;
>     }
>     ```
>
> * `__builtin_popcount(unsigned int) / __builtin_popcountll(unsigned long long)`
>
>   * 用法：返回括号内数的二进制表示数1的个数
>
>   * ```
>     #include <bits/stdc++.h>
>     using namespace std;
>       
>     int main(){
>     	/* 
>     	输出为4
>     	15 = 1111
>     	*/
>     	cout <<__builtin_popcount(15) << endl;
>     	return 0;
>     }
>     ```
>
> * `__builtin_parity(unsigned int) / __builtin_parity(unsigned long long)`
>
>   * 用法：判断括号中数的二进制数1的个数的奇偶性（偶数返回0，奇数返回1）
>
> * `__builtin_ffs(int) / __builtin_ffsll(long long) `
>
>   * 用法：返回括号中数的二进制表示的最后一个1在第几位（从后往前算）
>
> * `__builtin_sqrt()/__builtin_sqrtf()`
>
>   * 用法：快速开平方

# 趣味快谈

## `cin.tie(0)`

> 
>
> ​	`cin.tie(0)`操作解除`cin`和`cout`关联（绑定时，`cin`之前会将`cout`输出缓冲区的数据刷新到输出文件），可以避免当`cout`缓冲区的数据没有刷新到文件中时使用了`cin`，这样就可能发生数据不一致，通过绑定可以在输入前将`cout`数据输出到文件中（其实是输出到内核页缓冲区中）。默认是绑定的。

## `ios::sync_with_stdio(false)`

![img](https://pic1.xuehuaimg.com/proxy/https://cdn.jsdelivr.net/gh/moshang1314/myBlog@main/image/20210711101338148.png)
