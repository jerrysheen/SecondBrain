---
source: "<URL 或 论文/文档名>"
author: ""
year: ""
type: "paper|blog|doc|code"
---
## 函数指针基本概念 和 函数回调
### 执行规则
函数指针相当于记录了一个函数的内存地址，运行的时候能通过call 内存地址， 调用对应的函数，然后执行对应的函数。
1. 函数指针记录的是函数的入口地址，传递一个函数指针，相当于把函数的入口地址传递给别的使用方去使用，
2. 函数指针的声明方式：  
```
 using func = int (*)(int x, int y);
 
 // 这样就可以定义一个回调函数
 void MyCallBack(int x, int y)
 {
	 int c = x + y ...
 }
 
// 后续就可以在别的地方调用这个函数：
// 1.
func callback = &MyClassBack;
callback(1,2);

```

注意这边的 ** 是一个指针的意思，前面的int / void 则表示的是返回的类型。
### callback：
首先回调是什么？ 回调本质上是库去call你定义的函数。
也就是说在库中注册你的函数，随后在适当的时候执行。

```cpp
typedef void (*OnMsg)(int code, void* User)
// 这个地方相当于声明了一个函数指针， 名字是OnMsg， 类型是void
void register_cb(OnMsg cb, void * User)
{
	g_cb = cb;
	g_user = user
}
// 声明一个register_cb， 接收一个函数指针， 和void*数据
static OnMsg g_cb = nullptr;
static void* g_user = nullptr;
// 后续执行
void Library_Execute()
{
	g_cb(code, g_user);
}


C++侧：
// 声明一个类似register_cb的回调函数，
void on_msg_impl(int code, void* user)
{
	....
}

register_cb(&on_msg_impl, &ctx)
```

### 我的解释（不要省）
这一段中，描述了函数指针是怎么样的， 怎么样声明一个函数指针。
函数指针即是：
```cpp
int (*)(int a, int b);
```
声明一个函数指针可以是：
~~typedef int (*Function)(int a, int b);~~
错误的！， 不能写 int a， int b， 只能写 int ， int
```cpp
typedef int (*Function)(int, int);
等价于
using Function = int(*)(int, int);
// 这个Function就是被声明的一个函数指针

// 后续：
void Test(Function a, int v);
表示这个地方传入的是一个函数指针Function
// 能用这个Function去定义别的嘛？ Function相当于一个类型**别名**， 不能用Function =  而是Function variable = ...
int somefunction(int a,int b){return 1;};
Function func = &somfunction;
Function func = somefunction; 也可以写
// 这两行能写在一块吗？ 下面的写法对吗
Function func = [](int a, int b){return 1;};
Function func = +[](int a, int b){return 1;}; 这两个等价， 注意lambda传参数在哪里传

```
~~Function func = int(*)(int a , int b){return 1;};~~
<font color="#ff0000">错误的写法，这个地方在赋值，不能再去写一个 类型+匿名函数体</font>


以及描述了如何写一个Callback， callback本质就是将你写的函数去库中注册， 然后在某个时间点去执行这个函数，
## 成员函数指针
```cpp
struct A {
    int mul(int x) { return k * x; }
    int k = 2;
};

int (A::*pmf)(int) = &A::mul;   // “指向 A::mul 的指针”
A a;
int r = (a.*pmf)(3);            // 通过对象调用

```
也就是说声明了一个指向 成员函数的指针， 随后创建一个指向成员函数的对象，再执行这个对象。
pmf是我们自己定义的变量， 不是内建变量， 具体的后续拓展
[[成员变量指针和虚函数指针]]
### 我的解释（不要省）
- 这段说了什么（自我语言，避免“纯复制”）
- 我以前哪里误解了？
- 对哪些永久卡有贡献？→ ## 摘录要点
> 引文/代码片段/图（注明定位）


## lambda表达式
### 非捕获Lambda
[]中不捕获变量
```cpp
auto f = [](int a, int b){};
int f(1,2);
```
### 捕获Lambda——闭包对象
这时 g 是一个有状态的对象（closure），本质是个结构体，里面存 bias，并定义了 operator().
捕获 lambda 不能转换成函数指针（因为它需要那份状态）。可存入 std::function（见下）。
[[C++Lambda表达式]]

### 我的解释（不要省）
- 这段说了什么（自我语言，避免“纯复制”）
- 我以前哪里误解了？
- 对哪些永久卡有贡献？→ ## 摘录要点
> 引文/代码片段/图（注明定位）

## `std::function` / `std::invoke` / `std::bind`：现代 C++ 里的“可调用体”
前文提到 lambda表达式带捕获的，没办法用一个指针函数表示，所以就用std::function进行了大一统。 只需要输入类型一致，就可以都囊括进去。  **不过这个地方就好奇带捕获的数据是怎么传递得了**， 也是一样的方式，产生一个结构体，
<font color="#4bacc6">但是这个地方其实还需要注意一下底层原理和生成方式</font> [[todo]]

