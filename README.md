# [《Kotlin 协程完全教程》](https://www.youtube.com/watch?v=FE1XVkvFuvQ&t=711s&ab_channel=%E6%89%94%E7%89%A9%E7%BA%BF)配套源码

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


