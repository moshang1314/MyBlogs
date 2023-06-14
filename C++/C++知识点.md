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

# 2 C++ 单例模式

## 2.1 什么是单例模式

> ​	单例模式（Singleton Pattern，也称为单件模式），使用最广泛的设计模式之一。其意图是保证一个类仅有一个实例，并提供一个访问它的全局访问点，该实例被所有程序模块共享。
>
> **定义一个实例类：**
>
> 1. 私有化它的构造函数，以防止外界创建单例类的对象；
> 2. 使用类的私有静态指针变量指向类的唯一实例。
> 3. 使用一个公有的静态方法获取该实例。
>
> 注意：
>
> * 通常情况下，单例类会将构造函数、拷贝构造函数和赋值运算符都声明为私有(private)，以禁止从外部进行对象的复制和赋值。这样做可以确保只有一个实例，并防止通过赋值运算符创建新的实例。但某些情况下，可以自定义拷贝构造和赋值运算符来确保单例对象的正确行为。
> * 私有化析构函数可以防止在类外部通过`delete`关键字删除单例对象。

## 2.2 懒汉版（Lazy Singleton）

```c++
class Singleton{
private:
	static Singleton* instance;
private:
	Singleton(){}
	~Singleton(){}	
	Singleton(const Singleton&) = delete;
	Singleton(const Singleton&&) = delete;
	Singleton& operator=(const Singleton&) = delete;
public:
	static Sigleton* getInstance(){
	if(instance == NULL)
		instance = new Singleton();
	return instance;
	}
};
Singleton* Singleton::instance = NULL;


int main() {
    Singleton* p = Singleton::getInstance();
    return 0;
}
```

> 存在内存泄漏的问题，有两种解决方法：
>
> 1. 使用智能指针
> 2. 使用静态的嵌套对象
>
> 对于第一种解决方法，代码如下：
>
> ```c++
> #include <iostream>
> #include <memory>
> using namespace std;
> 
> class Singleton {
> private:
> 	static shared_ptr<Singleton> instance;
> private:
>     Singleton() {
>         cout << "构造函数" << endl;
>     }
>     Singleton( const Singleton& ) = delete;
> 	Singleton(const Singleton&&) = delete;
> 	Singleton& operator=(const Singleton&) = delete;
> public:
>     /* 此时，析构函数必须为公有，因为通过智能指针在类外调用类的析构，否则编译通过 */
>     ~Singleton() {
>         cout << "析构函数" << endl;
>     }
>     static shared_ptr<Singleton>& getInstance() {
>         if (instance == nullptr)
>         {
>             instance = shared_ptr<Singleton>(new Singleton());
>             //instance.reset( new Singleton() );
>         }
>         return instance;
> 	}
> };
> 
> shared_ptr<Singleton> Singleton::instance = nullptr;
> 
> int main() {
>     shared_ptr<Singleton>& p = Singleton::getInstance();
>     return 0;
> }
> ```
>
> 第二种解决方案，代码如下：
>
> ```c++
> #include <iostream>
> #include <memory>
> using namespace std;
> 
> class Singleton {
> private:
> 	static Singleton *instance;
> private:
>     Singleton() {
>         cout << "构造函数" << endl;
>     }
> 
>      ~Singleton() {
>         cout << "析构函数" << endl;
>      }
>      
>      Singleton( const Singleton& ) = delete;
>      Singleton& operator=( const Singleton& ) = delete;
> private:
>     class Deletor {
>     public:
>         ~Deletor() {
>             if (Singleton::instance != nullptr) {
>                 delete Singleton::instance;
>             }
>         }
>     };
>     static Deletor deletor;
> public:
>     static Singleton* getInstance() {
>         if (instance == nullptr)
>         {
>             instance = new Singleton();
>         }
>         return instance;
> 	}
> };
> 
> Singleton* Singleton::instance = nullptr;
> /* 必须对静态对象在类外初始化，否则可能被编译器优化删除 */
> Singleton::Deletor Singleton::deletor = Singleton::Deletor();
> 
> int main() {
>     Singleton* p = Singleton::getInstance();
>     return 0;
> }
> ```
>
> ​	在程序结束时，系统会调用静态成员`deletor`的析构函数，该析构函数会释放单例的唯一实例。使用这种方法释放单例对象有以下特征：
>
> * 在单例类内部定义专有的嵌套类
>
> * 在单例类内定义私有的专门用于释放的静态成员。
>
> * 利用程序在程序结束时析构静态变量的特性，选择最终的释放时机。
>
>
>   上述代码在单线程环境下是正确无误的，但是当拿到多线程环境下时，就会出现`race condition`，不是线程安全的。要使其线程安全，能在多线程环境下实现单例模式，我们首先想到的是利用同步机制来正确保护我们的`shared data`。可以使用双检测锁模式（DCL: Double-Checked Locking Pattern）：
>
>   ```c++
>   static Singleton* getInstance(){
>   	if(instance == NULL){
>   		Lock lock;	//基于作用域的加锁，超出作用域，自动调用析构函数解锁
>   		if(instance == NULL){
>   			instance = new Singleton();
>   		}
>   	}
>   	return instance;
>   }
>   ```
>
>   加入DCL后，其实还是有问题的，关于memory model。在某些内存模型中（虽然不常见）或者是由于编译器的优化以及运行时优化等等原因，使得instance虽然已经不是nullptr但是其所指对象还没有完成构造，这种情况下，另一个线程如果调用getInstance()就有可能使用到一个不完全初始化的对象。换句话说，就是代码中第2行：if(instance == NULL)和第六行instance = new Singleton();没有正确的同步，在某种情况下会出现new返回了地址赋值给instance变量而Singleton此时还没有构造完全，当另一个线程随后运行到第2行时将不会进入if从而返回了不完全的实例对象给用户使用，造成了严重的错误。在C++11没有出来的时候，只能靠插入两个memory barrier（内存屏障）来解决这个错误，但是C++11引进了memory model，提供了Atomic实现内存的同步访问，即不同线程总是获取对象修改前或修改后的值，无法在对象修改期间获得该对象。
>
>   ```c++
>   atomic<Widget*> Widget::pInstance{ nullptr };
>   Widget* Widget::Instance() {
>       if (pInstance == nullptr) { 
>           lock_guard<mutex> lock{ mutW }; 
>           if (pInstance == nullptr) { 
>               pInstance = new Widget(); 
>           }
>       } 
>       return pInstance;
>   }
>   ```
>
> Best of All:
>
> C++11规定了local static在多线程条件下的初始化行为，要求编译器保证了内部静态变量的线程安全性。在C++11标准下，《Effective C++》提出了一种更优雅的单例模式实现，使用函数内的 local static 对象。这样，只有当第一次访问getInstance()方法时才创建实例。这种方法也被称为Meyers' Singleton。C++0x之后该实现是线程安全的，C++0x之前仍需加锁。
>
> ```c++
> class Singleton
> {
> private:
> 	Singleton() { };
> 	~Singleton() { };
> 	Singleton(const Singleton&);
> 	Singleton& operator=(const Singleton&);
> public:
> 	static Singleton& getInstance() 
>         {
> 		static Singleton instance;
> 		return instance;
> 	}
> };
> ```
>
> 

