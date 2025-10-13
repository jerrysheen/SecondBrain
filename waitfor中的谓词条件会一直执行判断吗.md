# 2025-10-05 14:21 mCVIsNotFull.wait_for(lk, [&]{return !IsFull();});

还有就是std::memory order 的使用，怎么区分的。

- 想法：<一句话问题/灵感>
- 背景/触发：
比如以下代码，isFull在什么时候会检查？
```cpp
        // 阻塞直到push成功
        void PushBlocking(const DrawCommand& cmd)
        {
            while(!TryPush(cmd))
            {
                std::unique_lock<std::mutex> lk(m_mu);
                mCVIsNotFull.wait_for(lk, [&]{return !IsFull();});

            }
            mCVIsNotEmpty.notify_all();
        };
```
- 下一步：- [ ] 转Literature / - [ ] 直接Permanent？
tags: [fleeting]
