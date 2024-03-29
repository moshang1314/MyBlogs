> ![image-20230530105820911](https://pic1.xuehuaimg.com/proxy/https://cdn.jsdelivr.net/gh/moshang1314/myBlog@main/image/image-20230530105820911.png)

# 1 类中的const成员

> 1) **一个const对象不能调用非const成员函数**，即使成员函数并没有改变对象成员的值，编译器也会认为其改变了对象。所以要想调用哪个函数，就只能把那个函数设成const函数，也就是在函数后加const。
> 2) **const成员函数内部不能调用非const成员函数。**

# 2 输入缓冲区

> （1）当用户在键盘上输入数据时，数据首先被传递给操作系统，**并存储在操作系统内核的键盘输入缓冲区中。用户可以编辑数据，移动光标或删除字符，直到确认输入完成而按下 "Enter " 键**。此时，操作系统会将缓冲区中的数据**移动到正在等待输入的进程的输入缓冲区中**。当C++程序调用 `std::cin` 对象进行输入操作时，数据会从这个输入缓冲区中移动到程序的输入缓冲区中。当 `std::cin` 对象读取完缓冲区中的数据后，如果还需要更多的输入数据，它将阻塞等待，直到操作系统将更多的数据传输到输入缓冲区中。**在这个过程中，用户不能直接修改程序的输入缓冲区中的数据，只有在将数据传输到程序中之前，用户才可以输入和编辑数据**。
>
> （2）在输入过程中的**内核缓冲区对应的文件描述符就是标准输入描述符**。在Linux系统中，标准输入（stdin）是一个由操作系统内核维护和管理的文件描述符（文件句柄），**它是所有从标准输入流中输入的字符数据的源头**。标准输入与任何其他文件描述符一样，都是一个整数类型的文件句柄，**在Linux系统中它的值通常是0**。当从标准输入流中读取数据时，数据实际上是从内核缓冲区中读取的，内核缓冲区对应的文件描述符就是标准输入描述符。此时，如果使用 `read()` 系统调用从标准输入流中读取数据，就会使用标准输入描述符来访问内核缓冲区。因此，“标准输入描述符”与“内核缓冲区”之间是密切相关的
>
> （3）在C++中， `scanf()` 函数的输入缓冲区是由C标准库中的 `stdin` 文件流维护和管理的。`stdin` 文件流是一个全局变量，它在程序启动时自动创建，而输入缓冲区也是在此时被开辟的。
>
> 具体地说，当程序启动时，操作系统会在内存中为程序分配一块空间，用于存储程序的数据和代码。在此空间中，操作系统会为C标准库中的数据结构分配内存空间，包括 `stdin` 文件流的数据结构。在 `stdin` 文件流数据结构中，包含了一个指向输入缓冲区的指针，并将其初始化为一个能够容纳输入数据的缓冲区。当程序使用 `scanf()` 函数时，函数会从标准输入流中读取数据并将其存储在输入缓冲区中，缓冲区是由 `stdin` 文件流所管理的，而不是由 `scanf()` 函数自己单独开辟的。
>
> 需要注意的是，输入缓冲区的大小有限，当缓冲区中的数据满了之后，新的输入数据将无法存储在缓冲区中，这就会导致输入操作失败或输入错误。因此，在使用 `scanf()` 函数读取输入数据时，需要注意缓冲区的大小，为了避免缓冲区溢出和输入错误，建议使用输入限制和错误处理机制来处理输入数据。

# 3 STL中的兼容基本类型的默认值

> ```c
> int xx; //随机值
> double yy; //随机值
> 
> //加括号表示给定默认值
> int *x = new int(5); // *x=5
> int x = int(); // x=0
> char y = char(); // y='\0'
> 
> //比如unordered_map中
> unordered_map<int, int> m;
> m[0];	//此时m[0] = int() = 0
> ```
>

# 4 C++11中`using`引入的两种新用法

## 4.1 在子类中引用基类的成员

> 举例子类私有继承父类，私有继承会继承父类的`public`和`protect`部分为子类的`private`成员，这意味着无法使用子类对象直接调用父类成员。
>
> 但是可以使用`using`可以访问。
>
> eg:
>
> ```c++
> class father{
> public:
> 	father() : values(55){}
> 	virtual ~father(){}
> 	void test1() { cout << "father test1..." << endl; }
> protected:
> 	int value;
> };
> 
> class son : private father{
> public:
>     using father::test1;
>     using father::value;
>     void test2() { cout << "value is " << value << endl; }
> };
> 
> int main()
> {
>     son son;
>     son.test1();
>     son.test2();
> }
> ```

## 4.2 `using`定义别名

> 众所周知，在C++可以通过`typedef`重命名一个类型：
>
> ```c++
> typedef unsigned int uint_t;
> typedef void (*Fun) (int, const std::string&);
> ```
>
> 如果使用`using`的话，可以：
>
> ```c++
> using Fun = void (*) (int, const std::string&);
> ```

> 使用using定义别名看起来和typedef用法差不多，但实际上typedef无法重命名模板。
>
> C++ prime plus P482
>
> ```
> template<class T, T v>
> struct m_integral_constant
> {
>     static constexpr T value = v;
> };
> 
> template <bool b>
> using m_bool_constant = m_intergral_constant<bool, b>;
> 
> typedef m_bool_constant<true> m_true_type;
> typedef m_bool_constant<false> m_false_type;
> ```
>



