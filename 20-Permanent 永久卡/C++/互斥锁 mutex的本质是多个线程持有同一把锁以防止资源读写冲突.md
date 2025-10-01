---
id:
  "{ date:YYYYMMDDHHmm }":
type: permanent
tags:
---
# 命题：互斥锁的本质是多个线程持有同一把锁以防止资源读写冲突
## 定义 / 结论
- 多个线程持有同一把锁， 每次做修改的时候都先加锁，再解锁，这样子保证一次只执行一个线程的逻辑， 另外一个线程等待。

## 机制 / 因果
- mutex是互斥锁，  mutex.lock 只能在 mutex是unlock状态时才能操作。
<font color="#ff0000">- lk已经加锁了，这个地方不用重复写 lk.lock，会导致奇怪的行为</font>
````cpp
//线程A
std::unique_lock<std::mutex> lk(m); // 已加锁

//线程B
{ std::lock_guard<std::mutex> g(m); // 若 A 仍持锁，这里阻塞
  // 临界区...
} // 作用域结束 -> 解锁

````
则这个代码， 如果lk已经加锁了， 线程B只要创建出这个lock_guard 就会阻塞
## 操作 / 代码要点
### 1. 同一把互斥锁保护变量：
````cpp
std::mutex m;
int shared = 0;

void writer() {
    std::lock_guard<std::mutex> g(m);
    ++shared; // ✅ 受保护
}

int reader() {
    std::lock_guard<std::mutex> g(m);
    return shared;  // ✅ 受保护
}

````
### 2. 只做原子修改，无锁：
````cpp
std::atomic<int> counter{0};
void worker() { counter.fetch_add(1, std::memory_order_relaxed); } // ✅ 无锁、无数据竞争
````
### 3. 生产消费队列：
## 反例 / 边界
不想阻塞的几种方式：
- **试图不等：**
    `std::unique_lock<std::mutex> lk(m, std::defer_lock); if (lk.try_lock()) { /* 成功才进临界区 */ }`
    
- **带超时（需要 timed_mutex / recursive_timed_mutex）：**
    `std::unique_lock<std::timed_mutex> lk(tm, std::defer_lock); if (lk.try_lock_for(std::chrono::milliseconds(2))) { /* ... */ }`

## 引申/ 同类型
- <同类型的有什么>
- 读写锁？std::shared_mutex [[读写锁]]
- 无锁队列？
## 连接
- 定义 →【内容摘录】 永久卡A-四类卡片分工
- 因果 →【】这个概念从哪里来，到哪里去
- 实现 → 【具体的代码实现SOP 流程模板】SOP-永久卡模板（Templater版）
- 应用 →【Project中是否用到】 MOC-并发-导航
- 证据 → 【内容摘录、文献证明】[[新建一个多线程渲染]]  [[C++多线程并发基础入门教程]]
