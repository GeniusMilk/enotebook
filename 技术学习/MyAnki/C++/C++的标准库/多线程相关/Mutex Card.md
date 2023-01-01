#MutexCard
#八股文
#Anki

#C++
#C++的标准库

#myanki

#参考链接 
- [C++11 并发指南三(std::mutex 详解) - Haippy - 博客园 (cnblogs.com)](https://www.cnblogs.com/haippy/p/3237213.html)
- [c++11 多线程（2）mutex 总结 - 简书 (jianshu.com)](https://www.jianshu.com/p/8ff671d22aa8)

```ActivityHistory
/
```

```toc 
	style: bullet | number | inline (default: bullet) 
	min_depth: number (default: 2) max_depth: number (default: 6) 
	title: string (default: undefined) 
	allow_inconsistent_headings: boolean (default: false) 
	delimiter: string (default: |) 
	varied_style: boolean (default: false) 
```

## Mutex
Mutex 又称[[OS Card#互斥量|互斥量]]，C++ 11中与 Mutex 相关的类（包括锁类型）和函数都声明在 <mutex> 头文件中，需要使用 std::mutex，就必须包含 <mutex> 头文件。

## <mutex> 头文件介绍

### Mutex 系列类(四种)
==最基本的 Mutex 类==、==递归 Mutex 类==、==定时 Mutex 类==、==定时递归 Mutex 类==

-   std\:\:mutex，最基本的 Mutex 类。
-   std\:\:recursive_mutex，递归 Mutex 类。
-   std\:\:time_mutex，定时 Mutex 类。
-   std\:\:recursive_timed_mutex，定时递归 Mutex 类。

### Lock 类（两种）
==lock_guard==、==unique_lock==
-   std\:\:lock_guard，与 Mutex [[C++智能指针#3 1 1 RAII 的定义|RAII]] 相关，方便线程对[[OS Card#互斥量|互斥量]]上锁。
-   std\:\:unique_lock，与Mutex [[C++智能指针#3 1 1 RAII相关，方便线程对[[OS Card#互斥量|互斥量]]锁，但提供了==更好的上锁和解锁控制==。

### 其他类型
==once_flag==、==adopt_lock_t==、==defer_lock_t==、==try_to_lock_t==

-   std\:\:once_flag
-   std\:\:adopt_lock_t
-   std\:\:defer_lock_t
-   std\:\:try_to_lock_t

### 函数
==try_lock==、==lock==、==call_once==

-   std\:\:try_lock，尝试==同时对多个互斥量上锁==。
-   std\:\:lock，可以==同时对多个互斥量上锁==。
-   std\:\:call_once，如果==多个线程需要同时调用某个函数==，call_once 可以保==证多个线程对该函数只调用一次==。

## std\:\:mutex 介绍

### std\:\:mutex 的成员函数

#### 构造函数
std\:\:mutex不允许==拷贝构造==，也不允许 ==move 拷贝==，最初产生的 mutex 对象是处于 ==unlocked 状态==的。
#TODO

#### lock()
调用线程将==锁住该互斥量==。线程调用该函数会发生下面 3 种情况：
(1). 如果该互斥量当前没有被锁住，则调用线程==将该互斥量锁住==，直到调用 unlock之前，该线程一直拥有该锁。
(2). 如果当前互斥量被其他线程锁住，则当前的调用线程被==阻塞住==。
(3). 如果当前互斥量被当前调用线程锁住，则会==产生死锁(deadlock)。==

#### unlock()
解锁，释放对互斥量的所有权。

#### try_lock()
尝试锁住互斥量，如果互斥量被其他线程占有，则当前线程也==不会被阻塞==。线程调用该函数也会出现下面 3 种情况，
(1). 如果当前互斥量没有被其他线程占有，则该线程==锁住互斥==量，直到该线程调用 ==unlock 释放互斥量==。
(2). 如果当前互斥量被其他线程锁住，则当前调用线程==返回 false==，而并不会被==阻塞==掉。
(3). 如果当前互斥量被当前调用线程锁住，则会产生==死锁(deadlock)==

####  std\:\:mutex 的小例子

```cpp
#include <iostream>       // std::cout
#include <thread>         // std::thread
#include <mutex>          // std::mutex

volatile int counter(0); // non-atomic counter
std::mutex mtx;           // locks access to counter

void attempt_10k_increases() {
    for (int i=0; i<10000; ++i) {
        if (mtx.try_lock()) {   // only increase if currently not locked:
            ++counter;
            mtx.unlock();
        }
    }
}

int main (int argc, const char* argv[]) {
    std::thread threads[10];
    for (int i=0; i<10; ++i)
        threads[i] = std::thread(attempt_10k_increases);

    for (auto& th : threads) th.join();
    std::cout << counter << " successful increases of the counter.\n";

    return 0;
}
```

### std\:\:time_mutex 介绍

std\:\:time_mutex 比 std\:\:mutex 多了两个成员函数，==try_lock_for()==、==try_lock_until()。==

#### try_lock_for() 函数
try_lock_for() 函数接受一个==时间范围==，表示在这一段时间范围之内线程如果==没有获得锁则被阻塞住==（与 std\:\:mutex 的 try_lock() 不同，try_lock 如果被调用时没有获得锁则直接返回 false），如果在此期间其他线程==释放了锁==，则该线程可以获得对互斥量的锁，如果超时（即在指定时间内还是没有获得锁），则==返回 false==。

#### try_lock_until ()函数
try_lock_until ()函数则接受一个==时间点==作为参数，在指定时间点未到来之前线程如果没有获得锁则==被阻塞住==，如果在此期间其他线程释放了锁，则该线程可以获得对互斥量的锁，如果超时（即在指定时间内还是没有获得锁），则返回 ==false==。

####  std\:\:time_mutex 的用法

```cpp
#include <iostream>       // std::cout
#include <chrono>         // std::chrono::milliseconds
#include <thread>         // std::thread
#include <mutex>          // std::timed_mutex

std::timed_mutex mtx;

void fireworks() {
  // waiting to get a lock: each thread prints "-" every 200ms:
  while (!mtx.try_lock_for(std::chrono::milliseconds(200))) {
    std::cout << "-";
  }
  // got a lock! - wait for 1s, then this thread prints "*"
  std::this_thread::sleep_for(std::chrono::milliseconds(1000));
  std::cout << "*\n";
  mtx.unlock();
}

int main ()
{
  std::thread threads[10];
  // spawn 10 threads:
  for (int i=0; i<10; ++i){
  threads[i] = std::thread(fireworks);
  }

  for (auto& th : threads) {
	  th.join();
  }
  return 0;
}
```

### std\:\:recursive_timed_mutex 介绍

std\:\:recursive_mutex 与 std\:\:mutex 的关系一样，std\:\:recursive_timed_mutex 的特性也可以从 std\:\:timed_mutex 推导出来


recursive_mutex 类是==同步原语==，能用于保护共享数据免受从个多线程同时访问。
recursive_mutex 提供==排他性==递归所有权语义：
调用方线程在从它成功调用 ==lock ()==或== try_lock()== 开始的时期里占有 recursive_mutex 。此时期间，线程可以进行对 lock() 或 try_lock() 的附加调用。所有权的时期在线程调用 unlock 匹配次数时结束。
线程占有 recursive_mutex 时，若其他所有线程试图要求 recursive_mutex 的所有权，则它们将阻塞（对于调用 lock() ）或收到 false 返回值（对于调用 try_lock() ）。
可锁定 recursive_mutex 次数的最大值是==未指定==的，但抵达该数后，对 lock 的调用将抛出 ==std\:\:system_error== 而对 try_lock 的调用将返回 false 。
若 recursive_mutex 在仍为某线程占有时被==销毁==，则程序行为未定义。 recursive_mutex 类满足互斥体 (Mutex) 和标准布局类型 (StandardLayoutType) 的所有要求。

### Lock 类介绍

#### std\:\:lock_guard 介绍

与 Mutex [[C++智能指针#3 1 1 RAII 的定义|RAII]]  相关，方便线程对互斥量上锁。



##### std\:\:lock_guard 例子

```cpp
 #include <iostream>       // std::cout
#include <thread>         // std::thread
#include <mutex>          // std::mutex, std::lock_guard
#include <stdexcept>      // std::logic_error

std::mutex mtx;

void print_even (int x) {
    if (x%2==0) std::cout << x << " is even\n";
    else throw (std::logic_error("not even"));
}

void print_thread_id (int id) {
    try {
        // using a local lock_guard to lock mtx guarantees unlocking on destruction / exception:
        std::lock_guard<std::mutex> lck (mtx);
        print_even(id);
    }
    catch (std::logic_error&) {
        std::cout << "[exception caught]\n";
    }
}

int main ()
{
    std::thread threads[10];
    // spawn 10 threads:
    for (int i=0; i<10; ++i)
        threads[i] = std::thread(print_thread_id,i+1);

    for (auto& th : threads) th.join();

    return 0;
}
```
#### std\:\:unique_lock 介绍
与 Mutex  [[C++智能指针#3 1 1 RAII 的定义|RAII]] 相关，方便线程对互斥量上锁，但提供了更好的上锁和解锁控制。

##### stdd\:\:unique_lock 例子
```cpp
#include <iostream>       // std::cout
#include <thread>         // std::thread
#include <mutex>          // std::mutex, std::unique_lock

std::mutex mtx;           // mutex for critical section

void print_block (int n, char c) {
    // critical section (exclusive access to std::cout signaled by lifetime of lck):
    std::unique_lock<std::mutex> lck (mtx);
    for (int i=0; i<n; ++i) {
        std::cout << c;
    }
    std::cout << '\n';
}

int main ()
{
    std::thread th1 (print_block,50,'*');
    std::thread th2 (print_block,50,'$');

    th1.join();
    th2.join();

    return 0;
}
```

### Mutex 其他类型 介绍

<table>
<thead>
<tr>
<th></th>
<th></th>
<th></th>
</tr>
</thead>
<tbody>
<tr>
<td><a href="https://links.jianshu.com/go?to=http%3A%2F%2Fwww.cplusplus.com%2Freference%2Fmutex%2Fonce_flag%2F" target="_blank"><strong>once_flag</strong></a></td>
<td>Flag argument type for call_once (class )</td>
<td>once_flag</td>
</tr>
<tr>
<td><a href="https://links.jianshu.com/go?to=http%3A%2F%2Fwww.cplusplus.com%2Freference%2Fmutex%2Fadopt_lock_t%2F" target="_blank"><strong>adopt_lock_t</strong></a></td>
<td>Type of adopt_lock (class )</td>
<td>adopt_lock_t</td>
</tr>
<tr>
<td><a href="https://links.jianshu.com/go?to=http%3A%2F%2Fwww.cplusplus.com%2Freference%2Fmutex%2Fdefer_lock_t%2F" target="_blank"><strong>defer_lock_t</strong></a></td>
<td>Type of defer_lock (class )</td>
<td>defer_lock_t</td>
</tr>
<tr>
<td><a href="https://links.jianshu.com/go?to=http%3A%2F%2Fwww.cplusplus.com%2Freference%2Fmutex%2Ftry_to_lock_t%2F" target="_blank"><strong>try_to_lock_t</strong></a></td>
<td>Type of try_to_lock (class )</td>
<td>try_to_lock_t</td>
</tr>
</tbody>
</table>

#### once_flag
此类型的对象用作==call_once==的参数。  
在不同的线程中使用相同的对象在不同的调用上调用call_once，如果同时调用，则会==单个执行==。  
它是一个==不可复制的==、==不可移动的==、==可构造的==类
```cpp
struct once_flag {  
constexpr once_flag() noexcept;  
once_flag (const once_flag&) = delete;  
once_flag& operator= (const once_flag&) = delete;  
};
```
#### once_flag
这是一个空的类，用作==adopt_lock==的使用的类型。  
向==unique_lock==或==lock_guard==的构造函数传递==使用过的锁==，使该对象==不锁定互斥==对象，并假设它已经被当前线程锁定。  
`struct defer_lock_t {}; `// 空类，只是作为adopt_lock类型
```cpp
   constexpr adopt_lock_t adopt_lock {}; 
```   

#### defer_lock_t
try_to_lock_t是用于try_to_lock类型的空类。  
将==try_to_lock==传递给unique_lock的==构造函数==，使它通过调用它的try_lock成员来锁定互斥对象，代替lock。  
struct try_to_lock_t {};  
constexpr try_to_lock_t try_to_lock {};  
try_to_lock 用于unique_lock的构造函数的可能的参数；使用try_to_lock构造的unique_lock对象试图通过调用其try_lock成员而不是锁成员来锁定互斥对象。

#### try_to_lock_t
try_to_lock_t是用于==try_to_lock==类型的空类。  
将==try_to_loc==k传递给unique_lock的构造函数，使它通过调用它的try_lock成员来锁定互斥对象，代替lock。  
struct try_to_lock_t {};  
constexpr try_to_lock_t try_to_lock {};  
try_to_lock 用于unique_lock的构造函数的可能的参数；使用try_to_lock构造的unique_lock对象试图通过调用其==try_lock成员==而不是==锁成员==来锁定互斥对象。

  
  
作者：jorion  
链接：https://www.jianshu.com/p/8ff671d22aa8  
来源：简书  
著作权归作者所有。商业转载请联系作者获得授权，非商业转载请注明出处。

### Mutex 函数介绍

<table>
<thead>
<tr>
<th style="text-align:center"></th>
<th style="text-align:center"></th>
<th style="text-align:center"></th>
</tr>
</thead>
<tbody>
<tr>
<td style="text-align:center"><a href="https://links.jianshu.com/go?to=http%3A%2F%2Fwww.cplusplus.com%2Freference%2Fmutex%2Ftry_lock%2F" target="_blank"><strong>try_lock</strong></a></td>
<td style="text-align:center">Try to lock multiple mutexes (function template )</td>
<td style="text-align:center">试锁互斥量</td>
</tr>
<tr>
<td style="text-align:center"><a href="https://links.jianshu.com/go?to=http%3A%2F%2Fwww.cplusplus.com%2Freference%2Fmutex%2Flock%2F" target="_blank"><strong>lock</strong></a></td>
<td style="text-align:center">Lock multiple mutexes (function template )</td>
<td style="text-align:center">锁</td>
</tr>
<tr>
<td style="text-align:center"><a href="https://links.jianshu.com/go?to=http%3A%2F%2Fwww.cplusplus.com%2Freference%2Fmutex%2Fcall_once%2F" target="_blank"><strong>call_once</strong></a></td>
<td style="text-align:center">Call function once (public member function )</td>
<td style="text-align:center">call_once</td>
</tr>
</tbody>
</table>

#### try_lock
std\:\:try_lock是一个==模版函数==：

```cpp
template <class Mutex1, class Mutex2, class... Mutexes>
int try_lock (Mutex1& a, Mutex2& b, Mutexes&... cde);
```

使用try_lock成员函数==(非阻塞)式==的锁定对象a,b...等。函数为每个参数调用try_lock成员函数(首先是a，然后是b，最后是cde中的其他函数)，直到==所有调用都成功==，或者==调用失败(返回false或抛出异常)==。如果函数结束是因为调用失败，则对==所有调用try_lock成功的对象调用解锁==，函数将返回锁定失败的对象的参数序号。没有对参数列表中的其余对象执行其他调用。  

```cpp
// std::lock example
#include <iostream>       // std::cout
#include <thread>         // std::thread
#include <mutex>          // std::mutex, std::try_lock

std::mutex foo,bar;

void task_a () {
  foo.lock();
  std::cout << "task a\n";
  bar.lock();
  // ...
  foo.unlock();
  bar.unlock();
}

void task_b () {
  int x = try_lock(bar,foo);
  if (x==-1) {
    std::cout << "task b\n";
    // ...
    bar.unlock();
    foo.unlock();
  }
  else {
    std::cout << "[task b failed: mutex " << (x?"foo":"bar") << " locked]\n";
  }
}

int main ()
{
  std::thread th1 (task_a);
  std::thread th2 (task_b);

  th1.join();
  th2.join();

  return 0;
}
```

#### lock
 std\:\:==lock同样为一个模版函数==

```cpp
template <class Mutex1, class Mutex2, class... Mutexes>
void lock (Mutex1& a, Mutex2& b, Mutexes&... cde);
```

锁定所有的==参数互斥对象==，==阻塞==当前调用线程；该函数使用一个未指定的==调用序列==来锁定对象，该序列调用其==成员锁==、==try_lock==和==解锁==，以确保所有参数都被锁定在返回(不产生任何死锁)。  
如果函数不能锁定所有对象(例如，因为其中一个内部调用抛出异常)，则函数首先==解锁所有成功锁定的对象==(如果有的话)。  

```cpp
  // std::lock example
  #include <iostream>       // std::cout
  #include <thread>         // std::thread
  #include <mutex>          // std::mutex, std::lock

  std::mutex foo,bar;

  void task_a () {
    // foo.lock(); bar.lock(); // replaced by:
    std::lock (foo,bar);
    std::cout << "task a\n";
    foo.unlock();
    bar.unlock();
  }
  
  void task_b () {
    // bar.lock(); foo.lock(); // replaced by:
    std::lock (bar,foo);
    std::cout << "task b\n";
    bar.unlock();
    foo.unlock();
  }

  int main ()
  {
    std::thread th1 (task_a);
    std::thread th2 (task_b);

    th1.join();
    th2.join();

    return 0;
  }
```

#### callonce
std\:\:call_once ==公有模版函数==

```cpp
template <class Fn, class... Args>
void call_once (once_flag& flag, Fn&& fn, Args&&... args);
```

call_once调用将args 作为fn的参数调用fn，除非==另一个线程已经(或正在执行)使用相同的flag调用执行call_once==。如果已经有一个线程使用相同flag调用call_once,会使得当前变为==被动执行==，所谓被动执行不执行fn也不返回直到恢复执行后返回。这这个时间点上所有的并发调用这个函数相同的flag都是同步的。  
注意,一旦一个活跃调用返回了,所有当前被动执行和未来可能的调用call_once相同相同的flag也还不会成为积极执行。  

```cpp
  // call_once example
  #include <iostream>       // std::cout
  #include <thread>         // std::thread, std::this_thread::sleep_for
  #include <chrono>         // std::chrono::milliseconds
  #include <mutex>          // std::call_once, std::once_flag

  int winner;
  void set_winner (int x) { winner = x; }
  std::once_flag winner_flag;

  void wait_1000ms (int id) {
    // count to 1000, waiting 1ms between increments:
    for (int i=0; i<1000; ++i)
      std::this_thread::sleep_for(std::chrono::milliseconds(1));
    // claim to be the winner (only the first such call is executed):
    std::call_once (winner_flag,set_winner,id);
  }

  int main ()
  {
    std::thread threads[10];
    // spawn 10 threads:
    for (int i=0; i<10; ++i)
      threads[i] = std::thread(wait_1000ms,i+1);

    std::cout << "waiting for the first among 10 threads to count 1000   ms...\n";

    for (auto& th : threads) th.join();
    std::cout << "winner thread: " << winner << '\n';
  
    return 0;
  }
```