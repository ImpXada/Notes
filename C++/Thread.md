# Thread

## 传递临时对象作为线程参数

```cpp
void myprint(const int &i,char * pmybuf)
{
	cout << i << endl;
	cout << pmybuf << endl;
}
int main(int, char **)
{
	int mvar=1;
	int &refmvar = mvar;
	char mybuf[] = "this is a text";
	thread t(myprint, mvar, mybuf);
	t.join();
	cout << "main end" << endl;
}
```

程序正常运行，按顺序输出结果。

使用thread创建线程时，即使接收参数是引用，mvar也不是传入的引用。而是拷贝，不是真引用。

因此当使用detach后，i是安全的，因为是拷贝。

传入char[]类型时，当主线程结束后，会自动把传入的参数转换为string

## lock_guard和mutex

```cpp
class A
{
public:
	void inMsgRecvQueue()
	{
		lock_guard<std::mutex> sbguard(my_mutex);
		for (int i = 0; i < 100000; i++)
		{
			msgRecvQueue.push_back(i);
		}
		cout << "done:in" << endl;
	}
	void outMsgRecvQueue()
	{
		
		for (int i = 0; i < 100000; i++)
		{	lock_guard<std::mutex> sbguard(my_mutex);//执行lock_guard的构造函数相当于执行my_mutex.lock()
		//析构时执行unlock()
         //不灵活，可以通过调用析构函数或者使用{}设置作用域
			if (!msgRecvQueue.empty())
			{
				int command = msgRecvQueue.front();
				msgRecvQueue.pop_front();
			}
		}
		cout << "done:out" << endl;
	}

private:
	mutex my_mutex;
	list<int> msgRecvQueue;
};
int main(int, char**)
{
	A myobj;
	thread myOutMsgThread(&A::outMsgRecvQueue, &myobj);
	thread myInMsgThread(&A::inMsgRecvQueue, &myobj);
	myInMsgThread.join();
	myOutMsgThread.join();
}
```

### std::lock 同时上锁多个锁

```cpp
void outMsgRecvQueue()
{
	
	for (int i = 0; i < 100000; i++)
	{	
		std::lock(my_mutex1, my_mutex2);
		if (!msgRecvQueue.empty())
		{
			int command = msgRecvQueue.front();
			msgRecvQueue.pop_front();
		}
		my_mutex1.unlock();
		my_mutex2.unlock();
	}
}
```

### 使用lock_guard配合std::lock，进行自动unlock

```cpp
void outMsgRecvQueue()
{
	for (int i = 0; i < 100000; i++)
	{	
		std::lock(my_mutex1, my_mutex2);
		lock_guard<mutex> guard1(my_mutex1,std::adopt_lock);
		lock_guard<mutex> guard2(my_mutex2,adopt_lock);
		if (!msgRecvQueue.empty())
		{
			int command = msgRecvQueue.front();
			msgRecvQueue.pop_front();
		}
	}
}
传入std::adopt_lock参数，让lock_guard构造函数不调用
```

### recursive_mutex 递归互斥量

能够多次lock的mutex

效率低，有次数限制，尽量使用mutex

### 带超时的互斥量 std::timed_mutex 和 std::recursive_timed_mutex

 std::timed_mutex：是带超时的独占互斥量

#### try_lock_for()：

等待一段时间，如果拿到了锁，或者超时了未拿到锁，就继续执行（有选择执行）如下

```cpp
std::chrono::milliseconds timeout(100);
if (my_mymutex.try_lock_for(timeout)){
    //......拿到锁返回ture
}
else{
    std::chrono::milliseconds sleeptime(100);
    std::this_thread::sleep_for(sleeptime);
}
```

#### try_lock_until()：

参数是一个未来的时间点，在这个未来的时间没到的时间内，如果拿到了锁头，流程就走下来，如果时间到了没拿到锁，流程也可以走下来。

```cpp
std::chrono::milliseconds timeout(100);
if (my_mymutex.try_lock_until(chrono::steady_clock::now() + timeout)){
    //......拿到锁返回ture
}
else{
    std::chrono::milliseconds sleeptime(100);
    std::this_thread::sleep_for(sleeptime);
}
```



## unique_lock

1. 常规使用和lock_guard一样

   `unique_lock<mutex> ulock(my_mutex);`

