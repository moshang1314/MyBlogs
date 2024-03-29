# 一 C++语言基础篇

## 1. 说一下你理解的C++中的四种智能指针

> 智能指针其作用是管理一个指针，避免我们程序员申请的空间在函数结束时忘记释放，造成泄露这种情况发生。智能指针其实就是一个类对象，当超出了类对象的生命周期，会自动调用其析构函数，析构函数中会自动释放资源。
>
> **常用接口**
>
> ```c++
> T* get();
> T& operator*();
> T* operator->();
> T& operator=(const T& val);
> T* release();
> void reset(T* ptr=nullptr);
> ```
>
> 下面分别说一下哪四种：
>
> 1) `auto_ptr`（C++98方案， C++11已经抛弃）采用所有权模式。
>
>    1) ```c++
>       auto_ptr<std::string> p1 (new string("hello"));
>       auto_ptr<std::string> p2;
>       p2 = p1; //auto_ptr不会报错
>       ```
>
>    2) 此时不会报错，`p2`剥夺了`p1`的所有权，但是当程序运行时访问`p1`将会报错。所以`auto_ptr`的缺点是：存在潜在的内存崩溃问题。
>
> 2) `unique_ptr`
>
>    1) > `unique_ptr`实现独占式或严格拥有的概念，保证同一时间内只有一个智能指针可以指向该对象。它对于避免资源泄露特别有用。其内部实现移动拷贝，但删除了拷贝构造，因此可以使用将亡值或函数返回的局部对象赋值，此时会调用移动构造，转移所有权。
>       >
>       > ```c++
>       > unique_ptr<string> p3(new string(auto));
>       > unique_ptr<string> p4;
>       > p4 = p3; //此时会报错
>       > ```
>       >
>       > 因为拷贝构造被删除，所以上述语句会报错，因此`unique_ptr`比`auto_ptr`更安全。
>
> 3) `share_ptr`（共享型，强引用）
>
>    1) > `share_ptr`实现共享式拥有概念，多个智能指针可以指向相同对象，该对象和其相关资源会在”最后一个引用被销毁“时候释放。从名字share就可以框出了资源可以被多个对象共享，它使用计数机制来表明资源被几个指针共享。
>       >
>       > ​	可以通过成员函数`use_count()`来查看资源所有者个数，除了可以通过`new`来构造，还可以通过传入`auto_ptr`，`unique_ptr`，`weak_ptr`来构造。当我们调用`release()`时，当前指针会释放资源所有权，计数减一。当计数等于0时，资源会被释放。
>       >
>       > ​	`share_ptr`是为了解决`auto_ptr`在对象所有权上的局限性（`auto_ptr`是独占的），在使用计数的机制上提供了可以共享所有权的智能指针。
>
> 4) `weak_ptr`（弱引用）
>
>    1) > `weak_ptr`是一种不控制对象生命周期的智能指针，它指向一个`share_ptr`管理的对象。进行该对象的内存管理的是那个强引用的`share_ptr`。`weak_ptr`只是提供了管理对象的一个访问手段。`weak_ptr`设计的目的是为配合`share_ptr`而引入一种智能指针来协助`share_ptr`工作，它只可以从一个`share_ptr`或另一个`weak_ptr`对象构造，它的构造和析构不会引起引用计数的增加或减少。
>       >
>       > ​	`weak_ptr`是用来解决`share_ptr`相互引用时的死锁问题，如果说两个`share_ptr`相互引用，那么两个指针的引用计数永远不可能下降为0，也就是资源永远不会释放。它是对对象的一种弱引用，不会增加对象的引用计数，和`share_ptr`之间可以相互转化，`share_ptr`可以直接赋值给它，它可以通过调用`lock`函数来获得`share_ptr`。
>       >
>       > ​	两个智能指针都是`share_ptr`类型的时候，析构时两个资源引用计数会减1，但两者的引用计数还是为1，导致跳出函数时资源没有被释放，解决方法：把其中一个改为`weak_ptr`就可以了。

## 2 C++中内存分配情况

> **栈：**由编译器管理分配和回收，存放局部变量和函数参数。
>
> **堆：**由程序员管理，需要手动`new`、`malloc`和`delete`、`free`进行分配和回收，空间较大，但可能出现内存泄漏和空闲碎片的情况。
>
> **全局/静态存储区：**分为初始化和未初始化两个相邻区域，存储初始化和未初始化的全局变量和静态变量。
>
> **常量存储区：**存储常量，一般不允许修改。
>
> **代码区：**存放程序的二进制代码。

## 3 C++ 中的指针参数传递和引用参数传递

> ​	指针参数传递本质上是值传递，它所传递的是一个地址值。值传递过程中，被调用的形式参数作为被调函数的局部变量处理，会在栈中开辟内存空间以存放由主调函数传递进来的实参值，从而形成了实参的一个副本。值传递的特点是，被调函数对形式参数的任何操作都是作为局部变量进行的，不会影响主调函数的实参变量的值（形参指针变了，实参指针不会变）。
>
> ​	引用参数传递过程中，被调函数的形式参数也作为局部变量在栈中开辟了内存空间，但是这时存放的是由主调函数放进来的实参变量的地址。被调函数对形参（本体）的任何操作都被处理成间接寻址，即通过栈中存放的地址访问主调函数中的实参变量（根据别名找到主调函数中的本体）。因此，被调函数对形参的任何操作都会影响主调函数中的实参变量。
>
> ​	从编译的角度来讲，程序在编译时分别将指针和引用添加到符号表上，符号表中记录的是变量名及变量所对应的地址。指针变量在符号表中对应的地址值为指针变量的地址值，而引用在符号表上对应的地址值为引用对象的地址值（与实参名字不同，地址相同）。符号表生成之后就不会再改，因此指针改变其指向的对象（指针变量中的值可以改），而引用对象则不能修改。

## 4 C++中`const`和`static`关键字

