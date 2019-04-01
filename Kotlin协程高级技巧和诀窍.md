Kotlin协程出奇地简单：仅仅让一些长期运行的操作放在launch里面，然后就好了，是这样的吧？对于简单的情况，当然如此了。但是很快，并发和并行固有的复杂性开始积累。

当你进入协程的坑时，下面内容是你需要知道的。

## 取消 + 阻塞式的任务 = 😈
没有办法绕过它：你必须在某些时候使用好Java流。 使用流的一个问题（很多😉之一）是它们阻塞当前线程。 在协程世界中这是个坏消息。 现在，如果要取消协程，则必须等待读取或写入完成后才能继续。

作为一个简单的可重现示例，假设您打开一个ServerSocket并等待1秒超时的连接：

```kotlin
runBlocking(Dispatchers.IO) {
    withTimeout(1000) {
        val socket = ServerSocket(42)
         // 我们被永远地卡在这里，直到某个东西接收。不想知道答案吗？😜
        socket.accept()
    }
}
```
应该工作吧？ 不。

现在你感觉有点像这样：😖。 那么我们该如何解决呢？

当Closeable API构建良好时，它们支持从任何线程关闭流并且将适当地失败。

> 注意：通常，来自JDK的API遵循这些最佳实践，但要注意，不是任何第三方Closeable API都遵循。 你被警告过了。

多亏suspendCancellableCoroutine函数，我们可以在取消协程时可以关闭任何流：

```kotlin
public suspend inline fun <T : Closeable?, R> T.useCancellably(
        crossinline block: (T) -> R
): R = suspendCancellableCoroutine { cont ->
    cont.invokeOnCancellation { this?.close() }
    cont.resume(use(block))
}
```
确保这适用于您正在使用的API！

现在我们的阻塞式的accept调用，被包装在useCancellably中，协程将在超时发生时失败。

```kotlin
runBlocking(Dispatchers.IO) {
    withTimeout(1000) {
        val socket = ServerSocket(42)

        // 以`SocketException: socket closed`爆破了. 耶!
        socket.useCancellably { it.accept() }
    }
}
```
成功了！

但是如果你根本不支持取消怎么办？ 以下是您需要注意的事项：

 - 如果你使用协程的外部类中的任何实例属性/函数，即使你取消了协程，它也会被泄露。 如果您正在清理onDestroy中的资源，这尤其重要。
**解决方法**：将协程移到ViewModel或其他非上下文类并订阅其结果。

- 确保使用Dispatchers.IO阻止运行，因为这样可以让Kotlin留出一些预期无限期等待的线程。

- 尽可能使用suspendCancellableCoroutine， 而不是suspendCoroutine。
---
## launch 和 async
由于关于这两个构建器的争论已经过时，但是我想我会再次提及他们的不同点。

#### launch 往上冒出异常
当协程崩溃时，其父协程将被取消，从而取消父协程的所有子协程。 一旦整个树中的协程取消结束，异常就会发送到当前上下文的异常处理程序。 在Android上，这意味着无论您使用的是什么调度器，您的应用*都会崩溃*。

#### async 保留它的异常
这意味着await() 显式处理所有异常，并且安置CoroutineExceptionHandler不会起作用。

#### launch “阻塞了” 父协程的作用域
虽然该函数立即返回，但在使用内建launch的所有协程以某种方式结束之后，其父作用域才会结束。 这样，如果您只是想等待那些协程完成，则无需在父协程末尾调用所有子job的join() 。

与您可能期望的不同，即使未调用await() ，外部作用域仍将等待async协程完成。

#### async 返回一个结果
这个非常简单：如果你需要协程的结果，async是你唯一的选择。 如果您不需要结果，请使用launch来创建副作用(注：函数副作用，也即返回值)。 并且只有在继续之前需要这些副作用完成时，才需要使用join()。
#### join() 和 await()
join()不会重新抛出异常，但是await()会。 但是，如果发生错误，join()会取消您的协程，这意味着不会调用挂起函数join()之后的任何代码。

---
### 异常日志
既然了解了异常的处理方式（具体取决于您使用的构建器），您仍然处于两难境地：您希望记录异常而不会崩溃（因此我们无法使用launch），但您不想手动地try/catch它们（所以我们不能使用async）。 所以这让我们......没有选择？ 谢天谢地不是这样的。

记录异常是CoroutineExceptionHandler派上用场的地方。 但首先，让我们花一点时间来了解在协程中抛出异常时实际上发生了什么：

1. 捕获异常，然后通过Continuation恢复。
2. 如果您的代码没有处理异常，并且它不是CancellationException，则通过当前的CoroutineContext请求第一个CoroutineExceptionHandler。
3. 如果未找到处理程序或它报错了，则会将异常发送到特定平台的代码。
4. 在JVM上，ServiceLoader用于定位全局处理程序。
5. 一旦调用了所有处理程序或其中一个报错了，就会调用当前线程的异常处理程序。
6. 如果当前线程没有处理异常，它会冒泡上升到线程组，最后到达默认的异常处理程序。
7. 崩溃！

考虑到这一点，我们有几个选择：

- 为每个线程安置一个处理程序，但这是不现实的。
- 安置默认处理程序，但是这样，主线程中的错误不会使您的应用程序崩溃，并且您将处于潜在的不良状态。
- [将处理程序添加为服务](https://gist.github.com/SUPERCILEX/f4b01ccf6fd4ef7ec0a85dbd59c89d6c)，当使用内建有launch的任何协程崩溃时将调用该服务（高超技巧吧）。
- 使用您自己的自定义作用域，带有附加的处理程序，而不是GlobalScope，或者将处理程序添加到您使用的每个范围，但这很烦人，并使日志记录是可选的而不是默认。

最后一个解决方案是首选，因为它灵活，同时需要最少的代码和修改。

对于应用程序范围的任务，您可以使用带有日志记录处理程序的AppScope。 对于任何其他任务，当日志记录比崩溃更适合时，您可以添加处理程序。

```kotlin
val LoggingExceptionHandler = CoroutineExceptionHandler { _, t ->
    Crashlytics.logException(t)
}
val AppScope = GlobalScope + LoggingExceptionHandler
```

```kotlin
class ViewModelBase : ViewModel(), CoroutineScope {
    override val coroutineContext = Job() + LoggingExceptionHandler

    override fun onCleared() = coroutineContext.cancel()
}
```
没那么难吧

### 结论
任何时候我们处理边缘情况，事情会很快变得混乱。 我希望这篇文章，可以帮助您了解在欠佳条件下可能遇到的各种问题，以及您可以应用哪些可能的解决方案。

快乐Kotlining！