2. unique_lock有第二个参数，起到标记作用

   1. std::adopt_lock，表示互斥量提前被lock。

      ```cpp
      my_mutex.lock();
      lock_guard<mutex> guard1(my_mutex,std::adopt_lock);
      ```

   2. std::try_to_lock，尝试调用mutex的lock，锁定失败立即返回。非阻塞模式

      ```cpp
      unique_lock<mutex> ulock(my_mutex,try_to_lock);
      	if (ulock.owns_lock())
      	{
      		if (!msgRecvQueue.empty())
      		{
      			int command = msgRecvQueue.front();
      			msgRecvQueue.pop_front();
      		}
      	}
      ```

   3. std::defer_lock（使用前不能先lock，否则会异常）初始化未加锁的mutex

      ```cpp
      unique_lock<mutex> ulock(my_mutex,defer_lock);
      ulock.lock();
      ulock.unlock();//可以手动解锁，一般情况还是自动释放即可
      ```

      

### unique_lock 成员函数

1. lock(),unlock();

   ```cpp
   unique_lock<mutex> ulock(my_mutex,defer_lock);
   ulock.lock();
   ulock.unlock();//可以手动解锁，一般情况还是自动释放即可
   ```

2. try_lock()

   ```cpp
   if(ulock.try_lock()==true)//true表示拿到锁了
   ```

3. release()，返回所管理的mutex对象指针，并释放所有权

   ```cpp
   std::mutex *ptr=ulock.release();
   //现在需要自行解锁mutex了
   ptr->unlock();
   ```

### 所有权转移

unique_lock会和一个mutex绑定

ulock拥有my_mutex的所有权

ulock可以把所有权转移给其他对unique_lock对象

`unique_lock<mutex> ulock2(std::move(ulock));`

## 单例模式

### 标准的单例模式

```cpp
class MyCAS
{
private:
	MyCAS() {}

public:
	static MyCAS* GetInstance()
	{
		if (m_instance == nullptr)
		{
			m_instance = new MyCAS;
			static Release cl;
		}
		return m_instance;
	}
	class Release
	{
	public:
		~Release()
		{
			if (MyCAS::m_instance)
			{
				delete MyCAS::m_instance;
				MyCAS::m_instance == nullptr;
			}
		}
	};
private:
	static MyCAS* m_instance;
};
//单例模式一般程序开始便初始化，结束会被自动释放，因此不用考虑析构的问题，11行，16-27行可以删除
```

### 线程不安全问题

```cpp
class MyCAS
{
private:
	MyCAS() {}

public:
	static MyCAS* GetInstance()
	{
		if (m_instance == nullptr)
		{
			m_instance = new MyCAS;
		}
		return m_instance;
	}
	void func()
	{
		cout << "func" << endl;
	}

private:
	
	static MyCAS* m_instance;
};
MyCAS* MyCAS::m_instance = nullptr;

void mythread()
{
	cout << "thread start" << endl;
	MyCAS* p_a = MyCAS::GetInstance();
	cout << "thread end" << endl;
}

int main(int, char**)
{
	thread myobj1(mythread);
	thread myobj2(mythread);
	myobj1.join();
	myobj2.join();
}
//obj1执行到11行时发生了上下文切换，obj2开始执行，因此new了两次，发生二义性
```

可以设置全局mutex，调用到Get Instance时加锁，但效率太低

```cpp
mutex resourse_mutex;
class MyCAS
{
private:
	MyCAS() {}

public:
	static MyCAS *GetInstance()
	{
		unique_lock<mutex> ulock(resourse_mutex);
		if (m_instance == nullptr)
		{
			m_instance = new MyCAS;
		}
		return m_instance;
	}
	void func()
	{
		cout << "func" << endl;
	}

private:
	static MyCAS *m_instance;
};
```

引入双重锁定

```cpp
static MyCAS *GetInstance()
{
	if (m_instance == nullptr)
	{
		unique_lock<mutex> ulock(resourse_mutex);
		if (m_instance == nullptr)
		{
			m_instance = new MyCAS;
		}
	}
	return m_instance;
}
//m_instance不为nullptr一定被new过
//==nullptr 不代表一定没被new过（未加mutex时）
```

#### call_once

该函数第二个参数是一个函数名，能够保证函数只被调用一次。

