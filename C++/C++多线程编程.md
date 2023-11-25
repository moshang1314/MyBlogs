# 第一章

## 1.1  `detach`和`join`

> 当`std::thread()`对象未正确的加入（`joined`）或分离（`detach`），则`std::thread`的析构函数会调用`std::terminate()`终止整个进程。当调用`join()`或`detach()`后，`std::thread`将不再与该线程有任何关联。因此，必须在`std::thread`对象未被销毁时调用`join()`等待或`detach()`分离。可以使用“资源获取即初始化方式”（`RALL, Resource Acquisition is Initialization`）。示例代码如下：
>
> ```c++
> class thread_guard{
> 	std::thread& t;
> public:
> 	explict thread_guard(std::thread& t_) : t(t_) {}
> 	~thread_guard(){
> 		if(t.joinable){
> 			t.join();
> 		}
> 	}
> 	thread_guard(thread_guard const&)=delete;
> 	thread_guard& operator=(thread_guard const&)=delete;
> };
> 
> struct func{
>     int& i;
>     func(int& i_) : i(i_) {}
>     void operator() (){
>         for(unsigned j = 0; j < 1000000; ++j){
>             do_something(i);		// 潜在问题隐患：悬空引用
>         }
>     }
> }；
> 
> void f(){
> 	int some_local_state = 0;
> 	func my_func(some_lcoal_state);
> 	std::thread t(my_func);
> 	thread_guard g(t);
> 	do_something_in_current_thread();
> }
> ```

## 1.2 后台运行线程

> ​	使用`detach()`会让线程在后台运行，这意味着，主线程不会等待这个线程结束；如果线程分离，那么`std::thread`对象将不再引用它，即`std::thread`对象与实际执行的线程无关了，且分离线程不能被加入（`join`）。不过`C++`运行时库保证，当线程退出时，相关资源能够正确回收，后台线程的归属和控制`C++`运行库都会处理。

## 1.3 向线程函数传递参数

> ​	默认参数是拷贝到线程独立的内存中，即使参数是引用的形式。
>
> ```sql
> void f(int i, std::string const& s);
> std::thread t(f, 3, "hello");
> ```
>
> ​	代码创建了一个调用`f(3, "hello")`的线程，传递参数时，会复制`"hello"`的地址到线程内存。
>
>  ```c++
>  void update_data_for_widget(widget_id w, widget_data& data);
>  void oops_again(widget_di w){
>  	widget_data data;
>  	std::thread t(update_data_for_widget, w, data);
>  	display_status();
>  	t.join();
>  	process_widget_data(data);
>  }
>  ```
>
> ​	此时，虽然`update_data_for_widget`的第二个参数期待传入一个引用，但是`std::thread`的构造函数并不知晓；构造函数会无视函数期待的参数类型，并盲目的拷贝已提供的变量。当线程调用`update_data_for_widget`函数时，传递给函数的参数是`data`变量内部拷贝的引用，而非数据本身的引用。因此，在线程中对`data`的修改不会影响主线程中的`data`变量。
>
> ​	使用`std::ref`将参数转换成引用的形式。这种情况下线程的调用，将改成以下形式：
>
> ```c++
> std::thread t(update_data_for_widget, w, std::ref(data));
> ```
>
> ​	此时，`update_data_for_widget`就会接收到一个`data`变量的引用，而非一个`data`变量拷贝的引用。
>
> 可以传递一个成员函数指针作为线程函数，并提供一个合适的对象指针作为第一个参数，后续参数则为成员函数的形参。如：
>
> ```c++
> class A{
> public:
> 	void do_lengthy_work(int);
> };
> A my_a;
> int num(0);
> std::thread t(&A::do_lengthy_work, &my_a, num);
> ```

## 1.4 转移线程所有权

> ​	`std::thread`所有权可以在多个实例中互相转移，因为这些示例是可移动（movable）但不可复制（aren't copyable）。在同一时间点，就能保证只关联一个执行线程；同时，也允许程序员能在不同的对象之间转移所有权。
>
> ​	C++标准库中有很多资源占有（resource-owning）类型，比如`std::ifstream`，`std::unique_ptr`还有`std::thread`都是可移动（movable），但不可拷贝。
>
> ```c++
> void some_function();
> void some_other_function();
> std::thread t1(some_function);
> std::thread t2 = std::move(t1); // 调用移动拷贝构造
> t1 = std::thread(some_other_function);	// 调用移动赋值函数
> std::thread t3;
> t3 = std::move(t2);
> t1 = std::move(t3);	// 赋值操作将使程序崩溃
> ```
>
> ​	`std::thread`支持移动，就意味着线程的所有权可以在函数外传递，如：
>
> ```c++
> std::thread fun(){
> 	void some_function();
> 	return std::thread(some_function);
> }
> 
> std::thread g(){
> 	void some_other_function(int);
> 	std::thread t(some_other_function, 42);
> 	return t;
> }
> ```
>
> ​	也意味着`std::thread`可以作为参数进行传递，如：
>
> ```c++
> void fun(std::thread t);
> void g(){
> 	void some_function();
> 	f(std::thread(some_function));
> 	std::thread t(some_function);
> 	f(std::move(t));
> }
> ```
>
> ​	为了确保在`std::thread`对象销毁前，加入（`join`）或分离（`detach`），可以定义一个`scoped_thread`类。
>
> ```c++
> class scoped_thread{
> 	std::thread t;
> public:
> 	explicit scoped_thread(std::thread t_) : t(std::move(t_)){
> 		if(!t.joinable()){
> 			throw std::logic_error("No thread");
> 		}
> 	}
> 	
> 	~scoped_thread(){
> 		t.join();
> 	}
> 	scoped_thread(scoped_thread const&) = delete;
> 	scoped_thread& operator=(scoped_thread const&) = delete;
> };
> ```
>
> ​	量产线程，并等待它们结束：
>
> ```c++
> void do_work(unsigned id);
> 
> void f(){
> 	std::vector<std::thread> threads;
> 	for(unsigned i = 0; i < 20; ++i){
> 		threads.push_back(std::thread(do_work ,i)); //产生线程
> 	}
> 	std::for_each(threads.begin(), threads.end(), std::mem_fn(&std::thread::join));
> }
> ```

## 1.5 运行时决定线程数量

> `std::thread::hardware_concurrency()`这个函数将返回硬件支持的最大并发线程数，在多核系统中，返回值是CPU核心的数量。返回值仅仅是一个提示，当系统无法获取时，函数会返回0。以下函数实现了一个并行的累加函数。
>
> ```c++
> #include <iostream>
> #include <memory>
> #include <thread>
> using namespace std;
> 
> void fun1(int *data) {
>     *data = 5;
> }
> 
> template<typename Iterator , typename T>
> struct accumulate_block {
>     void operator()( Iterator first , Iterator last , T& result ) {
>         result = std::accumulate( first , last , result );
>     }
> };
> 
> template<typename Iterator , typename T>
> T parallel_accumulate( Iterator first , Iterator last , T init ) {
>     unsigned long const length = std::distance( first , last );
>     if (!length) {
>         return init;
>     }
> 
>     unsigned long const min_per_thread = 25;
>     unsigned long const max_threads = ( length + min_per_thread - 1 ) / min_per_thread; // 向上取整
> 
>     unsigned long const hardware_threads = std::thread::hardware_concurrency();
>     unsigned long const num_threads = std::min( hardware_threads != 0 ? hardware_threads : 2 , max_threads );
> 
>     unsigned long const block_size = length / num_threads;
> 
>     std::vector<T> results( num_threads );
>     std::vector<std::thread> threads( num_threads - 1 );
> 
>     Iterator block_start = first;
>     for (unsigned long i = 0; i < ( num_threads - 1 ); ++i) {
>         Iterator block_end = block_start;
>         std::advance( block_end , block_size );
>         threads[i] = std::thread(
>             accumulate_block<Iterator , T>() ,
>             block_start , block_end , std::ref( results[i] )
>         );
>         block_start = block_end;
>     }
>     accumulate_block<Iterator , T>()(
>         block_start , last , results[num_threas - 1]);
>     std::for_each( threads.begin() , threads.end() ,
>         std::mem_fn( &std::thread::join )
>     )
> 
>     return std::acculate( results.begin() , results.end() , init );
> }
> ```
>
> ​	范围内元素总数量除以每个线程处理的数量，确定启动线程的最大数量，这样能避免计算资源的浪费。比如，一台32芯的机器上，只有5个数需要计算，却启动了32个线程。计算量的最大值和硬件支持线程数中，较小值为启动线程的数量。因为上下文频繁的切换回降低线程的性能，所以你肯定不想启动的线程数多于硬件支持的线程数量。当`std::threadware_concurrency()`返回0时，需要重新设置一个合适的线程数。

## 1.6 识别线程

> ​	线程标识类型是`std::thread::id`，可以通过两种方式获得，第一种，通过调用`std::thread`对象的成员函数`get_id()`直接获取。如果`std::thread`对象没有与任何执行线程相关联，`get_id()`将返回`std::thread::type`默认构造值，这个值表示“没有线程”。第二种，当前线程中调用`std::this_thread::get_id()`也可以获得线程标识。
>
> ​	`std::thread::id`对象可以自由的拷贝和对比，如果两个对象的`std::thread::id`相等，那么它们就是同一个线程，或者都“没有线程”；如果不等，代表了两个不同的线程，或者一个有线程，一个没有。标准库也提供了`std::hash<std::thread::id>`容器，所以`thread::thread::id`也可以作为无序容器的键值。
>
> ```c++
> std::thread::id master_thread;
> void some_core_part_of_algorithm(){
> 	if(std::this_thread::get_id() == master_thread){
> 		do_master_thread_work();
> 	}
> 	do_common_work();
> }
> ```

# 2 使用互斥保护共享数据

