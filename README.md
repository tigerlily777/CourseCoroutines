# [《Kotlin 协程完全教程》](https://www.youtube.com/watch?v=FE1XVkvFuvQ&t=711s&ab_channel=%E6%89%94%E7%89%A9%E7%BA%BF)配套源码

##1. Basic
### 1.1_launch
### 1.2 suspend
关注协程和线程的脱离。
suspend标记的方法讲在协程中执行，执行完毕之后返回到之前的线程中。
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