具备互斥量能力，效率比互斥量更高效

需要配合std::once_flag标记使用

## conditional_variable

### notify_one() and wait()

线程A：等待一个条件满足，等待消息队列有消息

线程B：往消息队列存数据，有数据就通知线程A，线程A从等待的地方执行

```cpp
class A
{
public:
	void inMsgRecvQueue()
	{
		lock_guard<std::mutex> sbguard(my_mutex);
		for (int i = 0; i < 100000; i++)
		{
			msgRecvQueue.push_back(i);
			my_cond.notify_one();//尝试把out函数中的wait的线程唤醒
		}
		cout << "done:in" << endl;
	}
	void outMsgRecvQueue()
	{
		int command = 0;
		for(int i=0;i<100000;++i)
		{
			unique_lock<mutex> sbguard(my_mutex);

			//wait是条件变量的成员函数，等待一个东西
			//lambda表达式是第二个参数，如果第二个参数返回值是false，那么wait将解锁互斥量，并阻塞到本行
			//阻塞到其他线程调用notify_one()成员函数为止。
			// 返回ture，wait直接返回。继续运行
			//如果wait()没有第二个参数：那么就跟第二个参数返回false效果一样。
			// 
			//当其他线程用notify_one将阻塞中的wait唤醒后，wait会不断尝试获取锁，获取不到就卡住等待获取成功
			//获取成功后会再次判断lambda，false->阻塞，true->执行，如果没有wait参数，就相当于true
			my_cond.wait(sbguard, [this] {
				if (!msgRecvQueue.empty())
				{
					return true;
				}
				return false;
			});
			//只要走到这，锁一定是lock状态
			command = msgRecvQueue.front();
			msgRecvQueue.pop_front();
			sbguard.unlock();
		}
		//for (int i = 0; i < 100000; i++)
		//{
		//	lock_guard<std::mutex> sbguard(my_mutex);//不停的加锁，效率太低
		//	if (!msgRecvQueue.empty())
		//	{
		//		int command = msgRecvQueue.front();
		//		msgRecvQueue.pop_front();
		//	}
		//}
		//cout << "done:out" << endl;
	}

private:
	mutex my_mutex;
	list<int> msgRecvQueue;
	condition_variable my_cond;
};
int main(int, char**)
{
	A myobj;
	thread myOutMsgThread(&A::outMsgRecvQueue, &myobj);
	thread myInMsgThread(&A::inMsgRecvQueue, &myobj);
	myInMsgThread.join();
	myOutMsgThread.join();
}
```

稳定比写的漂亮更好

### notify_all()

如果有两个线程都执行一个函数，那么notify_one只能通知一个执行此函数的线程

### 虚假唤醒

通过lambda表达式防止虚假唤醒



## std::async and future

async，创建异步任务

```cpp
#include <future>
using namespace std;

int mythread()
{
	cout << "this thread id: " << std::this_thread::get_id() << endl;
	std::chrono::milliseconds dura(5000);
	std::this_thread::sleep_for(dura);
	return 5;
}

int main(int, char**)
{
	//希望线程返回结果，不使用全局变量
	//std::async 是函数模板，启动异步任务，返回std::future对象
	//通过future的get获取结果
	//future提供了一种访问异步操作结果的机制，future会保存一个将来某个时刻能拿到的值
	cout << "main thread id: " << std::this_thread::get_id() << endl;
	std::future<int> result = std::async(mythread);
    //result.wait();//等待线程返回，并不返回结果
	cout << result.get() << endl;//get获取将来值，因此，会阻塞在这个位置，只有拿到手后才会继续执行
    //get只能调用一次
    //不执行get依然会等待async执行结束
	cout << "main end" << endl;
}
```

### 调用对象中的函数，和传递参数

```cpp
class A
{
public:
	int mythread(int mypar)
	{
		cout << "this thread id: " << std::this_thread::get_id() << endl;
		std::chrono::milliseconds dura(5000);
		std::this_thread::sleep_for(dura);
		return mypar;
	}
};

int main(int, char**)
{
	A a;
	int temp = 12;
	std::future<int> result = std::async(&A::mythread,&a,temp);
	std::cout << result.get() << endl;
	cout << "main end" << endl;
}
```