## 2.1在C++中使用互斥

> ​	在C++中，我们听过构造`std::mutex`的实例来创建互斥，调用成员函数`lock()`对其加锁，调用`unlock()`解锁。但通常不推荐直接调用成员函数的做法。原因是，若按此处理，那么我们就必须记住，在函数外的每条路径上都要调用`unlock()`，包括由于异常退出的路径。取而代之，C++标准库提供了类模板`std::lock_guard<>`，针对互斥类融合了RAII手法：**在构造时加锁，在析构时解锁，从而保证互斥总是能被正确解锁。**
>
> ```c++
> #include <mutex>
> #include <algorithm>
> 
> std::list<int> some_list;
> std::mutex some_mutex;
> void add_to_list(int new_value){
> 	std::lock_guard<std::mutex> guard(some_mutex);
> 	some_list.push_back(new_value);
> }
> bool list_contains(int value_to_find){
> 	std::lock_guard<std::mutex> guard(some_mutex);
> 	return std::find(some_list.begin(), some_list.end(), value_to_find) != some_list.end();
> }
> ```
>
> ​	普遍的做法是，将互斥和受保护的数据组成一个类，这是面向对象设计准则的典型应用。本例的`add_to_list`和`list_contains`将被写为类的成员函数，而互斥和受保护的共享数据都会变成类的私有成员。按此处理，就非常便于确定哪些代码可以访问数据，哪些代码需要锁住互斥。
>
> ​	但当成员函数返回数据的指针或引用，那么这种保护将出现漏洞，游离的指针或引用可以在任何地方访问数据，从而危及共享数据的安全。或是在成员函数中调用了别的函数，向它们传递了共享数据的指针或引用。

## 2.2 发现接口固有的条件竞争

> ​	考虑`std::stack`容器，**由于其接口的功能的分割操作**，将导致条件竞争。例如，当调用`st.empty()`判断栈非空后，再调用`st.top()`去获取栈顶元素，此时，若其它线程已经先一步移除了栈顶元素，使得栈为空，则在空栈上调用`st.top()`将导致未定义行为。
>
> 以下是一个线程安全的栈的封装：
>
> ```c++
> #include <exception>
> #include <iostream>
> #include <memory>
> #include <mutex>
> #include <stack>
> using namespace std;
> 
> struct empty_stack : std::exception {
>     const char* what() const throw( ) {
>         return "This is a empty stack!";
>     }
> };
> 
> template<typename T>
> class threadsafe_stack {
> private:
>     std::stack<T> data;
>     mutable std::mutex m;
> public:
>     threadsafe_stack() {}
>     threadsafe_stack( const threadsafe_stack& other ) {
>         std::lock_guard<std::mutex> lock( other.m );
>         data = other.data;
>     }
>     threadsafe_stack& operator=( const threadsafe_stack& ) = delete;
>     void push( T new_value ) {
>         std::lock_guard<std::mutex> lock( m );
>         data.push( std::move( new_value ) );
>     }
>     std::shared_ptr<T> pop() {
>         std::lock_guard<std::mutex> lock( m );
>         if (data.empty()) throw empty_stack();
>         std::shared_ptr<T> const res( std::make_shared<T>( data.top() ) );
>         data.pop();
>         return res;
>     }
> 
>     void pop( T& value ) {
>         std::lock_guard<std::mutex> lock( m );
>         if (data.empty()) throw empty_stack();
>         value = data.top();
>         data.pop();
>     }
> 
>     bool empty() const {
>         std::lock_guard<std::mutex> lock( m );
>         return data.empty();
>     }
> };
> 
> int main() {
>     threadsafe_stack<int> st;
>     try {
>         st.pop();
>     }
>     catch (std::exception& e) {
>         cout << e.what() << endl;
>     }
>     return 0;
> }
> ```

## 2.3 死锁问题和解决方法

> ​	防范死锁的建议通常是，**始终按相同的顺序对两个互斥加锁。**
>
> ​	C++标准库提供了`std::lock()`函数，专门解决这一问题。它可以同时锁住多个互斥，而不会发生死锁的风险，其打破了产生死锁的必要条件之一：请求和保持条件。`std::lock()`函数会按序锁住锁，当遇到获取某个锁产生异常时，则会对已经锁住的锁调用`unlock()`进行解锁，然后抛出异常。即`std::lock()`的语义是要么**全部锁住，要么全部没有锁住并抛出异常。**
>
> ```c++
> class some_big_object;
> void swap(some_big_object& lhs, some_big_object& rhs);
> class X{
> private:
> 	some_big_object some_detail;
> 	std::mutex m;
> public:
> 	X(some_big_object const& sd) : some_detail(sd){}
> 	friend void swap(X& lhs, X& rhs){
> 		if(&lhs == &rhs){
> 			return;
> 		}
> 		std::lock(lhs.m, rhs.m);
> 		std::lock_guard<std::mutex> lock_a(lhs.m, std::adopt_lock);
> 		std::lock_guard<std::mutex> lock_b(rms.m, std::adopt_lock);
> 		swap(lhs.some_detail, rhs.some_detail);
> 	}
> };
> ```
>
> ​	其中`std::adopt_lock`对象用以指明互斥锁已经被锁住，`std::lock_guard`实例应当据此接收锁的归属权，不得在构造函数内试图另行加锁。
>
> 另外，C++17提供了新的RAII类模板`std::scoped_lock<>`。`std::scoped_lock<>`和`std::lock_guard`几乎完全等价，但`std::scoped_lock`是可变参数模板，接收各种互斥型别作为模板参数列表，机制与`std::lock()`函数相同。因此上面的代码可以改写为：
>
> ```c++
> void swap(X& lhs, X& rhs){
> 	if(&lhs == &rhs){
> 		return;
> 	}
> 	std::scoped_lock guard(lhs.m, rhs.m);
> 	swap(lhs.some_detail, rhs.some_detail);
> }
> ```
>
> ​	C++17具有隐式类模板参数推导机制，依据传入构造函数的参数对象自动匹配，选择正确的互斥类别。

## 2.4 运用`std::unique_lock<>`灵活加锁

> `std::unique_lock<>`在构造时可以接收第二个参数，可以传入`std::adopt_lock`指明锁已经被上锁，传入`std::defer_lock`指明不要在`std::unique_lock<>`的构造函数中对锁加锁，等以后有需要时，再在`std::unique_lock<>`对象上调用`lock()`而获取锁，或把`std::unique_lock<>`对象交给`std::lock()`函数加锁。
>
> ```c++
> class some_big_object;
> void swap(some_big_object& lhs, some_big_object& rhs);
> class X{
> private:
> 	some_big_object some_detail;
> 	std::mutex m;
> public:
> 	X(some_big_object const& sd) : some_detail(sd) {}
> 	friend void swap(X& lhs, X& rhs){
> 		if(&lhs == &rhs)
> 			return;
> 		std::unique_lock<std::mutex> lock_a(lhs.m, std::defer_lock);
> 		std::unique_lock<std::mutex> lock_b(lhs.m, std::defer_lock);
> 		std::lock(lock_a, lock_b);
> 		swap(lhs.some_detail, rhs.some_detail);
> 	}
> }
> ```
>
> ​	因为`std::unique_lock<>`具有成员函数`lock()`、`try_lock()`和`unlock()`所以它的实例可以传给`std::lock()`函数。`std::unique_lock`实例在底层与目标互斥关联，其内部还有一个标志，随着上述函数的执行而更新，以表明关联的互斥目前是否被该类实例占据，进一步保证析构函数正确调用`unlock()`。如果`std::unique_lock`实例的确占据着互斥，则其析构函数必须调用`unlock`函数，若不然，便绝不能调用`unlock()`。该标志可以通过`owns_lock()`成员函数查询。

## 2.5 在不同作用域之间转移互斥归属权

> ​	因为`std::unique_lock`实例不占有与之关联的互斥，所以随着其实例的转移，互斥的归属权可以在多个`std::unique_lock`实例之间转移。即`std::unique_lock`属于可转移却不可复制的型别。转移有一种用途：准许函数锁定互斥，然后把互斥的归属权转移给函数调用者，好让他在同一个锁的保护下执行其他操作。
>
> ​	以下代码，`get_lock()`函数先锁定互斥，接着对数据做前期准备，再将归属权返回给调用者。
>
> ```c++
> std::unique_lock<std::mutex> get_lock(){
> 	extern std::mutex some_mutex;
> 	std::unique_lock<std::mutex> lk(some_mutex);
> 	prepare_data();
> 	return lk;
> }
> 
> void process_data(){
>     std::unique_lock<std::mutex> lk(get_lock());
>     do_something();
> }
> ```
>
> ​	函数内部返回的局部变量，会是一个临时无名对象，因为超出函数作用域后，其即将消亡。
>
> **这是c++标准规定的：**
> **当从同类型的右值（亡值或纯右值） (C++17 前)亡值 (C++17 起)初始化（直接初始化或复制初始化）对象时，会调用移动构造函数，情况包括：**
>
> **初始化：T a = std::move(b); 或 T a(std::move(b));，其中 b 的类型是 T ；**
> **函数实参传递：f(std::move(a));，其中 a 的类型是 T 且 f 是 Ret f(T t) ；**
> ***函数返回：在像 T f() 这样的函数中的 return a;，其中 a 的类型是 T，且 T 有移动构造函数。***
> **当初始化器是纯右值时，通常会优化掉 (C++17 前)始终不会进行 (C++17 起)对移动构造函数的调用，见复制消除。**
>
> **典型的移动构造函数“窃取”实参曾保有的资源（例如指向动态分配对象的指针，文件描述符，TCP socket，输入输出流，运行的线程，等等），而非复制它们，并使其实参遗留在某个合法但不确定的状态。例如，从 std::string 或从 std::vector 移动可以导致实参被置为空。但是不应该依赖此类行为。对于某些类型，例如 std::unique_ptr，移动后的状态是完全指定的。**

