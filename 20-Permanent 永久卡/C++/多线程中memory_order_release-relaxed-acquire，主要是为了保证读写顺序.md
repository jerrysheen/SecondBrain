---
id:
  "{ date:YYYYMMDDHHmm }":
type: permanent
tags:
---
# 命题：多线程中memory_order_release-relaxed-acquire，主要是为了保证读写顺序
## 定义 / 结论

### 1. 本线程的顺序保证
**`release` 确保在它之前的所有操作都不会被重排序到它之后。**
就像厨师的承诺："我说菜好了，那么切菜、炒菜、调味这些步骤肯定都已经完成了"

atomic'<'bool'>' flag
flag.load(std::memory_order_acquire);
这个操作，保证所有在 flag.store(true, std::memory_order_release)；之前的操作， 都能被确保已经执行了。
consumer看到flag = true时， 能看到global_x 和 global_y的值，但是无法保证 global_y 和 global_x 谁先写入。
```cpp
// 全局变量
int global_x = 0;
int global_y = 0;
std::atomic<bool> flag{false};

void producer() {
    global_x = 10;     // A：写X
    global_y = 20;     // B：写Y
    flag.store(true, std::memory_order_release);  // C
}

void consumer() {
    if (flag.load(std::memory_order_acquire)) {
        printf("x=%d, y=%d\n", global_x, global_y);
    }
}

```

## 机制 / 因果
- 在单线程执行的时候， 编译器会对代码进行优化， 比如 int a = 400， int b = 200，可能被调换顺序等操作，编译器应该会根据依赖关系进行一些优化、顺便调换了顺序。
- 比如， int y = 0；  int a = y + 3； 这种操作就不会调换顺序。
这保证了，单线程执行的时候，顺序是没问题的，不管再怎么优化，结果一定是对的。
但是多线程渲染的时候，就可能会有时序错误， 假设我们不用原子操作，另外一个线程去读取当前线程的某个数据的时候， 时序是不确定的。【本质是因为编译器没办法去优化别的线程对此线程的执行顺序，这就可能导致出错】
比如下面这个内容，可能b = 100先赋值，可能a = 0先赋值。
```cpp
int a = 0;
int b = 100;
```
flag.store(true, std::memory_order_release); 操作的时候， 其实就保证了，下一次flag.load的时候，前面的int a， int b 这俩操作已经被执行了。
## 操作 / 代码要点
1) <步骤>
2) <关键API/陷阱>

## 反例 / 边界
- <何时不成立>

## 引申/ 同类型
- <同类型的有什么>
## 连接
- 定义 →【内容摘录】 永久卡A-四类卡片分工
- 因果 →【】这个概念从哪里来，到哪里去
- 实现 → 【具体的代码实现SOP 流程模板】SOP-永久卡模板（Templater版）
- 应用 → [[TinyEngine实现一个多线程渲染：]]
- 证据 → 【内容摘录、文献证明】Lit-写作与知识颗粒度研究（作者-年份）