### std::launch，向async额外传递参数，位于async第一个参数

1. std::launch::deffered，表示线程入口函数被延迟到调用get或者wait才执行，不调用就不创建线程

​		`std::future<int> result = std::async(std::launch::deferred,&A::mythread,&a,temp);`

2. std::launch::async 强制异步任务在新线程中执行
3. 同时传入async | deffered：意味着async的行为可能是async或者deffered，取决于库的行为
4. 不带参数的默认值为async|deffered，等同于3

### async和thread的区别

如果系统资源紧张，thread可能创建失败，程序可能崩溃

async和thread最明显的不同是，有时并不创建线程，get才创建

```cpp
int mythread()
{
	std::cout<<__FUNCTION__<<" :"<<std::this_thread::get_id()<<std::endl;
	return 1;
}
int main()
{
	cout<<__FUNCTION__<<" :"<<std::this_thread::get_id()<<std::endl;
	std::future<int> result = std::async(std::launch::deferred,mythread);
	cout << result.get() << endl;
	return 0;
}
//此时延迟调用，不创建新线程，只在本线程中执行函数
```

std::thread()如果系统资源紧张可能出现创建线程失败的情况，如果创建线程失败那么程序就可能崩溃，而且不容易拿到函数返回值（不是拿不到）
std::async()创建异步任务。可能创建线程也可能不创建线程，并且容易拿到线程入口函数的返回值；

由于系统资源限制：

1. 如果用std::thread创建的线程太多，则可能创建失败，系统报告异常，崩溃。
2. 如果用std::async，一般就不会报异常，因为如果系统资源紧张，无法创建新线程的时候，async不加额外参数的调用方式就不会创建新线程。而是在后续调用get()请求结果时执行在这个调用get()的线程上。如果你强制async一定要创建新线程就要使用 std::launch::async 标记。承受的代价是，系统资源紧张时可能崩溃。

3. 根据经验，一个程序中线程数量 不宜超过100~200 。

### 确定async的具体行为

std::future<int> result = std::async(mythread);
问题焦点在于这个写法，任务到底有没有被推迟执行。

通过wait_for返回状态来判断：

```cpp
std::future_status status = result.wait_for(std::chrono::seconds(6));
//std::future_status status = result.wait_for(6s);
//wait_for重载了s，min
	if (status == std::future_status::timeout) {
		//超时：表示线程还没有执行完
		cout << "超时了，线程还没有执行完" << endl;
	}
	else if (status == std::future_status::ready) {
		//表示线程成功放回
		cout << "线程执行成功，返回" << endl;
		cout << result.get() << endl;
	}
	else if (status == std::future_status::deferred) {
		cout << "线程延迟执行" << endl;
		cout << result.get() << endl;
	}

```



### 再谈future

#### future_status

```cpp
int mythread(int mypar)
{
	cout << "this thread id: " << std::this_thread::get_id() << endl;
	std::chrono::milliseconds dura(5000);
	std::this_thread::sleep_for(dura);
	return mypar;
}


int main(int, char**)
{
	cout<<"main thread id:"<<this_thread::get_id()<<endl;
	int temp = 12;
	std::future<int> result = std::async(std::launch::deferred, mythread, temp);

	std::future_status status = result.wait_for(std::chrono::seconds(6));//等待1s
	if(status == std::future_status::timeout)//等待一秒钟希望返回结果
	{
		//wait for等待1s，线程没返回结果，就是超时
		cout << "超时，线程没执行完毕" << endl;
	}
	else if(status==std::future_status::ready)
	{
		//表示成功返回
		cout << "成功返回" << endl;
		cout << result.get() << endl;
	}
	else if(status==std::future_status::deferred)
	//如果async，第一个参数是deffered，则条件成立
	{//因为没有执行，因此没有等待6s，直接运行到这里
		cout << "deffered" << endl;
		cout<<result.get()<<endl;
		//此时，get会在主线程执行函数
	}
	cout << "main end" << endl;
}
```

#### shared_future

future的get使用了move，不能重复get，如果多个线程都要调用get就要使用shared_future