## 2.6 在初始化过程中保护共享数据

> ​	有时候，我们需要某个共享数据，然而创建起来开销不菲。因为创建它可能需要建立数据库连接或分配大量的内存，所以等到必要的时候才真正着手创建。这种方式被称为延迟初始化，常见于单线程代码。以下代码用互斥来保护延迟初始化：
>
> ```c++
> std::shared_ptr<some_resource> resource_ptr;
> std::mutex resource_mutex;
> void foo(){
> 	std::unique_lock<std::mutex> lk(resource_mutex);
> 	if(!resource_ptr){
> 		resource_ptr.reset(new some_resource);
> 	}
> 	lk.unlock();
> 	resource_ptr->do_something();
> }
> ```
>
> ​	以上代码模式司空见惯，但它毫无必要地迫使多个线程循序运行，即不管有没有初始化都将执行加锁解锁操作。
>
> ​	C++标准库中提供了`std::once_flag`类和`std::call_once()`函数，以专门处理该情况。所有线程共同调用`std::call_once()`函数，从而保证在该调用返回时，指针初始化由其中某个线程安全唯一地完成。必要的同步数据则由`std::once_flag`实例存储，每个`std::once_flag`对应一次不同的初始化。相比显式地调用互斥，`std::call_once()`函数的额外开销往往更低，尤其是在初始化已经完成的情况下。注意，`std::once_flag`既不可复制也不可移动，这与`std::mutex`类似。
>
> ```c++
> std::shared_ptr<some_resource> resource_ptr;
> std::once_flag resource_flag;
> void init_resource(){
> 	resource_ptr.reset(new some_resource);
> }
> void foo(){
>     std::call_once(resource_flag, init_resource);	/* 初始化函数准确地被唯一一次调用 */
>     resource_ptr->do_something();
> }
> ```
>
>
> ```c++
> class X{
> private:
> 	connection_info connection_details;
> 	connection_handle connection;
> 	std::once_flag connection_init_flag;
> 	void open_connection(){
> 		connection = connection_manager.open(connection_details);
> 	}
> public:
> 	X(connection_info const& connection_details_) : connection_details(connection_details_){}
> 	void send_data(data_packet const& data){
> 		std::call_once(connection_init_flag, &X::open_connection, this);
> 		connection.send_data(data);
> 	}
> 	data_packet receive(){
> 		std::once_(connection_init_flag, &X::open_connection, this);
> 		return connection.receive_data();
> 	}
> };
> ```
>
>
> ​	C++标准规定，只要控制流程第一次遇到静态数据的声明语句，变量即进行初始化。多个线程同时调用同一个函数，而它含有静态函数，则任意线程均可能首先到达其声明处，这就形成了条件竞争的隐患。C++11解决了这个隐患，**规定初始化只会在某一线程上单独发生，在初始化完成之前其他线程不会越过静态数据的声明而继续运行。**于是，这使得条件竞争原来导致的问题变为，初始化应当由哪个线程具体执行。某些类的代码只需用到唯一一个全局实例，这种情形可用以下方法代替`std::call_once`：
>
> ```c++
> class my_class;
> my_class& get_my_class_instance(){
> 	static my_class instance;	/* 线程安全的初始化，C++11标准确保其正确性 */
> 	return instance;
> }
> ```
>
> ​	多个线程可以安全地调用`get_my_class_instance()`，而无须担忧初始化的条件竞争。

## 2.7 保护甚少更新的数据结构

> ​	对于甚少更新的数据，通常使用读写互斥来进行保护，其允许单独一个“写线程”进行完全排他的访问，也允许多个“读线程”共享数据或并发访问。
>
> ​	C++17提供了`std::shared_mutex`和`std::shared_timed_mutex`。C++14标准库中只有`std::shared_timed_mutex`，而C++11标准库都没有。如果编译器尚未支持C++14，那么可以考虑使用boost程序库，它也提供了这两个互斥。
> ​	`std::shared_mutex`和`std::shared_timed_mutex`的区别在于后者支持更多的操作，所以若无须进行额外的操作，应选择`std::shared_mutex`。更新操作可用`std::lock_guard<std::shared_mutex>`和`std::unique_lock<std::shared_mutex>`锁定。对于无须更新的访问，可以另行改用共享锁`std::shared_lock<std::shared_mutex>`实现访问。C++14引入了共享锁的类模板，其工作原理是RAII过程，使用方式则与`std::unique_lock`相同，只不过多个线程能够同时锁住同一个`std::shared_mutex`实现共享访问。共享锁若被某些线程所持有，则别的线程试图获取排他锁，就会发生阻塞，直到那些线程释放该共享锁。反之，如果任一线程持有排他锁，那么其他线程全都无法获取共享锁或排他锁，直到持锁线程将排他锁释放为止。
>
> 注：共享锁即读锁，对应`std::shared_lock<std::shared_mutex>`；排他锁即写锁，对应`std:lock_guard<std::shared_mutex>`和`std::unique_lock<std::shared_mutex>`。
>
> ```c++
> #include <map>
> #include <string>
> #include <mutex>
> #include <shared_mutex>
> class dns_entry;
> class dns_cache{
> 	std::map<std::string, dns_entry> entries;
>     mutable std::shared_mutex entry_mutex;
> public:
>     dns_entry find_entry(std::string const& domain) const{
>         std::shared_lock<std::shared_mutex> lk(entry_mutex);
>         std::map<std::string, dns_entry>::const_iterator const it = entries.find(domain);
>         return (it == entries.end()) ? dns_entry() : it->second;
>     }
>     
>     void update_or_add_entry(std::string const& domain, dns_entry const& dns_details){
>         std::lock_guard<std::shared_mutex> lk(entry_mutex);
>         entries[domian] = dns_details;
>     }
> };
> ```

## 2.8 递归加锁

> ​	假如线程已经持有某个`std::mutex`实例，试图再次对其重新加锁就会出错，将导致未定义行为。但在某些场景下确有需要让线程在同一互斥上多次重复加锁，而无须解锁。C++标准提供了`std::recursive_mutex`，其工作方式与`std::mutex`类似，不同之处在于，其允许同一线程对其互斥的同一实例多次加锁。我们必须先释放全部的锁，才可以让另一个线程锁住该互斥。例如，若我们对它调用了3次`lock()`，就必须调用3次`unlock()`。只要正确地使用`std::lock_guard<std::recursive_mutex>`和`std::unique_lock<std::recursive_mutex>`，它们会处理好递归锁的余下细节。递归互斥常用于这种情形，对于某个类，每个公有函数都需先锁住互斥，然后才进行操作，最后解锁互斥。但有时在某些操作过程中，公有函数需要调用另一个公有函数。在这种情况下，后者将同样试图锁住互斥，如果采用`std::mutex`将导致未定义行为。
>
> ​	但一般不推荐这样做，因为这容易破坏不变量。我们通常可以采取更好的方法：根据这两个公有函数的共同部分，提取出一个新的私有函数，新函数由这两个公有函数调用，而它假定互斥锁已经被锁住，遂无须重复加锁。 

# 3 并发操作的同步

## 3.1 凭借条件变量等待条件成立

> ​	C++标准库提供了条件变量的两种实现：`std::condition_variable`和`std::condition_variable_any`。它们都在标准库的头文件`<condition_variable>`内声明。两者都需配合互斥，方能提供妥当的同步操作。`std::condition_variable`仅限于与`std::mutex`一起使用；然而，主要某一类型符合成为互斥的最低标准，足以充当互斥，`std::condition_variable_any`即可与之配合使用，因此它的后缀是"_any"。由于`std::condition_variable_any`更加灵活，可能产生额外的开销，涉及其性能，自身体积等，因此`std::condition_variable`应该予于优先采用，除非有必要令程序更灵活。
>
> ​	我们需要达到的效果是，令线程甲一直休眠，直到有数据需要处理时才被唤醒。
>
> ```c++
> std::mutex mut;
> std::queue<data_chunk> data_queue;
> std::condition_variable data_cond;
> void data_preparation_thread(){	//由线程乙运行
> 	while(more_data_to_prepare()){
> 		data_chunk const data = prepare_data();
> 		{
> 			std::lock_guard<std::mutex> lk(mut);
> 			data_queue.push(data);
> 		}
> 		data_cond.notify_one();
> 	}
> }	
> 
> void data_processing_thread(){	//由线程甲运行
> 	while(true){
> 		std::unique_lock<std::mutex> lk(mut);
> 		data_cond.wait(
> 			lk, []{return !data_queue.empty();}
> 		);
> 		data_queue.pop();
> 		lk.unlock();
> 		process(data);
> 		if(is_last_chunk(data)){
> 			break;
> 		}
> 	}
> }
> ```
>
> ​	线程乙使用`std::lock_guard`保护队列，并压入数据，然后调用`std::condition_variable`实例的`notify_one()`成员函数通知线程甲。线程甲首先对使用`std::unique_lock`对互斥加锁，然后在`std::condition_variable`实例上调用`wait()`函数，传入锁对象和应该lambda函数，后者用于表达需要等待成立的条件。`wait()`函数在内部调用传入的lambda函数，判断条件是否成立：若成立（lambda函数返回true），则`wait()`函数返回；否则（lambda函数返回false），`wait()`解锁互斥，并令线程进入阻塞状态或等待状态。线程乙准备好数据后，调用`notify_one()`通知条件变量，线程甲随之从休眠中觉醒（阻塞解除），重新在互斥上获取锁，再次查验条件：若条件成立，则从`wait()`函数返回，而互斥仍被锁住；若条件不成立，则线程甲解锁互斥，并继续等待。
>
> ​	在`wait()`调用期间，条件变量可以多次查验给定的条件，次数不受限；在查验时，互斥总会被锁住；另外，当且仅当传入的判定函数返回true时（它判定条件成立），`wait()`才会返回。如果线程甲获得互斥，并且查验条件，而这一行为不是直接响应线程乙的通知，则称之为伪唤醒。以下是`wait()`函数的一种合法实现，仅仅使用了一个简单的循环，但效率不尽人意。
>
> ```c++
> template <typename Predicate>
> void minimal_wait(std::unique_lock<std::mutex>& lk, Predicate pred){
>     std::unique_lock<mutex> lk(mt);
> 	while(!pred()){
> 		wait(lk);
> 	}
> }
> ```
>
> ​	除此之外，还有另外一种实现，它只能通过调用`notify_one()`或`notify_all()`才会使休眠线程觉醒。