> **static作用：控制变量的存储方式和可见性**
>
> **作用一：修饰局部变量：**一般情况下，对于局部变量在程序中是存放在栈区的，并且局部的生命周期在包含语句块结束时便结束了。但是如果用`static`关键字修饰的话，该变量便会存放在静态数据区，其生命周期就会延续至整个程序执行结束。但是要注意的是，虽然用`static`对局部变量进行修饰之后，其生命周期以及存储空间发生了变化，但其作用域并没有改变，作用域还是限制在其语句块中。
>
> **作用二：修饰全局变量**：对于一个全局变量，它既可以在本文件中访问到，也可以在同一个工程中其它源文件被访问（添加`extern`进行声明即可）。用`static`对全局变量进行修饰改变了其作用域范围，由原来的整个工程可见变成了本文件可见。
>
> **作用三：修饰函数：**用`static`修饰函数，情况和修饰全局变量类似，也是改变了函数的作用域。
>
> **作用四：修饰类：**如果C++中对类中的某个函数用`static`修饰，则表示该函数属于一个类而不是此类的任何特定对象；如果对类中某个变量进行`static`进行`static`修饰，则表示该变量以及所有的对象所有，存储空间中只存在一个副本，可以通过类和对象去调用。静态非常量数据成员，其只能在类外定义和初始化，在类内仅是声明而已。
>
> **作用五：类成员/类函数声明`static`：**
>
> * 函数体内`static`变量的作用范围为该函数体，不同于`auto`变量，该变量的内存只被分配一次，因此其值在下一次调用时仍维持上次的值。
> * 在类中的`static`成员变量属于整个类所拥有，对类的所有对象只有一份。
> * 在类中的`static`成员函数属于整个类所拥有，这个函数不接收`this`指针，因而只能访问类的`static`成员变量。
> * `static`对象必须要在类外进行初始化，`static`修饰的变量优先于对象存在，所以`static`修饰的变量要在类外初始化。
> * `static`成员函数不能被`virtual`修饰，`static`成员不属于任何对象或实例，所以加上`virtual`没有任何实际意义；静态成员函数没有`this`指针，虚函数的实现是为每个对象分配一个`vptr`指针，而`vptr`是通过`this`指针调用的，所以不能为`virtual`。虚函数的调用关系是`this->vptr->ctable->virtual function`。
>
> **const关键字：含义及实现机制**：
>
> **const修饰基本数据类型：**基本数据类型，修饰符 const 可以⽤在类型说明符前，也可以⽤在类型说明符后， 其结果是⼀样的。在使⽤这些常量的时候，只要不改变这些常量的值即可。
>
> **const修饰指针变量和引用变量**：如果 const 位于⼩星星的左侧，则 const 就是⽤来修饰指针所指向的变量，即指 针指向为常量；如果 const 位于⼩星星的右侧，则 const 就是修饰指针本身，即指针本身是常量。
>
> **const应用到函数中：**作为参数的const修饰符：调用函数的时候，用相应的初始化const常量，则在函数体中，按照const所修饰的部分进行常量化，保护了原对象的属性。参数const通常用于参数为指针或引用的情况；作为函数返回值的const修饰符：声明了返回值后，const按照修饰原则，起到了相应的保护作用。
>
> **const在类中的用法：**const成员变量，只在某个对象生命周期内是常量，而对于整个类而言是可以改变的。因为类可以创建多个对象，不同的对象其const数据成员值可以不同。所以不能在类的声明初始化const数据成员，因为类的对象在还没创建的时候，编译器不知道const数据成员的值是什么。const数据成员的初始化只能在类的构造函数的初始化列表中进行。const成员函数：const成员函数的主要目的是防止函数修改对象的内容。要注意，const关键字和static关键字对于成员函数来说不能同时使用的，因为static关键字修饰的静态成员函数不含有this指针，即不能实例化，const成员函数又必须具体到某一个对象。
>
> **const修饰类对象，定义常量对象：**常量对象只能调用常量函数，别的成员函数都不能调用。
>
> **C++中的const类成员函数（用法和意义）：**常量对象可以调用类中的const成员函数，但不能调用非const成员函数；（原因：对象调用成员函数时，在形参列表的最前面加了一个形参this，但这是隐式的。this指针式默认指向调用函数的当前对象的，所以，很自然，this是一个常量指针`test *const`，因为不可以修改this指针代表的地址。但成员函数的参数列表（即小括号）后加了const关键字`(void print() const;)`，此成员函数就成为了常量成员函数，此时它的隐式this形参变为了`const test* const`，即不可以通过this指针来改变指向的对象的值。非常量对象可以调用类中的const成员函数，也可以调用非const成员函数。

## 5 C和C++的区别（函数/类/struct/class）

> ​	C++有新增的语法和关键字，语法的区别有头文件的不同和命名空间的不同，C++允许我们定义自己的空间，C中不可以。关键字方面比附C++与C动态管理内存的方式不同，C++中在`malloc`和`free`的基础上增加了`new`和`delete`，而且C++中在指针的基础上增加了引用的概念，关键字例如C++还增加了`auto`，`explicit`体现显式和隐式转换的概念要求，还有`dynamic_cast`增加了类型安全方面的内容。
>
> **函数方面C++中有重载和虚函数的概念：**C++支持函数重载而C不支持，是因为C++函数的名字修饰与C不同，C++函数名字的修饰会将参数加在后面，例如，`int func(int,double)`经过名字修饰后变成了`_func_int_double`，而C中则会变成`_func`，所以C++中会支持不同参数调用不同函数。
>
> **C++还有虚函数概念，用以实现多态**。
>
> **类方面，C的struct和C++的类也有很大不同：**C++中的struct不仅可以有成员变量还可以有成员函数，而且对于struct增加了权限访问的概念，struct的默认成员访问权限和默认继承权限都是public，C++中除了struct还有class表示类，struct和class还有一点不同在于class的默认成员访问权限和默认继承权限都是private。
>
> **C++中增加了模板重用代码，提供了更加强大的STL标准库**
>
> C是一种结构化的语言，重点在于算法和数据结构。C程序的设计首先考虑的是如何通过一个代码，一个过程对输入进行运算处理输出。而C++首先考虑的是如何构造一个对象模型，让这个模型能够契合与之对应的问题领域，这样就能通过获取对象的状态信息得到输出。