```cpp
int mythread(int mypar)
{
	cout << "this thread id: " << std::this_thread::get_id() << endl;
	std::chrono::milliseconds dura(5000);
	std::this_thread::sleep_for(dura);
	return mypar;
}

int main(int, char**)
{
	packaged_task<int(int)> mypt(mythread);//相当于把函数包装起来
	thread t1(std::ref(mypt), 1);//1是函数参数
	t1.join();
    //shared_future <int> result(mypt.get_future());
	future<int> result = mypt.get_future();
	shared_future<int> result_shared(std::move(result));
	//shared_future<int> result_shared(result.share());
	//可以通过result.valid()判断是否有效
	for (int i = 0; i < 10; i++)
	{
		int res = result_shared.get();
	}
}
```





## std::packaged_task

模板参数是可调用对象，通过packaged_task将各种可调用对象包装起来，方便调用

```cpp
int mythread(int mypar)
{
	cout << "this thread id: " << std::this_thread::get_id() << endl;
	std::chrono::milliseconds dura(5000);
	std::this_thread::sleep_for(dura);
	return mypar;
}

int main(int, char**)
{
	packaged_task<int(int)> mypt(mythread);//相当于把函数包装起来
	thread t1(std::ref(mypt), 1);//1是函数参数
	t1.join();
	future<int> result = mypt.get_future();
	cout << result.get() << endl;
}
```

也可以直接调用

```cpp
	packaged_task<int(int)> mypt(mythread);//相当于把函数包装起来
	mypt(100);
	future<int> res = mypt.get_future();
	cout << res.get() << endl;
```

可以把packaged_task放入到容器中，要使用move，因为删除了拷贝构造函数

## std::promise

能够在某个线程中给他赋值，在其他线程中取出来用

```cpp
void mythread(std::promise<int>& temp, int calc)
{
	calc++;
	calc *= 10;
	chrono::milliseconds dura(5000);
	this_thread::sleep_for(dura);

	int result = calc;
	temp.set_value(result);//把结果保存到promise中

}

int main(int, char**)
{
	std::promise<int> myprom;
	thread t1(mythread, std::ref(myprom), 100);
	t1.join();

	//获取结果值
	//通过promise保存结果，再通过future获取这个值
	future<int> ful = myprom.get_future();
	cout << ful.get() << endl;
}
```

线程间通信

```cpp
void mythread(std::promise<int>& temp, int calc)
{
	calc++;
	calc *= 10;
	chrono::milliseconds dura(5000);
	this_thread::sleep_for(dura);

	int result = calc;
	temp.set_value(result);//把结果保存到promise中

}

void mythread2(std::future<int>& temp)
{
	auto result = temp.get();
	cout << "thread2: " << result << endl;
}

int main(int, char**)
{
	std::promise<int> myprom;
	thread t1(mythread, std::ref(myprom), 100);
	t1.join();

	//获取结果值
	//通过promise保存结果，再通过future获取这个值
	future<int> ful = myprom.get_future();
	thread t2(mythread2, ref(ful));
	t2.join();
}
```

## std::atomic

```cpp
atomic<int> g(0);
void mythread()
{
	for (int i = 0; i < 1000000; ++i)
	{
		g++;
	}
}
int main(int, char**)
{
	thread myobj1(mythread);
	thread myobj2(mythread);
	myobj1.join();
	myobj2.join();
	cout << "main end: " << g<<  endl;
}
```

```cpp
atomic<bool> g_ifend(false);//线程退出标记

void mythread()
{
	chrono::milliseconds dura(1000);
	while(g_ifend==false)
	{
		cout<<__FUNCTION__<<std::this_thread::get_id()<<"running"<<std::endl;
		this_thread::sleep_for(dura);
	}
}
int main(int, char**)
{
	thread myobj1(mythread);
	thread myobj2(mythread);

	this_thread::sleep_for(chrono::milliseconds(5000));
	g_ifend = true;
	myobj1.join();
	myobj2.join();
	cout << "end" << endl;
}
```

### 支持的操作

```cpp
atomic<int> g(0);
void mythread()
{
	for (int i = 0; i < 10000000; ++i)
	{
		//g++;//结果正确
		//g += 1;//结果正确
		g = g + 1;//结果错误
		//一般原子操作针对++,--,+=,-=,&=等是支持的，其他可能不支持
	}
}
```

### 原子读写成员函数 load() store()

```cpp
atomic<int> ato(8);
auto ato2(ato.load())
ato.s
```

