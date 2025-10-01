---
source: https://zhuanlan.zhihu.com/p/194198073
author: ""
year: ""
type: paper|blog|doc|code
---
## 什么是C++多线程渲染
>**线程**是操作系统能够进行CPU调度的**最小单位**，它被**包含在进程之中**，一个进程可包含单个或者多个线程。
>在Linux中可以更容易观察到并理解线程，写一个如下程序：

````cpp
// main.cpp
#include<thread>

void func()
{
        int i=0;
        while(true);
}

int main()
{
        std::thread th(func);
        th.detach();
        while(true);
        return 0;
}
````
````bash
$ ps -feL | grep 1747
UID          PID    PPID     LWP  C NLWP STIME TTY          TIME CMD
zhengzu+    1747     357    1747 98    2 23:02 pts/0    00:00:24 ./a.out
zhengzu+    1747     357    1748 98    2 23:02 pts/0    00:00:24 ./a.out

#PID为processid,进程ID
#LWP为light weight process orthread， 即线程ID
````
即可观察到，该代码以一个进程的方式运行，进程ID为1747，**该进程包含两个线程**：
一个是主线程1747，主线程ID同进程ID（只是ID相同，并不指两者是一个东西），**main函数在主线程中运行**；
另一个是子线程1748，由代码std::thread th(func)生成该线程，**函数func在子线程中运行**；
### 我的解释（不要省）
1. C++中起一个线程的方式为 std::thread th(func)， 当起来这个线程之后， 他就独立运行了，<font color="#4bacc6">th.detach()是什么？</font>
2. 可以看到，在linux中， 主线程的ID【LWP】和进程 PID是一样的， 而程序起的线程，则为1748，和PID不一样。



## 何时适合使用并发：
> 不使用并发的唯一原因就是收益（性能的增幅）比不上成本（代码开发的脑力成本、时间成本，代码维护相关的额外成本）。**运行越多的线程，操作系统需要为每个线程分配独立的栈空间，需要越多的上下文切换，这会消耗很多操作系统资源**，如果在线程上的任务完成得很快，那么实际执行任务的时间要比启动线程的时间小很多，所以在某些时候，增加一个额外的线程实际上会降低，而非提高应用程序的整体性能，此时收益就比不上成本。而且**多线程代码如果编写不当，运行中会出现很多问题**，诸如执行结果不符合预期、程序崩溃等问题。

### 我的解释（不要省）
1. 大量相同任务，适合起一个线程跑，原则就是主线程耗时要大于启动线程的时间，这个分摊压力才合理。

## 代码执行逻辑
> 1. std::thread th1(proc1) 之后，线程th1<font color="#4bacc6">就已经被执行了。</font>
> 2. 线程的实例化，需要至少传递函数名。如果是 void proc2(int a, int b)这种， 那么就需要th2(proc2, a, b) 这样子去实例化。
> 3. 如果是类成员函数，"void Utils::proc3(int a, int b)",那么创建线程的语句为"std::thread th3(&Utils::proc3, a, b);"（固定写法，记住即可，不用深究这里为什么有&符号） 。

### 我的解释（不要省）
- 这段说了什么（自我语言，避免“纯复制”）
1. 简单介绍了线程怎么创建， 初始化的过程中的语法。
2. <font color="#4bacc6">现在的问题，th1已经被执行了什么意思？ 是这个线程已经在了，还是说里面的函数已经开始跑了？看起来是后者？</font>

- 我以前哪里误解了？
1. 只要一实例化，这个函数就开始跑了， 后序只是继续描述我是否要等待这个函数执行完而已。
2. 通过join detach来判断是否要等待这个线程执行完。

## join() vs  detach()
> join()与detach()都是std::thread类的成员函数，是两种线程阻塞方法，两者的区别是**是否等待子线程执行结束**。
> join()等待调用线程运行结束后当前线程再继续运行，例如，主函数中有一条语句th1.join(),那么执行到这里，主函数阻塞，直到线程th1运行结束，主函数才能继续运行。
> 整个过程就相当于：
### 我的解释（不要省）
- 如果后续操作、不需要等待这个线程返回的数据结果， 那么就用detach， 如果依赖这个线程执行完毕的结果， 那么就用join()