## 6 说一下C++里是怎么定义常量的？常量存放在内存的哪个位置？

> ​	对于局部常量，存放在栈区；
>
> ​	对于全局常量，编译期一般不分配内存，放在符号表中以提高访问效率；
>
> ​	字面值常量，比如字符串，放在常量区。

## 7 C++中重载和重写，重定义的区别

> **重载**
>
> ​	翻译自overload，是指同一可访问区内被声明的几个具有不同参数列表的同名函数，依赖于函数名字的修饰会将参数类型加在后面，可以是参数类型的个数或顺序不同。根据参数列表决定调用哪个函数，重载不关心函数的返回类型。
>
> **重写（覆盖）**
>
> 翻译自override，派生类中重新定义父类除了函数体外完全相同的虚函数，注意被重写的函数不能是static的，一定要是虚函数，且其他一定要完全相同。要注意，重写和被重写的函数是在不同的类当中的，重写函数的访问修饰符可以是不同的，尽管virtual中是private的，派生类中重写可以改为public。
>
> **重定义（隐藏）**
>
> 派生类重新定义父类中相同名字的非virtual函数，参数列表和返回值类型都可以不同，即父类中除了定义成virtual且完全相同的同名函数才不会被派生类中的同名函数所隐藏（重定义）。

## 8 符号表的作用

> ​	编译器会生成一个叫做”符号表“的数据结构来维护变量名和内存地址直接的对应关系。它会搜集变量名，比如我们定义一个全局变量`int a;`，那么编译器会为程序预留4个字节（32位平台）的空间，比如起始地址为23456788（长度为4），并把变量名"a"和地址23456788保存到符号表，这样程序中对a进行相关操作时，它就会根据变量找到真正的地址位置（23456788），进行相关的操作。在机器执行程序的时候，会把变量名替换为内存地址（和长度），而不存在任何名称。

## 9 介绍C++所有的构造函数

> ​	类的对象被创建时，编译系统为对象分配内存空间，并自动调用构造函数，由构造函数完成成员的初始化工作。即构造函数的作用：初始化对象的数据成员。
>
> ​	**无参构造函数**：即默认构造函数，如果没有明确写出无参构造函数，编译器会自动生成默认的无参构造函数，函数体为空，什么也不做，如果不想使用自动生成的无参构造函数，必须要自己手动写出一个构造函数。
>
> ​	**一般构造函数：**也称重载构造函数，一般构造函数可以有各种参数形式，一个类可以有多个一般构造函数，前提是参数的个数或者类型不同，创建对象时根据传入参数不同调用不同的构造函数。
>
> ​	**拷贝构造函数：**拷贝构造函数的函数参数为对象本身类型的引用，用于根据一个已经存在的对象复制出一个新的该类的对象，一般在函数中会将已存在的对象的数据成员的值一一复制到新创建的对象中。如果没有显式的写拷贝构造函数，则系统会默认创建一个拷贝构造函数，但当类中有指针成员时，最好不要使用编译器提供的默认的拷贝构造函数。最好自己定义并且在函数中执行深拷贝。
>
> ​	**类型转换构造函数：**根据一个指定类型的对象创建一个本类的对象，也可以算是一般构造函数的一种，这里提出来，是想说有的时候不允许默认转换的话，要记得将其声明为`explict`的，来阻止一些隐式转换的发生。
>
> ​	**赋值运算符的重载：**注意，这个类似拷贝构造函数，将=右边的本类对象的值复制给=左边的对象，它不属于构造函数，=左右两边的对象必须已经被创建。如果没有显式的写赋值运算符的重载，系统也会生成默认的赋值运算符，做一些基本的拷贝工作。
>
> 这里区分：
>
> ```c++
> A a1, A a2; a1 = a2;	//调用赋值运算符
> A a3 = a1; //调用拷贝构造函数，因为进行的是初始化工作，a3并未存在
> ```

## 10 C++的四种强制类型转换