## 2.3 饿汉版（Eager Singleton)

> 饿汉版（Eager Singleton）：指单例实例在程序运行时被立即执行初始化
>
> ```c++
> class Singleton
> {
> private:
> 	static Singleton instance;
> private:
> 	Singleton();
> 	~Singleton();
> 	Singleton(const Singleton&);
> 	Singleton& operator=(const Singleton&);
> public:
> 	static Singleton& getInstance() {
> 		return instance;
> 	}
> }
>  
> // initialize defaultly
> Singleton Singleton::instance;
> ```
>
> 由于在main函数之前初始化，所以没有线程安全的问题。

> 懒汉单例和饿汉单例多线程场景分析
> 1."懒汉"模式虽然有优点，但是每次调用GetInstance()静态方法时，必须判断NULL == m_instance，使程序相对开销增大；
>
> 2.多线程中会导致多个实例的产生，从而导致运行代码不正确以及内存的泄露；
>
> 3.未提供释放资源的函数，存在内存泄漏问题。
>
> 由于C++中构造函数并不是线程安全的。C++中的构造函数简单来说分两步：
>
> 第一步：内存分配；
>
> 第二步：初始化成员变量；
>
> 由于多线程的关系，可能当我们在分配内存好了以后，还没来得急初始化成员变量，就进行线程切换，另外一个线程拿到所有权后，由于内存已经分配了，但是变量初始化还没进行，因此获取成员变量的相关值会发生不一致现象。

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
> shared_ptr<int> p=new int(1024);   //错误 
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
> * **如果自定义了一个构造函数，编译器还会提供默认的拷贝构造。**
> * **如果自定义了一个任意一种拷贝构造，编译器将不再提供默认拷贝构造。**
> * 如果自定义了一个析构函数，编译器将不在提供默认的析构函数。
> * 如果自定义了拷贝赋值运算符，编译器将不再提供默认的拷贝赋值运算符。
> * **如果定义了拷贝赋值运算符，编译器将不会生成默认的拷贝构造函数，同样地，如果定义了拷贝赋值运算符后，也不会生成默认的拷贝构造函数了。**
>
> 对于对象的复制操作（包括拷贝构造，移动拷贝构造，拷贝赋值运算符和移动拷贝赋值运算符），只要提供了任意一种，则编译器默认用户已经负责了对象的复制工作，将不会提供额外的默认复制操作函数。

# 6 观察者模式

```c++
#include <algorithm>
#include <vector>
#include <stdio.h>

class Observable;

/* 抽象的观察者 */
class Observer{
public:
	virtual ~Observer();
	virtual void update() = 0;
	
	void observe(Observable* s);
protected:
    /* 观察目标 */
	Obsersable* subject_;
};

/* 被观察者 */
class Observable{
public:
    void register_(Observer* x);
    void unregister(Observer* x);
    /* 通知每个具体的观察者 */
    void notifyObservers(){
        for(size_t i = 0; i < observers_.size(); ++i){
            Observer* x = observers_[i];
            if(x){
                x->update();
            }
        }
    }
private:
    std::vector<Observer*> observers_;
};

/* 从观察者中退出观察，使得被观察者不再通知该对象 */
Observer::~Observer(){
    subject_->unregister(this);
}

/* 注册观察对象, 使观察者可通知该对象 */
void Observer::observe(Observable* s){
    s->register_(this);
    subject_ = s;
}
/* 将观察者对象放到被观察者的一个vector容器中 */
void Observable::register_(Observer* x){
    observers_.push_back(x);
}

void Observable::unregister(Observer* x){
    std::vector<Observer*>::iterator it = std::find(observers_.begin(), observers_.end(), x);
    if(it != observers_.end()){
        std::swap(*it, observers_.back());
        observers_.pop_back();
    }
}

// 具体的观察者
class Foo : public Observer{
    virtual void update(){
        printf("Foo::update() %p\n", this);
    }
};

int main(){
    Foo* p = new Foo;
    Observable subject;
    p->observe(&subject);
    subject.notifyObservers();
    delete p;
    subject.notifyObservers();
}
```

