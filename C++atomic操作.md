---
source: https://chatgpt.com/c/68eb8cd1-bf50-8323-b02c-1106687e4579
author: ""
year: ""
type: paper|blog|doc|code
---
## 原子类型 & 基本操作
常见原子类型：`std::atomic<bool/int/uint64_t/size_t/void*>`、`std::atomic<T*>`、`std::atomic_flag` 等。  
它们的操作分三类：

1. **纯加载/存储（Load/Store）**
    - `x.load(order)`
    - `x.store(value, order)`
2. **读-改-写（RMW）**：返回**修改前**的值（除非是后缀式命名）
    - `fetch_add/sub/and/or/xor(...)`
    - `exchange(new, order)`：把值换成 `new`，返回旧值
    - `compare_exchange_weak/strong(expected, desired, success_order, failure_order)`
3. **位/标志**（轻量自旋锁等）
    - `std::atomic_flag test_and_set/clear`
> 你提到的 `sCount.fetch_add(1, std::memory_order_relaxed)` 就是**RMW**：把 `sCount` 加 1，并返回**加之前**的值；`relaxed` 只保证这一步是**原子**的，不做可见性顺序保证。

### 我的解释（不要省）
比如我们的操作是返回之前的值，并且对现在的值做一个操作，比如加，减，等操作，后面输入的是和什么值做操作。
然后需要规定memory order


## 内存序（memory order
| 内存序                    | 适用             | 作用/直觉                                        |
| ---------------------- | -------------- | -------------------------------------------- |
| `memory_order_relaxed` | load/store/RMW | 只保证这一步“原子”，**不提供**跨线程的先后可见性；适合**独立计数**、统计    |
| `memory_order_consume` | load           | 理论上是数据依赖排序；实现混乱，**实际等同 acquire**（多数实现）       |
| `memory_order_acquire` | load/RMW       | 读取到某个“发布”的标志后，**之后**的读写不可重排到它之前（向后看有序）       |
| `memory_order_release` | store/RMW      | 在“发布”一个标志之前，**之前**的读写不可重排到它之后（向前看有序）         |
| `memory_order_acq_rel` | RMW            | 同时具备 acquire+release（常用于 `fetch_add` 等作为“门”） |
| `memory_order_seq_cst` | 全部             | 最强，额外提供**全局总序**，最直观但可能更慢                     |
|                        |                |                                              |

### 我的解释（不要省）
- acquire 和 release来保证之前之后的操作有序，感觉就是记录了一个check point，
- 而relaxed就是值保证这一步自己的原子， 比如我两个线程同时访问，的时候relaxed能保证 确实+了2，但是哪个线程先后是不保证的，
这个用到的时候再来解释更多吧， 感觉还是没有彻底理解

## 摘录要点
> 引文/代码片段/图（注明定位）

### 我的解释（不要省）
- 这段说了什么（自我语言，避免“纯复制”）
- 我以前哪里误解了？
- 对哪些永久卡有贡献？→ ## 摘录要点
> 引文/代码片段/图（注明定位）

### 我的解释（不要省）
- 这段说了什么（自我语言，避免“纯复制”）
- 我以前哪里误解了？
- 对哪些永久卡有贡献？→ ## 摘录要点
> 引文/代码片段/图（注明定位）

### 我的解释（不要省）
- 这段说了什么（自我语言，避免“纯复制”）
- 我以前哪里误解了？
- 对哪些永久卡有贡献？→ ## 摘录要点
> 引文/代码片段/图（注明定位）

### 我的解释（不要省）
- 这段说了什么（自我语言，避免“纯复制”）
- 我以前哪里误解了？
- 对哪些永久卡有贡献？→ 
# 关联
- 近邻：



# AI文本拷贝
