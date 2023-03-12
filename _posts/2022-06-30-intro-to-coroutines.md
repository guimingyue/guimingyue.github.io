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
1   

## 协程的分类
根据协程的实现方式，可以将协程分为有栈协程（stackful coroutine）和无栈协程（stackless coroutine）。有栈协程的代表就是 Go 语言的 goroutine，Java 语言的协程（Virtual Thread）也是有栈协程。而 C++，C# 和 Kotlin 等语言则实现的是无栈协程，当然 C++ 语言也有许多有栈协程的实现。

### Go 的协程
在 Go 语言中使用协程（goroutine）非常简单，如下所示，下载一个 url 指定的网页数据，通过 `go` 关键字就能启动一个协程执行相应的函数`fetchUrl`，由于`fetchUrl`是在一个函数中执行的，所以函数中执行`Sleep`时只是协程会暂停，而线程则会继续执行其他可执行的计算，比如执行另一个协程。

```Go
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

```Kotlin
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

```Java

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

一个线程在暂停调度时，操作系统会保存线程执行的“现场”，当再次调度到该线程执行时则会恢复“现场”，所以协程也一样，暂停和恢复协程的执行也需要保存和恢复“现场”，这个过程就可以分为有栈的实现和非栈的实现。


## 总结
本文首先从计算机资源利用率的角度分析了多线程的存在的问题，异步编程可以解决多线程的问题，但是部分异步编程也会存在代码难以理解和排查不便的问题。协程也是异步编程的一种形态，但是语言自带的协程能解决这类问题。
## Reference

* [JEP 436: Virtual Threads (Second Preview)](https://openjdk.org/jeps/436)
* [Kotlin Coroutines: Design and Implementation](https://dl.acm.org/doi/pdf/10.1145/3486607.3486751)
