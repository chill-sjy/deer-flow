# Python 异步与协程学习笔记

> 作者视角：有 Java 多线程背景，通过对比建立 Python async/await 心智模型。

---

## 1. 核心区别：Java 多线程 vs Python 协程

| 维度 | Java 多线程 | Python async 协程 |
|---|---|---|
| 并发模型 | 多线程并行（操作系统调度） | 单线程协程交替执行（用户态调度） |
| I/O 等待时 | 线程阻塞，释放 CPU，但线程资源被占用 | 协程暂停，线程（事件循环）继续跑其他协程 |
| 唤醒机制 | 操作系统中断唤醒线程 | 事件循环通过 epoll/kqueue 感知 I/O 就绪，调用 `coroutine.send()` 恢复协程 |
| 资源开销 | 每线程约 1MB 栈空间 | 每协程约几 KB |
| 锁 | `synchronized` / `ReentrantLock` | `asyncio.Lock()` |

---

## 2. 协程是什么

协程（Coroutine）是一种**可以被暂停和恢复的函数**。

- Java 中没有原生协程概念（Java 21 的虚拟线程是类似概念）
- Python 用 `async def` 定义协程函数，用 `await` 标记暂停点

```python
async def initialize_mcp_tools() -> list[BaseTool]:
    # 遇到 await，协程可能在此暂停
    _mcp_tools_cache = await get_mcp_tools()
    return _mcp_tools_cache
```

类比 Java 的 `CompletableFuture`：

```java
// Java：在另一个线程执行
CompletableFuture<List<Tool>> future = CompletableFuture.supplyAsync(() -> getMcpTools());
List<Tool> tools = future.get();  // 阻塞当前线程等待结果
```

```python
# Python：在同一个线程内暂停和恢复
async def main():
    tools = await initialize_mcp_tools()  # 协程暂停，线程去跑其他协程
    # tools 在这里一定有值，下一行可以安全使用
    print(tools)
```

---

## 3. async def 和 await 的语法规定

### `async def` 是确定性声明

`async def` 不是"可能用到协程"，而是**确定地声明这个函数是一个协程函数**。

调用普通函数 vs 调用协程函数，行为完全不同：

```python
# 普通函数：调用就立即执行，直接返回结果
def foo():
    return 1

result = foo()   # result = 1，立即执行

# 协程函数：调用不执行！只返回一个协程对象
async def bar():
    return 1

result = bar()         # result = <coroutine object bar>，什么都没执行！
result = await bar()   # result = 1，必须 await 才真正执行
```

### `await` 只能在 `async def` 内使用

这是**语法强制规定**，在普通函数里使用 `await` 会直接报语法错误：

```python
def normal_func():
    result = await initialize_mcp_tools()  # ❌ SyntaxError，不允许

async def async_func():
    result = await initialize_mcp_tools()  # ✅ 正确
```

类比 Java：就像 `yield`（生成器）只能写在生成器方法里，`await` 只能写在 `async def` 里，这是语言层面的约束。

### 传染性：async 会向上扩散

一旦某个函数内部用了 `await`，它自身必须是 `async def`，调用它的函数如果也想 `await`，也必须是 `async def`，依此类推，一直传到程序入口。

```
asyncio.run(main())          ← 程序唯一的同步入口
    └── async def main()
            └── await foo()
                    └── async def foo()
                            └── await bar()
                                    └── async def bar()
```

---

## 3. await 的正确理解（重点，易混淆）

### 错误理解 ❌
> "await 之后，当前函数的下一行会立刻执行，等IO完成时再回来赋值"

### 正确理解 ✅

`await` 对协程内部和对线程有不同的含义：

