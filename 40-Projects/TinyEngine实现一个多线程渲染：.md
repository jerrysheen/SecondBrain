[[C++多线程并发基础入门教程]]
[[起一个线程的本质是执行一个函数]]

[[C++的Unoin的内存分布以及使用]]
[[RingBuffer的实现细节]]


## 结构定义：

``` cpp
enum class Op : uint8_t {
    kInvalid = 0,
    kBeginFrame,
    kEndFrame,
    kSetPipeline,
    kSetVBIB,
    kSetViewport,
    kSetScissor,
    kDrawIndexed,
    // ... 未来要扩展就在这里加
    // 细节， 不要抽象， 需要对应好底层的具体渲染指令。
};
```

```cpp
struct Payload_BeginFrame { uint64_t frameIndex; };
struct Payload_EndFrame   { uint64_t frameIndex; };

struct Payload_SetPipeline {
    uint32_t psoId; // 你的 PSO 表的索引/句柄
};

struct Payload_SetVBIB {
    uint32_t vbHandle;
    uint32_t ibHandle;
    uint32_t vbStride;
    uint32_t ibFormat; // DXGI_FORMAT_R16_UINT / R32_UINT 等自定义枚举
};

struct Payload_SetViewport {
    float x, y, w, h, minDepth, maxDepth;
};
```

```cpp
struct Command {
    Op op{Op::kInvalid};
    CommandData data{};
};
```

## 渲染线程负责：
相当于整个Renderer， 现在都会在渲染线程中，
主线程通过发送内容到渲染线程， 渲染线程Renderer消化这个内容。

### 主线程：
```cpp
// main.cpp
#include "Renderer.h"

int main() {
    Renderer r;

    for (uint64_t frame = 0; frame < 3; ++frame) {
        r.BeginFrame(frame);
        r.SetPipeline(1);
        r.SetViewport(0,0,1920,1080);
        r.SetScissor(0,0,1920,1080);
        r.SetVBIB(/*vb*/10, /*ib*/20, /*stride*/32, /*fmt*/0);
        r.DrawIndexed(36, 0, 0);
        r.EndFrame(frame);
    }
    // 析构器会优雅退出渲染线程
    return 0;
}

```

渲染线程提供调用的接口，
1. 录制DrawCommand进队列，
2. ~~Update的时候进行消费~~
这个时候不是Update的事情了，renderingthread自己也是一个while循环，通过是否有命令需要执行去单独执行， 而不是主线程调用Update更新renderer的逻辑了。
```cpp
// Renderer.h
#pragma once
#include "SpscRing.h"
#include <thread>
#include <atomic>
#include <iostream>

class Renderer {
public:
    Renderer() : m_running(true), m_thread(&Renderer::renderLoop, this) {}
    ~Renderer() {
        // 发送最后的退出命令（简单粗暴：置标志+唤醒）
        m_running.store(false, std::memory_order_release);
        // 防止消费者阻塞
        m_cmdCV.notify_all();
        if (m_thread.joinable()) m_thread.join();
    }

    // --- 生产者 API（主线程调用） ---
    void BeginFrame(uint64_t frameIndex) {
        Command c; c.op = Op::kBeginFrame; c.data.beginFrame = { frameIndex };
        m_queue.push_blocking(c);
    }

    void EndFrame(uint64_t frameIndex) {
        Command c; c.op = Op::kEndFrame; c.data.endFrame = { frameIndex };
        m_queue.push_blocking(c);
    }

    void SetPipeline(uint32_t psoId) {
        Command c; c.op = Op::kSetPipeline; c.data.setPipeline = { psoId };
        m_queue.push_blocking(c);
    }

    void SetVBIB(uint32_t vbHandle, uint32_t ibHandle, uint32_t stride, uint32_t ibFmt) {
        Command c; c.op = Op::kSetVBIB; c.data.setVBIB = { vbHandle, ibHandle, stride, ibFmt };
        m_queue.push_blocking(c);
    }

    void SetViewport(float x, float y, float w, float h, float minD=0.f, float maxD=1.f) {
        Command c; c.op = Op::kSetViewport; c.data.setViewport = { x,y,w,h,minD,maxD };
        m_queue.push_blocking(c);
    }

    void SetScissor(int32_t x, int32_t y, int32_t w, int32_t h) {
        Command c; c.op = Op::kSetScissor; c.data.setScissor = { x,y,w,h };
        m_queue.push_blocking(c);
    }

    void DrawIndexed(uint32_t indexCount, uint32_t startIndex, int32_t baseVertex) {
        Command c; c.op = Op::kDrawIndexed; c.data.drawIndexed = { indexCount, startIndex, baseVertex };
        m_queue.push_blocking(c);
    }

private:
    void renderLoop() {
        // 初始化 D3D12 设备/队列/命令分配器...（略）
        uint64_t currentFrame = 0;

        while (m_running.load(std::memory_order_acquire)) {
            Command cmd;
            if (!m_queue.try_pop(cmd)) {
                // 没有命令就阻塞等待
                std::unique_lock<std::mutex> lk(m_waitMu);
                m_cmdCV.wait_for(lk, std::chrono::milliseconds(1));
                continue;
            }

            switch (cmd.op) {
                case Op::kBeginFrame: {
                    currentFrame = cmd.data.beginFrame.frameIndex;
                    // Reset allocator/list, set backbuffer RT, etc.
                    // cmdList->Reset(...);
                } break;

                case Op::kSetPipeline: {
                    auto& q = cmd.data.setPipeline;
                    // cmdList->SetPipelineState( PSO[q.psoId] );
                } break;

                case Op::kSetVBIB: {
                    auto& q = cmd.data.setVBIB;
                    // IASetVertexBuffers/IndexBuffer 通过句柄表查真正资源
                } break;

                case Op::kSetViewport: {
                    auto& q = cmd.data.setViewport;
                    // cmdList->RSSetViewports( ... );
                } break;

                case Op::kSetScissor: {
                    auto& q = cmd.data.setScissor;
                    // cmdList->RSSetScissorRects( ... );
                } break;

                case Op::kDrawIndexed: {
                    auto& q = cmd.data.drawIndexed;
                    // cmdList->DrawIndexedInstanced(q.indexCount, 1, q.startIndex, q.baseVertex, 0);
                } break;

                case Op::kEndFrame: {
                    // Close / ExecuteCommandLists / Present / Signal fence
                    // queue->ExecuteCommandLists(...);
                    // swapchain->Present(...);
                    // fence->Signal(frameFenceValue++);
                    // Flush when necessary
                } break;

                default: break;
            }
        }
        // 清理 GPU 资源/同步（略）
    }

private:
    SpscRing<1024> m_queue;
    std::atomic<bool> m_running;
    std::thread m_thread;

    // 仅用于渲染线程空转时短暂 sleep 的等待
    std::mutex m_waitMu;
    std::condition_variable m_cmdCV;
};

```