```cpp
std::fuction<int(int, int)> fn;

fn = [](int a, int b) {return a + b;}
fn = &add;

if(fn)
{
	int r = fn(3,3);
}
```

### 我的解释（不要省）
- 这段说了什么（自我语言，避免“纯复制”）
- 我以前哪里误解了？
- 对哪些永久卡有贡献？→ ## 摘录要点
> 引文/代码片段/图（注明定位）

## std::invoke
先来看看没有std invoke时候， 不同可调用的写法的区别：
```cpp
// 普通调用和函数指针
int add(int, int)
int (*p)(int, int) = &add;
int t1 = add(1,1);
int t2 = p(1,2);

// 仿函数：
// 相当于重写了 （）操作符， 所以plus（）相当于在调用
struct Plus( int operator()(int a, int b) const{});
Plus f;
int r1 = f(1,1);
auto lam = [](int a, int b){return a + b;};
int r4 = lam(1,2);

// 成员函数指针：
struct A {int mul(int x){return x * k;};  int k = 10;};
int (A::*member_func)(int) = A::&mul;
A a;
A* pa = &a;
int r5 = (a.*member_func)(3);
int r6 = (pa->*meber_func)(3);
```
不同的函数有不同的调用方式，但是用invoke，就可以统一这些写法：
```cpp
using std::invoke;

invoke(p, 1, 2);          // 函数指针
invoke(f, 1, 2);          // 仿函数 / lambda
invoke(&A::mul, a, 3);    // 成员函数指针 + 对象/指针/ref_wrapper 都行
invoke(&A::k, a);         // 数据成员指针：返回 a.k
```
### 我的解释（不要省）
感觉也没怎么统一起来。。  还有一种作用，就是统一这些写法
![[Pasted image 20251001184152.png]]
# 关联
- 近邻：



# AI文本拷贝

```

我觉得是时候来了解下C++的函数指针部分了，我发现好多与函数相关的东西， 我都不了解，请从底层讲到上层把这件事情好好说一遍，比如 callback fn以及 std里面各种与之相关的操作。
其实我觉得第一个问题就是我不明白这个东西底层运行的机制是什么， 返回 function指针和返回function() 之间的区别，相当于函数指针就是内存中指向函数地址的指针是么，函数 传递指针，就是相当于把这块函数后面的内容全部传递过去， C++用执行函数的方式执行后面的内容，直到这个函数末尾？
```

太好了，这个话题把“**回调/函数指针/λ表达式/std::function**”一锅端正合适。下面从**底层→语法→现代C++封装→工程实践**一步步讲清楚，并顺手把几个常见误区纠正。

---

# 0. 先纠正两个常见误解

- **“函数指针=把函数的代码整块传过去”** ❌  
    不是。函数指针只是**一个地址**（或更复杂的句柄）——告诉 CPU “从这儿开始执行”。代码本体还在原处，没有被拷来拷去。
    
- **“可以返回一个函数本体”** ❌  
    C/C++ 里**函数不是对象**，不能按值返回/赋值。你能返回的是**函数指针**或**函数引用**（引用很少用）。上层封装（如 `std::function`）可以“像对象”一样返回，因为那是**包装器**。
    

---

# 1. 底层机制：CPU 是怎么“跳到一个函数地址”的？

一次函数调用（不论通过名字还是函数指针）典型会发生：

1. **参数传递**：按调用约定（ABI / calling convention）把参数放进寄存器或栈。
    
2. **保存返回地址**：`call` 指令把下一条指令地址压栈（或寄存器保存）。
    
3. **跳转到目标地址**：若是函数指针，就是 `call [ptr]`；直接调用是 `call symbol`。
    
4. **被调函数执行**：建立/使用栈帧、局部变量等。
    
5. **返回**：`ret` 弹出返回地址，跳回调用点。
    

**函数指针 = 能被 `call` 的地址**。类型必须匹配（参数/返回值/调用约定），否则是未定义行为。

---

# 2. 纯 C 风格的函数指针

## 2.1 声明与使用

```cpp
// 函数类型：int(int, int)
using Fn = int(int,int);          // 注意：这是“函数类型”，不是指针
using FnPtr = int(*)(int,int);    // 这是“指向该函数的指针类型”

int add(int a, int b) { return a+b; }

FnPtr p = &add; // 或直接 p = add;
int r = p(2, 3); // 通过指针调用，等价于 add(2,3)
```

- **不能返回“函数本体”**，但可以返回 `FnPtr`：
    

