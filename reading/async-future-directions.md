# 异步编程的过去、现在与未来

> 基于 [What Async Promised and What it Delivered](https://causality.blog/essays/what-async-promised/)（Josh Segall, April 2026）及相关最新进展整理

---

## 一、核心论题：每次修复都制造新的结构性代价

文章作者把异步编程的演进描述为一个"击鼓传花"式的故事：每一波方案解决了上一波的最大痛点，同时引入新的结构性成本。

### 演进脉络

| 阶段 | 解决的问题 | 引入的新问题 |
|------|-----------|------------|
| **回调（Callbacks）** | 线程-连接比过高，C10K 问题 | 控制流倒置、错误处理碎片化、回调地狱 |
| **Promise/Future** | 嵌套金字塔、错误处理集中化 | 只能一次性 resolve、错误静默吞噬、轻度类型分裂 |
| **Async/Await** | 线性序列的人体工程学 | **函数着色**、生态系统碎片化、新型死锁、**顺序性陷阱** |

---

## 二、文章最重要的两个概念

### 2.1 函数着色（Function Coloring）

Bob Nystrom 2015 年的经典比喻：把 async 函数比作"红色函数"，sync 函数比作"蓝色函数"。红色函数可以调用蓝色，但蓝色调用红色必须进行特殊处理，且这个属性会病毒式传播。

**实际代价：**

- **函数级别**：加一个 I/O 调用，函数签名改变，所有调用方必须跟着改。
- **库级别**：Python 的 `requests`（同步）和 `aiohttp`（异步）是两个独立项目；`httpx` 才提供统一接口，但这个统一本不该是问题。
- **生态系统级别**：Rust 的 Tokio 和 async-std 运行时不兼容，库作者必须选边站，或者维护双份 API。

### 2.2 顺序性陷阱（The Sequential Trap）

这是文章最犀利的观察，也是最少被讨论的问题：

```javascript
// 看起来干净，实则顺序执行了本可并行的操作
async function loadDashboard(userId) {
    const user = await getUser(userId);
    const orders = await getOrders(user.id);          // 等 orders 完成
    const recommendations = await getRecommendations(user.id); // 才开始这个
    return render(user, orders, recommendations);
}
```

`orders` 和 `recommendations` 相互独立，完全可以并行，但 `async/await` 的顺序语法**主动隐藏了依赖关系**——而依赖关系正是判断"什么可以并行"的唯一依据。

并行版本需要程序员手动分析依赖并打破顺序语法：

```javascript
const [orders, recommendations] = await Promise.all([
    getOrders(user.id),
    getRecommendations(user.id)
]);
```

**这是一个根本性矛盾**：async/await 以"让异步代码看起来像同步代码"为卖点，而顺序语法天然掩盖了并发机会。

---

## 三、当前最新进展（2025-2026）

### 3.1 Java Project Loom：虚拟线程的大规模落地

Java 21 稳定引入虚拟线程，2025 年已被 Netflix、Uber、Stripe、Amazon、LinkedIn 等公司用于生产环境。

**核心思路**：让线程本身变轻，而不是改变编程模型。

- 虚拟线程由 JVM 管理，创建百万级线程内存开销极低
- 阻塞 I/O 调用在 JVM 层自动转为非阻塞，程序员写普通同步代码
- **完全没有函数着色**——没有 async/await，没有回调，就是普通的 `try/catch`
- Spring Boot 3.2+ 已默认支持，一个微服务处理 100 万并发 WebSocket 连接内存减少 80%

Java 25（JEP 505）同步推进结构化并发 API（`StructuredTaskScope`），让并发任务的生命周期和错误传播更可预测。

**局限**：对 CPU 密集型任务无帮助；资源瓶颈转移到连接池等外部约束。

### 3.2 Zig：从语言层面彻底击败函数着色

Zig 的新 I/O 设计（2025 年 Kristoff 博客，计划随 0.16.0 发布）提出了目前最激进的解法。

**核心思路**：把 I/O 实现从语言关键字降级为可注入的接口参数，类比 Allocator。

```zig
// 新 Zig：I/O 实现由调用者注入
fn saveData(io: Io, data: []const u8) !void {
    const file = try Io.Dir.cwd().createFile(io, "save.txt", .{});
    defer file.close(io);
    try file.writeAll(io, data);
}
```

**关键设计：分离异步性（asynchrony）与并发性（concurrency）**

- `io.async(fn, args)` 声明"这些操作之间没有必要的顺序"——表达**异步性**
- `io.asyncConcurrent(fn, args)` 明确要求真正的并发执行
- 同一份代码，换不同的 `Io` 实现（阻塞/线程池/绿色线程/stackless 协程），行为自动适配
- **函数签名不变**，着色问题从根本上消除

标准库将提供：阻塞 I/O、线程池、基于 `io_uring` 的绿色线程、Stackless 协程（兼容 WASM）。

### 3.3 Rust：异步生态趋向成熟，但结构问题仍存

2025 年 Rust 异步编程的主要进展：

- **async fn in traits** 稳定化，之前这是重大缺失
- Tokio 与 async-std 的 API 兼容性有所改善
- `io_uring` 深度集成，I/O 密集型场景性能大幅提升
- 生成器（generators）和异步生成器进入推进阶段，简化流处理

但 O'Connor（2026）记录的 "futurelock" 死锁问题揭示 Rust async 仍有深层陷阱：future 持有锁后停止被 poll，导致死锁，需要 core dump + 反汇编才能诊断。函数着色问题也未根本解决。

### 3.4 Java 结构化并发与 Swift 结构化并发

**结构化并发**（Structured Concurrency）是当前另一条主流路线：把并发任务的生命周期绑定到词法作用域，确保子任务不会在父任务退出后泄漏。

- Java：`StructuredTaskScope`（JEP 505，Java 25 预览），与虚拟线程配合使用
- Swift 6.2：改进了 Actor 隔离语义，`@concurrent` 属性显式标记后台执行，默认 `@MainActor` 减少 UI 竞态

结构化并发不解决函数着色，但提升了错误传播、取消传播的可预测性。

### 3.5 代数效应（Algebraic Effects）：学术前沿

OCaml 5 已落地 effect handler，Koka、Effekt 等研究语言围绕效应系统构建。

**潜力**：效应可以统一描述异常、状态、异步、协程——理论上一套机制解决所有副作用建模问题，彻底消解 sync/async 二元对立。

**现状**：性能开销、与现有类型系统的集成仍是障碍，主流语言落地遥遥无期。

---

## 四、未来方向的判断

### 4.1 短期（1-3 年）：虚拟线程路线将进一步扩散

Java Loom 证明了"让线程变轻"是工程上最务实的路线——不改变编程模型，不引入着色，直接通过运行时解决资源问题。Go 的 goroutine 早已验证了这个思路。

预计：其他 JVM 语言（Kotlin、Scala）将深度整合虚拟线程；.NET 也会持续改善类似机制。

### 4.2 中期（3-5 年）：依赖图显式化

文章提出的核心问题是：async/await 隐藏了依赖关系。未来的工具/语言可能：

- 静态分析工具自动检测"可并行但被顺序写法遮蔽"的 await 序列
- 编译器辅助的依赖推断，自动生成 `Promise.all` 等并行模式
- 新的语法糖，让程序员直接声明"这些操作相互独立"，而不是靠顺序代码暗示

Zig 的 `io.async` 是这个方向的先行者：**明确声明异步性，而非通过语法顺序隐式表达**。

### 4.3 长期：重新思考抽象层次

文章最后的结论值得重读：

> "approaches that start by asking 'how do we manage concurrent execution?' keep generating new problems at every level of abstraction."

每一次的修补都是在当前框架内打补丁。真正的突破可能需要换一个问题：

- 不问"如何调度并发执行"，而问"**如何表达数据依赖**"
- 代数效应、依赖类型、数据流编程等路线在各自探索这个方向
- Zig 的"异步性不等于并发性"是目前最清晰的概念区分

---

## 五、小结

| 路线 | 代表 | 解决函数着色？ | 解决顺序陷阱？ | 生产成熟度 |
|------|------|--------------|--------------|-----------|
| 虚拟线程 | Java Loom, Go goroutine | ✅ 彻底 | ❌ 程序员仍需手动并行 | ✅ 高 |
| Io 接口参数化 | Zig | ✅ 彻底 | 部分（需显式 io.async） | 🔄 开发中 |
| 改良 async/await | Rust, Python, JS | ❌ 本质未变 | ❌ 未变 | ✅ 高 |
| 结构化并发 | Java StructuredTaskScope, Swift | ❌ 未解决 | ❌ 未解决 | 🔄 预览/早期 |
| 代数效应 | OCaml 5, Koka | 理论上✅ | 理论上✅ | ❌ 学术为主 |

**核心矛盾仍未解决**：主流语言的 async/await 让"写单个函数"很舒适，但让"维护大型代码库、发现并行机会"更困难。虚拟线程路线是当下最务实的工程答案；Zig 的 Io 接口是最有概念突破性的语言层设计；代数效应是理论上最彻底但工程距离最远的方向。

---

*整理日期：2026-05-06*
*主要参考：[causality.blog](https://causality.blog/essays/what-async-promised/)、[kristoff.it/blog/zig-new-async-io](https://kristoff.it/blog/zig-new-async-io/)、Java 25 JEP 505、Swift 6.2 并发更新*

---

## 附录：虚拟线程详解——对 JavaScript / Python 用户的意义

### A.1 概念精确定义：四种"线程"的区别

理解虚拟线程，需要先把几个容易混淆的概念分清楚。

#### OS 线程（操作系统线程）

由内核管理。创建、切换都走 syscall，进入内核态。Linux 默认每个线程保留 8MB 虚拟地址空间作为栈。同时跑 1 万个 OS 线程，内存和调度开销都很可观——这就是 C10K 问题的根源。

#### 虚拟线程 / 绿色线程（Virtual Thread / Green Thread）

由**用户态运行时**（JVM、Go runtime、gevent 等）管理，内核完全不知道它们的存在。

- 栈很小（几 KB 起步），可动态增长
- 切换在用户态完成，无 syscall 开销
- **关键**：当虚拟线程遇到阻塞 I/O，运行时把它挂起，把底层的 OS 线程让给其他虚拟线程。I/O 完成后再恢复

这是 Java Loom（`Virtual Thread`）、Go goroutine 和 Python gevent（`greenlet`）共享的基本机制。它们本质上是同一类东西，只是集成深度不同。

#### 协程 / async-await（Coroutine）

协程是**无栈**的——编译器把 `async` 函数改写成状态机，用堆上的结构体保存局部变量。

- 切换点是程序员显式写下的 `await`，而不是运行时自动检测阻塞
- 因此需要**函数着色**：调用方必须知道被调方是不是协程，否则无法 `await`
- Python 的 `asyncio`、JavaScript 的 `async/await`、Rust 的 `Future` 都是这个模型

#### 对比一览

| | OS 线程 | 虚拟线程/绿色线程 | 协程（async/await） |
|--|---------|-----------------|-------------------|
| 谁管理 | 内核 | 用户态运行时 | 编译器生成状态机 |
| 有没有"真实的栈" | 有 | 有（可增长） | **没有**（状态机） |
| 切换触发方式 | 内核抢占 | 运行时检测到 I/O 阻塞 | 程序员写 `await` |
| 函数着色 | 无 | **无** | **有** |
| 可同时存在数量 | 几千 | 几十万~百万 | 几十万~百万 |

**虚拟线程和协程解决的是同一个问题（I/O 并发），但虚拟线程对程序员完全透明，协程需要程序员参与。**

---

### A.2 Python：最接近虚拟线程的是 gevent，不是 asyncio

#### Python 的困境：GIL

CPython（标准实现）有全局解释器锁（GIL），同一时刻只有一个 OS 线程可以执行 Python 字节码。这让"多线程"在 Python 里对 I/O 有效（线程等 I/O 时会释放 GIL），对 CPU 密集型几乎无效。

Python 3.13 引入了实验性的 free-threaded 模式（移除 GIL），但这不是虚拟线程——它只是让 OS 线程能真正并行，代价是更复杂的内存模型，且 2025 年还在实验阶段，大量 C 扩展不兼容。

#### 路线一：asyncio（函数着色，现状主流）

```python
import asyncio
import aiohttp  # 必须用 async 兼容库，不能用 requests

async def fetch_user(session, user_id):          # 必须标 async
    async with session.get(f"/users/{user_id}") as r:
        return await r.json()

async def fetch_orders(session, user_id):        # 必须标 async
    async with session.get(f"/orders/{user_id}") as r:
        return await r.json()

async def load_dashboard(user_id):
    async with aiohttp.ClientSession() as session:
        # 顺序陷阱：这两个请求其实可以并行
        user   = await fetch_user(session, user_id)
        orders = await fetch_orders(session, user_id)

        # 正确写法：显式并行
        user, orders = await asyncio.gather(
            fetch_user(session, user_id),
            fetch_orders(session, user_id),
        )
        return render(user, orders)

asyncio.run(load_dashboard(42))
```

**代价**：整个调用栈都必须 async。`requests` 不能用，必须换 `aiohttp` 或 `httpx`。已有的同步代码库迁移成本极高。

#### 路线二：gevent（绿色线程，透明并发）

gevent 是 Python 里最接近 Java Loom 理念的方案。它的核心技术是两件事：

1. **greenlet**：用户态的绿色线程，有真实的栈，切换在用户态完成
2. **monkey-patch**：在程序启动时替换标准库的 socket、threading 等模块，让所有阻塞调用变成协作式——遇到 I/O 自动让出控制权，无需程序员标 `await`

```python
from gevent import monkey
monkey.patch_all()          # 必须在所有 import 之前调用

import gevent
import requests             # 注意：这里用的是普通 requests，不是 aiohttp！

def fetch_user(user_id):    # 普通函数，不需要 async
    return requests.get(f"http://api/users/{user_id}").json()

def fetch_orders(user_id):  # 普通函数，不需要 async
    return requests.get(f"http://api/orders/{user_id}").json()

def load_dashboard(user_id):
    # spawn 启动绿色线程，立即返回 Greenlet 对象
    g_user   = gevent.spawn(fetch_user,   user_id)
    g_orders = gevent.spawn(fetch_orders, user_id)

    gevent.joinall([g_user, g_orders])   # 等两者都完成

    user   = g_user.value
    orders = g_orders.value
    return render(user, orders)

load_dashboard(42)
```

关键对比：`fetch_user` 和 `fetch_orders` 是**完全普通的同步函数**，不带任何 async 标记。现有的同步代码库（requests、数据库驱动等）在 monkey-patch 之后大多可以直接复用。

#### monkey-patch 的工作原理

```python
# monkey.patch_all() 做的事情，简化版：
import socket as _orig_socket
import gevent.socket as _gevent_socket

# 把标准库的 socket 替换成 gevent 版本
socket.socket = _gevent_socket.socket
# ... 同样替换 ssl、threading、time.sleep 等
```

`requests` 底层用的是 `socket`。patch 之后，当 `socket.recv()` 等待数据时，gevent 的事件循环自动把当前 greenlet 挂起，切换到另一个就绪的 greenlet——对 `requests` 完全透明。

#### 路线三：Python 3.13 free-threaded（移除 GIL）

这不是虚拟线程，是另一条路：让 OS 线程真正并行。

```python
# Python 3.13，需要安装 free-threaded 版本（python3.13t）
import threading
import requests

results = {}

def fetch(key, url):
    results[key] = requests.get(url).json()   # 线程真正并行执行

threads = [
    threading.Thread(target=fetch, args=(i, url))
    for i, url in enumerate(urls)
]
for t in threads: t.start()
for t in threads: t.join()
```

**现状（2025）**：实验性，默认不开启。大量 C 扩展（NumPy、Pandas 等）需要专门适配，生产环境使用尚早。它解决的主要是 CPU 密集型场景，I/O 并发原本 GIL 就不是主要瓶颈。

#### Python 三条路线对比

| | asyncio | gevent | free-threaded OS 线程 |
|--|---------|--------|----------------------|
| 函数着色 | 有 | **无** | 无 |
| 能用 requests | 不能 | **能** | 能 |
| 迁移现有代码 | 代价高 | **代价低** | 中等 |
| 机制 | 协程状态机 | 绿色线程 | OS 线程（真并行） |
| CPU 密集 | 差 | 差 | 好 |
| 生产成熟度 | 高 | 高 | 实验性 |
| 主要风险 | 着色传染 | monkey-patch 脆弱性 | C 扩展兼容性 |

---

### A.3 JavaScript：事件循环就是答案，虚拟线程不适用

#### JS 为什么不需要虚拟线程

JavaScript 从设计之初就是单线程 + 事件循环。它从来没有"一个连接一个 OS 线程"的阶段，所以也没有 C10K 问题需要解决。Node.js 的高并发本来就是靠事件循环实现的。

虚拟线程解决的问题（"我有很多 OS 线程在阻塞 I/O，浪费资源"）在 JS 里根本不存在——JS 始终只有一个 OS 线程在跑用户代码。

但 JS 有自己的代价：函数着色。

#### async/await 的着色问题在 JS 里的样子

```javascript
// sync 版本
function getUserName(userId) {
    const user = db.getUser(userId);   // 假设这是同步的（不可能）
    return user.name;
}

// 加了一个 DB 调用，整条链都必须变成 async
async function getUserName(userId) {
    const user = await db.getUser(userId);
    return user.name;
}

// 所有调用方也必须变
async function renderProfile(userId) {        // 被迫变成 async
    const name = await getUserName(userId);   // 被迫加 await
    return `<h1>${name}</h1>`;
}

// 一直往上传染，直到顶层
app.get('/profile/:id', async (req, res) => {
    const html = await renderProfile(req.params.id);
    res.send(html);
});
```

#### 顺序陷阱在 JS 里的常见形态

```javascript
// ❌ 常见错误：看起来并行，实际串行
async function getStats(userId) {
    const posts    = await getPosts(userId);      // 等完才开始下一个
    const comments = await getComments(userId);   // posts 和 comments 互不依赖
    const likes    = await getLikes(userId);      // 三个请求完全可以同时发
    return { posts, comments, likes };
}

// ✅ 正确：显式声明独立性
async function getStats(userId) {
    const [posts, comments, likes] = await Promise.all([
        getPosts(userId),
        getComments(userId),
        getLikes(userId),
    ]);
    return { posts, comments, likes };
}

// ✅ 另一种写法：先启动，后等待（更接近"声明依赖"的思维）
async function getStats(userId) {
    const postsP    = getPosts(userId);       // 立即开始，不 await
    const commentsP = getComments(userId);    // 立即开始，不 await
    const likesP    = getLikes(userId);       // 立即开始，不 await

    return {
        posts:    await postsP,
        comments: await commentsP,
        likes:    await likesP,
    };
}
```

第三种写法揭示了一个规律：**把 Promise 的创建和 await 分开，就是在手动表达"这些操作相互独立"**——这正是 Zig 的 `io.async` 想自动化的事情。

#### Node.js Worker Threads：不是虚拟线程，是真 OS 线程

```javascript
// worker_threads 是给 CPU 密集型任务用的，不是 I/O 并发
const { Worker, isMainThread, parentPort } = require('worker_threads');

if (isMainThread) {
    const worker = new Worker(__filename);
    worker.on('message', result => console.log(result));
    worker.postMessage({ n: 40 });
} else {
    parentPort.on('message', ({ n }) => {
        // 这个 fib 跑在独立 OS 线程里，不阻塞主线程
        parentPort.postMessage(fib(n));
    });
}

function fib(n) {
    return n <= 1 ? n : fib(n-1) + fib(n-2);
}
```

Worker Threads 每个都是真正的 OS 线程，开销和 Java 的老式线程池一样。它解决的是"CPU 密集型任务阻塞事件循环"的问题，和虚拟线程的目标完全不同。

#### JS 并发的当前状态总结

JS 没有虚拟线程，也不太需要。它的并发模型是：

- **I/O 并发**：事件循环 + async/await，性能已经很好，但有着色问题和顺序陷阱
- **CPU 并行**：Worker Threads（真 OS 线程），适合计算密集型，但通信开销大
- **未来**：TC39 没有虚拟线程相关提案，方向是继续改善 async/await 的人体工程学（更好的错误追踪、`Promise.withResolvers` 等细节改进）

---

### A.4 如果你写 JS/Python，现在该怎么办

**JavaScript**：

1. 养成检查"顺序陷阱"的习惯。每次写 `await`，问自己：这个操作依赖上一行的结果吗？不依赖就用 `Promise.all` 或先启动后等待。
2. 函数着色无法消除，接受它。保持 async 在边界处（路由 handler、顶层入口），不要让 async 无谓地扩散到纯计算逻辑里。
3. CPU 密集型用 Worker Threads，但尽量少用——架构上应该把 CPU 密集型任务推给专门的服务。

**Python**：

- **新项目，I/O 密集**：asyncio + httpx/aiohttp 是主流，生态最好
- **旧代码库迁移，不想改函数签名**：gevent 是最低阻力路径，monkey-patch 让现有同步代码"免费"获得并发
- **CPU 密集型**：multiprocessing（多进程）仍是最稳妥的，free-threaded 模式等 2026 年后再评估生产可用性

```python
# Python 中最容易踩的顺序陷阱
import asyncio
import httpx

async def bad_example():
    async with httpx.AsyncClient() as client:
        # ❌ 串行，每个请求等上一个完成
        r1 = await client.get("https://api.example.com/a")
        r2 = await client.get("https://api.example.com/b")
        r3 = await client.get("https://api.example.com/c")

async def good_example():
    async with httpx.AsyncClient() as client:
        # ✅ 并行，三个请求同时发出
        r1, r2, r3 = await asyncio.gather(
            client.get("https://api.example.com/a"),
            client.get("https://api.example.com/b"),
            client.get("https://api.example.com/c"),
        )
```

---

---

### A.5 虚拟线程 vs JS 异步：同一场景的两种写法

用一个具体场景对比：**电商后台聚合页——同时拉取用户信息、订单列表、推荐商品，三个接口独立，最后合并渲染。**

这个场景典型：三个 I/O 操作相互独立，是顺序陷阱最容易出现的地方。

#### Java（虚拟线程，Java 21+）

```java
// 依赖：jdk.httpserver（标准库），Java 21+
import java.net.URI;
import java.net.http.HttpClient;
import java.net.http.HttpRequest;
import java.net.http.HttpResponse;
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;
import java.util.concurrent.Future;

HttpClient http = HttpClient.newHttpClient();

// 普通同步函数，没有任何 async 标记
String fetchUser(String userId) throws Exception {
    var req = HttpRequest.newBuilder()
        .uri(URI.create("https://api.example.com/users/" + userId))
        .build();
    // send() 是阻塞调用——但在虚拟线程里阻塞不会占用 OS 线程
    return http.send(req, HttpResponse.BodyHandlers.ofString()).body();
}

String fetchOrders(String userId) throws Exception {
    var req = HttpRequest.newBuilder()
        .uri(URI.create("https://api.example.com/orders?user=" + userId))
        .build();
    return http.send(req, HttpResponse.BodyHandlers.ofString()).body();
}

String fetchRecommendations(String userId) throws Exception {
    var req = HttpRequest.newBuilder()
        .uri(URI.create("https://api.example.com/recommendations?user=" + userId))
        .build();
    return http.send(req, HttpResponse.BodyHandlers.ofString()).body();
}

// 聚合函数——用虚拟线程并行执行三个独立请求
DashboardData loadDashboard(String userId) throws Exception {
    // newVirtualThreadPerTaskExecutor：每个任务一个虚拟线程，开销极小
    try (ExecutorService exec = Executors.newVirtualThreadPerTaskExecutor()) {

        Future<String> userF    = exec.submit(() -> fetchUser(userId));
        Future<String> ordersF  = exec.submit(() -> fetchOrders(userId));
        Future<String> recoF    = exec.submit(() -> fetchRecommendations(userId));

        // get() 看起来是阻塞等待，但当前虚拟线程挂起时不消耗 OS 线程
        return new DashboardData(userF.get(), ordersF.get(), recoF.get());
    }
}
```

注意几件事：

- `fetchUser`、`fetchOrders`、`fetchRecommendations` 是**普通方法，没有任何异步标记**。你可以在任何地方调用它们，不需要调用方知道它们内部有 I/O。
- `http.send()` 是**阻塞调用**，但 JVM 在底层用 `io_uring`/`epoll` 实现了"阻塞虚拟线程，释放 OS 线程"。程序员看到的是阻塞，实际上是非阻塞。
- `Future.get()` 也是阻塞调用，同理。
- 整个代码里**没有一个 `async`、`await`、`callback`、`Promise`**。

如果服务器用虚拟线程处理 HTTP 请求（Spring Boot 3.2+ 默认开启），每个进来的请求就是一个虚拟线程，`loadDashboard` 直接跑在里面，不需要任何额外改造。

#### JavaScript（async/await）

```javascript
// 同一个场景，JS 的写法
async function fetchUser(userId) {              // 必须标 async
    const res = await fetch(`/api/users/${userId}`);
    return res.json();
}

async function fetchOrders(userId) {            // 必须标 async
    const res = await fetch(`/api/orders?user=${userId}`);
    return res.json();
}

async function fetchRecommendations(userId) {   // 必须标 async
    const res = await fetch(`/api/recommendations?user=${userId}`);
    return res.json();
}

// 版本一：顺序陷阱（常见错误）
async function loadDashboard_BAD(userId) {
    const user  = await fetchUser(userId);           // ~200ms
    const orders = await fetchOrders(userId);        // 再等 ~150ms
    const reco  = await fetchRecommendations(userId); // 再等 ~180ms
    return { user, orders, reco };                   // 总计 ~530ms
}

// 版本二：正确的并行写法
async function loadDashboard(userId) {
    const [user, orders, reco] = await Promise.all([
        fetchUser(userId),           // 三个同时发出
        fetchOrders(userId),
        fetchRecommendations(userId),
    ]);
    return { user, orders, reco };   // 总计 ~200ms（最慢的那个）
}

// Express 路由
app.get('/dashboard/:id', async (req, res) => {   // 路由 handler 也必须 async
    const data = await loadDashboard(req.params.id);
    res.json(data);
});
```

#### 两种写法的本质差异

```
Java 虚拟线程模型：
  OS 线程 #1 ──► 虚拟线程 A (fetchUser)  ──► [等待 I/O，挂起]
                                               │
              ──► 虚拟线程 B (fetchOrders) ──► [等待 I/O，挂起]
                                               │
              ──► 虚拟线程 C (fetchReco)   ──► [等待 I/O，挂起]
                                               │
  I/O 返回后，OS 线程 #1（或其他 OS 线程）依次恢复三个虚拟线程

  程序员视角：写了三个"阻塞"调用，运行时自动并发

JS 事件循环模型：
  OS 线程 #1 ──► 发出 fetch(user) 请求，注册回调，继续
             ──► 发出 fetch(orders) 请求，注册回调，继续
             ──► 发出 fetch(reco) 请求，注册回调，继续
             ──► [事件循环空转，等待 I/O 事件]
             ──► I/O 事件到达，执行对应的 Promise resolve 回调

  程序员视角：必须用 Promise.all 显式声明"这三个独立"，否则串行
```

两种模型在性能上可以达到同样效果（都是 ~200ms 完成三个并行请求），但**认知负担不同**：

- Java：运行时承担了"检测 I/O 阻塞、挂起虚拟线程、调度其他任务"的责任，程序员写阻塞代码
- JS：程序员承担了"分析哪些操作独立、手动用 Promise.all 并行"的责任，写错就变串行

**函数着色带来的另一个隐性代价**：JS 代码里，`fetchUser` 是 async 函数。如果你在非 async 环境里（比如某个工具函数、类的构造函数）想调用它，你无法直接调用——必须改造调用方。Java 的 `fetchUser` 没有这个约束，任何地方都能调。

#### 一个 Java 虚拟线程更能体现优势的场景：大批量并发

```java
// Java：同时处理 1000 个用户的仪表盘请求
List<String> userIds = loadBatchUserIds(); // 1000 个 userId

try (ExecutorService exec = Executors.newVirtualThreadPerTaskExecutor()) {
    List<Future<DashboardData>> futures = userIds.stream()
        .map(id -> exec.submit(() -> loadDashboard(id)))
        .toList();

    List<DashboardData> results = futures.stream()
        .map(f -> {
            try { return f.get(); }
            catch (Exception e) { throw new RuntimeException(e); }
        })
        .toList();
}
// 1000 个虚拟线程，每个都写的是阻塞代码，内存总开销约几十 MB
```

```javascript
// JS：同一场景
const userIds = await loadBatchUserIds();

// Promise.all 1000 个请求——需要手动管理并发数，否则可能打爆连接池
const results = await Promise.all(
    userIds.map(id => loadDashboard(id))  // 1000 个并发 fetch
);

// 实际生产中通常要加并发限制
import pLimit from 'p-limit';
const limit = pLimit(50);  // 最多 50 个并发

const results = await Promise.all(
    userIds.map(id => limit(() => loadDashboard(id)))
);
```

Java 版本里，虚拟线程的调度由运行时自动处理，JVM 内部会根据底层线程池控制实际并发。JS 版本需要程序员自己引入 `p-limit` 这类库来控制并发数，且必须理解为什么需要这样做。

**这是虚拟线程路线最大的工程优势**：把并发管理的复杂度从应用层推进了运行时层，让程序员专注于"做什么"，不必操心"怎么调度"。

---

*附录补充日期：2026-05-06*
