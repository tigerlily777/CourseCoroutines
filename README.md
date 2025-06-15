# [《Kotlin 协程完全教程》](https://www.youtube.com/watch?v=FE1XVkvFuvQ&t=711s&ab_channel=%E6%89%94%E7%89%A9%E7%BA%BF)源码 + notes

##1. Basic
### 1.1_launch
### 1.2 suspend
关注协程和线程的脱离。
suspend标记的方法will be running on a coroutine，执行完毕之后返回到之前的线程中。
### 1.3 coroutine in Android project
典型用法：
```kotlin
private fun coroutinesStyle() = CoroutineScope(Dispatchers.Main).launch {
    val contributors = gitHub.contributors("square", "retrofit")
    showContributors(contributors)
  }
```
coroutineScope ->

viewModelScope ->

lifecycleScope -> bind to activity or fragment lifecycle. all the coroutine will be cancelled in ```onDestroy()```

### 1.4 withContext
```kotlin
  override fun onCreate(savedInstanceState: Bundle?) {
    super.onCreate(savedInstanceState)
    setContentView(R.layout.layout_1)
    infoTextView = findViewById(R.id.infoTextView)


    CoroutineScope(Dispatchers.Main).launch {
      val data = withContext(Dispatchers.IO) {
        // 网络代码
        "data"
      }
      val processedData = withContext(Dispatchers.Default) {
        // 处理数据
        "processed $data"
      }
      println("Processed data: $processedData")
    }
  }
```
第二段会等待上一段执行完再执行


### 1.5 custom suspend
### 1.6 withContext pt2


## 杠精式问答
### Q1.什么叫从当前的线程中挂起?
🧠 一句话解释：

“从当前线程中挂起”就是：
➤ 暂时中止这个 coroutine 的执行，释放当前线程给别人用，等条件成熟了再回来继续执行。

⸻
🎭 类比一下：

想象你是个打工人（coroutine），在公司唯一的会议室（线程）开会。
	•	普通函数 call：你进去就霸着会议室不走，别人都得等你讲完；
	•	挂起函数 call：你说：“哦，这一段我等老板批准才能继续，那我先出去吃个饭，会议室给别人用吧！”
	•	老板批准之后你再回来，从你中断的位置继续讲下去。

⸻

🔍 技术角度解释：

在协程中，如果你调用了一个挂起函数（比如 delay()、withContext()、await()），这个 coroutine 会：
	1.	暂停自己
	2.	当前线程不再被你占用
	3.	调度器会安排别的 coroutine 用这条线程
	4.	当条件满足（比如 delay 时间到了，或者网络结果返回了）时：
	5.	你被恢复继续执行，从挂起的位置继续往下走
```
CoroutineScope(Dispatchers.Default).launch {
    println("A - ${Thread.currentThread().name}") // 打印当前线程名
    delay(1000) // 这里挂起，当前线程可以去干别的事了
    println("B - ${Thread.currentThread().name}") // 可能恢复在同一个线程，也可能不是
}
```
output may be:
```
A - DefaultDispatcher-worker-1
...等1秒...
B - DefaultDispatcher-worker-2 (有可能)
```
说明你这个 coroutine 中途挂起了，而且恢复执行时可能换了个线程！

### Q2.如果我把io类型的任务发到default中会怎么样
✅ 答案是：可以运行，不会崩，但不是最优！

🧠 为什么？

1. Dispatchers.Default 的特点：
	•	固定大小线程池，线程数 ≈ CPU 核心数（比如手机是 8 核，就有 8 个线程）
	•	设计目标是跑 CPU 密集型任务：计算、压缩、加密、业务逻辑等等

2. I/O 操作的特点：
	•	比如网络请求、数据库读写、文件读写，大多数时间都在等待外部响应
	•	而不是在「计算」，所以对 CPU 压力小，但会卡住线程
💣 把 I/O 操作放在 Dispatchers.Default 会有什么后果？

👉 会让有限的线程池资源被卡住

举个栗子🌰：
```
CoroutineScope(Dispatchers.Default).launch {
    val result = slowNetworkRequest() // 这里如果用的是阻塞式 HTTP 请求
    println("Result: $result")
}
```
如果你启动了 100 个这样的任务：
	•	它们都在 DefaultDispatcher 上运行
	•	如果你使用的是 阻塞 I/O 操作（例如 Retrofit 的同步请求、File API 等）
	•	那你这几个 precious CPU 核心线程会被挡住了……别的 coroutine 就没得跑了！

💥 这会影响并发性能，让 app 变卡！

Dispatchers.IO 的特点：
	•	线程池可以自动扩容（最大64个线程）：
	•	适合「等待型」任务，多任务互不阻塞