```cpp
FnPtr pick(bool plus) {
    return plus ? +[](int a,int b){return a+b;} // 非捕获lambda可转函数指针
                 : +[](int a,int b){return a-b;};
}
```

## 2.2 回调（callback）在 C API 里的典型形式

C 风格常见签名：**函数指针 + `void* userData`** 作为上下文。

```c
typedef void (*OnMsg)(int code, void* user);
void register_cb(OnMsg cb, void* user);
```

C++ 侧使用：

```cpp
struct Context { int base; };

void on_msg_impl(int code, void* user){
    auto* ctx = static_cast<Context*>(user);
    // 用 ctx 携带的状态做事
}

Context ctx{42};
register_cb(&on_msg_impl, &ctx);
```

> 这就是为什么很多 C 库要你传 `void*`：**回调是纯函数指针，状态另给一个通用指针**。

---

# 3. 指向“成员函数”的指针（Pointer-to-member）

与普通函数指针不同，**成员函数多了一个隐式 `this`**。语法与调用都不同：

```cpp
struct A {
    int mul(int x) { return k * x; }
    int k = 2;
};

int (A::*pmf)(int) = &A::mul;   // “指向 A::mul 的指针”
A a;
int r = (a.*pmf)(3);            // 通过对象调用
```

要点：

- 类型写作 `Ret (Class::*)(Args...)`。
    
- 调用时必须有对象（或指针）：`(obj.*pmf)(...)` / `(ptr->*pmf)(...)`。
    
- **不要把它当成普通函数指针强转**；其内部表示可能是“代码地址 + 调整信息/虚表索引”等，不同 ABI 不同。
    
- 若 `mul` 是虚函数，经由指向成员函数调用会走**动态分派**（尊重虚函数机制）。
    

---

# 4. Lambda（λ 表达式）与“可调用对象”

## 4.1 非捕获 lambda

```cpp
auto f = [](int a,int b){ return a+b; };
int r = f(1,2);
```

- **非捕获**（`[]` 中不捕变量）时，lambda 可**隐式转换为函数指针**：
    

```cpp
using FnPtr = int(*)(int,int);
FnPtr p = f;       // OK：无捕获
```

## 4.2 捕获 lambda = “闭包对象”

```cpp
int bias = 5;
auto g = [bias](int x){ return x + bias; };
```

- 这时 `g` 是一个**有状态的对象**（closure），本质是个结构体，里面存 `bias`，并定义了 `operator()`.
    
- **捕获 lambda 不能转换成函数指针**（因为它需要那份状态）。可存入 `std::function`（见下）。
    

---

# 5. `std::function` / `std::invoke` / `std::bind`：现代 C++ 里的“可调用体”

## 5.1 `std::function<R(Args...)>`

- 一个**类型抹消（type-erasure）**的包装器：能装**函数指针、捕获/非捕获 lambda、仿函数对象、`bind` 结果**等。
    
- 像普通函数一样调用：
    

```cpp
#include <functional>

std::function<int(int,int)> fn;

fn = [](int a,int b){ return a+b; };  // 捕获/非捕获皆可
fn = &add;                             // 函数指针也行

if (fn) {
    int r = fn(3,4); // 调用
}
```

- **实现细节**：内部一般有 **小对象优化（SBO）** 的缓冲区（通常大小≈几个指针）。
    
    - 小型可调用体放栈内；超出则**堆分配**。
        
    - 因此 `std::function` 可能有**分配/虚派发**开销；热路径用时要注意。
        

## 5.2 `std::invoke`

- 一个**统一的调用工具**：不管是函数指针、成员函数指针、lambda、仿函数，`std::invoke(callable, args...)` 都能正确调用。
    

```cpp
#include <functional>

std::invoke(p, 1, 2);            // p 是函数指针
std::invoke(f, 1, 2);            // f 是 lambda/仿函数
std::invoke(&A::mul, a, 3);      // 成员函数指针 + 对象
```

## 5.3 `std::bind`（老兵不死，但**更推荐 lambda**）

```cpp
auto h = std::bind(&A::mul, &a, std::placeholders::_1);
int r = h(7); // 等价于 a.mul(7)
```

- `bind` 早期很有用，但**可读性/调试性差**，现代 C++ **优先用 lambda**：
    

```cpp
auto h = [&a](int x){ return a.mul(x); }; // 更直观
```

## 5.4 “轻量非拥有”的 `function_ref` 思想（了解）

- 大型项目里常用一种“**不拥有目标，仅作引用的回调**”类型，避免 `std::function` 的堆分配开销。
    
- 标准里将引入 `std::function_ref`（C++26），各库（LLVM/absl）已有类似实现。
    
- 用法像：`void foo(function_ref<void(int)> cb) { cb(42); }` ——**不拷贝目标，不分配**。
    

---

# 6. 回调落地：从 C 到现代 C++