[C++ static_cast、dynamic_cast、const_cast和reinterpret_cast（四种类型转换运算符） (biancheng.net)](http://c.biancheng.net/view/2343.html)

### 10.1 `static_cast`

> **static_cast**
>
> 用法：`static_cast<type-id>(expression)`
>
> 该运算符把expression转换为type-id类型，**但没有运行时类型检查来保证转换的安全性。**
>
> * 编译器隐式执行的任何类型转换都可以由`static_cast`来转换，比如`int`与`float`、`double`与`char`、`enum`与`int`之间的转换等。
>
>   * > ```c++
>     > double a = 1.999;
>     > int b = static_cast<double>(a);
>     > ```
>     >
>     > ​	当编译器隐式执行类型转换时，大多数的编译器都会给出一个警告，使用`static_cast`可以明确告诉编译器，这种损失精度的转换是在知情的情况下进行的，也可以让阅读程序的其他程序员明确你转换的目的不是由于疏忽。把精度大的类型转换为精度小的类型，`static_cast`使用截断处理。
>
> * `static_cast`可以找回存放在`void*`指针中的值。
>
>   * ```c++
>     double a = 1.999;
>     void *vptr = &a;
>     double *dptr = static_cast<double*>(vptr);
>     cout << *dptr << endl;	//输出1.999
>     ```
>
> * `static_cast`也可以用于在基类与派生类指针或引用类型之间的转换。
>
>   * > ​	然而它不做运行时的检查，不如`dynamic_cast`安全。`static_cast`仅仅是依靠类型转换语句中提供的信息来进行转换，而`dynamic_cast`则会遍历整个类继承体系进行类型检查，因此`dynamic_cast`在执行效率上比`static_cast`要差一些。注意！！！**从下向上转换时安全的，从上向下的转换不一定安全。**
>     >
>     > 现在我们有父类与其派生类如下：
>     >
>     > ```c++
>     > #include <iostream>
>     > 
>     > using namespace std;
>     > 
>     > class ANIMAL {
>     > public:
>     > 	ANIMAL() : _type("ANIMAL"){}
>     > 	virtual void OutPutname(){ cout << "ANIMAL" << endl;}
>     > private:
>     > 	string _type;
>     > };
>     > 
>     > class DOG : public ANIMAL{
>     > public:	
>     > 	DOG() : _name("大黄"), _type("DOG"){}
>     >     void OutPutname() { cout << _name << endl; }
>     > 	void OutPuttype(){ cout << _type << endl; }
>     > private:
>     > 	string _name;
>     > 	string _type;
>     > };
>     > 
>     > int main(){
>     >     //基类指针转为派生类指针，且该基类指针指向基类对象
>     >     ANIMAL * ani1 = new ANIMAL;
>     >     DOG* dog1 = static_cast<DOG*>( ani1 );
>     >     dog1->OutPuttype();   //错误，在运行时出现错误
>     > 
>     >     //基类指针转换为派生类指针，且该基类指针指向派生类对象
>     >     ANIMAL* ani3 = new DOG;
>     >     DOG* dog3 = static_cast<DOG*>( ani3 );
>     >     dog3->OutPuttype(); //正确
>     > 
>     >     //子类指针转为派生类指针
>     >     DOG* dog2 = new DOg;
>     >     ANIMAL* ani2 = static_cast<DOG*>( dog2 );
>     >     ani2->OutPutname(); //正确
>     > }
>     > ```
>
> * 另外，与`const_cast`相比，`static_cast`不能转换掉变量的`const`属性，也包括`volitale`或者`__unaligned`属性。

### 10.2 `dynamic_cast`

> ​	`dynamic_cast`用于在类的继承层次之间进行类型转换，它既允许向上转型，也允许向下转型。向上转型是无条件的，不会进行任何检测，所以都能成功。向下转型的前提必须是安全的，要借助RTTI进行检测。`dynamic_cast`是最常用的RTTI组件，只适用于**包含虚函数的类**的继承关系间的**指针或引用**间的转换。
>
> > 哪些转换是安全的？
> >
> > 当使用 dynamic_cast 对指针进行类型转换时，程序会先找到该指针指向的对象，再根据对象找到当前类（指针指向的对象所属的类）的类型信息，并从此节点开始沿着继承链向上遍历，如果找到了要转化的目标类型，那么说明这种转换是安全的，就能够转换成功，如果没有找到要转换的目标类型，那么说明这种转换存在较大的风险，就不能转换。
> >
> > 假设有下面这样的类层次结构：
> >
> > ```c++
> > class Grand { // has virtual methods
> > };
> > class Superb : public Grand{...};
> > class Magnificent : public Superb {...};
> > 
> > Grand *pg = new Grand;
> > Grand *ps = new Superb;
> > Grand *pm = new Magnificent;
> > 
> > Magnificent *p1 = (Magnificent*) pg;	//此向下转换不安全，因为它将基类对象的地址给派生类指针
> > Superb *p2 = (Magnificent*) pm; //向上转换安全
> >   
> > //dynamic_cast的语法
> > Superb *pm = dynamic_cast<Superb*>(pg);
> > ```
> >
> > ​	通常情况下，如果指向对象`(*pt)`的类型为`Type`或者是从`Type`直接或间接派生而来的类型，则下面的表达式是安全的，可以成功将`pt`转换为`Type`类型的指针。否则将返回空指针。
> >
> > ```c++
> > dynamic_cast<Type*>(pt);
> > 
> > Magnificent *p2 = dynamic_cast<Magnificent*> ps;	//向下转换，*ps是Superb类型的对象，本质上依旧是向下转换，不安全，返回nullptr
> > 
> > Superb *p3 = dynamic_cast<Superb*> ps; //向下转换，*ps是Superb类型的对象，本质上是同级转换，安全
> > ```
> >
> > 当使用`dynamic_cast`来转换引用时，规则相似，但由于没有空指针对应的引用值，因此无法使用特殊的引用值来指示失败。当转换不安全时，`dynamic_cast`将引发类型为`bad_cast`的异常，这种异常时从`exception`类派生而来的，它是在头文件`typeinfo`中定义的。

## 11 指针和引用的区别

> ​	指针和引用都是一种内存地址的概念，区别呢，指针是一个实体，引用只是一个别名。
>
> 在程序编译的时候，将指针和引用都添加到符号表中。指针它指向一块内存，指针的内容是所指向的内存的地址，在编译的时候，则是将”指针变量名-指针变量的地址”添加到符号表中，所以说，指针包含的内容是可以改变的，允许拷贝和赋值，有const和非const区别，甚至可以为空，`sizeof`指针得到的是指针类型的大小。
>
> ​	而对于引用来说，它只是一块内存的别名，在添加到符号表的时候是将“引用变量名-引用对象地址”添加到符号表中，符号表一经完成不能改变，所以引用必须而且只能在定义时被绑定到一块内存上，后续不能更改，也不能为空，也没有`const`和非`const`的区别。
>
> ​	`sizeof`引用得到代表对象的大小。而`sizeof`指针得到的是指针本身的大小。另外在参数传递中，指针需要被解引用后才可以对对象进行操作，而直接对引用进行的修改会直接作用到引用对象上。
>
> ​	作为参数时也不同，传指针的实质是传值，传递的值是指针中保存的地址；传引用的实质是传地址，传递的是变量的地址。

## 12 野（wild）指针与悬空（dangling）指针有什么区别？如何避免？

> ​	野指针（wild pointer）：就是没有被初始化过的指针。用`gcc -Wall`编译，会出现`used uninitialized`警告。
>
> ​	悬空指针：是指针最初指向的内存已经被释放了的一种指针。
>
> 无论是野指针还是悬空指针，都是指向无效内存区域（这里的无效指的是“不安全不可控”）的指针。访问“不安全不可控”的内存区域将导致 未定义行为。
>
> ​	如何避免使用野指针？在平时的编码中，养成在定义指针后再使用之前完成初始化的习惯或者使用智能指针。

## 13 说一下const修饰指针如何区分？

> ```c++
> const int *p1;	//指向整型常量，它指向的值不能修改
> int* const p2;	//指向整型的常量指针，它不能再指向别的变量，但指向（变量）的值可以修改。
> const int* const p3;	//指向整型常量的常量指针，它既不能再指向别的常量，指向的值也不能修改。
> ```

## 14 简单说一下函数指针

> ​	定义：函数指针是指向函数的指针变量。函数指针本身首先是一个指针变量，该指针变量指向一个具体的函数。这正如用指针变量可指向整型变量，字符型、数组一样，这里是指向函数。
>
> ​	在编译时，每一个函数都有一个人口地址，该入口地址就是函数指针所指向的地址。有了指向函数的指针变量后，可用该指针变量调用函数，就如同用指针变量可引用其它类型变量一样。
>
> ​	其次是用途：调用函数和做函数的参数，比如回调函数。
>
> ```c++
> char *fun(char *p) {...} //函数fun
> char *(*pf)(char* p); //函数指针pf
> pf = fun;	//函数指针pf指向函数fun
> pf(p);		//通过函数指针pf调用函数fun
> ```

## 15 堆和栈的区别

> **栈**
>
> 由编译器进行管理，在不需要的时候自动回收空间，一般保存的是局部变量和函数参数等。
>
> 连续的内存空间，在函数调用的时候，首先入栈的主函数的下一条可执行指令的地址（记录函数返回地址），然后是函数的各个参数。大多数编译器中，**参数是从右往左入栈**（原因在于采用这种顺序，是为了让程序员在使用C/C++的“函数参数长度可变”这个特性时更方便。如果是从左往右压栈，第一个参数（即描述可变参数表各变量类型的那个参数）将被放在栈底，由于可变参的函数第一步就需要解析可变参数表的各参数类型，即第一步就需要得到上述参数，因此，将它放在栈底是很不方便的。）本次函数调用结束时，局部变量先出栈，然后是参数，最后是栈顶指针最开始存放的地址，程序由该点继续运行，不会产生碎片。
>
> 栈是高地址向低地址扩展，空间较小。
>
> **堆**
>
> 由程序员管理，需要手动`new malloc delete free`进行分配和回收，如果不进行回收的话，会造成内存泄漏的问题。不连续的空间，实际上系统中有一个空闲链表，当有程序申请的时候，系统遍历空闲链表找到第一个大于等于申请大小的空间分配给程序，一般在分配的程序的时候，也会在空间的头部写入内存大小，方便`delete`回收空间大小。当然如果有剩余的，也会将剩余的插入到空闲链表中，这也是产生内存碎片的原因。
>
> 堆是低地址向高地址扩展，空间较大，较为灵活。

## 16 `new / delete, malloc/free`区别

> ​	都可以用来在堆上分配和回收空间。`new / delete`是操作符，`malloc / free`是库函数。
>
> ​	执行`new`实际上执行两个过程：1）分配未初始化的内存空间（`void* operator new()`）；2）使用对象的构造函数对空间进行初始化；返回空间的首地址。如果在第一步分配空间中出现问题，则会抛出`std::bad_alloc`异常，或被某个设定的异常处理函数捕获处理；如果在第二步构造函数时出现异常，则自动调用`delete`释放内存。
>
> ​		执行`delete`实际上也有两个过程：1）使用析构函数对对象析构；2）回收内存空间（`operator delete()`）。
>
> ​	以上也可以看出`new`和`malloc`的区别，`new`得到的时经过初始化的空间，而`malloc`得到的时未初始化的空间。所以`new`是`new`一个类型，而`malloc`则是`malloc`若干字节长度的空间。`delete`和`free`同理，`delete`和`free`同理，`delete`不仅释放空间还析构对象，`delete`一个类型，`free`若干字节长度的空间。
>
> ​	为什么有了`malloc / free`还需要`new / delete`，因为对于非内部数据类型而言，光用`malloc / free`无法满足动态对象的要求。对象在创建的同时需要自动执行构造函数，对象消亡以前要自动执行析构函数。由于`malloc / free`是库函数而不是运算符，不在编译器控制权限之内，不能够把执行的构造函数和析构函数的任务强加于`malloc / free`，所以有了`new / delete`操作符。 

## 17 `volatile`和`extern`关键字

> ​	**`volatile`三个特性**
>
> ​	易变性：在汇编层面反映出来，就是两条语句，下一条语句不会直接使用上一条语句对应的`volatile`变量的寄存器中的内容，而是重新从内存中读取。
>
> ​	不可优化性：`volatile`告诉编译器，不要对我这个变量进行各种激进的优化，甚至直接消除，保证程序员写在代码中的指令，一定会被执行。
>
> ​	顺序性：能够保证`volatile`变量直间的顺序性，编译器不会乱序优化。
>
> **`extern`**
>
> ​	在C语言中，修饰符`extern`用在变量或者函数的声明前，用来说明”此变量/函数是在别处定义的，要在此处引用“。
>
> ​	注意`extern`声明的位置对其作用域也有关系，如果是在`main`函数中进行声明的，则只能在`main`函数中调用，在其它函数中不能调用。
>
> ​	在C++中`extern`还有另外一种作用，用于指示C或者C++函数的调用规范。比如在C++中调用C库函数，就需要在C++程序中使用`extern C`声明要引用的函数。这是给链接器用的，告诉链接器在链接的时候用C函数规范来链接。主要原因是C++和C程序在编译完成后目标代码中的命名规则不同，用此来解决名字匹配的问题。

## 19 `define` 和`const`区别

> ​	对于`define`来说，宏定义实际上是预编译阶段进行处理，没有类型，也就没有类型检查，仅仅做的是遇到宏定义就进行字符串展开，遇到多少次就展开多少次，而且这个简单的展开过程中，很容易出现边界效应，达不到预期的效果。因为`define`宏定义仅仅是展开，因此运行时系统并不为宏定义分配内存，但是从汇编的角度来讲，`define`却以立即数的方式保留了多份数据的拷贝。
>
> ​	对于`const`来说，`const`是在编译期间进行处理的，`const`是有类型，也有类型检查的，程序运行时系统会为`const`常量分配内存，而从汇编的角度讲，`const`常量出现的地方保留的是真正数据的内存地址，只保留了一份数据的拷贝，省去了不必要的内存空间。而且，有时编译器不会为普通的`const`常量分配内存，而是直接将`const`常量添加到符号表中，省去了读取和写入内存的操作，效率更高。

## 20 计算下面几个类的大小

> ```c++
> class A{}; sizeof(A) = 1; //空类在实例化时得到一个独一无二的地址，所以为1
> class A{virtual Fun(){}};	sizeof(A) = 4(32bit)/8(64bit);//当C++类中有虚函数的时候，会有一个指向虚函数表的指针
> class A{static int a;}; sizeof(A) = 1; 
> class A{int a;} sizeof(A) = 4;
> class A{static int a; int b;} sizeof(A) = 4;
> ```

## 21面向对象的三大特性

> C++面向对象的三大特征是：封装、继承、多态。
>
> **封装**
>
> 就是把客观事务封装成抽象的类，并且类可以把自己的数据和方法只让信任的类或对象操作，对不可信的进行信息隐藏。一个类就是一个封装了数据以及操作这些数据的代码的逻辑实体。在一个对象内部，某些代码或某些数据可以是私有的，不能被外界访问。通过这种方式，对象对内部数据提供了不同级别的保护，以防止程序中无关的部分意外的改变或错误的使用了对象的私有部分。
>
> **继承**
>
> 是指可以让某个类型的对象获得另外一个对象的属性的方法。它支持按级别分类的概念。继承是指这样一种能力：它可以使用现有类的所有功能，并在无需重新编写原来类的情况下对这些功能进行扩展。通过继承创建的新类称为“子类”或“派生类”，被继承的类称为“基类”、“父类”或“超类”。继承的过程，就是从一般到特殊的过程。要实现继承，可以通过“继承”和“组合”来实现。
>
> 继承概念的实现方式有两类：
>
> 实现继承：实现继承是直接使用基类的属性和方法而无须额外的编码的能力。
>
> 接口继承：接口继承是指仅使用属性和方法的名称，但是子类必须提供实现的能力。
>
> **多态**
>
> 就是向不同的对象发送同一个消息，不同的对象在接收时会产生不同的行为。即一个接口，可以实现多种方法。
>
> 多态与非多态的实质区别就是函数地址是早绑定还是晚绑定的。如果函数的调用，在编译期间就可以确定函数的调用地址，并产生代码，则是静态的，即地址早绑定。而如果函数调用的地址不能在编译器期间确定，需要在运行时才确定，这就属于晚绑定。

## 22 多态的实现

> 多态其实一般就是指继承加虚函数实现的多态，对于重载来说，实际上基于的原理是，编译器为函数生成符号表时的不同规则，重载只是一种语言特性，与多态无关，与面向对象也无关，但这又是C++中增加的新规则，所以也算属于C++，所以如果非要说重载算是多态的一种，那就可以说：多态分为静态多态和动态多态。
>
> 静态多态其实就是重载，因为静态多态是在编译时期就决定了调用哪个函数，根据参数列表来决定；
>
> 动态多态是指通过子类重写父类的虚函数来实现的，因为是在运行时决定调用的函数，所以称为动态多态。
>
> 动态多态的实现与虚函数表、虚函数指针相关。
>
> 扩展：子类是否要重写父类的虚函数？子类继承父类时，父类的纯虚函数必须重写，否则子类也是一个虚类不可实例化。**定义纯虚函数是为了实现一个接口，起到一个规范的作用，规范继承这个类的程序员必须实现这个函数。**

## 23 虚函数相关（虚函数表，虚函数指针），虚函数的实现原理

> ​	C++多态的表象，在基类的函数前加上`virtual`关键字，在派生类中重写该函数，运行时将会根据对象的实际类型来调用相应的函数。如果类型是派生类，就调用派生类的函数，如果是基类，就调用基类的函数。
>
> ​	实际上，当一个类中包含了虚函数时，编译器会为该类生成一个虚函数表，保存该类中虚函数的地址，同样，派生类继承基类，派生类中自然一定有虚函数，所以编译器也会为派生类生成自己的虚函数表。当我们定义一个派生类对象时，编译器检测该类型有虚函数，所以为这个派生类对象生成一个虚函数指针，指向该类型的虚函数表，这个虚函数指针的初始化是在构造函数中完成的。
>
> ​	后续如果有一个基类类型的指针，指向派生类，那么当调用虚函数时，就会根据所指的真正对象的虚函数表指针去寻找虚函数的函数地址，也就可以调用派生类的虚函数表中的虚函数以此实现多态。
>
> 补充：如果基类中没有定义`virtual`，那么进行`Base B; Derived D; Base *p = D; p->function();`这种情况下调用的则是`Base`中的`function()`。因为基类和派生类中都没有虚函数的定义，那么编译器就会认为不用留给多态多态的机会，就事先进行函数地址的绑定（早绑定），详细过程就是，定义了一个派生类对象，首先要构造基类的空间，然后构造派生类的自身内容，形成一个派生类对象，那么进行类型转换时，直接截取基类的部分的内容，编译器认为类型就是基类，那么（函数符号表（不同于虚函数表的另一个表）中绑定的函数地址也就是基类中的函数的地址，所以执行的是基类的函数。

## 24 编译器处理虚函数表应该如何处理

> 对于派生类来说，编译器建立虚函数表的过程其实一共是三个步骤：
>
> * 拷贝基类的虚函数表，如果是多继承，就拷贝每个虚函数的虚函数表。
> * 当然还有一个基类的虚函数表和派生类自身的虚函数表共用了一共虚函数表，也称为某个基类为派生类的主基类。
> * 查看派生类中是否有重写基类中的虚函数，如果有，就替换成已经重写的虚函数地址；查看派生类是否有自己的虚函数，如果有，就追加到自身的虚函数到自身的虚函数表中。
>
> `Derived *pd = new D(); B *pb = pd; C *pc = pd;`其中`pb, pd, pc`的指针位置是不同的，要注意的是派生类的自身的内容是追加在主基类的内存块后。
>
> ![image-20230729142256802](https://pic1.xuehuaimg.com/proxy/https://cdn.jsdelivr.net/gh/moshang1314/myBlog@main/image/image-20230729142256802.png)

## 25 析构函数一般写成虚函数的原因

> ​	直观的讲：是为了降低内存泄漏的可能性。举例来说就是，一个基类的指针指向一共派生类的对象，在使用完毕准备销毁时，如果基类的析构函数没有定义成虚函数，那么编译器根据指针类型就会认为当前对象的类型是基类，调用基类的析构函数（该对象的析构函数的函数地址早就被绑定成了基类的析构函数），仅执行基类的析构函数，派生类的自身内容将无法被析构，造成内存泄漏。
>
> ​	如果基类的析构函数定义成虚函数，那么编译器就可以根据实际对象类型，执行派生类的析构函数，再执行基类的析构函数，成功释放内存。

## 26 构造函数为什么一般不定义为虚函数

> * 虚函数调用只需要知道“部分的”信息，即只需要知道函数接口，而不需要知道对象的具体类型。但是，我们要创建一个对象的话，是需要知道对象的完整类型信息的。特别是，需要知道要创建对象的确切类型，因此，构造函数不应该被定义为虚函数。
> * 而且从目前编译器实现虚函数进行多态的方式来看，虚函数的调用是通过实例化之后的对象的虚函数表指针来找到虚函数的地址进行调用的，如果说构造函数是虚的，那么虚函数表指针则是不存在的，无法找到对应的虚函数表来调用虚函数，那么这个调用实际上也是违反了先实例化后调用的准则。

## 27 构造函数或析构函数中调用虚函数会怎样

> ​	实际上是不应该在构造函数或析构函数中调用虚函数的，因为这样的调用其实并不会带来所想要的效果。
>
> 举例来说就是，有一个动物的基类，基类中定义了一个动物本身行为的虚函数`action_type()`，在基类的构造函数中调用了这个虚函数。
>
> 派生类中重写了这个虚函数，我们期望着根据对象的真实类型不同，而调用各自实现的虚函数，但实际上当我们创建一个派生类对象时，首先会创建派生类的基类部分，执行基类的构造函数，此时，派生类自身部分还没有被初始化，对于这种还没有初始化的东西，C++选择当它们还不存在作为一种安全的方法。
>
> ​	也就是说构造派生类的基类部分是，编译器会认为这就是一个基类类型的对象，然后调用基类类型中的虚函数实现，并没有按照我们想要的方式进行。即对象在派生类构造函数执行前并不会成为一个派生类对象。
>
> ​	在析构函数中也是同理，派生类执行了析构函数后，派生类的自身成员呈现未定义的状态，那么在执行基类的析构函数中是不可能调用到派生类重写的方法的。所以说，我们不应该在析构函数或构造函数中调用虚函数，就算调用一般也不会达到我们想要的结果。

## 28 析构函数的作用，如何起作用

> ​	构造函数只是起初始化值的作用，但实例一个对象的时候，可以通过实例去传递参数，从主函数传递到其它函数里面，这样就使其它的函数里面有值了。规则，只要你一实例化对象，系统自动回调一个构造函数，就是你不写，编译器也会自动调用一次。
>
> ​	析构函数与构造函数的作用相反，用于撤销对象的一些特殊任务处理，可以是释放对象分配的内存空间；特点：析构函数与构造函数同名，但该函数前面加~。
>
> ​	析构函数没有参数，也没有返回值，而且不能重载，在一个类中只能有一个析构函数。当撤销对象时，编译器也会自动调用析构函数。每个类必须有一个析构函数，用户可以自定义析构函数，也可以是编译器自动生成默认的析构函数。一般析构函数定义为类的公有成员。

## 29 构造函数的执行顺序？析构函数的执行顺序？

> **构造函数顺序**
>
> * 基类构造函数。如果有多个基类，则构造函数的调用顺序是某类在类派生表中出现的顺序，而不是它们在成员初始化列表中的顺序。
> * 成员类对象构造函数。如果有多个成员对象则构造的调用顺序是对象在类中被声明的顺序，而不是它们出现在成员初始化表中的顺序。
> * 派生类构造函数
>
> **析构函数顺序**
>
> * 调用派生类的析构函数
> * 调用成员类对象的析构函数
> * 调用基类的析构函数

## 30 纯虚函数（应用于接口继承和实现继承）

> ​	实际上，纯虚函数的出现就是为了让继承可以出现多种情况：
>
> * 有时我们希望派生类只继承成员函数的接口
> * 有时我们又希望派生类既继承成员函数的接口，又继承成员函数的实现，而且可以在派生类中重写成员函数以实现多态。
> * 有时我们又希望派生类在继承成员函数接口和实现的情况下，不能重写缺省的实现。
>
> ​    其实，声明一个纯虚函数的目的就是为了让派生类只继承函数的接口，而且派生类中必须提供一个这个纯虚函数的实现，否则含有纯虚函数的类将是抽象类，不能进行实例化。
>
> ​     对于纯虚函数来说，我们其实是可以给它提供实现代码的，但是由于抽象类不能实例化，调用这个实现的唯一方式就是在派生类对象中指出其`class`名称来调用。

## 31 静态绑定和动态绑定的介绍

> ​	静态类型就是它在程序中被声明时所采用的类型，在编译期间确定。动态类型则是指目前所指对象的实际类型，在运行时确定。静态绑定，又名早绑定，绑定的是静态类型，所对应的函数或属性依赖于对象的静态类型，发生在编译期间。动态绑定，又名晚绑定，绑定的是动态类型，所对应的函数或属性依赖于动态类型，发生在运行期间。
>
> ​	比如说，`virtual`函数是动态绑定的，非虚函数是静态绑定的，缺省参数值也是静态绑定的。这里呢，就需要注意，我们不应该重新定义继承而来的缺省参数，因为即使我们重定义了，也不会起到效果。因为一个基类的指针指向一个派生类对象，在派生类的对象中针对虚函数的缺省值进行了重定义，**但是缺省参数值是静态绑定的，静态绑定的是静态类型相关的内容**，所以会出现一种派生类的虚函数实现方式结合了基类的缺省参数值的调用效果，这个与期望效果不同。

## 32 深拷贝和浅拷贝的区别（举例说明深拷贝的安全性）

> ​	当实现类的等号赋值时，会调用拷贝函数，在未定义显式拷贝构造函数的情况下，系统会调用默认的拷贝函数——即浅拷贝，它能够完成成员的一一复制。当数据成员中没有指针时，浅拷贝时安全的。
>
> ​	但当数据成员中又指针时，如果采用简单的浅拷贝，则两类中的两个指针指向同一个地址，当对象快要结束时，会调用两次析构函数，而导致野指针的问题。
>
> ​	所以，这时必需采用深拷贝。深拷贝和浅拷贝之间的区别就在于深拷贝会在堆内存中申请另外的空间来存储数据，从而解决了野指针问题。简而言之，当数据成员中有指针时，必需采用深拷贝更加安全。

## 33 什么情况下会调用拷贝构造函数（三种情况）

> ​	类的对象需要拷贝时，拷贝构造函数将会被调用，以下情况都会调用拷贝构造函数：
>
> * 一个对象以值传递的方式传入函数，需要拷贝构造函数吵架呢一个临时对象压入到栈空间中。
> * 一个对象以值传递的方式从函数返回，需要执行拷贝构造函数创建一个临时对象作为返回值。
> * 一个对象需要通过另外一个对象进行初始化

## 34 为什么拷贝构造函数必需使用引用传递，不能是值传递？

> ​	为了防止递归调用。当一个对象需要以值进行传递时，编译器会生成代码调用它的拷贝构造函数生成一个副本，如果类A的拷贝构造函数的参数不是引用传递，而是采用值传递，那么就又需要为了创建传递给拷贝构造函数的参数的临时对象，而又一次调用类A的拷贝构造函数，这就是一个无限递归。

## 35 结构体内存对齐方式和为什么要进行内存对齐

> 对齐规则：
>
> * 对于结构体中的各个成员，第一个成员位于偏移为0的位置，以后的每个数据成员的偏移量必须时`min(#pragma pack()指定的对齐数，数据成员本身长度)`的倍数。
> * 在所有的数据成员完成各自对齐之后，结构体或联合体本身也要进行对齐，整体长度是`min(#pragma pack()指定的对齐数， 长度最长的数据成员的长度)`的倍数。
>
> 内存对齐的作用是什么？
>
> * 结果内存对齐之后，CPU的内存访问速度大大提升。因为CPU把内存当成是一块一块的，块的大小可以是2、4、8、16字节，因此CPU在读取内存的时候是一块一块进行读取的，块的大小被称为内存读取粒度。比如说CPU要读取一个4个字节的数据到寄存器（假设内存的读取粒度是4），如果数据是从0字节开始的，那么直接价格呢0~3四个字节的完全读到寄存器中进行处理即可。
> * 如果数据是从1字节开始的，就首先将前4个字节读取到寄存器，并再次读取4~7个字节数据到寄存器，接着把0字节，5，6，7字节的数据提出，最后合并1，2，3，4字节的数据加入寄存器，所以说，当内存没有对齐时，寄存器进行了很多额外的操作，大大降低了CPU的性能。
> * 另外，还有一个就是，有的CPU遇到未进行内存对齐的处理直接拒绝处理，不是所有的硬件平台都能访问任意地址上的任意数据，某些硬件平台只能在某些地址取某些特定类型的数据，否则抛出异常。所以内存对齐还有利于平台移植。

## 36 内存泄漏的定义，如何检测与避免

> ​	定义：内存泄漏简单的说就是申请了一块内存空间，使用完毕后没有释放掉。它的一般表现方式是程序运行时间越长，占用内存越多，最终耗尽全部内存，整个系统崩溃。由程序申请的一块内存，且没有任何一个指针指向它，那么这块内存就泄漏了。
>
> **如何检测内存泄漏**
>
> * 首先可以通过观察猜测是否可能发生内存泄漏，Linux中使用`swap`命令观察还有多少交换空间，在一两分钟内键入该命令三到四次，看看可用的交换区是否在减少。
> * 还可以使用其他一些`/usr/bin/stat`工具如`netstat`、`vmstat`等。如发现波段有内存被分配且从不释放，一个可能的解释就是有个进程出现了内存泄漏。 
> * 当然也有用于内存调试，内存泄漏检测以及性能分析的软件开发工具`valgrind`这样的工具来进行内存泄漏的检测。

## 37 说一下平衡二叉树、高度平衡二叉树（AVL）

> ​	二叉树：任何节点最多只允许有两个子节点，称为左子节点和右子节点，以递归的方式定义二叉树为，一个二叉树如果不为空，便是由一个根节点和左右两个子树构成，左右子树都可能为空。
>
> ​	二叉搜索树：二叉搜索树可以提供对数时间复杂度的元素插入和访问。节点的放置规则是：任何节点的键值一定大于其左子树的每一个节点的键值，并小于其右子树中的每一个节点的键值。因此一直向左走可以取得最小值，一直往右走可以得到最大值。插入：从根节点开始，遇键值较小则向右，与键值较大则向左，直到尾端，即插入点。删除：如果删除点只有一个子节点，则直接将其子节点连至父节点。如果删除点有两个子节点，以右子树的最小值代替要删除的位置。
>
> ​	平衡二叉树：其实对于树的平衡与否没有一个绝对的标准，“平衡”的大致意思是：没有任何一个节点过深，不同的平衡条件会造就出不同的效率表现。以及不同的实现复杂度。有种特殊的结构如AVL-tree、RB-tree、AA-tree，均可以实现平衡二叉树。
>
> ​	AVL-tree：高度平衡的二叉树（严格的平衡二叉树）AVL-tree是要求任何节点的左右子树的高度相差做多为1的平衡二叉树。当插入的新节点破坏了平衡性，从下往上找到第一个不平衡点，需要进行单旋转，或者双旋转进行调整。