64？
是因为 Kotlin 团队设了一个 安全的默认上限，来平衡：
	•	系统资源消耗
	•	并发性能
	•	阻塞 I/O 的容忍度
🧠 深入解释一下：

🔧 1. Dispatchers.IO 的本质是什么？

它是 Kotlin 协程为阻塞式 I/O 操作（网络请求、文件操作、数据库等）准备的调度器。
这个线程池：
	•	是共享的后台线程池
	•	可以动态扩容线程数（不同于 Default）
	•	但有最大限制：64

⸻

🧱 2. 为什么设上限，而不是无限扩？

❗因为无限扩 = 系统资源爆炸，内存用光，线程调度器崩溃！

线程不是免费的，每个线程都要：
	•	栈内存（通常 1MB）
	•	上下文切换开销
	•	OS调度资源

如果你启动了上千个阻塞线程，那可比 coroutine 重多了，会卡到地狱。

### Q3. 如果我同时开启了65个coroutine发到io去做，也会只有64个最多同时进行？
✅ 正确！你说得完全对：

如果你同时启动了 65 个 coroutine，全部 launch(Dispatchers.IO)，
那么最多只有 64 个 coroutine 会被同时调度到线程中执行，
剩下的就会 排队等资源释放。

🧠 发生了什么？

你启动 65 个 coroutine：
```
repeat(65) {
    CoroutineScope(Dispatchers.IO).launch {
        println("Start $it")
        Thread.sleep(5000) // 👈 假设是阻塞型 I/O 操作
        println("End $it")
    }
}
```
这些 coroutine 都会被送去 Dispatchers.IO 的线程池里。

🔗 然而 Dispatchers.IO 最多 只能并发运行 64 个任务（默认设置）。
	•	前 64 个：立刻被线程池调度，开始执行
	•	第 65 个：💤 会被挂起等待前面有线程空出来

⸻

🧪 想验证这个现象？

你可以用这种方式测试它：
```
val runningJobs = mutableListOf<Job>()

repeat(100) {
    val job = CoroutineScope(Dispatchers.IO).launch {
        println("[$it] Start at ${System.currentTimeMillis()}")
        Thread.sleep(5000)
        println("[$it] End at ${System.currentTimeMillis()}")
    }
    runningJobs.add(job)
}

runBlocking {
    runningJobs.forEach { it.join() }
}
```
你会观察到：
	•	前 64 个是同时执行的（几乎同一时刻开始）
	•	后面的都是批次执行的（等前面空了再开始）
⚠️ 注意事项！

这个限制只对 阻塞任务 有明显影响！

如果你是使用的 非阻塞挂起函数（例如 delay() 或 Retrofit 的 suspend 调用）：
```
launch(Dispatchers.IO) {
    delay(5000) // 非阻塞，不会卡线程
}
```

那就不受线程池限制！因为挂起时它会让出线程，线程是空出来的，别人可以用！

### Q4. 那理论上，如果我的每个coroutine都用了suspend fun来包裹耗时操作，那我能不能同时运行一万或者十万个coroutine在IO线程？ 有上限吗
🎯 短答：

理论上 YES，可以同时运行一万、十万个 coroutine（甚至更多）在 Dispatchers.IO 上，
只要你用的都是非阻塞的挂起函数（suspend fun）！🌈
但并发 coroutine 的数量也不是完全无限，系统资源最终还是有上限。

🧠 原理揭秘：

✅ 挂起函数 suspend fun 的神奇之处：

当你调用一个真正“挂起”的函数（例如 delay()、suspend Retrofit call()）时：
	•	它不会阻塞线程
	•	coroutine 会被挂起，当前线程被释放，线程可以去执行别的 coroutine
	•	所以即使你启动十万个 coroutine，它们并不会十万个线程！

这就是 Kotlin 协程最强大的魔法之一：轻量级线程（fiber-like）

💡 举个例子：十万个 coroutine delay
``` fun main() = runBlocking {
    repeat(100_000) {
        launch(Dispatchers.IO) {
            delay(1000) // 👈 非阻塞挂起
        }
    }
    println("All coroutines launched")
}
```
1 秒后，全部完成✅
⚠️ 但注意：不是所有 suspend fun 都是真挂起！
	•	如果你写的 suspend fun 里面偷偷用了 阻塞 API（比如 Thread.sleep()，或 Retrofit 的同步调用），那就 GG 了，会卡线程！
 所以重点不是你有没有写 suspend，而是你用了什么操作！
### 💥 那到底有没有 coroutine 数量上限？有，但非常高！几个因素：
 | 限制来源         | 描述                                                                 |