## 3.2  线程安全的队列容器

> 与之前的栈容器相似，`std::queue<>`的接口还是存在固有的条件竞争。所以我们需要将`front()`和`pop()`合并成一个函数。当队列用于线程间的数据传递时，负责接收的线程常常需要等待数据压入。我们要对外提供`pop()`的两个变体，`try_pop()`和`wait_and_pop()`：它们都试图弹出队首元素，前者总是立即返回，即便队列内空无一物（通过返回值示意操作失败）；后者会一直等到有数据压入可供获取。
>
> ```c++
> #include <queue>
> #include <memory>
> #include <mutex>
> #include <condition_variable>
> 
> template<typename T>
> class threadsafe_queue{
> private:
> 	mutable std::mutex mut;	/* 互斥必须用mutable修饰（针对const对象，准许其数据成员发生变动）*/
> 	std::queue<T> data_queue;
> 	std::condition_variable data_cond;
> public:
> 	threadsafe_queue(){}
>     
> 	threadsafe_queue(threadsafe_queue const& other){
> 		std::lock_guard<std::mutex> lk(other.mut);
> 		data_queue = other.data_queue;
> 	}
>     
>     void push(T new_value){
>         std::lock_guard<std::mutex> lk(mut);
>         data_queue.push(new_value);
>         data_cond.notify_one();
>     }
>     /* 通过引用传递，是为了防止返回结果时分配空间失败，导致数据丢失 */
>     void wait_and_pop(T& value){
>         std::unique_lock<std::mutex> lk(mut);
>         data_cond.wait(lk, [this]{return !data_queue.empty();});
>         value = data_queue.front();
>         data_queue.pop();
>     }
>     
>     std::shared_ptr<T> wait_and_pop(){
>         std::unique_lock<std::mutex> lk(mut);
>         data_cond.wait(lk, [this]{return !data_queue.empty();});
>         std::shared_ptr<T> res(std::make_shared<T>(data_queue.front()));
>         data_queue.pop();
>         return res;
>     }
>     
>     bool try_pop(T& value){
>         std::lock_guard<std::mutex> lk(mut);
>         if(data_queue.empty())
>             return false;
>         value = data_queue.front();
>         data_queue.pop();
>         return true;
>     }
>     
>     std::shared_ptr<T> try_pop(){
>         std::lock_guard<std::mutex> lk(mut);
>         if(data_queue.empty())
>             return std::shared_ptr<T>();
>         std::shared_ptr<T> res(std::make_shared<T>(data_queue.front()));
>         data_queue.pop();
>         return res;
>     }
>     
>     bool empty() const{
>         std::lock_guard<std::mutex> lk(mut);
>         return data_queue.empty();
>     }
> }
> ```
>
>  	条件变量也适用于多个线程都在等待同一个目标事件的情况。若要将工作负荷分配给多个线程，那么，对于每个通知应该仅有一个线程响应。如果有多个线程因执行`wait()`而同时等待，每当有新数据就绪并加入`data_queue`时，`notify_one()`的调用就会触发其中一个线程去查验条件，让它从`wait()`返回。此方式并不能确定会通知到具体哪个线程，甚至不能保证正好有线程在等待通知，因为可能不巧，负责处理数据的全部线程都尚未执行到此。
>
> ​	如果几个线程都在等待同一个目标事件，那么还存在另一种可能的行为方式：它们全部需要做出响应。以上行为会在两种情形下发生：一个是共享数据的初始化，所有负责处理的线程都用到同一份数据，但都需要等待数据初始化完成；二是所有线程都要等待共享数据更新。尽管条件变量适用于第一种情形，当我们可以选择更好的处理方式，如`std::call_once()`函数。`notify_all`会通知当前所有执行`wait()`等待的线程，让它们去查验等待的条件。
>
> ​	假定，某个线程按计划仅仅等待一次，只要条件成立一次，它就不再理会条件变量。条件变量未必是这种同步模式的最佳选择。若我们所等待的条件需要判定某份数据是否可用，上述论断就非常正确。`future`更适合此场景。

## 3.3 使用`futrue`等待一次性事件发生

> ​	C++标准库使用`future`来模拟一次性事件：若线程需要等待某个特定的一次性事件发生，则会以恰当的方式取得一个`future`，它代表目标事件，接着，该线程就能一边执行其它任务，一边在`future`上等待；同时，它以短暂的间隔反复查验目标事件是否已经发生。这个线程可以转换运行模式，先不等待目标事件发生，直接暂缓当前任务，而切换到别的任务，乃至必要时，才回头等待`future`就绪。`future`可能与数据关联，也可能未关联。一旦目标事件发生，`future`就进入就绪状态，无法重置。
>
> ​	C++标准库中有两种`future`，声明在标准库的头文件`<future>`内：独占`future(unique future, 即std::future<>)`和共享`future(shared future, 即std::shared_future<>)`。同一个事件仅仅允许关联一个`std::future`实例，但可以关联多个`std::shared_future`实例。只要目标事件发生，与后者关联的实例都会同时就绪，并且它们都可以访问与该目标事件关联的任何数据。如果没有关联数据，应该使用特化的模板：`std::future<void>`和`std::shared_future<void>`。
>
> ​	虽然`future`能用于线程间通信，但`future`对象本身不提供同步访问。若多个线程需访问同一个`future`对象，必须用互斥或其它同步方式进行保护。后续章节将说明，一个`std::shared_future<>`对象可能派生出多个副本，这些副本都指向同一个异步结果，由多个线程分别独占，它们可访问属于自己的副本而无须相互同步。
>
> ​	最基本的一次性事件是，置于后台运行的计算任务完成，得出结果。

### 3.3.1 从后台返回值

> ​	因为`std::thread`没有提供直接回传结果的方法，所以函数模板`std::async()`应运而生（其声明也在`<future>`中）。只要我们并不急线程运算的值，就可以使用`std::async()`按异步方式启动任务。我们从`std::async()`函数处获得`std::future`对象，运行的函数一旦完成，其返回值就由该对象最后持有。若要用到这个值，只需在`future`对象上调用`get()`，当前线程就会阻塞，以便`future`准备妥当并返回该值。例如：
>
> ```c++
> #include <future>
> #include <iostream>
> 
> int find_the_answer_to_ltuae();
> void do_other_stuff();
> int main(){
> 	std::future<int> the_answer = std::async(find_the_answer_to_ltuae); /* 异步启动一个线程，并由其返回的future来等待其计算结果 */
> 	do_other_stuff();
> 	std::cout << "The answer is " << the_answer.get() << std::endl;
> }
> ```
>
> ​	`std::async()`可以接收附加参数，进而传递给任务函数作为参数，此方式与`std::thread`构造函数相同。如果`std::async()`参数是右值，则通过移动原始参数构建副本。
>
> ```c++
> #include <string>
> #include <future>
> struct X{
> 	void foo(int, std::string const&);
> 	std::string bar(std::string const&);
> };
> X x;
> auto f1 = std::async(&X::foo, &x, 42, "hello");	/* 调用p->foo(42, "hello")，其中p的值是&x */
> auto f2 = std::async(&X::bar, x, "goodbye");
> 
> struct Y{
> 	double operator()(double);
> }
> Y y;
> auto f3 = std::async(Y(), 3.141);
> audo f4 = std::async(std::ref(y), 2.718);
> X baz(X&);
> std::async(baz, std::ref(x));
> ```
>
> ​	按默认情况下，`std::async()`的具体实现会自行决定——等待`future`时，是启动新线程，还是同步执行任务。我们可以给`std::async()`补充一个参数，以指定采用哪种运行方式。参数类型是`std::launch`（`std::launch`是C++11引入的枚举类类型(enum class))，取值可以是`std::launch::defered`或`std::launch::async`。前者指定在线程上延后调用任务函数，得到在`future`上调用了`wait()`或`get()`，任务函数才会执行；该参数的值还可以是`std::launch::defered | std::launch::async`表示由`std::async`的实现自行选择运行方式，它是参数的默认值。若延后调用任务函数，则任务函数有可能永远不会运行。
>
> ```c++
> auto f6 = std::async(std::launch::async, Y(), 1.2);	//运行新线程
> auto f7 = std::async(std::launch::defered, baz, std::ref(x));	//在wait()或get()内部运行任务函数
> auto f8 = std::async(std::launch::defered : std::launch::async, baz, std::ref(x)); //交由实现自行选择运行方式
> auto f9 = std::async(baz, std::ref(x));
> f7.wait(); //新线程被延后，到这里才运行
> ```
>
> **`std::thread`和`std::async`的区别**
>
> ```c++
> // std::thread 和 std::async 的区别
>     // std::thread： 创建线程，如果系统资源紧张，那么创建线程就会失败，则执行到这句时，可能会导致崩溃
>     //               拿到线程的返回值会比较难，需要用一个全局变量来接
>     // std::async：  创建异步任务，可能创建线程，也可能不创建线程
>     //               很容易拿到线程的返回值，可以用future来接
>     // 由于系统资源的限制
>     // （1）如果用 std::thread() 创建的线程太多，则可能创建失败，导致系统报告异常，崩溃
>     // （2）如果用 std::async() 一般不会报告异常，
>     //        如果系统资源紧张，则这种不加额外参数的调用，就不会创建新线程
>     //        后续如果谁调用了result.get() 来请求结果，那么这个异步任务就运行在这条get()语句所在的线程上
>     // （3）总结：一个程序里，线程数量不宜超过100-200。
> 
>     // std::async() 不确定性
>     // 不带参数的情况，存在不确定性，所以需要判断其是否创建了线程
>     std::future<int> result = std::async(std::launch::deferred, &MyClass::MyThread, &ele, 5);
>     std::future_status status = result.wait_for(std::chrono::seconds(0));  // 只需要等0秒就可以判断是否被延迟
>     if (status == std::future_status::deferred)
>     {
>         // 可以通过状态是否被延迟调用，来判断是否创建了新线程
>         result.get();     // 这个时候才会在当前的线程中调用MyThread 入口函数
>     }
> ```

