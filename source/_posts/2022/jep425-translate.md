---
title: JEP-425 Virtual Threads (Preview)
categories: 翻译
tags: [JEP, Java, 翻译, 协程]
abbrlink: 910d18dc
date: 2022-04-30
---

# JEP 425: Virtual Threads (Preview)

> 提示：本文翻译的时候，文档最新版本是 2022/06/08，所以本文可能存在一定的时效性问题，请读者注意。

## 总览
将*虚拟线程*引入到 Java 平台。虚拟线程是轻量级的线程，可以大大减少编写、维护和监测高吞吐量并发应用程序的工作量。这是 [预览 API](https://openjdk.java.net/jeps/12) .

## 目标
- 使以简单的线程每请求方式编写的服务器应用程序能够以接近最佳的硬件利用率进行扩展。
- 让使用 `java.lang.Thread` API 的现有代码能够以最小的改动采用虚拟线程。
- 使用现有的 JDK 工具轻松地对虚拟线程进行故障排除、调试和分析。

## 非目标
- 我们的目标不是移除线程的传统实现，也不是默默地将现有的应用程序迁移到使用虚拟线程。
- 它的目标不是改变 Java 的基本并发模型。
- 在 Java 语言或 Java 库中提供新的数据并行结构不是我们的目标。流 API 仍然是并行处理大型数据集的首选方式。

<!-- more -->

## 动机
近三十年来，Java 开发者一直依赖线程作为并发服务器应用程序的构建基础模块。每个方法中的每条语句都在一个线程中执行，由于 Java 是多线程的，所以多个执行线程同时发生。线程是 Java 的并发单元：一段顺序代码，与其他此类单元同时运行，而且基本上是独立的。每个线程都提供一个堆栈来存储局部变量和协调方法调用，以及出错时的上下文：异常是由同一线程中的方法抛出和捕获的，因此开发人员可以使用线程的堆栈跟踪来找出发生了什么。线程也是工具的一个核心概念：调试器通过线程方法中的语句，剖析器将多个线程的行为可视化，以帮助了解其性能。

### 线程每请求的风格
服务器应用程序通常处理相互独立的并发用户请求，因此，应用程序通过在整个请求期间为该请求分配一个线程来处理请求是合理的。这种线程每请求的风格容易理解，容易编程，也容易调试和分析，因为它使用操作系统平台的最小并发单元来代表应用程序的并发单元。

服务器应用程序的可扩展性受 Little 定律的制约，该定律将延迟、并发性和吞吐量联系起来。对于一个给定的请求处理时间（即延迟），应用程序同时处理的请求数量（即并发性）必须与到达率（即吞吐量）成比例增长。例如，假设一个平均延迟为 50ms 的应用程序，通过并发处理 10 个请求，达到每秒 200 个请求的吞吐量。为了使该应用程序扩展到每秒 2000 个请求的吞吐量，它将需要同时处理 100 个请求。如果每个请求都在一个线程中处理，那么，为了使应用程序跟上，线程的数量必须随着吞吐量的增长而增长。

不幸的是，可用的线程数量是有限的，因为 JDK 将线程作为操作系统线程的封装来实现。操作系统线程的调度成本很高，所以我们不能有太多的线程，这使得该实现不适合线程每请求的方式。如果每个请求在其持续时间内消耗一个线程，也就是一个操作系统线程，那么线程的数量往往在其他资源（如 CPU 或网络连接）耗尽之前就已经成为限制因素了。JDK 目前对线程的实现将应用程序的吞吐量限制在一个远远低于硬件所能支持的水平。即使线程是池化的，这种情况也会发生，因为池化有助于避免启动新线程的高成本，但不会增加线程的总数。

### 用异步风格提高可扩展性
一些希望最大限度地利用硬件的开发者已经放弃了线程每请求风格，而选择了线程共享风格。请求处理代码不是从头到尾在一个线程上处理一个请求，而是在等待一个 I/O 操作完成时将其线程返回到一个池中，以便该线程可以为其他请求提供服务。这种细粒度的线程共享，即代码只在执行计算时保留线程，而不是在等待 I/O 时保留线程，允许大量的并发操作而不消耗大量的线程。虽然它消除了操作系统线程稀缺性对吞吐量的限制，但它的代价很高。它需要所谓的异步编程风格，采用一套独立的 I/O 方法，不等待 I/O 操作的完成，而是在以后向回调发出完成信号。如果没有专门的线程，开发者必须将他们的请求处理逻辑分解成小的阶段，通常写成 lambda 表达式，然后用 API 将它们组成一个顺序管道（例如 CompletableFuture，或所谓的"响应式"框架）。因此，他们放弃了语言的基本顺序组合操作符，如循环和 try/catch 块。

在异步风格中，一个请求的每个阶段可能在不同的线程上执行，每个线程以交错的方式运行属于不同请求的阶段。这对理解程序行为有深刻的影响。堆栈跟踪没有提供可用的上下文，调试器不能穿过请求处理逻辑，剖析器不能将一个操作的成本与它的调用者联系起来。当使用 Java 的流 API 来处理短管道中的数据时，组成 lambda 表达式是可以管理的，但当应用程序中所有的请求处理代码都必须以这种方式编写时，就会出现问题。这种编程风格与 Java 平台不一致，因为应用程序的并发单元，即异步流水线，不再是平台的并发单位了。

### 使用虚拟线程保留线程线程每请求每请求的风格
为了使应用程序能够扩展，同时与平台保持和谐，我们应该努力通过更有效地实现线程来保留线程每请求的风格，这样它们就会更多。操作系统无法更有效地实现操作系统线程，因为不同的语言和运行时以不同的方式使用线程栈。然而，Java 运行时有可能以一种切断与操作系统线程的一对一对应关系的方式来实现 Java 线程。就像操作系统通过将大量的虚拟地址空间映射到有限的物理 RAM 上，给人以内存充足的错觉一样，Java 运行时也可以通过将大量的虚拟线程映射到少量的操作系统线程上，给人以线程充足的错觉。

虚拟线程是 `java.lang.Thread` 的一个实例，它不与特定的操作系统线程相联系。相比之下，平台线程是以传统方式实现的 `java.lang.Thread` 的一个实例，作为操作系统线程的一个封装。

线程每请求风格的应用程序代码可以在请求的整个过程中在虚拟线程中运行，但虚拟线程只在 CPU 上执行计算时消耗一个操作系统线程。其结果是与异步风格相同的可扩展性，只是它是以透明方式实现的。当在虚拟线程中运行的代码调用 java.* API 中的阻塞 I/O 操作时，运行时会执行一个非阻塞的操作系统调用，并自动暂停虚拟线程，直到以后可以恢复。对 Java 开发者来说，虚拟线程只是创建成本低且几乎无限多的线程。硬件利用率接近最佳，允许高水平的并发，因此，具有很高的吞吐量，与此同时应用程序仍然与 Java 平台的多线程设计及其工具相协调。

### 虚拟线程的影响
虚拟线程开销低，数量众多，因此没必要池化使用：每个任务都应创建一个新的虚拟线程。因此，大多数虚拟线程都是短命的，而且调用栈很浅，只执行一次 HTTP 客户端调用或一次 JDBC 查询。相比之下，操作系统线程是重量级和昂贵的，因此往往必须是池化的。它们往往寿命很长，有很深的调用栈，并在许多任务之间共享使用。

总之，虚拟线程保留了与 Java 平台设计相协调的可靠的线程每请求风格，同时优化了硬件的利用。使用虚拟线程不需要学习新的概念，尽管它可能需要忘掉为应对今天的高线程成本而养成的习惯。虚拟线程不仅能帮助应用开发者，还能帮助框架设计者提供易于使用的 API，这些 API 与平台的设计兼容，同时又不影响扩展性。

## 描述
今天，JDK 中 java.lang.Thread 的每个实例都是一个平台线程。平台线程在底层操作系统线程上运行 Java 代码，并在代码的整个生命周期内占用操作系统线程。平台线程的数量受限于操作系统线程的数量。

虚拟线程是 java.lang.Thread 的一个实例，它在底层操作系统线程上运行 Java 代码，但在代码的整个生命周期中不占用操作系统线程。这意味着许多虚拟线程可以在同一个操作系统线程上运行他们的 Java 代码，有效地共享它。虽然一个平台线程垄断了宝贵的操作系统线程，但虚拟线程却没有。虚拟线程的数量可以比操作系统线程的数量大得多。

虚拟线程是一种轻量级的线程实现，由 JDK 而不是操作系统提供。它们是用户模式线程的一种形式，在其他多线程语言中也很成功（例如 Go 中的 goroutines 和 Erlang 中的 process）。用户模式线程甚至在 Java 的早期版本中作为所谓的"绿色线程"出现，当时操作系统线程还没有成熟和普及。然而，Java 的绿色线程都共享一个操作系统线程（M:1 调度），并最终被作为操作系统线程的封装器（1:1 调度）的平台线程所超越。虚拟线程采用 M: N 调度，即大量（M）虚拟线程被调度到较少数量（N）的操作系统线程上运行。

### 使用虚拟线程 VS 使用平台线程
开发人员可以选择到底是使用虚拟线程还是平台线程。下面是一个创建大量虚拟线程的示例程序。该程序首先获得一个 ExecutorService，它将为每个提交的任务创建一个新的虚拟线程。然后，它提交了 10,000 个任务，并等待它们全部完成：
```Java
try (var executor = Executors.newVirtualThreadPerTaskExecutor()) {
    IntStream.range(0, 10_000).forEach(i -> {
        executor.submit(() -> {
            Thread.sleep(Duration.ofSeconds(1));
            return i;
        });
    });
}  // executor.close() is called implicitly, and waits
```
这段示例代码里面的任务十分简单，就是休眠一秒，现在的硬件轻松能支持一万个虚拟线程并发执行这样的代码。然而在底层，JDK 仅在少量的操作系统线程上运行代码，也许就一个。

如果这个程序使用 ExecutorService，为每个任务创建一个新的平台线程，例如 `Executors.newCachedThreadPool()`，情况就会大不相同。ExecutorService 将试图创建 10,000 个平台线程，这会导致创建 10,000 个操作系统线程，而程序可能会崩溃，这取决于机器和操作系统。

如果程序使用 ExecutorService，从一个线程池中获取平台线程，例如 `Executors.newFixedThreadPool(200)`，情况也不并不会好多少。ExecutorService 将创建 200 个平台线程，由所有 10,000 个任务共享，因此许多任务将按顺序运行，而不是并发运行，程序将需要很长时间才能完成。对于这个程序，有 200 个平台线程的池子只能达到每秒 200 个任务的吞吐量，而虚拟线程的吞吐量约为每秒 10000 个任务（经过充分预热）。此外，如果将示例程序中的 10_000 改为 1_000_000，那么该程序将提交 1,000,000 个任务，创建 1,000,000 个并发运行的虚拟线程，并（在充分预热后）实现大约 1,000,000 个任务/秒的吞吐量。

如果这个程序中的任务是进行一秒钟的计算（例如，对一个巨大的数组进行排序），而不仅仅是休眠，那么增加超过处理器内核的线程数量将没有帮助，无论它们是虚拟线程还是平台线程。虚拟线程不是更快的线程 - 它们运行代码的速度并不比平台线程快。它们的存在是为了提供规模（更高的吞吐量），而不是速度（更低的延时）。它们可以比平台线程多得多，所以根据 [利特尔法则](https://zh.wikipedia.org/zh/%E5%88%A9%E7%89%B9%E7%88%BE%E6%B3%95%E5%89%87) ，它们可以实现更高的并发，从而获得更高的吞吐量。

换句话说，在以下情况，虚拟线程可以显著提高应用程序的吞吐量
- 并发任务的数量很高（超过几千），并且
- 工作负载不受 CPU 限制，因为在这种情况下，拥有比处理器内核更多的线程不能提高吞吐量。

虚拟线程有助于提高典型的服务器应用程序的吞吐量，正是因为这种应用程序由大量的并发任务组成，它们的大部分时间都在等待。

> 译者注：上面这些一句话说就是虚拟线程对于 CPU 密集型任务吞吐量的改善不明显，主要是显著提升 IO 密集型并发任务的吞吐量。

一个虚拟线程可以运行任何平台线程可以运行的代码。特别是，虚拟线程支持线程局部变量和线程中断，就像平台线程一样。这意味着，现有的处理请求的 Java 代码将很容易在虚拟线程中运行。许多服务器框架会选择自动完成这一工作，为每个传入的请求启动一个新的虚拟线程，并在其中运行应用程序的业务逻辑。

下面是一个服务器应用程序的例子，它聚合了另外两个服务的结果。一个假设的服务器框架（不具体指哪个）为每个请求创建一个新的虚拟线程，并在该虚拟线程中运行应用程序的处理代码。而应用程序代码又创建了两个新的虚拟线程，通过与第一个例子相同的 ExecutorService 并发地获取资源。

```Java
void handle(Request request, Response response) {
    var url1 = ...
    var url2 = ...
 
    try (var executor = Executors.newVirtualThreadPerTaskExecutor()) {
        var future1 = executor.submit(() -> fetchURL(url1));
        var future2 = executor.submit(() -> fetchURL(url2));
        response.send(future1.get() + future2.get());
    } catch (ExecutionException | InterruptedException e) {
        response.fail(e);
    }
}
 
String fetchURL(URL url) throws IOException {
    try (var in = url.openStream()) {
        return new String(in.readAllBytes(), StandardCharsets.UTF_8);
    }
}
```

上述服务器应用程序的例子里，即使包含了直接的阻塞代码，也很容易扩展，因为它调用的是大量的虚拟线程。

`Executor.newVirtualThreadPerTaskExecutor()` 不是创建虚拟线程的唯一方法。下面讨论的新的 `java.lang.Thread.Builder` API，可以创建和启动虚拟线程。此外， [Structured Concurrency](https://openjdk.java.net/jeps/428) 提供了一个更强大的API来创建和管理虚拟线程，特别是在类似于这个服务器例子的代码中，线程之间的关系对于平台和工具来说是透明的。

### 虚拟线程是预览特性，默认处于禁用状态
上面的程序使用了 `Executors.newVirtualThreadPerTaskExecutor()` 方法，所以要在 JDK 19 上运行它们，你必须按以下方法启用预览 API：
- 用 `javac --release 19 --enable-preview Main.java` 编译程序，用 `java --enable-preview Main` 运行程序
- 当使用源代码启动器时，用 `java --source 19 --enable-preview Main.java` 运行该程序
- 当使用 jshell 时，用 `jshell --enable-preview` 启动它

### 不要池化虚拟线程
开发人员通常会将应用程序代码从传统的基于线程池的 `ExecutorService` 迁移到虚拟线程每任务的 `ExecutorService`。线程池，就像所有的资源池一样，是为了共享昂贵的资源，但虚拟线程并不昂贵，而且从来没有必要把它们放在一起。

开发人员有时会使用线程池来限制对有限资源的并发访问。例如，如果一个服务不能处理超过 20 个并发请求，那么通过提交给一个容量为 20 的线程池来执行对该服务的所有访问将确保这一点。由于平台线程的高成本使得线程池无处不在，这个模式也变得无处不在，但开发者不应该为了限制并发而特地虚拟线程池。应该使用专门为此目的而设计的结构，如 `semaphores`，来保护对有限资源的访问。这比线程池更有效、更方便，也更安全，因为不存在线程本地数据意外地从一个任务泄漏到另一个任务的风险。

### 观测虚拟线程
编写清晰的代码并不是全部的内容。对运行中的程序状态的清晰呈现对于故障诊断、维护和优化也是至关重要的，JDK 早已提供了调试、剖析和监控线程的机制。这些工具也应该为虚拟线程做同样的事情（也许要对它们的数量做一些调整），因为它们毕竟是 java.lang.Thread 的实例。

Java 调试器可以穿过虚拟线程，显示调用堆栈，并检查堆栈框架中的变量。JDK Flight Recorder（JFR）是 JDK 的低开销剖析和监控机制，它可以将应用程序代码的事件（如对象分配和 I/O 操作）与正确的虚拟线程联系起来。对于以异步风格编写的应用程序，这些工具则无能为力。在那种风格中，任务与线程之间没有联系，所以调试器不能显示或操作任务的状态，剖析器也不能知道一个任务花了多少时间来等待 I/O。

线程转储是另一个流行的工具，用于排除以线程每请求风格编写的应用程序的故障。不幸的是，JDK 的传统线程转储，即用 `jstack` 或 `jcmd` 获得的线程转储，呈现的是一个平面的线程列表。这适合于几十或几百个平台线程，但不适合于几千或几百万个虚拟线程。因此，我们不会将传统的线程转储扩展到包括虚拟线程，而是在 `jcmd` 中引入一种新的线程转储，将虚拟线程与平台线程放在一起，并以一种有意义的方式进行分组。当程序使用 [Structured Concurrency](https://openjdk.java.net/jeps/428) 时，可以显示线程之间更丰富的关系。

因为可视化和分析大量的线程可以从工具化中受益，所以 `jcmd` 除了纯文本外，还可以以 JSON 格式发起新的线程转储。

```bash
$ jcmd <pid> Thread.dump_to_file -format=json <file>
```

新的线程转储格式列出了在网络 I/O 操作中被阻塞的虚拟线程，以及由上面显示的 new-thread-per-task `ExecutorService` 创建的虚拟线程。它不包括对象地址、锁、JNI 统计、堆统计，以及其他出现在传统线程转储中的信息。此外，由于它可能需要列出大量的线程，生成一个新的线程转储并不会暂停应用程序。

下面是这样一个线程转储的例子，它取自一个与上面第二个例子类似的应用程序，在 JSON 浏览器中呈现（点击放大）:
![json 格式的线程转储](https://img.jiangchen.tech/202206121646348.png)
由于虚拟线程是在 JDK 中实现的，并不与任何特定的操作系统线程相联系，因此它们对操作系统来说是不可见的，而操作系统也不知道它们的存在。操作系统级别的监控将观察到 JDK 进程使用的操作系统线程比虚拟线程的数量少。

### 虚拟线程的调度
为了进行有用的工作，线程需要被调度，也就是说，被分配到一个处理器核心上执行。对于作为操作系统线程实现的平台线程，JDK 依赖于操作系统的调度器。相比之下，对于虚拟线程，JDK 有自己的调度器。JDK 的调度器不是直接将虚拟线程分配给处理器，而是将虚拟线程分配给平台线程（这就是前面提到的虚拟线程的 M：N 调度）。然后，平台线程由操作系统照常调度。

JDK 的虚拟线程调度器是一个基于工作窃取算法的 `ForkJoinPool`，以 FIFO 模式运行。调度器的并行性是指可用于调度虚拟线程的平台线程的数量。默认情况下，它等于可用处理器的数量，但它可以通过系统属性 `jdk.virtualThreadScheduler.parallelism` 来调整。请注意，这个 `ForkJoinPool` 与普通池不同，后者用于实现并行流，并且以后进先出的模式运行。

调度器分配给一个虚拟线程的平台线程被称为虚拟线程的载体。一个虚拟线程在其生命周期中可以被调度在不同的载体上；换句话说，调度器并不在虚拟线程和任何特定的平台线程之间保持亲和力。从 Java 代码的角度来看，一个正在运行的虚拟线程在逻辑上是独立于其当前载体的。

载体的身份对虚拟线程来说是不可用的。Thread.currentThread()返回的值始终是虚拟线程本身。

载体和虚拟线程的堆栈痕迹是分开的。在虚拟线程中抛出的异常将不包括载体的堆栈帧。线程转储不会在虚拟线程的堆栈中显示载体的堆栈帧，反之亦然。

载体的线程本地变量对于虚拟线程来说是不可用的，反之亦然。

此外，从 Java 代码的角度来看，一个虚拟线程和它的载体暂时共享一个操作系统线程的事实是看不见的。相比之下，从本地代码的角度来看，虚拟线程和其载体都运行在同一个本地线程上。因此，在同一个虚拟线程上被多次调用的本地代码在每次调用时可能会观察到不同的操作系统线程标识符。

目前，调度器并没有为虚拟线程实现时间共享。时间共享是对已经消耗了一定数量的 CPU 时间的线程的强制抢占。虽然当平台线程数量相对较少且 CPU 利用率为 100% 时，时间共享可以有效地减少一些任务的延迟，但不清楚时间共享在一百万个虚拟线程中是否会同样有效。

### 虚拟线程的运行
要利用虚拟线程的优势，没有必要重写你的程序。虚拟线程不要求或期望应用程序代码明确地将控制权交还给调度器；换言之，虚拟线程不是合作性的。用户代码不能对虚拟线程如何或何时分配给平台线程做出假设，就像它对平台线程如何或何时分配给处理器内核做出假设一样。

为了在虚拟线程中运行代码，JDK 的虚拟线程调度器通过将虚拟线程挂载在平台线程上，将虚拟线程分配到平台线程上执行。这使得平台线程成为虚拟线程的载体。之后，在运行一些代码后，虚拟线程可以从其载体上卸载。这时，平台线程是自由的，所以调度器可以在其上挂载一个不同的虚拟线程，从而使其再次成为载体。

通常情况下，当一个虚拟线程在 I/O 或 JDK 中的一些其他阻塞操作（如 BlockingQueue.take()）上阻塞时，它就会被卸载。当阻塞操作准备完成时（例如，在 socket 上已经收到了字节），它将虚拟线程提交回调度器，调度器将把虚拟线程挂载到载体上以继续执行。

虚拟线程的挂载和卸载频繁而透明地发生，并且不会阻塞任何操作系统线程。例如，前面显示的服务器应用程序包括以下一行代码，它包含对阻塞操作的调用：
```Java
response.send(future1.get() + future2.get());
```
这些操作将导致虚拟线程多次挂载和卸载，一般来说，每次调用 get() 都会有一次，在 send(...) 中执行 I/O 的过程中可能会有多次。

JDK 中的绝大多数阻塞操作都会解除对虚拟线程的挂载，从而释放其载体和底层操作系统线程，以承担新的工作。然而，JDK 中的一些阻塞操作并没有解除对虚拟线程的挂载，因此会阻塞其载体和底层 OS 线程。这是因为操作系统层面（如许多文件系统操作）或 JDK 层面（如 Object.wait()）的限制。这些阻塞操作的实现将通过暂时扩大调度器的并行性来补偿对操作系统线程的捕获。因此，调度器的 ForkJoinPool 中的平台线程数量可能会暂时超过可用处理器的数量。调度器可用的最大平台线程数可以通过系统属性 jdk.virtualThreadScheduler.maxPoolSize 来调整。

有两种情况下，虚拟线程在阻塞操作中不能被卸载，因为它在载体上自旋：
- 当它在一个同步块或方法中执行代码时
- 当它执行一个本地方法或一个外来函数时

自旋并不会使应用程序变得不正确，但它可能会阻碍其可扩展性。如果一个虚拟线程在自旋时执行了一个阻塞操作，如 I/O 或 BlockingQueue.take()，那么它的载体和底层操作系统线程在操作的过程中就会被阻塞。虚拟线程频繁的长时间自旋会因为捕获载体而损害应用程序的可扩展性。

调度器不能通过扩展其并行性来补偿自旋的情况。相反，通过修改频繁运行的同步块或方法，避免频繁和长时间的自旋，应该使用 [java.util.concurrent.locks.ReentrantLock](https://docs.oracle.com/en/java/javase/18/docs/api/java.base/java/util/concurrent/locks/ReentrantLock.html) 代替，去保护潜在的长 I/O 操作。没有必要替换那些不经常使用（例如，只在启动时执行）或守护内存操作的同步块和方法。一如既往，努力保持锁策略的简单和清晰。

新的诊断方法有助于将代码迁移到虚拟线程，以及评估是否应该用 java.util.concurrent 锁取代 synchronized 的特定使用：
- 当一个线程在阻塞自旋时，会发出一个 JDK 飞行记录器（JFR）事件（见 [JDK Flight Recorder](https://openjdk.java.net/jeps/425#JDK-Flight-Recorder-JFR) ）。
- 系统属性 jdk.tracePinnedThreads 会在线程在自旋被阻塞时触发堆栈跟踪。使用 `-Djdk.tracePinnedThreads=full ` 运行时，当一个线程在自旋被阻塞时，会打印出完整的堆栈跟踪，并突出显示本地帧和持有监视器的帧。使用 `-Djdk.tracePinnedThreads=short` 运行时，输出的结果只限于有问题的帧。

在未来的版本中，我们可能会去掉上面的第一个限制（在同步锁中自旋）。第二个限制是为了与本地代码正确互动。

### 内存占用以及对垃圾回收的影响
虚拟线程的堆栈作为堆栈块对象存储在 Java 的垃圾收集堆中。堆栈随着应用程序的运行而增长和缩小，这既是为了提高内存效率，也是为了适应任意深度的堆栈（最多到 JVM 配置的平台线程堆栈大小）。这种效率使大量的虚拟线程成为可能，从而使服务器应用程序中的线程每请求风格继续成为可能。

在上面的第二个例子中，回顾一下，一个假定的框架通过创建一个新的虚拟线程并调用 handle 方法来处理每个请求；即使它在深层调用堆栈的末端调用 handle（在认证、事务等之后），handle 本身也会产生多个虚拟线程，这些虚拟线程只执行短暂的任务。因此，对于每个具有深层调用堆栈的虚拟线程来说，会有多个具有浅层调用堆栈的虚拟线程，消耗的内存很少。

一般来说，虚拟线程所需的堆空间和垃圾收集器的活动量很难与异步代码相比。一百万个虚拟线程至少需要一百万个对象，但一百万个任务共享一个平台线程池也是如此。此外，处理请求的应用程序代码通常在 I/O 操作中持有数据。请求每线程风格的代码可以将这些数据保存在本地变量中，这些变量存储在堆中的虚拟线程栈中，而异步代码必须将相同的数据保存在堆对象中，这些对象从流水线的一个阶段传递到下一个阶段。一方面，虚拟线程所需的堆栈框架布局比紧凑对象的布局更浪费；另一方面，虚拟线程可以在许多情况下改变和重用它们的堆栈（取决于底层的 GC 交互），而异步管道总是需要分配新对象，因此虚拟线程可能需要较少的分配。总的来说，线程每请求风格的代码与异步代码的堆消耗和垃圾收集器活动量应该是大致相似的。随着时间的推移，我们希望能使虚拟线程栈的内部表示能够显著的更紧凑。

与平台线程栈不同，虚拟线程栈不是 GC roots，因此其中包含的引用不会被执行并发堆扫描的垃圾收集器（如 G1）在 stop-the-world 暂停中遍历。这也意味着，如果一个虚拟线程在例如 `BlockingQueue.take()` 上被阻塞，并且没有其他线程可以获得对该虚拟线程或队列的引用，那么该线程就可以被垃圾收集。这点很好，因为该虚拟线程永远不能被中断或解除阻塞。当然，如果虚拟线程正在运行，或者它被阻塞，并且可能被解除阻塞，那么它将不会被垃圾收集。

目前虚拟线程的一个限制是，G1 GC 不支持巨大的堆栈块对象。如果一个虚拟线程的堆栈达到区域大小的一半，比如小到 512KB，那么可能会抛出一个 StackOverflowError。

### 变更细节
剩下的几个小节详细描述了我们提出的整个 Java 平台及其实现的变化：
- [java.lang.Thread](#java.lang.Thread)
- [Thread-local variables](#Thread-local variables)
- [java.util.concurrent](#java.util.concurrent)
- [Networking](#Networking)
- [java.io](#java.io)
- [Java Native Interface (JNI)](#Java Native Interface (JNI))
- [Debugging (JVM TI, JDWP, and JDI)](#Debugging (JVM TI, JDWP, and JDI))
- [JDK Flight Recorder (JFR)](#JDK Flight Recorder (JFR))
- [Java Management Extensions (JMX)](#Java Management Extensions (JMX))
- [java.lang.ThreadGroup](#java.lang.ThreadGroup)

#### java.lang.Thread
我们更新了下列的 [java.lang.Thread](https://download.java.net/java/early_access/loom/docs/api/java.base/java/lang/Thread.html) ：
- Thread.Builder, Thread.ofVirtual(), 和 Thread.ofPlatform() 是新的创建虚拟线程和平台线程的 API。例如，
```Java
Thread thread = Thread.ofVirtual().name("duke").unstarted(runnable);
```
创建了一个处于 `unstarted` 状态的名为 "duke" 的线程。
- `Thread.startVirtualThread(Runnable)` 是一种创建和启动虚拟线程的便捷方式。
- `Thread.Builder` 既可以创建一个线程，也可以创建一个 ThreadFactory，然后它可以创建多个属性相同的线程。
- `Thread.isVirtual()` 测试一个线程是否是一个虚拟线程。
- `Thread.join` 和 `Thread.sleep` 的新重载接受作为 `java.time.Duration` 实例的等待和睡眠时间。
- 新的 final 方法 `Thread.threadId()` 返回一个线程的标识符。现有的非 final 方法 `Thread.getId()` 现在已被弃用。
- `Thread.getAllStackTraces()` 现在返回所有平台线程的映射，而不是所有线程。

`java.lang.Thread` API 在其他方面没有变化。和以前一样，Thread 类定义的构造函数可以创建平台线程。没有新的公共构造函数。

虚拟线程和平台线程之间的主要 API 差异是：
- 公共线程构造函数不能创建虚拟线程。
- 虚拟线程总是守护线程。`Thread.setDaemon(boolean)` 方法不能将一个虚拟线程改为非守护线程。
- 虚拟线程有一个固定的优先级，即 `Thread.NORM_PRIORITY`。`Thread.setPriority(int)` 方法对虚拟线程没有影响。这一限制可能会在未来的版本中被修改。
- 虚拟线程不是线程组的活动成员。当在一个虚拟线程上调用时，`Thread.getThreadGroup()` 返回一个名称为 "VirtualThreads" 的占位符线程组。`Thread.Builder` API 并没有定义一个方法来设置虚拟线程的线程组。
- 在设置了 `SecurityManager` 的情况下运行时，虚拟线程没有权限。
- 虚拟线程不支持 `stop()`、`suspend()` 或 `resume()` 方法。当在虚拟线程上调用这些方法时，会抛出一个异常。

#### Thread-local variables
虚拟线程支持线程本地变量（ThreadLocal）和可继承的线程本地变量（InheritableThreadLocal），就像平台线程一样，所以它们可以运行使用现有代码的线程本地变量。然而，由于虚拟线程可能非常多，所以在使用线程本地变量时要仔细考虑。特别是，不要使用线程本地变量来在线程池中共享同一线程的多个任务之间汇集昂贵的资源。虚拟线程不应该被集中起来，因为每个线程在其生命周期内只用于运行一个任务。我们已经从 `java.base` 模块中删除了许多线程本地变量变量的使用，为虚拟线程做准备，以减少数百万线程运行时的内存占用。

此外：
- `Thread.Builder` API 定义了一种方法，可以在创建线程时选择不使用线程本地变量。它还定义了一个方法，可以选择不继承可继承的线程本地变量的初始值。当从一个不支持线程本地变量的线程中调用时，`ThreadLocal.get()` 返回初始值，而 `ThreadLocal.set(T)` 则抛出一个异常。
- 现在，传统的上下文类加载器被指定为像可继承的线程本地变量一样工作。如果 `Thread.setContextClassLoader(ClassLoader)` 在一个不支持线程本地变量的线程上被调用，那么它会抛出一个异常。

在某些情况下，局部变量可能被证明是线程本地变量的一个更好的替代品。

#### java.util.concurrent
支持锁定的原始 API， [java.util.concurrent.LockSupport](https://docs.oracle.com/en/java/javase/18/docs/api/java.base/java/util/concurrent/locks/LockSupport.html) ，现在支持虚拟线程。挂起一个虚拟线程会释放底层的平台线程去做其他工作，而恢复一个虚拟线程则会安排它继续工作。对`LockSupport`的这一改变使得所有使用它的API（锁、Semaphores、阻塞队列等）在虚拟线程中被调用时能够优雅地挂起。

此外：

- `Executors.newThreadPerTaskExecutor(ThreadFactory)` 和 `Executors.newVirtualThreadPerTaskExecutor()` 创建一个 `ExecutorService`，为每个任务创建一个新线程。这些方法能够实现与使用线程池和 `ExecutorService` 的现有代码的迁移和互操作性。
- `ExecutorService` 现在扩展了 `AutoCloseable`，从而允许该 API 与 try-with-resource 结构一起使用，如上面的例子所示。
- `Future` 现在定义了一些方法来获取已完成任务的结果或异常，以及获取任务的状态。结合起来，这些新增的方法使得使用 `Future` 对象作为流的元素变得很容易，过滤一个 `Future` 流以找到已完成的任务，然后通过映射来获得一个结果流。这些方法在为 [structured concurrency](https://openjdk.java.net/jeps/8277129) 提出的 API 添加中也很有用。

#### Networking
`java.net` 和 `java.nio.channels` 包中的网络 API 的实现现在可以与虚拟线程一起工作。在虚拟线程上进行的操作，如建立网络连接或从 socket 中读取，会释放底层平台线程来做其他工作。

为了允许中断和取消，由 `java.net.Socket`、`ServerSocket` 和 `DatagramSocket` 定义的阻塞式 I/O 方法现在被指定为在虚拟线程中调用时可中断。中断一个在 socket 上阻塞的虚拟线程将取消该线程并关闭该 socket。当从 `InterruptibleChannel` 获取时，这些类型的 socket 上的阻塞 I/O 操作一直是可中断的，因此这一变化使这些 API 在用构造函数创建时的行为与从 `channel` 获取时的行为一致。

#### java.io
`java.io` 包提供了字节和字符流的 API。这些 API 的实现具有很强的同步性，当它们在虚拟线程中使用时，需要进行修改以避免自旋。

作为背景，面向字节的输入/输出流没有被指定为线程安全的，也没有指定当线程在读或写方法中被阻塞时调用 `close()` 的预期行为。在大多数情况下，从多个并发线程中使用一个特定的输入或输出流是没有意义的。面向字符的读写器也没有被指定为线程安全的，但它们确实为子类暴露了一个锁对象。除了自旋之外，这些类的同步是有问题的，也是不一致的；例如，`InputStreamReader` 和 `OutputStreamWriter` 使用的流解码器和编码器是在流对象而不是锁对象上同步的。

为了防止自旋，现在的实现工作如下：

- `BufferedInputStream`、`BufferedOutputStream`、`BufferedReader`、`BufferedWriter`、`PrintStream` 和 `PrintWriter` 现在在直接使用时使用显式锁而不是监视器。当这些类被子类化时，它们会像以前一样同步。
- 由 `InputStreamReader` 和 `OutputStreamWriter` 使用的流解码器和编码器现在使用与包裹 `InputStreamReader` 或 `OutputStreamWriter` 相同的锁。

更进一步，要消除所有这些经常不必要的锁，已经超出了本 JEP 的范围。

此外，`BufferedOutputStream`、`BufferedWriter` 和 `OutputStreamWriter` 的流编码器所使用的缓冲区的初始大小现在更小了，以便在堆中有许多 `streams` 或 `writers` 时减少内存的使用（如果有一百万个虚拟线程，每个线程在一个套接字连接上有一个缓冲流，就可能出现这种情况）。

#### Java Native Interface (JNI)
JNI 定义了一个新的函数，`IsVirtualThread`，用来测试一个对象是否是一个虚拟线程。

JNI 规范在其他方面没有变化。

#### Debugging (JVM TI, JDWP, and JDI)
调试架构由三个接口组成：JVM 工具接口（JVM TI）、Java Debug Wire Protocol（JDWP）和 Java Debug Interface（JDI）。这三个接口现在都支持虚拟线程。

对 JVM TI 的更新是：
- 大多数用 jthread（即对 Thread 对象的 JNI 引用）调用的函数可以用对虚拟线程的引用来调用。少数函数，即 `PopFrame`、`ForceEarlyReturn`、`StopThread`、`AgentStartFunction` 和 `GetThreadCpuTime`，在虚拟线程上不被支持。`SetLocal` *函数仅限于设置在断点或单步事件中被暂停的虚拟线程的最顶层帧中的局部变量。
- `GetAllThreads` 和 `GetAllStackTraces` 函数现在被指定为返回所有平台线程而不是所有线程。
- 所有的事件，除了那些在早期虚拟机启动期间或堆迭期间发布的事件外，都可以在虚拟线程的上下文中调用事件回调。
- 暂停/恢复实现允许虚拟线程被调试器暂停和恢复，并且允许平台线程在虚拟线程被挂载时暂停。
- 一个新的能力，`can_support_virtual_threads`，让代理对虚拟线程的线程开始和结束事件有更精细的控制。
- 新函数支持虚拟线程的批量暂停和恢复；这些需要 `can_support_virtual_threads` 能力。

现有的 JVM TI 代理大多会像以前一样工作，但如果他们调用不支持虚拟线程的函数，可能会遇到错误。当一个不知道虚拟线程的代理与一个使用虚拟线程的应用程序一起使用时，就会出现这些错误。对 `GetAllThreads` 的改变是返回一个只包含平台线程的数组，这对一些代理来说可能是个问题。启用 `ThreadStart` 和 `ThreadEnd` 事件的现有代理可能会遇到性能问题，因为他们缺乏将这些事件限制在平台线程的能力。

JDWP 的更新是：
- 一个新的命令允许调试员测试一个线程是否是一个虚拟线程。
- `EventRequest` 命令的一个新修改器允许调试者将线程开始和结束事件限制在平台线程上。

对 JDI 的更新是：
- `com.sun.jdi.ThreadReference` 中的一个新方法测试一个线程是否是一个虚拟线程。
- `com.sun.jdi.request.ThreadStartRequest` 和 `com.sun.jdi.request.ThreadDeathRequest` 中的新方法将为请求产生的事件限制为平台线程。

如上所述，虚拟线程不被认为是线程组中的活动线程。因此，由 JVM TI 函数 `GetThreadGroupChildren`、JDWP 命令 `ThreadGroupReference`/`Children` 和 JDI 方法 `com.sun.jdi.ThreadGroupReference.threads()` 返回的线程列表只包括平台线程。

#### JDK Flight Recorder (JFR)
JFR 通过几个新的事件支持虚拟线程：
- `jdk.VirtualThreadStart` 和 `jdk.VirtualThreadEnd` 表示虚拟线程开始和结束。这些事件在默认情况下是禁用的。
- `jdk.VirtualThreadPinned` 表示一个虚拟线程在自旋时被挂起，即没有释放其平台线程（见 [讨论](#虚拟线程的运行) ）。这个事件默认是启用的，阈值为 20ms。
- `jdk.VirtualThreadSubmitFailed` 表示启动或取消挂起虚拟线程失败，可能是因为资源问题。该事件默认为启用。

#### Java Management Extensions (JMX)
`java.lang.management.ThreadMXBean` 只支持对平台线程的监控和管理。`findDeadlockedThreads()` 方法可以找到处于死锁状态的平台线程的周期；它不会找到处于死锁状态的虚拟线程的周期。

`com.sun.management.HotSpotDiagnosticsMXBean` 中的一个新方法可以生成上述的新式线程转储。这个方法也可以通过平台的 `MBeanServer` 从本地或远程的 JMX 工具间接调用。

#### java.lang.ThreadGroup
`java.lang.ThreadGroup` 是一个用于分组线程的传统 API，在现代应用中很少使用，也不适合用于分组虚拟线程。我们现在废止并降级了它，并期望在未来引入一个新的线程组织结构，作为 [structured concurrency](https://openjdk.java.net/jeps/8277129) 的一部分。

作为背景，`ThreadGroup` API 可以追溯到 Java 1.0。它最初的目的是提供作业控制操作，如停止一个组中的所有线程。现代代码更倾向于使用 Java 5 中引入的 java.util.concurrent 包的线程池 API。在早期的 Java 版本中，`ThreadGroup` 支持小程序的隔离，但 Java 的安全架构在 Java 1.2 中发生了很大的变化，线程组不再发挥重要作用。线程组的目的也是用于诊断，但这个作用被 Java 5 中引入的监控和管理功能（包括 `java.lang.management` API）所取代了。

除了在很大程度上无关紧要之外，`ThreadGroup` 的 API 和实现还有一些重大问题：
- 销毁线程组的 API 和机制是有缺陷的。
- 该 API 要求实现者拥有对组中所有实时线程的引用。这给线程创建、线程启动和线程终止增加了同步和竞争的开销。
- API 定义了 `enumerate()` 方法，这些方法本质上是荒谬的。
- API 定义了 `suspend()`，`resume()`，和 `stop()` 方法，这些方法本身就容易造成死锁，而且不安全。

现在，`ThreadGroup` 被规范、废弃，并被降级如下：
- 删除了显式销毁线程组的能力：最终被废弃的 `destroy()` 方法什么也没做。
- 守护线程组的概念已被删除：被废弃的 `setDaemon(boolean)` 和 `isDaemon()` 方法所设置和检索的守护状态被忽略了。
- 该实现不再保留对子组的强引用。当线程组中没有活的线程，并且没有其他东西保持线程组的存活时，该组现在有资格被垃圾回收。
- 已被废弃的 `suspend()`、`resume()` 和 `stop()` 方法总是抛出一个异常。

## 备选方案
- 继续依赖异步 API。异步 API 很难与同步 API 集成，造成了同一 I/O 操作的两种表现形式的分裂世界，并且没有提供统一的操作序列概念，平台可以将其作为故障诊断、监控、调试和剖析的上下文。

- 在 Java 语言中增加语法上的无堆栈程序（即 Async/await）。这比用户模式的线程更容易实现，并将提供一个统一的结构，代表一连串操作的上下文。

    然而，这种结构是新的，与线程分开，在许多方面与它们相似，但在一些细微的方面又不同。它将在为线程设计的 API 和为轮子设计的 API 之间分割开来，并需要将新的类似线程的构造引入平台的所有层面及其工具。这将花费更长的时间让生态系统采用，并且不像用户模式的线程那样优雅，与平台和谐相处。

    大多数已经支持了语法式协程的语言之所以这样做，是因为无法实现用户模式线程（如 Kotlin）、遗留的语义保证（如固有的单线程 JavaScript）或语言特定的技术限制（如 C++）。这些限制并不适用于 Java。

- 引入一个新的公共类来表示用户模式的线程，与 `java.lang.Thread` 无关。这将是一个抛弃 Thread 类 25 年来所积累的不必要的包袱的机会。我们探索了这种方法的几种变体，并进行了原型设计，但在每一种情况下，都要解决如何运行现有代码的问题。

     主要的问题是，`Thread.currentThread()` 被直接或间接地用于现有代码中（例如，在确定锁的所有权，或用于线程局部变量）。这个方法必须返回一个代表当前执行线程的对象。如果我们引入一个新的类来代表用户模式的线程，那么 `currentThread()` 就必须返回某种包装对象，它看起来像一个 `Thread`，但却委托给了用户模式的线程对象。

     让两个对象来代表当前的执行线程会很混乱，所以我们最终得出结论，保留旧的 `Thread` API 并不是一个重大的障碍。除了 `currentThread()` 等少数方法外，开发人员很少直接使用 `Thread` API；他们大多使用 `ExecutorService` 等更高级别的 API 进行交互。随着时间的推移，我们将通过废弃和删除过时的方法来抛弃 `Thread` 类以及 `ThreadGroup` 等相关类中不需要的包袱。

## 测试
- 现有的测试将确保我们在这里提出的修改不会在众多的配置和执行模式中造成任何意外的退化
- 我们将扩展 jtreg 测试工具，允许现有的测试在虚拟线程的上下文中运行。这将避免许多测试需要有两个版本。
- 新的测试将执行所有新的和修订的 API，以及所有支持虚拟线程的领域。
- 新的压力测试将针对那些对可靠性和性能至关重要的领域。
- 新的微观基准测试将针对对性能至关重要的领域。
- 我们将使用一些现有的服务器，包括 Helidon 和 Jetty，进行大规模的测试。

## 风险和假设
这个建议的主要风险是由于现有 API 及其实现的变化而导致的兼容性问题：

- 对 `java.io.BufferedInputStream`、`BufferedOutputStream`、`BufferedReader`、`BufferedWriter`、`PrintStream` 和 `PrintWriter` 类中使用的内部（和无记录的）锁定协议的修改，可能会影响那些假定 I/O 方法在它们被调用的流上进行同步的代码。这些变化并不影响扩展这些类并假定由超类锁定的代码，也不影响扩展 `java.io.Reader` 或 `java.io.Writer` 并使用由这些 API 暴露的锁定对象的代码。
- `java.lang.ThreadGroup` 不再允许销毁线程组，不再支持守护线程组的概念，其 `suspend()`、`resume()` 和 `stop()` 方法总是抛出异常。

有几个与源代码不兼容的 API 变化，以及一个与二进制不兼容的变化，可能会影响扩展 `java.lang.Thread` 的代码：
- 如果现有源文件中的代码扩展了 `Thread`，并且子类中的某个方法与任何新的 `Thread` 方法相冲突，那么该文件将无法编译，无需更改。
- `Thread.Builder` 被添加为一个嵌套接口。如果现有源文件中的代码扩展了 `Thread`，导入了一个名为 `Builder` 的类，并且子类中的代码引用了 "Builder" 作为一个简单的名称，那么该文件将不会被编译而不被更改。
- `Thread.threadId()` 被添加为一个最终方法，用于返回线程的标识符。如果现有源文件中的代码扩展了 `Thread`，并且子类声明了一个名为 `threadId` 的方法，没有参数，那么它将不会被编译。如果现有的已编译代码扩展了 `Thread`，而子类定义了一个名为 `threadId` 的方法，其返回类型为 long，没有参数，那么如果子类被加载，在运行时就会抛出 `IncompatibleClassChangeError`。

当现有代码与利用虚拟线程或新 API 的较新代码混合时，可能会观察到平台线程和虚拟线程之间的一些行为差异：
- `Thread.setPriority(int)` 方法对虚拟线程没有影响，它们的优先级总是 `Thread.NORM_PRIORITY`。
- `Thread.setDaemon(boolean)` 方法对虚拟线程没有影响，它们总是守护线程。
- `Thread.stop()`、`suspend()` 和 `resume()` 方法在虚拟线程上调用时，会抛出一个 `UnsupportedOperationException`。
- `Thread` API 支持创建不支持线程本地变量的线程。`ThreadLocal.set(T)` 和 `Thread.setContextClassLoader(ClassLoader)` 在一个不支持线程局部变量的线程的上下文中调用时，会抛出一个 `UnsupportedOperationException`。
- `Thread.getAllStackTraces()` 现在返回所有平台线程的映射，而不是所有线程的映射。
- 由 `java.net.Socket`、`ServerSocket` 和 `DatagramSocket` 定义的阻塞式 I/O 方法现在在虚拟线程的上下文中被调用时可以被中断。当阻塞在 socket 操作上的线程被中断时，现有的代码可能会中断，这将唤醒该线程并关闭 socket。
- 虚拟线程不是 `ThreadGroup` 的活动成员。在一个虚拟线程上调用 `Thread.getThreadGroup()` 会返回一个空的假 "VirtualThreads" 组。
- 在设置了 `SecurityManager` 的情况下运行时，虚拟线程没有权限。
- 在 JVM TI 中，`GetAllThreads` 和 `GetAllStackTraces` 函数并不返回虚拟线程。启用 `ThreadStart` 和 `ThreadEnd` 事件的现有代理可能会遇到性能问题，因为它们缺乏将事件限制在平台线程的能力。
- `java.lang.management.ThreadMXBean` API 支持对平台线程的监控和管理，但不支持虚拟线程。
- -XX:+PreserveFramePointer 标志对虚拟线程的性能有极大的负面影响。

## 依赖
- [JEP 416 (Reimplement Core Reflection with Method Handles)](https://openjdk.java.net/jeps/416) in JDK 18 removed the VM-native reflection implementation. This allows virtual threads to park gracefully when methods are invoked reflectively.
- [JEP 353 (Reimplement the Legacy Socket API)](https://openjdk.java.net/jeps/353) in JDK 13, and [JEP 373 (Reimplement the Legacy DatagramSocket API)](https://openjdk.java.net/jeps/373) in JDK 15, replaced the implementations of java.net.Socket, ServerSocket, and DatagramSocket with new implementations designed for use with virtual threads.
- [JEP 418 (Internet-Address Resolution SPI)](https://openjdk.java.net/jeps/418) in JDK 18 defined a service-provider interface for host name and address lookup. This will allow third-party libraries to implement alternative java.net.InetAddress resolvers that do not pin threads during host lookup.

## 参考文档：
 [JEP 425: Virtual Threads (Preview)](https://openjdk.java.net/jeps/425)
