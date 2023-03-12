---
layout: post
title: 协程简介
category: Coroutine
---

最近几年，协程渐渐成为各种编程语言的一个重要的特性，Go 语言在早期就提供了其协程实现 Goroutine，C++ 20 也定义了其协程的标准，Java 语言也终于在 Java 19 中推出了其协程的实现 Virtual Thread。

## 多线程的困境
在多核时代，多线程技术是提高计算机利用率的主要手段，当一个线程某些原因（比如 IO )而阻塞时，线程就无法继续运行，那么操作系统会调度其他线程运行，这样 CPU 就会保持运行。但是对于操作系统来说，线程切换是有成本的，如果频繁的发生线程切换，那也是一笔不小的开销。另一方面，操作系统在分配线程时，也需要为线程分配其内存栈，也有一定的内存开销，所以一个操作系统能够创建的线程数量也是有限的。
考虑一种场景，针对每个请求，假设接收到请求以后，需要等待 1 秒再进行处理，那么使用多线程的一个请求一个线程的处理方式可能就是这样的。

```java
void handle(Request request) {
    Thread.sleep(1000);
    doSomework(request);
}
```
这样处理会让处理的线程在 sleep 的时候让出 CPU，但是线程确还是一直存在的，占用着资源。
对于多数 IO 密集型的应用（比如 Web 应用服务器Tomcat），一般采用一个线程对应一个请求的方式，但是对于 IO 密集型的应用，大多数时间都在阻塞等待 IO 事件的完成，而在等待过程中，线程却一直被占用着，有限的线程没有发挥出其应有的能力，而线程数过多也带来了一笔不小的 CPU 切换开销。

## 异步编程
为了有效的利用资源，可以将一个线程处理一个请求的处理方式改为异步处理。针对上面的请求处理使用异步的方式可以这样来实现。

```java
 ScheduledExecutorService scheduledExecutorService = new ScheduledThreadPoolExecutor(Runtime.getRuntime().availableProcessors());

 void handle(Request request) {
    scheduledExecutorService.schedule(() -> {
            doSomework(request);
        }, 1000, TimeUnit.MILLISECONDS);
}     
```
这样就实现了在另外一个异步处理请求，只是换了一个线程处理该请求。再看另一个异步处理的场景是：假设一个请求会有多次 IO 操作，如果几次 IO 操作都进行同步处理，那么 IO 处理的总时间为这几次 IO 时间的总和，但是如果能将这几次 IO 处理并行化，那 IO 处理的总时间则为 IO 处理中最长时间的一次 IO 时间，对于请求处理线程，在 IO 的时候，还可以处理不依赖 IO 结果的操作，这将大大减少这次请求处理的总时间，如下面一段代码所示。

```java
Future<Response1> f1 = ioRequest1();
Future<Response1> f2 = ioRequest1();

execSome();

computeResult(f1.get(), f2.get());
```

异步编程可以改善资源利用率，但是也会存在代码不容易理解，部分异步模型会导致排查问题不方便，所以需要探索更合适的异步编程方案。

## 协程
协程也是一种异步编程的方式，一般来说它是指可以一段暂停和恢复的函数。一般来说，函数只有从开始执行和执行结束，中间是无法暂停的，那么为什么要
暂停函数呢？为了提高计算机的资源利用率。从前文可以知道，线程会有切换开销和内存占用的开销，那如果线程不是由操作系统创建的，而是在用户态根据需求创建并且调度的，那就可以解决使用线程的问题。从这个角度来讲，协程就是用户态的线程，它在用户态创建，运行在线程之上，由编程语言的运行时调度器进行调度。当函数（协程）执行到一段 IO 这样耗时的操作时，就可以让出线程资源，等待 IO 完成后继续被调度到。

协程还是会由操作系统的线程来执行，而线程则是在处理器上执行的，所以协程，线程和处理器的关系可以用下图表示。
![coroutine-thread-processor](/images/coroutine/processor-thread-coroutine.drawio.png)

每个线程在同一时刻只会调度一个协程执行，协程间的切换在用户态执行，无需操作系统参与，运行开销比较小。如果某个正在执行中的协程遇到 IO 事件，那么就暂停该协程的执行，再调度其他待执行的协程，等 IO 事件完成，就再等待调度。这样就实现了协程的暂停和恢复。