|:------------------|:----------------------------------------------------------------------|
| 📦 内存           | 每个 coroutine 大约消耗几 KB 内存（包含栈帧、状态机等）               |
| 🔢 调度器开销     | coroutine 太多会增加调度队列长度，导致调度延迟                        |
| 💻 系统能力       | 受平台影响，比如 Android 和 JVM 后端服务器的调度能力不同              |
| 🌪 IO Dispatcher  | 虽然 coroutine 数量理论无限，但底层线程池有限，线程忙不过来时会排队挂起 |

一般在 10 万 coroutine 级别系统才开始感到压力。🚀

### Q:🧠 如何鉴别一个 suspend fun 是否“真挂起”？

✅ 1. 看内部用的是不是：
	•	delay() ✅
	•	withContext() ✅
	•	suspend fun 接口本身 ✅
	•	Flow ✅
	•	suspendCoroutine {} ✅

❌ 2. 如果你看到：
	•	Thread.sleep() ❌
	•	原生 Java 的 I/O 操作（Socket、File）❌
	•	同步数据库、同步 HTTP ❌

那它就是阻塞的！

### Q6:如果我发送一个网络请求， fun getData，但是我不标注成suspend function，当我运行的时候，我也在coroutineScope里去叫了getData，会发生什么？和我用suspend 包裹起来再叫，有神马区别
🎯 先来直接回答：

如果你不标注成 suspend，但在 coroutineScope 里调用 getData()，
那这个函数就会 直接在当前线程里同步执行 ——
⚠️ 不会挂起，也不能让出线程！

🔬 举个实际例子分析：

🧪 1. 同步版本（不是 suspend）
```kotlin
fun getData(): String {
    // 模拟网络请求（阻塞）
    Thread.sleep(3000) 
    return "Hello"
}

CoroutineScope(Dispatchers.IO).launch {
    val result = getData()
    println(result)
}
```
会发生什么？
	•	getData() 在 IO 线程中直接执行
	•	Thread.sleep() 把线程卡住了 3 秒
	•	在这 3 秒内，这个线程啥也不能干，别的 coroutine 也不能调度上来
	•	你只是把阻塞的任务塞进了协程里，本质没有任何异步化！❌

 ✅ 2. 正确写法（加 suspend）
 ```kotlin
suspend fun getData(): String {
    delay(3000) // 👈 假设使用 Retrofit + suspend 或 delay
    return "Hello"
}

CoroutineScope(Dispatchers.IO).launch {
    val result = getData()
    println(result)
}
```
这才是真正的挂起操作：
	•	调用 getData() 时，coroutine 会挂起，释放线程
	•	等待 3 秒后继续执行，但线程早就被还给调度器了，其他 coroutine 可以继续使用这个线程
	•	🌈 高效！不卡！并发友好！

 💣 捅破天的关键点：

launch { getData() } 本身不能帮你“自动挂起同步代码”！
就算你在 coroutine 里调用 getData()，
只要 getData() 不是 suspend 的，它就完全同步执行，直接占线程！

⸻

🧠 举个生活类比理解：

想象 coroutine 是一个「图书馆座位调度系统」：
	•	suspend 函数会在任务等待时「让出座位」给其他人
	•	非-suspend 函数一屁股坐下死都不起来（比如 Thread.sleep()）

你在 coroutine 里喊一个不肯让座的家伙（非-suspend函数）干事，结果就是：
别人全排队，这人独占资源，线程效率低爆了！

### Q: 这个例子里面，getData的时候 这个协程会从IO线程上suspend，等到getData拿到结果的时候再回来继续执行对吧，那getData发生的过程中，数据一点点传输回来，这件事在哪发生的？ 还有println(result) 这句也是在IO线程上执行的吗
```
CoroutineScope(Dispatchers.IO).launch {
    val result = getData()
    println(result)
}
```
A: 🎬 第一步：CoroutineScope(Dispatchers.IO).launch
	•	新建了一个 coroutine，调度器选的是 Dispatchers.IO
	•	所以这整个协程最初会被调度到 IO 线程池 的某个线程上执行

⸻

🛫 第二步：执行到 getData()（suspend function）

如果 getData() 是一个真正挂起的 suspend 网络请求，比如 Retrofit + Kotlin Coroutine Adapter，那事情是这样发生的：

⚠️ 在调用 getData() 时，协程会「挂起」，释放当前线程。

然后：
	1.	Retrofit 会发起一个 HTTP 请求（通常底层由 OkHttp 处理）
	2.	OkHttp 会在 自己内部的线程池（跟 coroutine 无关！） 中进行网络通信，包括连接服务器、读取字节流、写入 buffer、监听 socket 等
	3.	数据是一点点地通过 OkHttp 的 socket 线程 接收的，而不是 coroutine 的线程

