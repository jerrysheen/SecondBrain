---
id:
  "{ date:YYYYMMDDHHmm }":
type: permanent
tags:
---
# 命题：C++回调的本质是库去Call你的函数
## 定义 / 结论
也就是说库在适当的时候调用你注册进去的函数。
所以我应该是在自己执行的地方，写一个函数， 然后注册到库中。

## 机制 / 因果
- <为什么对>

## 操作 / 代码要点
```cpp
#include <cstdio>

// === 库侧 ===
using OnMsg = void(*)(int code, void* user);

static OnMsg  g_cb   = nullptr;
static void*  g_user = nullptr;

void register_cb(OnMsg cb, void* user) { // 你调用库
    g_cb = cb; g_user = user;
}

void library_event_happened(int code) {  // 某个时机库触发事件
    if (g_cb) g_cb(code, g_user);        // 库调用你（回调）
}

// === 你（业务）侧 ===
struct Context { int base; };

void on_msg_impl(int code, void* user) {
    auto* ctx = static_cast<Context*>(user);
    std::printf("got code=%d, base=%d\n", code, ctx->base);
}

int main() {
    Context ctx{42};
    register_cb(&on_msg_impl, &ctx);  // 1) 注册

    library_event_happened(200);      // 2) 事件 → 库回调你
    // 输出: got code=200, base=42
}

```
可以看到，main函数内， 第一步注册我的函数到库中，
随后，库只需要在合适的时间点拉起函数，就执行了我注册的代码

## 反例 / 边界
- <何时不成立>

## 引申/ 同类型
std::function 和 lambda表达式的限制。
lambda表达式可以吗？ **非捕获式的可以** 也就是不携带上下文，没有 [=]和[&]的


## 连接
- 定义 →【内容摘录】 永久卡A-四类卡片分工
- 因果 →【】这个概念从哪里来，到哪里去
- 实现 → 【具体的代码实现SOP 流程模板】SOP-永久卡模板（Templater版）
- 应用 →【Project中是否用到】 MOC-并发-导航
- 证据 → 【内容摘录、文献证明】[[C++函数指针相关的内容]]
