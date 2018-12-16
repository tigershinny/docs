[原文在此,可能需要翻墙](https://proandroiddev.com/kotlin-coroutines-patterns-anti-patterns-f9d12984c68e)

### 简介
 决定写一些东西，在我看来当你使用Kotlin coroutines时，这些东西你应该做和不应该做(或者说，至少试着避免做)。

### 处理异常时，用coroutineScope或者使用SupervisorJob包装异步调用
❌如果异步代码块可能抛出异常，不要依赖于用try/catch代码块包装

```kotlin
val job: Job = Job()
val scope = CoroutineScope(Dispatchers.Default + job)
// 可能抛出异常
fun doWork(): Deferred<String> = scope.async { ... }   // (1)
fun loadData() = scope.launch {
    try {
        doWork().await()                               // (2)
    } catch (e: Exception) { ... }
}
```
在上面例子中，doWork函数开启了新的协程(1)，这个协程可能抛出一个未处理的异常。如果你试着用try/catch块包装doWork(2)，它照样会崩溃。

**这是因为，任意job的子job的失败，会直接导致它的父job的失败**

✅避免崩溃的一种方式是使用[SupervisorJob](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/-supervisor-job.html)(1)

> 子job的失败或者取消，不会导致Supervisor job的失败，也不会影响其他子job

```kotlin
val job = SupervisorJob()                               // (1)
val scope = CoroutineScope(Dispatchers.Default + job)
// 可能抛出异常
fun doWork(): Deferred<String> = scope.async { ... }
fun loadData() = scope.launch {
    try {
        doWork().await()
    } catch (e: Exception) { ... }
}
```
注意：只有显式地用SupervisorJob在协程域上运行async，才能行得通。所以，下面的代码仍然会使得应用崩溃，因为async实在父协程的域上启动的(1)。

```kotlin
val job = SupervisorJob()                               
val scope = CoroutineScope(Dispatchers.Default + job)
fun loadData() = scope.launch {
    try {
        async {                                         // (1)
            // 可能抛出异常
        }.await()
    } catch (e: Exception) { ... }
}

```
✅ 避免崩溃的另外一种方法，也是更可取的方式，是通过用coroutineScope包裹async(1)。现在，当异常发生在async内部时，它将会取消这个域内创建的所有其他协程，而没有影响外面的域(2)。

```kotlin
val job = SupervisorJob()                               
val scope = CoroutineScope(Dispatchers.Default + job)
//可能抛出异常
fun doWork(): Deferred<String> = coroutineScope {     // (1)
    async { ... }
}
fun loadData() = scope.launch {                       // (2)
    try {
        doWork().await()
    } catch (e: Exception) { ... }
}
```
或者，你可以在async块内部处理异常。

### 主协程最好选择Main调度器

❌ 如果你需要做后台的工作而后在主协程中更新UI，不要在非Main调度器中启动它。

```kotlin
val scope = CoroutineScope(Dispatchers.Default)          // (1)
fun login() = scope.launch {
    withContext(Dispatcher.Main) { view.showLoading() }  // (2)  
    networkClient.login(...)
    withContext(Dispatcher.Main) { view.hideLoading() }  // (2)
}
```
在上面的例子红，我们使用Default调度器的域启动了主协程(1)。使用这个方式，当每次我们接触用户界面，我们不得不切换上下文(2)。

✅ 大多数情况下，使用Main调度器创建域是更可取的，这会使得代码更加简洁和更少的上下文显式切换。

```kotlin
val scope = CoroutineScope(Dispatchers.Main)
fun login() = scope.launch {
    view.showLoading()    
    withContext(Dispatcher.IO) { networkClient.login(...) }
    view.hideLoading()
}
```
### 避免使用多余的async/await

❌如果你使用async函数，然后随着直接使用await，那么你应该避免这么做。
```kotlin
launch {
    val data = async(Dispatchers.Default) { /* 代码 */ }.await()
}
```
✅ 如果你想要切换协程上下文，而且挂起父协程，那么withContext是更好的选择。

```kotlin
launch {
    val data = withContext(Dispatchers.Default) { /* code */ }
}
```
性能虽然不是一个太大的关注点(虽然async创建了新协程来做这些工作)，但是语义上async暗示着，你想要在后台开启多个协程，然后在它们上await。

### 避免取消域的job

❌ 如果你需要取消协程，首先不要取消域的job。

```kotlin
class WorkManager {
    val job = SupervisorJob()
    val scope = CoroutineScope(Dispatchers.Default + job)
    fun doWork1() {
        scope.launch { /* 做一些事情 */ }
    }
    fun doWork2() {
        scope.launch { /* 做一些事情 */ }
    }
    fun cancelAllWork() {
        job.cancel()
    }
}
fun main() {
    val workManager = WorkManager()
    workManager.doWork1()
    workManager.doWork2()
    workManager.cancelAllWork()
    workManager.doWork1() // (1)
}
```
上面代码的问题在于，当我们取消job时，我们把它放入到completed的状态。在completed的job的域中启动的协程，不会被执行(1)。

✅ 当我们想要取消特定域中所有的协程，我们可以使用[cancelChildren](https://github.com/Kotlin/kotlinx.coroutines/issues/787)函数。提供取消单个job的可能，也是一个良好实践(2)。

```kotlin
class WorkManager {
    val job = SupervisorJob()
    val scope = CoroutineScope(Dispatchers.Default + job)
    fun doWork1(): Job = scope.launch { /* 做一些事情 */ } // (2)
    fun doWork2(): Job = scope.launch { /* 做一些事情 */ } // (2)
    fun cancelAllWork() {
        scope.coroutineContext.cancelChildren()         // (1)                             
    }
}
fun main() {
    val workManager = WorkManager()
    workManager.doWork1()
    workManager.doWork2()
    workManager.cancelAllWork()
    workManager.doWork1()
}
```
### 避免编写用隐式调度器的挂起函数

❌ 不要编写这样的挂起函数，它依赖于来自特定协程调度器的执行。

```kotlin
suspend fun login(): Result {
    view.showLoading()
    val result = withContext(Dispatcher.IO) {  
        someBlockingCall() 
    }
    view.hideLoading()
    return result
}
```
上面例子中，login函数是个挂起函数，当你在使用了非Main调度器的协程执行它的时候，这个函数会崩溃。

```kotlin
launch(Dispatcher.Main) {     // (1) 不崩溃
    val loginResult = login()
    ...
}
launch(Dispatcher.Default) {  // (2) 造成崩溃
    val loginResult = login()
    ...
}
```

> CalledFromWrongThreadException: 在创建了视图层级的原来线程，才能接触它的视图

✅ 要以这样一种方式设计你的挂起函数：它可以在任何协程调度器中执行。

```kotlin
suspend fun login(): Result = withContext(Dispatcher.Main) {
    view.showLoading()
    val result = withContext(Dispatcher.IO) {  
        someBlockingCall() 
    }
    view.hideLoading()
	return result
}
```
现在，你可以在任何调度器中调用login函数。

```kotlin
launch(Dispatcher.Main) {     // (1) 不会崩溃
    val loginResult = login()
    ...
}
launch(Dispatcher.Default) {  // (2) 也不会崩溃
    val loginResult = login()
    ...
}
```
### 避免使用全局域
❌ 如果你在你的Android应用中到处使用GlobalScope，你需要停止这么做了。

```kotlin
GlobalScope.launch {
    // 代码
}
```

> 全局域使用在最顶层协程中，它在整个应用的生命周期中运行，而且不能够永久地取消。
> 应用代码通常使用定义应用的[CoroutineScope](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/-coroutine-scope/index.html)，在[GlobalScope](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/-global-scope/index.md)的实例上使用[async](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/async.html)或者[launch](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/launch.html) ， 是非常不建议的。

✅Android中的协程，可以很容易使用到Activity、Fragment、View或者ViewModel的生命周期中。

```kotlin
class MainActivity : AppCompatActivity(), CoroutineScope {

    private val job = SupervisorJob()
    
    override val coroutineContext: CoroutineContext
        get() = Dispatchers.Main + job
        
    override fun onDestroy() {
        super.onDestroy()
        coroutineContext.cancelChildren()
    }
    
    fun loadData() = launch {
        // code
    }
}
```
特别感谢：Andrey Mischenko, Louis CA, DBradyn Poulsen, Tolriq, Dave A.