## 6.1 C 风格（函数指针 + userData）

```c
typedef void (*OnMsg)(int, void*);
void register_cb(OnMsg, void*);
```

C++ 侧桥接捕获状态：

```cpp
struct Ctx { int k; };
Ctx ctx{10};

register_cb(
    [](int code, void* user){
        auto* c = static_cast<Ctx*>(user);
        // ...
    },
    &ctx
);
```

## 6.2 现代 C++（`std::function`）

若你自己定义接口，优先这样：

```cpp
using OnMsg = std::function<void(int)>;

void Register(OnMsg cb); // 直接让对方传任意可调用体

Register([k=10](int code){
    // 捕获到状态了，简单直接
});
```

**注意**：如果这是**高频回调**，且你担心分配/虚派发开销，考虑 `function_ref`-风格或模板参数化：

```cpp
template <class F>
void Register(F&& cb) {        // 泛型回调
    callback_ = std::forward<F>(cb); // 自己决定怎么存（可能依然是 std::function）
}
```

---

# 7. 线程/异步场景中的回调

- **跨线程**时，回调里**不要直接触碰非线程安全对象**；转交到目标线程（比如渲染线程）再执行。
    
- 如果要“**立即执行并拿结果**”，用**同步 RPC**（你前面我们讨论过的 `CallNow` 模式）：
    
    - 回调类型可以是 `std::function<R(Args...)>`，内部用 `promise/future` 或条件变量把结果带回来。
        
- **异常与 noexcept**：给回调签名标上 `noexcept`（或在内部捕获异常），防止跨 ABI 传异常导致崩溃。
    

---

# 8. 性能与选择建议（超实用）

|需求|推荐工具|说明|
|---|---|---|
|极致性能、固定签名|**函数指针**|无分配、无类型擦除，C 兼容|
|成员函数回调|**指向成员函数** + 对象|用 `std::invoke(pmf, obj, args...)` 调|
|需要捕获状态，低复杂度|**lambda**|大多情况最清晰|
|统一装各种可调用体|**`std::function`**|简洁，可能分配，热路径谨慎|
|高频回调、避免分配|**function_ref 风格** 或模板化|轻量、零拥有；暂属库扩展/自实现|
|绑定部分参数|**lambda** 替代 `std::bind`|可读性更好|
|任意可调用体统一调用|**`std::invoke`**|覆盖函数/成员函数/仿函数|

---

# 9. 小抄（可以直接贴到你的代码库里）

```cpp
// 1) 普通函数指针
using Fn = int(*)(int,int);
int add(int,int);
Fn p = &add; int r = p(1,2);

// 2) 成员函数指针
struct S { int f(int) {return 0;} };
int (S::*pmf)(int) = &S::f; S s; (s.*pmf)(42);

// 3) 非捕获 lambda -> 函数指针
using FP = void(*)(int);
FP fp = +[](int){ /*...*/ }; // “+” 强制转函数指针可读性更好

// 4) 捕获 lambda 存 std::function
std::function<int(int)> g = [bias=5](int x){ return x+bias; };

// 5) 统一调用
std::invoke(&S::f, s, 10); // 成员函数指针
std::invoke(fp, 3);        // 函数指针
std::invoke(g, 7);         // std::function / 仿函数

// 6) C 回调桥接（函数指针 + void*）
extern "C" void reg(void(*cb)(int, void*), void*);
struct Ctx{int v;}; Ctx ctx{1};
reg([](int c, void* u){ auto* x=(Ctx*)u; /*...*/; }, &ctx);
```

---

# 10. 回到你渲染系统的落地点

- 你现在用的“**lambda + 渲染线程队列**”正是现代 C++ 的“**以可调用体为一等公民**”的思路。
    
- 需要**C 接口**时（比如图形驱动/三方库）：就用**函数指针 + userData**；  
    需要**C++ 接口**时：优先 `std::function` / 模板参数 / `function_ref` 风格；
    
- 线程安全时，别把**对象所有权**和**调用时序**搞乱：回调只是“**怎么调用**”，**何时/在哪个线程**执行仍由你掌控（队列/栅栏/安全点）。
    

---

如果你愿意，我可以给你一份**“回调模式全家桶”小库**（头文件），里头实现：

- `function_ref` 的简洁版本（零拥有、零分配）
    
- 一个 `InvokeResult` 工具类型（提取回调返回值类型）
    
- C 回调桥接模板（把捕获 lambda 包装成 `void(*)(..., void*)` + 静态 trampoline）
    
- 以及你渲染线程 `CallNow/Enqueue` 的统一可调用体签名设计示例
    

这样你在项目中就能**按需挑选**：极致性能 → 函数指针 / function_ref；灵活易用 → std::function；需要 C 兼容 → trampoline。