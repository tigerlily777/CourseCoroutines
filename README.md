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