✅ 所以：数据接收、解析这些底层活，根本不在 coroutine 的线程中完成！

⸻

💾 第三步：收到结果后 resume coroutine

当网络响应完成后，OkHttp + CoroutineAdapter 会：
	•	在某个调度器（通常是原来的 IO dispatcher）上 resume coroutine
	•	也就是，协程会「回到 Dispatchers.IO 上的某个线程」继续执行下一行代码

 🖨️ 第四步：println(result) 是在哪个线程执行的？

✅ 答案是：

仍然在 Dispatchers.IO 的某个线程上！

协程 resume 后的代码，会继续在它当初挂起前所在的 dispatcher（IO）上执行，除非你手动 withContext() 切换了 dispatcher！

🧪 实验验证一下！

来个简单代码片段：
```
suspend fun getData(): String {
    println("getData() running on ${Thread.currentThread().name}")
    delay(1000) // 模拟挂起的 IO 操作
    return "Fetched Data"
}

fun main() = runBlocking {
    CoroutineScope(Dispatchers.IO).launch {
        val result = getData()
        println("Result: $result on ${Thread.currentThread().name}")
    }.join()
}
```
output:
```
getData() running on DefaultDispatcher-worker-3
Result: Fetched Data on DefaultDispatcher-worker-3
```
🧠 表示：
	• 启动和恢复执行都在 IO dispatcher 上的线程！
	• 真正的数据传输并不在这个线程，是 OkHttp 内部搞的！

### Q: 比如我有个网络请求suspend fun A(), 🎯 问题核心：如果 suspend fun A()（网络请求）执行时会 从当前线程挂起，真正的数据传输是由 OkHttp 自己的线程池完成，那我还要用 Dispatchers.IO 干嘛？是不是没意义？
✅ 简短回答：

虽然 挂起点释放了 IO dispatcher 的线程，
但使用 Dispatchers.IO 仍然非常重要，因为整个函数的同步逻辑、调用链、恢复 resume 都在这个 dispatcher 里运行。

⸻
🧵 1. suspend fun A() 做了什么？

比如这样：
```kotlin
suspend fun fetchData(): String {
    println("Start fetch on ${Thread.currentThread().name}")
    val response = api.getData() // Retrofit + suspend
    println("Got response on ${Thread.currentThread().name}")
    return response.body()?.string() ?: ""
}
```
	• api.getData() 是个 挂起函数
	• 在挂起点之前，代码（如打印、参数准备）会在当前线程执行
	• 在挂起点之后，Coroutine 会暂停当前协程，并释放这个线程
	• 网络请求是由 OkHttp 的后台线程 去跑的
	• 当网络返回后，Coroutine 会在原 dispatcher（IO）中恢复执行
	• 继续执行 println(...)
	• 返回结果处理、UI event 发射、Log 打印等

⸻

⚠️ 如果你用了错误的 dispatcher，会怎么样？

❌ 用 Dispatchers.Main 来做请求：
```
withContext(Dispatchers.Main) {
    fetchData() // suspend 网络请求
}
```
	•	虽然 coroutine 会在挂起点暂停、释放主线程，但 resume 后还会跑回主线程！
	•	可能 resume 的逻辑非常重（比如 response.body()?.string() 会解析 JSON），会卡住主线程！导致 UI 卡顿！

⸻

✅ 用 Dispatchers.IO：
```withContext(Dispatchers.IO) {
    fetchData()
}
```
	•	挂起前的准备逻辑、挂起后的 resume 逻辑都在 IO 线程
	•	即使数据传输是 OkHttp 的事，挂起函数周围的逻辑仍然可能很重
	•	所以我们建议把这些逻辑全放在 IO dispatcher 里跑，避免污染 UI 线程或 Default dispatcher

⸻

🧠 换个角度理解：

Coroutine 调度器是为「协程自己」服务的，
网络请求线程是 OkHttp 自己的兵马。
你用哪个 dispatcher，决定的是你自己的 coroutine 上哪里开工、哪里收尾！

```
Coroutine (IO dispatcher)      OkHttp (socket threads)
      |                                 |
start |----> prepare params             |
      |                                 |
      |----> suspend fun call ----------|---> 网络传输
      |                                 |
挂起 |←--- release IO thread           |
      |                                 |
resume|----> back to IO thread         <---- 完成后 callback
      |----> 处理返回 + 数据解析
```

✅ 小结句，

虽然 suspend function 的挂起过程释放了 dispatcher 的线程，
但整个 coroutine 的上下文（包括 resume 后）仍在 dispatcher 指定的线程中完成。
所以网络请求应使用 Dispatchers.IO，避免阻塞主线程或 Default 线程池，是为了安全地执行 resume 后的后处理逻辑！

⸻