## join() detach()的使用场景：
思考一下， 什么时候使用Join() 是有意义的，<font color="#4bacc6">首先是否可以join多次？</font>比如我每一帧之间有一个join来等待上一帧的结果？ 看起来是不行的， join只能**有且只有调用一次**。
- 我以前哪里误解了？
	-看起来我没有充分理解这两个功能，这两个功能只能调用一次，它就不像GPU的围栏值，
考虑两类工作内容：
比如我起一个渲染线程， 我应该会用detach() 表示这个线程独立运行，后续只需要用生产者消费者模型就ok了？<font color="#4bacc6">但是数据怎么填充是个问题</font>
比如我起一个job工作，我是需要在之后完成这个工作之后将内容回收回来的，那么这个肯定需要等？，因为后续有步骤依赖，首先这个等的过程我有点没想通， 是通过回调？ 还是怎么的？
目前第一种渲染线程天生适合多线程，问题是生产消费数据中数据怎么塞。
第二种job，怎么等待，怎么接收回调，是一个问题，值得后续看一下
### 离开作用域之后【注意析构】：
离开作用域之前， 也就是函数析构之前， 必须调用一次join 或者detach来终止线程，避免奔溃。
### 我的解释（不要省）
- 这段说了什么（自我语言，避免“纯复制”）
- 我以前哪里误解了？
	没有充分理解Join的，等待执行结果，意味着什么？
- 对哪些永久卡有贡献？→ 

## 线程的同步
ok，上面写到了线程怎么起，detach和join， 马上就聊到了这个关键问题， 就是线程的数据问题。
### 互斥锁 / 死锁：
有一个共享数据A，不同的线程都需要使用它，假设我为这个共享数据加上了互斥锁，那么同一时间只能由一个线程去使用它，每个线程使用这个共享数据之前都去unlock它，用完之后去lock。
并且判断此时有没有别的线程在使用它，【线程A lock共享数据之后，其余线程需要等待这个线程A unlock，才能使用这个共享数据，此时会阻塞等待】
**死锁** ： 就是在这个过程中，线程A和线程B都lock了某些数据，并且都在等待另外一些数据被unlock，且这个过程中每个进程都不放弃自己的数据，也不能剥夺别人的数据。
产生条件就是 1. 需要互斥资源 2.不释放自己占有的资源 3. 不抢夺别人的资源 4. 循环等待。
### 读写锁 shared_mutex：



### 我的解释（不要省）
- 这段说了什么（自我语言，避免“纯复制”）
- 我以前哪里误解了？
- 对哪些永久卡有贡献？→ 




## 指针、引用数据的有效性注意
> 引文/代码片段/图（注明定位）

### 我的解释（不要省）
- 这段说了什么（自我语言，避免“纯复制”）
- 我以前哪里误解了？
- 对哪些永久卡有贡献？→ 


## 多线程参数的传递
> 引文/代码片段/图（注明定位）

### 我的解释（不要省）
- 这段说了什么（自我语言，避免“纯复制”）
- 我以前哪里误解了？
- 对哪些永久卡有贡献？→ 

## 临界区、信号量、互斥量（锁）的区别与联系？？？？
> 引文/代码片段/图（注明定位）
不知道是什么 先空着
### 我的解释（不要省）
- 这段说了什么（自我语言，避免“纯复制”）
- 我以前哪里误解了？
- 对哪些永久卡有贡献？→ 

## mutex
互斥锁，
std::mutex mtx;
lock(mtx);  unlock(mtx)
std::lock_guard<std::mutex> lock(mtx);
// std::lock_guard 自动管理锁，能够在声明的时候加锁，lock析构的时候自动解锁。

### 我的解释（不要省）
用锁来判断哪个线程正在使用数据，

## ### std::condition_variable 线程之间互相等待
线程同步工具， 允许一个线程等待另外一个线程的信号