### SPSC 环形队列：
```cpp
// SpscRing.h
#pragma once
#include "Command.h"

template <size_t CapacityPow2>
class alignas(64) SpscRing {
    static_assert((CapacityPow2 & (CapacityPow2 - 1)) == 0, "Capacity must be power of two");
public:
    bool try_push(const Command& cmd) {
        auto head = m_head.load(std::memory_order_relaxed);
        auto next = (head + 1) & (CapacityPow2 - 1);
        if (next == m_tail.load(std::memory_order_acquire)) {
            return false; // full
        }
        m_buf[head] = cmd;                     // 单生产者：安全的非原子写
        m_head.store(next, std::memory_order_release);
        return true;
    }

    // 生产者阻塞 push（推荐开发期用）
    void push_blocking(const Command& cmd) {
        while (!try_push(cmd)) {
            std::unique_lock<std::mutex> lk(m_mu);
            m_cv_not_full.wait(lk, [&]{ return !is_full(); });
        }
        // 唤醒消费者
        m_cv_not_empty.notify_one();
    }

    bool try_pop(Command& out) {
        auto tail = m_tail.load(std::memory_order_relaxed);
        if (tail == m_head.load(std::memory_order_acquire)) {
            return false; // empty
        }
        out = m_buf[tail];                     // 单消费者：安全的非原子读
        m_tail.store((tail + 1) & (CapacityPow2 - 1), std::memory_order_release);
        return true;
    }

    // 消费者阻塞 pop
    void pop_blocking(Command& out) {
        while (!try_pop(out)) {
            std::unique_lock<std::mutex> lk(m_mu);
            m_cv_not_empty.wait(lk, [&]{ return !is_empty(); });
        }
        // 唤醒生产者
        m_cv_not_full.notify_one();
    }

    bool is_full() const {
        auto head = m_head.load(std::memory_order_relaxed);
        auto next = (head + 1) & (CapacityPow2 - 1);
        return next == m_tail.load(std::memory_order_acquire);
    }

    bool is_empty() const {
        return m_tail.load(std::memory_order_relaxed) ==
               m_head.load(std::memory_order_acquire);
    }

private:
    std::array<Command, CapacityPow2> m_buf{};
    std::atomic<size_t> m_head{0}; // 生产者写，消费者读
    std::atomic<size_t> m_tail{0}; // 消费者写，生产者读

    // 背压与唤醒
    mutable std::mutex m_mu;
    std::condition_variable m_cv_not_full;
    std::condition_variable m_cv_not_empty;
};

```