```
┌──────────────────────────────────────────────────────────┐
│ 协程内部的视角（等同于 Java 函数内的顺序执行）：             │
│                                                          │
│   tools = await initialize_mcp_tools()                   │
│   print(tools)  ← 一定在 await 完成后才执行               │
│                                                          │
│   → await 对协程内部是顺序的，tools 必然有值              │
└──────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────┐
│ 线程（事件循环）的视角：                                    │
│                                                          │
│   协程A 在 await 期间，线程没有空等                         │
│   线程去执行了协程B、协程C...                               │
│   I/O 完成后，事件循环回来恢复协程A，继续执行               │
│                                                          │
│   → await 对线程是非阻塞的                                │
└──────────────────────────────────────────────────────────┘
```

对比 Java：

| | Java 阻塞 I/O | Python await |
|---|---|---|
| 函数内部顺序 | 有保证，等 I/O 完成才继续 | **同样有保证**，等 I/O 完成才继续 |
| 线程是否阻塞 | 线程阻塞，啥也干不了 | 线程不阻塞，去跑其他协程 |

---

## 4. 事件循环：协程调度器

事件循环（Event Loop）是 Python async 的核心，类比 **Java NIO 的 Selector**。

### Java NIO Selector（I/O 多路复用）

```java
Selector selector = Selector.open();
channel1.register(selector, SelectionKey.OP_READ);
channel2.register(selector, SelectionKey.OP_READ);

while (true) {
    selector.select();  // 阻塞等待，直到至少有一个 I/O 就绪
    Set<SelectionKey> readyKeys = selector.selectedKeys();
    // 处理就绪的 I/O
}
```

### Python 事件循环（原理等价）

```python
# asyncio 事件循环核心逻辑（简化示意）
class EventLoop:
    def __init__(self):
        self.selector = selectors.DefaultSelector()  # 封装 epoll/kqueue
        self.pending_tasks = {}  # socket -> 协程

    def run_forever(self):
        while True:
            # 1. 等待任意 I/O 就绪（单线程唯一阻塞点）
            events = self.selector.select()  # 对应 Java 的 selector.select()

            # 2. 处理就绪的 I/O
            for key, mask in events:
                socket = key.fileobj
                coroutine = self.pending_tasks[socket]

                # 3. 恢复协程执行（唤醒机制）
                coroutine.send(None)  # 相当于"唤醒"协程
```

底层系统调用：
- Linux：`epoll`
- macOS：`kqueue`
- Windows：`IOCP`

---

## 5. 执行时序图

假设有两个协程同时运行：

```
时间 ──────────────────────────────────────────────────────────►

事件循环（单线程）:

  [协程A执行] → 遇到 await network_request() → 注册到 selector → [协程A暂停]
                                                                         ↓
  [协程B执行] → 遇到 await database_query()  → 注册到 selector → [协程B暂停]
                                                                         ↓
  selector.select() ← 阻塞等待（epoll/kqueue 监听所有注册的 socket）
                                                                         ↓
  【操作系统通知：协程A的 socket 有数据了】
                                                                         ↓
  select() 返回 → 事件循环恢复协程A → [协程A继续执行后续代码]
```

---

## 6. 如何调用 async 函数

async 函数不能直接调用，有三种方式：

```python
# ❌ 错误：直接调用只返回协程对象，不执行
result = initialize_mcp_tools()

# ✅ 方式1：在另一个 async 函数中用 await（最常见）
async def main():
    result = await initialize_mcp_tools()

# ✅ 方式2：用 asyncio.run() 运行（程序入口）
asyncio.run(initialize_mcp_tools())

# ✅ 方式3：在已有事件循环中提交任务
loop = asyncio.get_event_loop()
loop.run_until_complete(initialize_mcp_tools())
```

---

## 7. 并发执行多个协程

类比 Java 的 `CompletableFuture.allOf()`：

```java
// Java：并行等待多个 Future
CompletableFuture.allOf(future1, future2).join();
```

```python
# Python：并发执行多个协程
await asyncio.gather(task_a(), task_b())
```

`asyncio.gather()` 会让两个协程交替执行，总耗时接近最慢的那个，而非两者之和。

---

## 8. 项目中的实际例子

`src/mcp/cache.py` 中的 `initialize_mcp_tools()`：