```cpp
#include <condition_variable>

std::mutex mtx;
std::condition_variable cv;
bool ready = false;

// 等待线程
void waiter() {
    std::unique_lock<std::mutex> lock(mtx);
    cv.wait(lock, [] { return ready; });  // 等待ready为true
    // 条件满足后继续执行
}

// 通知线程
void notifier() {
    {
        std::lock_guard<std::mutex> lock(mtx);
        ready = true;  // 改变条件
    }
    cv.notify_one();  // 通知等待的线程
}
```

### 1. cv.wait()
``` cpp
std::unique_lock<std::mutex> lock(mtx); 
cv.wait(lock, [] { return ready; }); // 等待ready为true，不是等待lock

// 模拟wait()的内部逻辑
void wait_simulation(std::unique_lock<std::mutex>& lock, auto condition) {
    while (!condition()) {  // 1. 检查条件
        lock.unlock();      // 2. 如果条件不满足，释放锁
        // 3. 线程进入睡眠，等待被notify唤醒
        wait_for_notification();
        lock.lock();        // 4. 被唤醒后，重新获取锁
        // 5. 回到while循环，再次检查条件
    }
    // 6. 条件满足，继续执行，锁保持锁定状态
}



```
这个地方的wait，是在wait ready为true，
如果ready是false，就**解锁lock**，等待线程被唤醒。【让其他线程能够修改ready】
类似于上面这个， condition为false的时候，就释放lock的控制权，等待ready 为true，
``` c++
void notifier() {
    // 模拟一些工作
    std::this_thread::sleep_for(std::chrono::seconds(2));
    
    std::cout << "Notifier is setting ready = true\n";
    
    // 修改条件
    {
        std::lock_guard<std::mutex> lock(mtx);
        ready = true;
    }
    
    // 通知等待的线程
    std::cout << "Notifier is calling notify_all()\n";
    cv.notify_all();  // 唤醒所有等待线程
}
```
这个地方就很简单，把ready设置为true，<font color="#4bacc6"> 然后告诉所有线程进行一次同步？</font>cv.notify_one();  cv.notify_all();
### 2. unique_lock   cv.notify_one();  cv.notify_all();
unique_lock 比 lock_guard更加灵活， 可以支持手动lock unlock，
因为**wait内部需要去调用lock unlock**，所以用unique_lock
现在没怎么明白的其实还是这个线程，是运行一次， 还是说持久的运行下去，
cv.wait(lock, [] { return ready; });
看起来只是让线程停顿在这个地方， 好像运行一次和持续运行， 和这个东西没关系， 更多的是看线程中的函数表达， 线程就理解为执行某个函数，这个函数中如果有while
### 我的解释（不要省）
- 这段说了什么（自我语言，避免“纯复制”）
- 我以前哪里误解了？
- 对哪些永久卡有贡献？→ 

### std::atomic（原子操作）
```` c++
#include <atomic>

std::atomic<int> counter{0};
std::atomic<bool> flag{false};

void thread_function() {
    counter++;  // 原子递增，无需加锁
    flag.store(true);  // 原子写入
    
    int value = counter.load();  // 原子读取
}
````

### 我的解释（不要省）
- 能保证操作的不可分割性， 也就是读/写这个数据的这个时间段这个数据不会被改写，<font color="#4bacc6">当然是否还有更多的点？比如一定按照顺序之类的？</font>
## 什么是多线程渲染
> 引文/代码片段/图（注明定位）

### 我的解释（不要省）
- 这段说了什么（自我语言，避免“纯复制”）
- 我以前哪里误解了？
- 对哪些永久卡有贡献？→ 

## 什么是多线程渲染
> 引文/代码片段/图（注明定位）

### 我的解释（不要省）
- 这段说了什么（自我语言，避免“纯复制”）
- 我以前哪里误解了？
- 对哪些永久卡有贡献？→ ## 什么是多线程渲染
> 引文/代码片段/图（注明定位）

### 我的解释（不要省）
- 这段说了什么（自我语言，避免“纯复制”）
- 我以前哪里误解了？
- 对哪些永久卡有贡献？→ ## 什么是多线程渲染
> 引文/代码片段/图（注明定位）

### 我的解释（不要省）
- 这段说了什么（自我语言，避免“纯复制”）
- 我以前哪里误解了？
- 对哪些永久卡有贡献？→ 
# 关联
- 近邻