协程的优势是足够的轻量，比如下面的这段代码使用协程（`vt`，Virtual Thread）可以正常执行，而使用线程（`pt`, Platform Thread），就会出现 `OutOfMemoryError` 错误。
```java
public static void main(String[] args) throws ExecutionException, InterruptedException {
    // 虚拟线程，也就是协程
    ExecutorService vt = Executors.newVirtualThreadPerTaskExecutor();
    // 平台线程，也就是操作系统的线程
    ExecutorService pt = Executors.newThreadPerTaskExecutor(Thread::new);
    for (int i = 0; i < 100_000; i++) {
        vt.submit(() -> {
            try {
                Thread.sleep(5000L);
            } catch (InterruptedException e) {
                throw new RuntimeException(e);
            }
            System.out.print(".");
        });
    }
}
```

## 协程的分类
如何在支持协程的编程语言中使用协程呢？那么先看看如何基于协程从下载一个网页数据，Go，Kotlin 和 Java 语言分别如下。

### Go 的协程
在 Go 语言中使用协程（goroutine）非常简单，如下所示，下载一个 url 指定的网页数据，通过 `go` 关键字就能启动一个协程执行相应的函数`fetchUrl`，由于`fetchUrl`是在一个函数中执行的，所以函数中执行`Sleep`时只是协程会暂停，而线程则会继续执行其他可执行的计算，比如执行另一个协程。

```go
func handle(request string) {
    ch := make(chan []byte, 1)
    // 启动一个协程
    go fetchUrl(request, ch)
    return <- ch
}

func fetchUrl(url string, ch chan []byte) {
    time.Sleep(time.Second)
    resp, _ := http.Get(url)
    defer resp.Body.close()
    d, _ := ioUtil.ReadAll(resp.Body)
    ch <- d
} 
```

### Kotlin 的协程
Kotlin 语言的协程与 Go 语言不一样，实现与上述同样的功能代码如下。

```kotlin
suspend fun handle(url: String): ByteArray = coroutineScope { 
    val deferred: Deferred<ByteArray> = async(Dispatchers.IO) {
        delay(1000)
        fetchUrl(url)
    }
    deferred.await()
}

fun fetchUrl(url: String): ByteArray {
    with(URL(url).openConnection() as HttpURLConnection) {
        return inputStream.readBytes()
    }
}
```
### Java 的协程
Java 语言在 Java 19 版本中发布了协程的第一个预览版本，实现上述的功能代码如下。Java 语言的协程的命名为虚拟线程（Virtual Thread），在使用时，它就是一个`java.lang.Thread`的子类`java.lang.VirtualThread`实例对象。

```java

// 创建协程的 ExecutorService
ExecutorService executorService = Executors.newVirtualThreadPerTaskExecutor();
    
byte[] handle(String url) throws ExecutionException, InterruptedException {
    Future<byte[]> future = executorService.submit(() -> {
        try {
            TimeUnit.SECONDS.sleep(1);
            return fetchURL(url);
        } catch (Exception e) {
            throw new RuntimeException(e);
        }
    });
    return future.get();
}

byte[] fetchURL(String url) throws IOException {
    URL u = new URL(url);
    try (var in = u.openStream()) {
        return in.readAllBytes();
    }
}
```
### 有栈协程与无栈协程
从上面 Go，Kotlin 和 Java 这几种语言的协程使用上来看 Java 和 Go 比较像，只是 Go 语言的协程使用更简洁，而 Kotlin 则通过 `suspend`，`asyn` 等关键字来使用协程，这两种不同的形式到底有什么区别呢？
主要是编程语言实现协程的实现方式有区别，协程的实现方式可以分为有栈协程（stackful coroutine）和无栈协程（stackless coroutine）。有栈协程的代表就是 Go 语言的 goroutine，Java 语言的协程（Virtual Thread）也是有栈协程。而 Kotlin，C++，C# 和 等语言则实现的是无栈协程，当然 C++ 语言也有许多有栈协程的实现（比如 libeasy）。
协程本质上是一个可暂停执行的函数，也就是说线程在执行一个函数时，可以暂停这个函数的执行，转而去执行其他的函数。正常情况下，从一个函数到另一个函数有两种方式，调用函数和从调用的函数返回，但是协程的暂停函数不是这种情况，而是在函数执行的中间暂停。那么又如何实现函数的暂停呢？

