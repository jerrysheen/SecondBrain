---
id:
  "{ date:YYYYMMDDHHmm }":
type: permanent
tags:
---
# 命题：C++：父类的静态变量，是每个继承的子类共享的
## 定义 / 结论
也没什么原理，就是多个子类共享， 因为父类只有一个静态变量，子类只是取它的值。
要注意之前潜意识里面觉得好像子类自己独享一个，那是因为我们用的是 Manager T这种模板类型，这个和父子继承不太一样，
因为模板类会为各个子类生成独立的一个class，所以他们不共享静态成员。
```cpp
// 编译器为SceneManager生成
class Manager_SceneManager {
    static std::unique_ptr<SceneManager> s_Instance;  // 独立的
};

// 编译器为WindowManager生成
class Manager_WindowManager {
    static std::unique_ptr<WindowManager> s_Instance;  // 独立的
};
```
## 机制 / 因果
- <为什么对>

## 操作 / 代码要点
1) <步骤>
2) <关键API/陷阱>

## 反例 / 边界 CRTP模式
CRTP模式下，
也就是用template写的父类， 则不会有这种情况。
```cpp
template<Typename T>
class Manager
{
	static std::unique_ptr<T> s_Instance;
};

class ResourceManager : Manager<ResouceManager>
{
	ResourceManaer* GetInstance(){return s_Instance.Get();}
}
.cpp
std::unique_ptr<ResouceManager> ResourceManager::s_Instance = nullptr;
```

## 引申/ 同类型
- <同类型的有什么>
## 连接
- 定义 →
- 因果 →
- 实现 →
- 应用 →
- 证据 →