### 3.3.2 关联`future`实例和任务

> ​	`std::packaged_task<>`连结了`future`对象与函数。`std::package_task<>`对象在执行任务时，会调用关联的函数，把返回值保存为`future`的内部数据，并令`future`准备就绪。
>
> ​	`std::packaged_task<>`是类模板，其模板参数是函数签名（function signature）：譬如，`void()`表示一个函数，不接收参数，也没有返回值。由于模板先行指定了函数签名，因此传入的函数必须与之相符，即它应接收指定类型的参数，返回值也必须是可以转换为指定类型。
>
> ​	`std::packaged_task<>`具有成员函数`get_future()`，它返回`std::future<>`实例，该`future`的特化类型取决于函数签名所指定的返回值。`std::packaged_task<>`还具备函数调用操作符，它的参数取决于函数签名的参数列表。
>
> ```c++
> template<>
> class packaged_task<std::string(std::std::vector<char>*, int)>{
> public:
> 	template <typename Callable>
> 	explicit packaged_task(Callable&& f);
> 	std::future<std::string> get_future();
> 	void operator()(std::vector<char>*, int);
> }
> ```
>
> ​	`std::packaged_task`对象是可调用对象，我们可以直接调用，还可以将其包装在`std::future`对象内，当作线程函数传递给`std::thread`对象，也可以传递给需要可调用对象的函数。若`std::packaged_task`作为函数对象而被调用，它就会通过函数调用操作符接收参数，并进一步传递给包装在内的任务函数，由其异步运行得出结果，并将结果保存到`std::future`对象内部，再通过`get_future`获取此对象。因此，为了在未来的适当时刻执行某项任务，我们可以将其包装在`std::packaged_task`对象内，取得对应的`future`之后，才把该对象传递给其他线程，由它触发任务执行。等到需要使用结果时，我们静候`future`准备就绪即可。
>
> **在线程间传递任务**
>
> ​	许多图形用户界面框架都设立了专门的线程，作为更新界面的实际执行者。若别的线程需要更新界面，就必须向它发送消息，由它执行操作。该模式可以运用`std::packaged_task`实现。
>
> ```c++
> #include <deque>
> #include <mutex>
> #include <future>
> #include <thread>
> #include <utility>
> 
> std::mutex m;
> std::deque<std::packaged_task<void()>> tasks;
> bool gui_shutdown_message_received();
> void get_and_process_gui_message();
> 
> void gui_thread(){
> 	while(!gui_shutdown_message_received()){
> 		get_and_process_gui_message();
> 		std::packaged_task<void()> task;
> 		{
> 			std::lock_guard<std::mutex> lk(m);
> 			if(tasks.empty())
> 				continue;
> 			task = std::move(tasks.front);
> 			tasks.pop_front();
> 		}
> 		task();
> 	}
> }
> 
> std::thread gui_bg_thread(gui_thread);
> template<typename Func>
> std::future<void> post_task_for_gui_thread(Func f){
> 	std::packaged_task<void()> task(f);
> 	std::future<void> res = task.get_future();
> 	std::lock_guard<std::mutex> lk(m);
> 	tasks.push_back(std::move(task));
> 	return res;
> }
> ```
>
> ​	注意：直接调用`packaged_task<>`实例时并不会创建新的线程。
>
> ```c++
> #include<iostream>
> #include<future>
> using namespace std;
> int main()
> {
> 	cout << "Main start id = " << this_thread::get_id() << endl;
> 	packaged_task<int(int)> package([](int num) {
> 		cout << "Thread start id = " << this_thread::get_id() << endl;
> 		chrono::milliseconds time(5000); //休息5秒
> 		this_thread::sleep_for(time);
> 		cout << "Thread end id = " << this_thread::get_id() << endl;
> 		return num * 2;
> 	});
> 	  
> 	package(5);                                 //直接调用，并未创建新线程
> 	future<int> result = package.get_future();  //future对象里保存线程的执行结果
> 	cout << "result = " << result.get() << endl;
> 	return 0;
> }
> ```

### 3.3.3 创建`std::promise`

> ​	`	std::promise<T>`给出了一种异步求值的方法（类型为T），某个`std::future<T>`对象与结果关联，能延后读出需要求取的值。其工作机制是：等待数据的线程在`future`上阻塞，而提供数据的线程利用相配的`promise`设定关联的值，使`future`准备就绪。
>
> ​	调用`get_future`成员函数，可以获得`std::promise`实例的`std::future`对象。`promise`的值通过成员函数`set_value`设置，设置好，`future`即准备就绪，凭借它能获取该值。如果`std::promise`在被销毁时仍未曾设置值，保存的数据则由异常代替。

### 3.3.4 保存异常到`future`

> ​	考虑下面的代码片段。假设我们向函数`square_root()`传入-1，它就会抛出异常，为调用者所发现。
>
> ```c++
> double square_root(double x){
> 	if(x < 0){
> 		throw std::out_of_range("x < 0");
> 	}
> 	return sqrt(x);
> }
> ```
>
> ​	现在，我们改为异步调用的形式运行该函数：
>
> ```c++
> std::future<double> f = std::async(square_root, -1);
> double y = f.get();
> ```
>
> ​	若经由`std::async`调用的函数抛出异常，则会被保存到`future`中，代替本该设定的值，`future`随之进入就绪状态，等到其成员函数`get()`被调用，存储在内的异常即被重新抛出。
>
> ​	同样地，`std::promise`也是如此，它通过成员函数的显式调用实现。当发生异常时，应该调用成员函数`set_exception()`。若`calcutlate_value()`发生异常，则会在后续的`catch`块中捕获并装填到`promise`。
>
> ```c++
> extern std::promise<double> some_promise;
> try{
> 	some_promise.set_value(calculate_value());
> }
> catch(...){
> 	some_promise.set_exception(std::current_exception());
> }
> ```
>
> ​	这里的`std::current_exception()`用于捕获抛出的异常。此外，我们还能用`std::std::make_exception_ptr()`直接保存异常，而不触发抛出异常行为：
>
> ```c++
> some_promise.set_exception(std::make_exception_ptr(std::logic_error("foo")));
> ```
>
> ​	如果`std::future`未能就绪，就销毁`std::promise`或`std::packaged_task`，其析构函数都会将异常`std::future_error`存储为异步任务的状态数据，它的值是错误代码`std::future_errc::broken_promise`。我们一旦创建`future`对象，便是许诺会按异步方式给出值或异常，但刻意销毁产生它们的来源，就无法提供所求的值或出现的异常，导致许诺被破坏。这时候，倘若编译器不向`future`存入任何数据，则等待的线程有可能永远等不到结果。

### 3.3.5 多个线程一起等待

> ​	若我们在多个线程上访问同一个`std::future`对象，而不采取额外的同步措施，将引发数据竞争并导致未定义行为。这是`std::future`的特性：它模拟了对异步结果的独占行为，`get()`仅能被有效调用唯一一次。只有一个线程可以获取目标值，原因是第一次调用`get()`会进行移动操作，之后该值不复存在。
>
> ​	`std::future`仅能移动构造 和移动赋值，所以归属权可在多个实例之间转移，但在相同时刻，只会有唯一一个`future`实例指向特定的结果；`std::shared_future`的实例则能复制出副本，因此，我们可以持有该类的多个对象，它们全指向同一个异步任务的状态数据。
>
> ​	`std::future`的首选方式是，向每个线程传递`std::shared_future`对象的副本，它们为各线程独自所有，并视为局部变量。因此，这些副本就作为各线程的内部数据，由标准库正确地同步，可以安全地访问。
>
> ​	`future`和`promise`都有成员函数`valid()`，用于判别异步状态是否有效。
>
> ​	`std::shared_future`的实例依据`std::future`的实例构造而得，前者所指向的异步状态由后者决定。因为`std::future`对象独占异步状态，其归属权不为其它任何对象共有，所以需要用`std::move()`调用移动构造，转移归属权。
>
> ```c++
> std::promise<int> p;
> std::future<int> f(p.get_future());
> assert(f.valid());
> std::shared_future<int> sf(std::move(f)); //对象f将不再有效
> assert(!f.valid());
> assert(sf.valid()); //对象sf生效
> ```
>
> ​	`std::future`还具备成员函数`share()`，直接创建新的`std::shared_future`，并向它转移归属权。
>
> ```c++
> std::shared_future<int> sf = f.share();
> ```

## 3.4 延时等待