对于一个线程在暂停调度时，操作系统会保存线程执行的“现场”，当再次调度到该线程执行时则会恢复“现场”，所以协程也一样，暂停和恢复协程的执行也需要保存和恢复“现场”，这个过程的实现方式可以分为有栈的实现和无栈的实现。

#### 有栈协程的实现
有栈协程顾名思义，也就是会有一个协程自有的栈（一段堆内存）来存储其执行的状态，当协程需要暂停时，就将当前的状态保存到栈中，当协程需要恢复执行时，就使用栈中保存的状态恢复执行，但是这个过程中不会有操作系统线程的切换，而是在用户态完成的。那么什么时候需要切换协程呢？比如需要延迟当前的执行（sleep），又比如在进行 IO 时，都可以讲线程释放出来，待需要的时候就继续执行。在 Java 19 的虚拟线程的实现中，对虚拟线程进行 sleep 时，会调用到一个 native 的 `jdk.internal.vm.Continuation#doYield` 方法，这个方法的实现是一段汇编代码，各个平台的汇编代码实现可以参考 loom 项目代码中的 `gen_continuation_yield` 函数的实现。
需要注意的是，有栈协程不需要编译器的支持，只需要语言的 runtime 层面的支持。

#### 无栈协程的实现
无栈协程的实现则是由语言的编译器根据定义写成的关键字将写成编译成状态机来实现。以 Kotlin 语言为例，一个可暂停的方法如下。`suspend`来定义一个方法，标识其在协程内部使用，它可以调用其他`suspend`的方法，比如`delay`这样可以暂停协程，但是不会阻塞线程的方法。对于 Kotlin 语言，它支持在 JVM 平台上实现协程，而 JVM 平台目前还没有稳定的方式能实现方法的暂停（Java 虚拟线程在 Java 19 中出现并且是预览版本），所以 Kotlin 需要从编译器层面来实现协程。

```kotlin
suspend fun doWorld() {
    delay(1000L)
    println("World!")
}
```
编译器在编译协程代码时，会识别代码中每一个可暂停的点，对齐进行标记，假设有某个方法 N 个可暂停的点，M 个 return 语句，那么就会生成 N + M 个状态，每个状态都代表当前方法执行到的一个状态，每次暂停都会给状态进行赋值，恢复执行时会根据状态的值确定执行到哪个代码块了，比如 Kotlin 语言的协程例子中对 handle 方法反编译成 Java 后的不分代码如下。`invokeSuspend` 就是每次恢复执行时会调用到的方法。

```java
int label;

@Nullable
public final Object invokeSuspend(@NotNull Object $result) {
Object var2 = IntrinsicsKt.getCOROUTINE_SUSPENDED();
switch (this.label) {
    case 0:
        ResultKt.throwOnFailure($result);
        Continuation var10001 = (Continuation)this;
        this.label = 1;
        if (DelayKt.delay(1000L, var10001) == var2) {
            return var2;
        }
        break;
    case 1:
        ResultKt.throwOnFailure($result);
        break;
    default:
        throw new IllegalStateException("call to 'resume' before 'invoke' with coroutine");
}

return HandleKt.fetchUrl(url);
}
```
## 总结
本文首先从计算机资源利用率的角度分析了多线程的存在的问题，异步编程可以解决多线程的问题，但是部分异步编程也会存在代码难以理解和排查不便的问题。协程也是异步编程的一种形态，它能很好的解决异步编程存在的问题。根据协程的实现方式的不同，可以分为有栈协程和无栈协程。
## Reference

* [JEP 436: Virtual Threads (Second Preview)](https://openjdk.org/jeps/436)
* [Kotlin Coroutines: Design and Implementation](https://dl.acm.org/doi/pdf/10.1145/3486607.3486751)
* [Coroutines | Kotlin Documentation](https://kotlinlang.org/docs/coroutines-overview.html)