```python
async def initialize_mcp_tools() -> list[BaseTool]:
    async with _initialization_lock:        # 异步锁，防止并发重复初始化
        if _cache_initialized:
            return _mcp_tools_cache or []

        # await：等待网络连接 MCP Server，期间线程可处理其他请求
        _mcp_tools_cache = await get_mcp_tools()
        _cache_initialized = True
        return _mcp_tools_cache
```

- `async with _initialization_lock`：异步锁，对应 Java 的 `synchronized`，防止多个协程同时初始化
- `await get_mcp_tools()`：网络 I/O，协程在此暂停，等连接 MCP Server 完成后恢复

---

## 9. CPU 密集任务：协程的"饿死"问题

### 问题根源

事件循环**只在 `await` 处切换协程**。如果一个协程有大量 CPU 计算（中间没有 `await`），事件循环就会被卡住，其他协程即使 I/O 已经完成，也无法得到执行。

```
协程A: [CPU计算 2秒（无await）] → await IO → [CPU计算 2秒（无await）]
协程B: I/O早已完成，但一直在等协程A释放事件循环

时间线：
  ├── [协程A CPU 2秒，事件循环被卡死]
  │        ↑ 协程B就算IO完成了，在这2秒内完全无法执行
  ├── [协程A await IO，事件循环切换]
  ├── [协程B执行]
  ├── [协程A CPU 2秒，事件循环再次被卡死]
  └── ...
```

**对比 Java**：Java 多线程中，CPU 密集任务只影响那一个线程，其他线程由操作系统独立调度，不受影响。Python async 是单线程，一个协程的 CPU 任务会阻塞整个事件循环。

### 解决方案：把 CPU 密集任务丢到线程池

```python
import asyncio

def cpu_heavy_function():
    # 模拟耗时 CPU 计算
    result = 0
    for i in range(10**8):
        result += i
    return result

async def main():
    loop = asyncio.get_event_loop()

    # ❌ 直接在协程里做CPU计算，事件循环被卡死
    result = cpu_heavy_function()

    # ✅ 用 run_in_executor 丢到线程池，协程在此暂停，事件循环可以跑别的
    result = await loop.run_in_executor(None, cpu_heavy_function)
    #                                   ↑
    #                          None 表示使用默认线程池
```

`run_in_executor` 类比 Java 的 `CompletableFuture.supplyAsync()`：

```java
// Java：把任务提交到线程池，当前线程不阻塞
CompletableFuture<Integer> future = CompletableFuture.supplyAsync(() -> cpuHeavyFunction());
Integer result = future.get();
```

```python
# Python：把任务提交到线程池，协程暂停，事件循环不阻塞
result = await loop.run_in_executor(None, cpu_heavy_function)
```

### 适用场景总结

| 任务类型 | 做法 | 原因 |
|---|---|---|
| I/O 密集（网络、文件） | 直接 `await` | asyncio 天然擅长 |
| CPU 密集（计算、加密） | `run_in_executor` 丢线程池 | 避免卡死事件循环 |
| 混合（I/O + CPU） | I/O 用 `await`，CPU 用 `run_in_executor` | 分开处理 |

---

## 10. 总结

| 概念 | Java | Python |
|---|---|---|
| 异步执行 | `new Thread()` 或线程池 | `async def` 定义协程 |
| 等待结果（不阻塞线程） | `CompletableFuture` | `await` |
| 并发调度器 | 操作系统线程调度 | `asyncio` 事件循环 |
| I/O 多路复用 | `NIO Selector` | `asyncio`（封装 epoll/kqueue）|
| 锁 | `synchronized` / `ReentrantLock` | `asyncio.Lock()` |
| 并发任务 | `CompletableFuture.allOf()` | `asyncio.gather()` |

> **一句话**：Python async 是单线程的协程调度，通过 `await` 主动让出控制权，由事件循环基于 I/O 多路复用决定何时恢复，而非依赖操作系统线程调度。