> ​	有两种超时（timeout）机制可供选择：一是迟延超时（duration-based timeout），线程根据指定的时长而继续等待（如30毫秒）；而是绝对超时（absolute timeout），在某特定时间点（time point）来临之前，线程一直等待。大部分等待函数都具有变体，专门处理这两种机制的超时。处理迟延超时的函数变体以`_for`为后缀，而处理绝对超时的函数变体以`_until`为后缀。
>
> ​	例如，`std::condition_variable`含有成员函数`wait_for()`和`wait_until()`，它们各自具备两个重载，分别对应`wait()`的两个重载：其中一个重载停止等待的条件是收到信号、超时，或发生伪唤醒；我们需要向另一个重载函数提供断言，在对应线程被唤醒之时，只有该断言成立，它才会返回，如果超时，这个重载函数也会返回。
>
> ​	支持迟延超时的等待需要用到`std::chrono::duration<>`实例。例如，我们等待某个`future`进入就绪状态，并以35毫秒为限：
>
> ```c++
> std::future<int> f = std::async(some_task);
> if(f.wait_for(std::chrono::milliseconds(35)) == std::future_status::ready)
> 	do_something_with(f.get());
> ```
>
> ​	所有等待函数都返回一个状态值，指明是否超时或者目标事件是否已发生。上例中，我们借助`future`进行等待，所以一旦超时，函数就返回`std::future_status::timeout`；假如准备就绪，则函数返回`std::future_status::ready`；若`future`的相关任务被延后，函数返回`std::future_status::defered`（即线程运行被延迟到成员函数`get()或wait()`，共享状态包含延迟函数，只有显示请求才计算结果）。迟延超时的等待需要一个参照标准，它采用了标准库内部的恒稳时钟，只要代码指定了等待35毫秒，那现实中等待的时间就是35毫秒，即使期间系统时钟发生调整。
>
> ​	时间点（`std::chrono::time_point`）用于带有后缀`"_util"`的等待函数的变体。
>
> ```c++
> #include <condition_variable>
> #include <mutex>
> #include <chrono>
> 
> std::condition_variable cv;
> bool done;
> std::mutex m;
> bool wait_loop(){
> 	auto const timeout = std::chrono::steady_clock::now() + std::chrono::miliseconds(500);
> 	std::unique_lock<std::mutex> lk(m);
> 	while(!done){
> 		if(cv.wait_until(lk, timeout) == std::cv_status::timeout)
> 			break;
> 	}
> 	return done;
> }
> ```

## 3.5 接受超时时限的函数

> ​	超时时限的最简单用途是，推迟特定线程的处理过程，若它无所事事，就不会占用其它线程的处理时间。
>
> ​	`std::this_thread::sleep_for()`和`std::this_thread::sleep_until()`，它们的功能就像简单的闹钟：线程或采用`sleep_for`按指定的时长休眠，或采用`sleep_until()`休眠直到指定时刻为止。`sleep_until()`函数允许我们在特定的时刻唤醒线程。这可以允许我们在特定的时间唤醒线程。这可以用于夜间数据备份程序，或在早晨6:00运行工资单输出程序，或播放影像时在两帧画面的刷新之间暂停线程。
>
> ​	超时时限可以配合条件变量，或配合`future`一起使用。只要互斥支持，我们在尝试给互斥加锁的时候，甚至也能设定超时时限。普通的`std::mutex`和`std::recursive_mutex`不能限时加锁，但`std::timed_mutex`和`std::recursive_timed_mutex`可以。这两种锁都含有成员函数`try_lock_for()`和`try_lock_until()`，前者尝试在给定的时长内获取锁，后者尝试在给定的时间点之前获取锁。
>
> ​	下表列出了C++标准库中接受超时时限的函数，并说明了其参数和返回值。
>
> | 类/命名空间                                                  | 函数                                                         | 返回值                                                       |
> | ------------------------------------------------------------ | ------------------------------------------------------------ | ------------------------------------------------------------ |
> | `std::this_thread`命名空间                                   | `sleep_for(duratio)`，<br />`sleep_until(time_point)`        | 无                                                           |
> | `std::condition_variable`或<br />`std::condition_variable_any`，<br />`wait_for(lock, duration)`，<br />`wait_unitl(lock, time_point)` | `std::cv_status::timeout`或<br />`std::cv_status::no_timeout`，<br />`wait_for(lock, duration, predicate)`，<br />`wait_until(lock, time_point, predicate)` | `Bool`——被唤醒时断言的返回值                                 |
> | `std::timed_mutex`，<br />`std::recursive_timed_mutex`，<br />`std::shared_timed_mutex` | `try_lock_for(duration)`，<br />`try_lock_until(time_point)` | `Bool`——若获取了锁，则返回`true`，否则返回`false`            |
> | `std::shared_timed_mutex`                                    | `try_lock_shared_for(duration)`，<br />`try_lock_shared_until(time_poine)` | `Bool`——若获取了锁，则返回`true`，否则返回`false`            |
> | `std::unique_lock<TimedLockable>`，<br />`unique_lock<lockable, duration>`，<br />`unique_lock<lockable, time_point>` | `try_lock_for(duration)`，<br />`try_lock_until(time_point)` | 无——如果在构建的对象上获取了锁，那么`owns_lock()`返回`true`，否则返回`false`。<br />`Bool`若获取了锁，则返回`true`，否则返回`false`。 |
> | `std::shared_lock<SharedTimedLockable>`，<br />`shared_lock(lockable, duration)`，<br />`shared_lock(lockable, time_point)` | `try_lock_for(duration)`，<br />`try_lock_until(time_point)` | 无——如果在构建的对象上获取了锁，那么`owns_lock()`返回`true`，否则返回`false`。<br />`Bool`若获取了锁，则返回`true`，否则返回`false`。 |
> | `std::future<ValueType>`或`std::shared_future<ValueType>`    | `wait_for(duraion)`，<br />`wait_until(time_point)`          | 如果等待超时则返回`std::future_status::timeout`，如果`future`已就绪则返回`std::future_status::ready`。如果`future`上的函数按推迟方式执行，且尚未开始执行，则返回`std::future_status::deferred`。 |

## 3.6 使用同步操作简化代码

### 3.6.1 利用`future`进行函数式编程

> ​	若要以C++实现函数式编程风格的并发编程，`future`则是画龙点睛之笔，使之真正切实可行；`future`对象可以在线程间传递，所以一个计算任务可以依赖另一个任务的结果，却不必显式地访问共享数据。
>
> ```c++
> template<typename T>
> std::list<T> sequential_quick_sort( std::list<T> input ) {
>     if (input.empty()) {
>         return input;
>     }
>     std::list<T> result;
>     result.splice( result.begin() , input , input.begin() );
>     T const& pivot = *result.begin();
> 
>     auto divide_point = std::partition( input.begin() , input.end() [ & ]( T const& t ) {return t < pivot;} );
>     std::list<T> lower_part;
>     lower_part.splice( lower_part.end() , input , input.begin() , divide_point );
>     auto new_lower( sequential_quick_sort( std::move( lower_part ) ) );
>     auto new_higher( sequential_quick_sort( std::move( input ) ) );
>     result.splice( result.end() , new_higher );
>     result.splice( result.begin() , new_lower );
>     return result;
> }
> ```
>
> **并发快速排序**
>
> ```c++
> template<typename T>
> std::list<T> parallel_quick_sort( std::list<T> input ) {
>     if (input.empty()) {
>         return input;
>     }
>     std::list<T> result;
>     result.splice( result.begin() , input , input.begin() );
>     T const& pivot = *result.begin();
> 
>     auto divide_point = std::partition( input.begin() , input.end() [ & ]( T const& t ) {return t < pivot;} );
>     std::list<T> lower_part;
>     lower_part.splice( lower_part.end() , input , input.begin() , divide_point );
>     std::future<std::list<T>> new_lower(std::async(&parallel_quick_sort<T>, std::move(lower_part)));
>     std::future<std::list<T>> new_higher( std::async(&parallel_quick_sort<T>, std::move(input)) );
>     result.splice( result.end() , new_higher );
>     result.splice( result.begin() , new_lower );
>     return result;
> }
> ```
>
> ​	如果递归层数太大，比如10层，则将有1024个线程同时运行。所以一旦线程库断定所产生的任务过多（原因也许是任务数目超出了可调配的硬件并发资源），就有可能转为按同步方式生成新任务。假如向其它线程传递任务无助于提升性能，那么负责调用`get()`的线程就会亲自执行新任务，而不再发起新线程，从而减小开销。

### 3.6.2 等待多个`future`

> ​	假定有大量数据需要处理，而且每项数据都能独立完成，此时，我们只要生成一组异步任务，分别处理数据，使之通过`future`各自返回处理妥当的项目，就能充分利用手头的硬件资源。我们可能需要专门生成一个线程，交由该任务统筹等待`future`就绪，或不停查验`future`状态，等到它们全部准备就绪才生成新任务，最后汇总。
>
> ```c++
> std::future<FinalResult> process_data(std::vector<MyData>& vec){
> 	size_t const chunk_size = whatever;
> 	std::vector<std::future<ChunkResult>> results;
> 	for(auto begin = vec.begin(), end = vec.end(); beg != end;){
> 		size_t const remaining_size = end - begin;
> 		size_t const this_chunk_size = std::min(remaining_size, chunk_size);
> 		results.push_back(std::async(process_chunk, begin, begin_this_chunk_size));
> 		begin += this_chunk_size;
> 	}
> 	return std::async([all_result=std::move(results)](){
> 		std::vector<ChunkResult> v;
> 		v.reverse(all_result.size());
> 		for(auto& f : all_results){
> 			v.push_back(f.get());
> 		}
> 		return gather_results(v);
> 	});
> }
> ```
>
> ​	这段代码新生成了一个异步任务，专门等待收集分项结果，只要这些结果全部得出，就进行最后的汇总处理。然后，因为它等待每个任务，但凡有分项结果得出，就会被系统任务调度器唤醒。不过，若它发现有任务尚未得出结果，就会再次休眠。这既会占据等待的线程，更会在各个`future`就绪时，导致多余的上下文切换，带来额外的开销。
>
> ​	上述“等待-切换”的行为实属无谓。采用`std::experimental::when_all()`函数即可避免。我们向该函数传入一系列需要等待的`future`，由它生成并返回一个新的总的总领`future`，等到传入的`future`全部就绪，此总领`future`也随之就绪。接下来就能运用这个总领`future`安排更多工作。例如：
>
> ```c++
> 
> std::experimental::future<FinalResult> process_data(std::vector<MyData>& vec){
> 	size_t const chunk_size = whatever;
> 	std::vector<std::experimental::future<ChunkResult>> results;
> 	for(auto begin = vec.begin(), end = vec.end(); begin != end;){
> 		size_t const remining_size = end - begin;
> 		size_t const this_chunk_size = std::min(remaining_size, chunk_size);
> 		results.push_back(spawn_async(
> 		process_chunk, begin, begin+this_chunk_size));
> 		begin += this_chunk_size;
> 	}
> 	return std::experimental::when_all(
> 	results.begin, results.end()).then([](std::future<std::vector<std::experimental::future<ChunkResult>>> ready_results){
> 	std::vector<std::experimental::future<ChunkResult>>
> 		all_results = ready_results.get();
> 	std::vector<ChunkResult> v;
> 	for(auto& f : all_results){
> 		v.push_back(f.get());
> 	}
> 	return gather_results(v);
> 	});
> }
> ```

