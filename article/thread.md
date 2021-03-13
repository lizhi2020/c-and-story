本文主要介绍C++的thread库用法，并以此为工具探究互斥锁，条件变量，信号量和原字操作等概念

首先引入入一个样例代码
```
void f() {
	cout << "f start" << endl;
}
int main() {
	thread p(f);
	return 0;
}
```
这个程序是有问题的，在windows下调试运行会报错'abort has been called'
原因是thread对象在析构时会检查线程状态

`if joinable()==true,then std::terminate()`

而`terminate`会导致**整个程序**终止！

也就是说如果你显式定义了一个`thread`，则必须显式的在某个地方调用`join()`或者`depatch()`

如果你既不想`join()`也不想`depatch()`,可以动态申请thread的指针:`thread* p=new thread(f)`,并且不`delete`它。但这不是好的编程写法，如非必要，不要这么做。

现在我们将程序改写：
```
void f() {
	cout << "f start" << endl;
};
int main() {
	thread *p=new thread(f);
    p->join();
	return 0;
}
```
程序正常执行，很好，再将代码改的复杂一点
```
static int a = 0;
void f(int t) {
	for (int i = 0; i < t; i++)
		a += 2;
};
int main() {
	thread* p = new thread(f, INT_MAX/4);
	thread* m = new thread(f, INT_MAX/4);
	p->join();
	m->join();
	cout << a;
	return 0;
}
```
自行验证一下就会发现`a`最终的值并非预期，这是一个典型的多线程编程错误例子。
出问题的语句是`a+=2`。
如果只有一个物理处理单元(单核CPU)，那么我们只需要保证`a+=2`是原子的即可。
如果有多个执行单元，那么我们应当保证`a+=2`这个操作是互斥的！
在这个例子中，互斥体现为永远只能有最多一个线程在执行`a+=2`(也可以一个线程都没有)。我们称`a+=2`这部分代码为关键区

那我们该如何实现互斥呢？
用一个flag来表示哪一个线程在执行关键区？如果仅仅只用了C++语言的赋值和判断运算符的话是不可行的。

必须使用原子操作的原语或者锁，条件变量等特殊的数据结构，才能实现互斥。

我们可以先试着使用互斥锁`std::mutex`看看
一个线程可以对一个mutex对象进行lock(),和unlock()操作。

我们可以将mutex类比为一扇门。如果这扇门是开的，任何一个人（线程）都可以尝试进入，然后将门锁上，那么后来者就无法进入了。
后来的人（线程）想要lock()一个已经被别人锁上的门，就只能等待（被阻塞）直到里面的人打开了门unlock()然后出来。

修改代码如下：
```
mutex mtx;
void f(int t) {
	for (int i = 0; i < t; i++) {
		mtx.lock();
		a += 2;
		mtx.unlock();
	}
};
```
代码现在看起来很不错，但你实际运行起来会发现有一个大问题：太慢了！
至少本人的代码在20秒内没有结束！

聪明的写法:
```
mutex mtx;
void f(int t) {
    mtx.lock();
	for (int i = 0; i < t; i++) {
		a += 2;	
	};
    mtx.unlock();
};
```

但直接使用mutex是有隐患的,下面代码来自cppreference.com:
```
std::mutex m;
 
void bad() 
{
    m.lock();                    // acquire the mutex
    f();                         // if f() throws an exception, the mutex is never released
    if(!everything_ok()) return; // early return, the mutex is never released
    m.unlock();                  // if bad() reaches this statement, the mutex is released
}
 
void good()
{
    std::lock_guard<std::mutex> lk(m); // RAII class: mutex acquisition is initialization
    f();                               // if f() throws an exception, the mutex is released
    if(!everything_ok()) return;       // early return, the mutex is released
}                                      // if good() returns normally, the mutex is released
```

不过此处有个问题，如果模板类型不是mutex会怎么样?
比如如下定义`unique_lock<int> mtx`

虽然有了互斥锁，但更多情况下我们还会用到条件变量。条件变量用于等待特定事件发生。
试想这样的一个场景：线程1，2对变量a不断+1，线程3不断检测a,直到a>=100，然后对a加2

如果只使用互斥锁，你会发现线程3的代码很不优雅:
```
mtx.lock()
if(**){
    **
};
mtx.unlock()
```
线程3会造成CPU时间的浪费：它拿到了锁，发现不符合自己的条件，又释放了锁。这个过程其实并没有做有意义的事。

所以我们需要条件变量。

C++ 还提供异步操作和原子操作来实现多线程之间的同步，但并没有信号量的实现。