# 4 C++内存模型和原子操作

> ​	C++标准将“对象”定义为“某一存储范围”（a region of storage）。C++中每个变量都是对象，对象的数据成员也是对象；每个对象都占用至少一块内存区域；若变量属于内建基本类型（如`int`或`char`），则不论是其大小，都占用一块内存区域（且仅此一块），即便它们的位置相邻或是它们是数列中的元素；相邻的位域属于同一内存区域。
>
> ​	要避免条件竞争，就必须强制两个线程按一定的次序访问。这个次序可以固定不变，即某一访问总是先于另一个；也可以变动，即随应用程序的运行而间隔轮换访问的次序。我们可以使用互斥来保证每次仅容许一个线程访问目标内存区域，遂一个访问必然先于另一个；另一种方法是，利用原子操作的同步性质，在目标内存区域（或相关内存区域）采取原子操作，从而强制两个线程遵从一定的访问程序次序。

## 4.1 C++中的原子操作及其类别

> ​	原子操作是不可分割的操作。在系统的任一线程内，我们都不会观察到这种操作处于半完成状态；它或者完全做好，或者完全没做。与之相反，非原子操作在完成一半的时候，有可能为另一个线程所见。假定由原子操作组合出非原子操作，例如向结构体的原子数据成员赋值，那么别的线程有可能观察到其中的某些原子操作已经完成，而某些却还没开始，若多个线程同时赋值，而底层操作相交进行，本来意图完整存入的数据就会彼此错综复杂。

## 4.2 标准原子类型

> ​	标准原子类型的定义位于`<atomic>`内。这些类型的操作全是原子化的，并且，根据语言的定义，C++内建的原子操作也仅仅支持这些类型，尽管通过采用互斥，我们也能够令其它操作实现原子化。它们全部都具备成员函数`is_lock_free()`，准许使用者判定某一给定类型上的操作是由原子指令直接实现（`x.is_lock_free()`返回`true`），还是要借助编译器和程序库的内部锁来实现（`x.is_lock_free()`返回`false`）。假设原子操作内部本身也使用了互斥，就很可能无法到达所期望的性能的提升，而更好的做法是基于互斥的方式，该方式更加直观其不易出错。
>
> ​	只有一个原子类型不提供`is_lock_free()`成员函数：`std::atomic_flag`。它是简单的布尔标志，因此必须采取无锁操作。只要利用这种简单的无锁布尔标志，我们就能实现一个简单的锁，进而基于该锁实现其它所有原子类型。这里说简单，确实如此：类型`std::atomic_flag`的对象在初始化时清零，随后即可通过成员函数`test_and_set()`查值并设置成立，或者由`clear`清零。整个过程只有两个操作。没有赋值，没有拷贝构造，没有“查值并清零”的操作，也没有任何其他的操作。
>
> ​	C++中的大多数内建类型的原子类型都是通过模板`std::atomic`特化得出的，功能更加齐全，但可能不属于无锁结构。**由于不具备拷贝构造函数和拷贝赋值操作符**，因此按照传统的做法，**标准的原子类型对象无法复制，也无法赋值。然而，它们其实可以接受内建类型赋值，也支持隐式地转换成内建类型，还可以直接经由成员函数处理，**如`load()`和`store()`、`exchange()`、`compare_exchange_weak()`和`compare_exchange_strong()`等。它们还支持复合赋值操作，如`+=`、`-=`、`*=`和`|=`。而且，整型和指针的`std::atomic<>`特化都支持`++`和`--`运算。这些操作符都有对应的具名成员函数。赋值操作符的返回值是存入的值，而具名成员函数的返回值则是进行操作前的值。习惯上，C++的赋值操作符通常返回引用，指向赋值的对象，但原子类型的设计与此有别，要防止暗藏错误。否则，为了从引用获得存入的值，代码必须指向单独的读取操作，使赋值和读取操作之间存在间隙，让其它线程有机可乘，得以改动该值，结果形成条件竞争。
>
> ​	当然，类模板`std::atomic<>`并不局限于上述特化类型。它其实具有泛化模板（primary template），可依据用户自定义类型创建原子类型的变体。该泛化目标具备的操作仅限于以下几种：`load()`、`store()`（接受用户自定义类型的赋值，以及转换为用户自定义类型）、`exchange()`、`compare_exchange_weak()`、`compare_exchange_strong()`。
>
> ​	对于原子类型上的每一种操作，我们都可以提供额外的参数，从枚举类`std::memory_order`取值，用于设定所需的内存次序语义（memory-ordering semantics）。枚举类`std::memory_order`具有6个可能的值，包括`std::memory_order_relaxed`、`std::memory_order_acquire`、`std::memory_order_consume`、`std::memory_order_acq_rel`、`std::memory_order_release`和`std::memory_order_seq_cst`。
>
> ​	操作的类别决定了内存次序所准许的取值。若我们没有把内存次序显式地设定，则默认采用最严格的内存次序，即`std::memory_order_seq_cst`。目前，知道的操作可以划分为3类就已经足够了：
>
> * 存储（store）操作，可选用的内存次序有`std::memory_order_relaxed`、`std::memory_order_release`或`std::memory_order_seq_cst`。
> * 载入（load）操作，可选用的内存次序有`std::memory_order_relaxed`、`std::memory_order_consume`、`std::memory_order_acquire`或`std::memory_order_seq_cst`。
> * “读-改-写”（read-modify-write）操作，可选用的内存次序有`std::memory_order_relaxed`、`std::memory_order_acquire`、`std::memory_order_consume`、`std::memory_order_acq_rel`、`std::memory_order_release`和`std::memory_order_seq_cst`。

### 4.2.2 操作`std::atomic_flag`

> ​	`std::atomic_flag`是最简单的标准原子类型，表示一个布尔标志。该类型的对象只有两种状态：成立或置零。二者必居其一。
>
> ​	`std::atomic_flag`类型的对象必须由宏`ATOMIC_FLAG_INIT`初始化，它把标志初始化为置零状态：`std::atomic_flag f = ATOMIC_FLAG_INIT`。无论在哪里声明，也无论什么作用域，`std::atomic_flag`对象永远以置零状态开始，别无他选。全部原子类型中，只有`std::atomic_flag`必须采用这种特殊的初始化处理，它也是唯一保证无锁的原子类型。如果`std::atomic_flag`对象具有静态存储期，它就会保证以静态方式初始化，从而避免初始化次序问题。 对象在完成初始化后才会操作其标志。
>
> ​	完成`std::atomic_flag`对象的初始化后，我们只能对其执行3种操作：销毁、置零、读取原有值并设置标志成立。这分别对应于析构函数、成员函数`clear()`、成员函数`test_and_set()`。我们可以为`clear()`和`test_and_set()`指定内存次序。`clear()`是存储操作，因此无法采用`std::memory_order_acquire`或`std::memory_order_acq_rel`次序，而`test_and_set()`是“读-改-写”操作，因此可以采用任何内存次序。对于上面的两个原子操作，默认内存次序都是`std::memory_order_seq_cst`。
>
> ```c++
> f.clear(std::memory_order_release);
> bool x = f.test_and_set();
> ```
>
> ​	上面的代码中，`clear()`的调用显式的采用释放语义将标志清零，而`test_andO_set()`的调用采用默认的内存次序，获取旧值并设置标志成立。
>
> ​	我们无法从`std::atomic_flag`对象拷贝出另一个对象，也无法向一个对象拷贝赋值，这两个限制并非`std::atomic_flag`独有，所有原子类型都同样受限。
>
> ​	`std::atomic_flag`可以完美地扩展成自旋锁互斥（spin-lock mutex）。最开始令原子标志置零，表示互斥没有加锁。我们反复调用`test_and_set()`试着锁住互斥，一旦读取的值变成了`false`，则说明线程已将标志设置成立（其新值为`true`），则循环终止。而简单地将标志置零即可解锁互斥。
>
> ```c++
> class spinlock_mutex{
> 	std::atomic_flag flag;
> public:
> 	spinlock_mutex(): flag(ATOMIC_FLAG_INIT);
> 	void lock(){
> 		while(flag.test_and_set(std::memory_order_acquire));
> 	}
> 	void unlock(){
> 		flag.clear(std::memory_order_release);
> 	}
> };
> ```
>
> ​	这个互斥非常简单，但已经能够配合`std::lock_guard<>`运用自如，足以保证互斥发挥功效。然而，上述的自旋锁互斥在`lock()`内忙等待。因此，若我们不希望出现任何程度的竞争，那么该互斥远非最佳选择。后面我们会讲解内存次序的语义，让读者明白，它如何确保互斥锁强制施行必要的内存次序。
>
> ​	`std::atomic_flag`严格受限，甚至不支持单纯的无修改查值操作（nonmodifying query），无法用作普通的布尔标志。

### 4.2.3 操作`std::atomic<bool>`

> ​	`std::atomic<bool>`是基于整数的最基本的原子类型。相比`std::atomic_flag`，它是一个更齐全的布尔标志。尽管`std::atomic<bool>`也无法拷贝构造或拷贝赋值，但我们还是能依据非原子布尔量创建其对象，初始值是`true`或`false`皆可。该类型的实例还能接受非原子布尔量的赋值。
>
> ```c++
> std::atomic<bool> b(true);
> b = false;
> ```
>
> ​	我们还要注意一点，按C++惯例，赋值操作符通常返回一个引用，指向接受赋值的目标对象（等号左侧的对象）。而非原子布尔量也可以向`std::atomic<bool>`赋值，但该赋值操作符的行为有别于惯常做法：它直接返回赋予的布尔值。这是原子类型的又一个常见模式：它们所支持的赋值操作符不返回引用，而是按值返回（该值属于对应的非原子类型）。假设返回的是指向原子变量的引用，若有代码依赖赋值操作的结果，那它必须随之显式地加载该结果的值，而另一个线程有可能在返回和加载间改动其值。我们按值返回赋值操作的结果（该值属于非原子类型），就会避开多余的加载动作，从而确保获取的值正是赋予的值。
>
> ​	`std::atomic_flag`的成员函数`clear()`严格受限，而`std::atomic<bool>`的写操作有所不同，通过调用`store()`(`true`和`false`皆可)，它也能设定内存次序语义。类似的`std::atomic<bool>`提供了更通用的成员函数`exchange()`以代替`test_and_set()`，它获取原有的值，还让我们自行选定新值作为替换。
>
> ​	`std::atomic<bool>`还支持单纯的读取（没有伴随的修改行为）：隐式做法是将实例转换为普通布尔值，显式做法则是调用`load()`。
>
> ​	不难看出，`store()`是存储操作，而`load()`是载入操作，但`exchange()`是“读-改-写”操作。
>
> ```c++
> std::atomic<bool> b;
> bool x = b.load(std::memory_order_acquire);
> b.store(true);
> x = b.exchange(false, std::memory_order_acq_rel); //将b的值改为false，并返回先前的值
> ```
>
> ​	`std::atomic<bool>`还引入了一种操作：若原子对象当前的值符合预期，就赋予新值。它与`exchange`一样，同为“读-改-写”操作。
>
> **依据原子对象当前的值决定是否保存新值**
>
> ​	这一新操作被称为“比较-交换”（compare-exchange），实现形式是成员函数`compare_exchange_weak()`和`compare_exchange_strong()`。比较-交换操作是原子类型的编程基石。使用者给定一个期望值原子变量将与它比较，如果相等，就存入另一既定的值；否则，**更新期望值所属的变量，向它赋予原子变量的值**。比较-交换函数返回布尔类型，如果完成了保存动作（前提是两值相等），则操作成功，函数返回`true`；反之操作失败，函数返回`false`。
>
> ​	对于`compare_exchange_weak()`，即使原子变量的值与期望值相等，保存动作还是有可能失败，在这种情形下，原子变量维持原值不变，`compare_exchange_weak()`返回`false`。原子化的比较-交换必须由一条指令单独完成，而某些处理器没有这种指令，无从保证该操作按原子化方式完成。要实现比较-交换，负责的线程则须改为连续运行一系列指令，但在这些计算机上，只要出现线程数量多于处理器数量的情形，线程就有可能执行到中途因系统调度而切出，导致操作失败。这种计算机最有可能引发上述的保存失败，我们称之为佯败（spurious failure）。其败因不是变量值本身存在问题，而是函数执行时机不对，因为`compare_exchange_weak()`可能佯败，所以它往往必须配合循环使用。
>
> ```c++
> bool expected = false;
> extern atomic<bool> b;	//由其它源文件的代码设定的变量的值
> while(!b.compare_exchange_weak(expected, true) && !expected);
> ```
>
> ​	只要`expected`变量还是`false`，就说明`compare_exchange_weak()`的调用发生佯败，我们继续循环。
>
> ​	另一方面，只有当原子变量的值不符合预期时，`compare_exchange_strong()`才返回`false`。这让我们得以明确知悉变量是否成功修改，或者是否存在另一个线程抢先切入而导致佯败，从而能摆脱上例所示的循环。

## 4.3 同步操作和强制次序

> ​	假设有两个线程共同操作一个数据结构，其中一个负责增添数据，另一个负责读取数据。为了避免恶性条件竞争，写线程设置一个标志，用以表示数据已经存储妥当，而读线程则一直待命，等到标志成立才着手读取。
>
> ```c++
> #include <vector>
> #include <atomic>
> #include <iostream>
> std::vector<int> data;
> std::atomic<bool> data_ready(false);
> void reader_thread(){
> 	while(!data_ready.load()){	//读(1)
> 		std::this_thread::sleep(std::chrono::milliseconds(1));
> 	}
> 	std::cout << "The answer = " << data[0] << "\n"; //读(1)
> }
> 
> void write_thread(){
> 	data.push_back(42); //写(3)
> 	data_ready = true;  //写(4)
> }
> ```
>
> ​	在一份数据上进行非原子化的读(1)和写(3)，却没有强制这些访问服从一定的次序，将导致未定义行为，因此要让代码正确运作，就必须为其施行某种次序。
>
> ​	我们可以利用原子变量强制其它非原子操作遵从预定次序。原子变量`data_ready`的操作提供了所需的强制次序，它属于`std::atomic<bool>`，凭借两种内存模型关系“先行”（happens-before）和同步（synchronizes-with），这些操作确定了必要的次序。数据写出(3)在标志变量`data_ready`设置为成立(4)之前发生，标志判别(1)在数据读取(2)之前发生。等到从变量`data_ready`读取的值变成了`true`(1)，写出动作与该读取动作即达成同步，构成先行关系。数据写出(3)在标志设置成立(4)之前发生，且(4)在从标志读取`true`值(1)之前发生，而(1)在读取数据(2)之前发生，又因为先行关系可传递，所以这些操作被强制施行了预定的次序：数据写出在数据读取操作前面发生。
>
> ​	这些先行关系看起来相当直观：某个值的写出操作在其读取操作之前发生。这对默认原子操作自然成立（因为它就是按照默认次序），但在代码中还是应该明确设定采用这一次序，因为原子操作还能根据需要选取其它次序。

### 4.3.1 同步关系

> ​	同步关系只存在于原子类型的操作之间。如果一种数据结构含有原子类型，并且其整体操作都涉及恰当的内部原子操作，那么该数据结构的多次操作之间（如锁定互斥）就可能存在同步关系。同步关系的基本思想是：对变量`x`执行原子写操作$W$和原子读操作$R$，且两者都有适当的标记。只要满足下面其中一点，它们彼此同步：
>
> * $R$读取了$W$直接存入的值。
>
> * $W$所属线程随后还执行了另一原子写操作，$R$读取了后面存入的值。
>
> * 任意线程执行一连串“读-改-写”操作（如`fetch-add()`或`compare_exchange_weak()`），而其中第一个操作读取的值由$W$写出。
>
>   原子类型上的全部操作都默认添加适当的标记，它的含义是，在C++内存模型中，操作原子类型时所受的各种次序约束。

### 4.3.2 先行关系

> ​	先行关系和严格先行关系是程序确立操作次序的基本要素；它们的用途是清楚界定哪些操作能看见其它哪些操作产生的结果。在单一线程中，这种关系非常直观：若某项操作按控制流程顺序在另一项之前执行，前者即先于后者发生。具体而言，在源代码中，若甲操作的语句位于乙操作之前，那么甲就先于乙发生，且甲严格先于乙发生。但如果同一语句内出现多个操作，则它们之间通常不存在先行关系，因为C++标准没有规定执行次序。例如，
>
> ```c++
> #include <iostream>
> void foo(int a, int b){
> 	std::cout << a << "," << b << std::endl;
> }
> int get_num(){
> 	static int i = 0;
> 	return ++i;
> }
> int main(){
> 	foo(get_num(), get_num());	//get_num()发生两次调用，但没有明确的先后次序
> }
> ```
>
> ​	某些单一语句中含有的多个操作，还是会按一定的顺序执行：譬如由内建逗号操作符拼接而成的表达式；又如，一个表达式的结果可以充当另一个表达式的参数。但单一语句中的多个操作往往没有规定次序，它们之间不存在控制流程的先后关系，因而没有先行关系，一条语句的所有操作全在下一条语句的所有操作之前执行。
>
> ​	线程间先行关系依赖于同步关系：如果甲、乙操作分别由不同线程执行，且它们同步，则甲操作跨线程地先于乙操作发生。这也是可传递关系：若甲操作跨线程先于乙操作发生，且乙操作跨线程地先于丙操作发生，则甲跨线程地先于丙操作发生。

### 4.3.3 原子操作的内存次序

> ​	原子类型上的操作服从6种内存次序：`memory_order_relaxed`、`memory_order_consume`、`memory_order_acquire`、`memory_order_release`、`memory_order_acq_rel`和`memory_order_seq_cst`。其中，`memory_order_seq_cst`是可选的最严格的内存次序，各种原子类型的所有操作都默认遵从该次序，除非我们特意为某项操作另行指定。
>
> ​	虽然内存次序共有6种，但它们只代表3种模式：先后一致次序（`memory_order_seq_cst`）、获取-释放次序（`memory_order_comsume`、`memory_order_acquire`、`memory_order_release`和`memory_order_acq_rel`）、宽松次序（`memory_order_relaxed`）。

**1、先后一致次序**

