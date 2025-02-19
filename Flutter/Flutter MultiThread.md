# Flutter 多线程编程详解

Dart在**单线程中是以消息循环机制来运行的**，其中包含两个任务队列，一个是“微任务队列” **micro task queue**，另一个叫做“事件队列” **event queue**。微任务队列的执行优先级高于事件队列。

Dart大致运行原理：先开启app执行入口函数main()，执行完成之后，消息机制启动，先是会按照先进先出的顺序逐个执行微任务队列中的任务**micro task**，事件任务**event task**执行完毕后便会退出，但是，在事件任务执行的过程中也可以插入新的微任务和事件任务，在这种情况下，整个线程的执行过程便是一直在循环，不会退出，而Flutter中，主线程的执行过程正是如此。**Dart 中事件的执行顺序：Main > MicroTask > EventQueue**

**Isolate 线程会优先执行 Micro task queue 里的事件，当 Micro task queue 里的事件变成空了，才会去执行 Event queue 里的事件。如果正在执行 Microtask queue 里的事件，那么 Event queue 里的事件就会被阻塞，就会导致渲染、手势响应等都得不到响应（绘制图形，处理鼠标点击，处理文件IO等都是在 Event Queue 里完成）。**

**所以为了保证功能正常使用不卡顿，尽量少在 Micro task queue 做事情，可以放在 Event queue 做。**



**为什么单线程可以做一个异步操作呢？**

因为 APP 只有在你滑动或者点击操作的时候才会响应事件。没有操作的时候进入等待时间，两个队列里都是空的。这个时间正是可以进行异步操作的，所以基于这个特点，单线程模型可以在等待过程中做一些异步操作，因为等待的过程并不是阻塞的，所以给我们的感觉就像同时在做多件事情，但自始至终只有一个线程在处理事情。

**Future 为何卡顿**

再来说一下刚刚的问题，我已经放在异步里解析了，为什么还是会卡呢？

其实很简单，Future 里的代码可能需要执行10s，也就是 Event queue 需要10s才能执行完。那这个10s内其他代码肯定就无法执行了。所以 Future 里的代码执行时间过长，还是会卡 UI 的。



**1.Future 适合耗时小于 16ms 的操作**

**2.可以通过 compute() 进行耗时操作**

**3.Dart 是单线程原因，但也支持多线程，但是线程间数据不互通**

**4.Isolate 线程会优先执行 Microtask queue 里的事件，当 Microtask queue 里的事件变成空了，才会去执行 Event queue 里的事件。如果正在执行 Microtask queue 里的事件，那么 Event queue 里的事件就会被阻塞，就会导致渲染、手势响应等都得不到响应（绘制图形，处理鼠标点击，处理文件IO等都是在 Event Queue 里完成）。**

**5.Dart 中事件的执行顺序：Main > MicroTask > EventQueue**

**6.用await和Future包住的代码都是在eventQueue里运行的，同时没有延迟的Future代码的then也是在同一个eventQueue里执行的。只有延迟的代码Future和then会分开处理，此时then进入到了microTask队列里。**



在现代应用开发中，多线程技术是提高应用性能和响应能力的重要手段。Flutter 作为一个跨平台的 UI 框架，同样支持多线程编程。本文将详细探讨 Flutter 的多线程原理、使用方式、通信机制及使用场景。

### 1. Flutter 异步编程

Flutter 的异步编程主要通过 Dart 的异步特性来实现，允许你在不阻塞 UI 的情况下处理耗时操作。以下是异步编程的几个关键概念：

#### 1.1 Future

`Future` 是 Dart 中用于处理异步操作的核心类，代表一个可能在未来某个时间完成的值。通过 `Future`，你可以执行异步任务并在完成时获取结果。

##### 示例：

```dart
Future<String> fetchData() async {
  await Future.delayed(Duration(seconds: 2));
  return '数据加载完成';
}

void loadData() async {
  String result = await fetchData();
  print(result); // 输出: 数据加载完成
}
```

#### 1.2 async 和 await

- **async**：标记函数为异步函数，异步函数总是返回一个 `Future`。
- **await**：等待 `Future` 完成并获取结果，必须在 `async` 函数内使用。

#### 1.3 Stream

`Stream` 是处理一系列异步事件的类，可以处理多个值，比如实时数据流。

##### 示例：

```dart
Stream<int> numberStream() async* {
  for (int i = 1; i <= 5; i++) {
    await Future.delayed(Duration(seconds: 1));
    yield i; // 每秒发出一个值
  }
}

void listenToStream() {
  numberStream().listen((number) {
    print(number); // 输出: 1, 2, 3, 4, 5
  });
}
```

#### 1.4 错误处理

在异步编程中，错误处理非常重要。可以使用 `try-catch` 语句捕获异常：

```dart
Future<void> loadData() async {
  try {
    String result = await fetchData();
    print(result);
  } catch (e) {
    print('发生错误: $e');
  }
}
```

#### 1.5 小结

Flutter 的异步编程使得处理耗时任务（如网络请求、文件读写等）变得简单而高效。通过使用 `Future`、`Stream`、`async` 和 `await`，你可以轻松管理异步操作，确保应用保持响应。

### 2. 默认线程

在 Flutter 中，以下四种任务 Runner 分别负责不同的工作，运行在各自的线程中：

- **Platform Task Runner**：处理与平台相关的任务，如通过平台通道与原生代码交互。
- **UI Task Runner**：即 UI 线程，负责渲染 Flutter 的 UI，处理用户输入和动画。所有 UI 更新必须在此线程中进行。
- **IO Task Runner**：专门处理 I/O 操作（如文件读取、网络请求等），避免阻塞 UI 线程，确保界面流畅。
- **GPU Task Runner**：负责与 GPU 进行通信，执行图形绘制等任务，确保渲染过程中的图形计算高效进行。

通过这些任务 Runner，Flutter 可以实现更高效的并发处理，确保流畅的用户体验。

### 3. Flutter 多线程原理

Flutter 使用 Dart 语言进行开发，Dart 内建对异步编程的支持，但 Dart 本身是单线程的。实际的多线程功能是通过 Isolate 实现的。

#### 3.1 Isolate

- **Isolate** 是 Dart 中的基本并发单位。每个 Isolate 拥有自己的内存堆，不共享内存。Isolate 之间通过消息传递进行通信，避免了传统多线程中的共享状态问题。

### 4. 使用多线程的方式

#### 4.1 使用 Isolate

创建新的 Isolate 来处理耗时操作，避免阻塞主线程。

##### 示例代码：使用 Isolate

```dart
import 'dart:async';
import 'dart:isolate';

void entryPoint(SendPort sendPort) {
  int result = 0;
  for (int i = 0; i < 100000; i++) {
    result += i;
  }
  sendPort.send(result); // 发送结果回主线程
}

Future<void> runIsolate() async {
  final receivePort = ReceivePort();
  final isolate = await Isolate.spawn(entryPoint, receivePort.sendPort);
  
  receivePort.listen((data) {
    print('Result from isolate: $data');
    receivePort.close();
    isolate.kill();
  });
}
```

#### 4.2 使用 compute 函数

Flutter 提供 `compute` 函数，可以方便地将耗时计算放在后台线程执行。

##### 示例代码：使用 compute

```dart
import 'package:flutter/foundation.dart';

int heavyComputation(int input) {
  int result = 0;
  for (int i = 0; i < input; i++) {
    result += i;
  }
  return result;
}

Future<void> runComputeExample() async {
  final result = await compute(heavyComputation, 100000);
  print('Computed result: $result');
}
```

### 5. 多线程间如何通信

Isolate 之间的通信通过消息传递实现，使用 `SendPort` 和 `ReceivePort` 来建立通信通道：

- **SendPort**：发送消息的端口。
- **ReceivePort**：接收消息的端口。

在示例中，`entryPoint` 函数通过 `sendPort.send(result)` 将结果发送回主线程，主线程监听 `ReceivePort` 接收结果。

### 6. 何时该使用多线程

决定何时开启新的线程（通常是通过创建新的 Isolate 或使用 IO Task Runner）取决于以下几个指标和情境：

#### 6.1 耗时操作

- **网络请求**：如果请求时间较长，建议在新线程中处理。
- **文件读取/写入**：大文件操作可能导致 UI 卡顿。
- **复杂计算**：耗时的算法或数据处理（如图像处理）。

#### 6.2 具体指标

虽然没有绝对的时间阈值，但一般建议：

- **超过 100 毫秒**：如果某个任务的执行时间超过 100 毫秒，建议将其放在后台线程中执行。
- **多次调用的累积时间**：如果某个操作短时间内多次调用，累计时间可能导致性能问题，应考虑异步处理。

#### 6.3 用户体验

- **动画和交互**：确保 UI 交互和动画流畅，任何可能影响用户体验的操作应放在新线程中。
- **实时反馈**：需要实时响应的操作（如用户输入）应谨慎选择是否开启新线程。

### 7 注意事项

##### 7.1 内存管理

- **内存开销**：每个 Isolate 都有独立的内存堆，因此创建过多的 Isolate 可能导致内存占用过高。应合理控制 Isolate 的数量。
- **避免内存泄漏**：确保在不需要时释放 Isolate，避免长时间占用资源。

##### 7.2 传递数据的限制

- **只能传递可序列化的数据**：Isolate 之间的通信使用消息传递，因此只能传递可序列化的对象（如基本数据类型、List、Map 等），不能直接传递 Dart 对象或复杂类型。
- **数据复制**：传递的数据会被复制到目标 Isolate，因此大对象的传递可能导致性能开销。应尽量传递小而简单的数据。

##### 7.3 线程安全

- **没有共享内存**：Isolate 之间没有共享内存，所有的数据共享必须通过消息传递。这意味着需要仔细设计数据流，确保数据一致性。

##### 7.4 性能开销

- **创建和销毁成本**：创建 Isolate 是一个开销较大的操作，因此频繁创建和销毁 Isolate 会影响性能。使用 Isolate 池可以帮助减少开销。
- **异步任务**：在 Isolate 中执行任务时，需要注意任务的执行时间，确保不会造成过长的阻塞。

##### 7.5 错误处理

- **捕获异常**：在 Isolate 中抛出的异常不会直接传递到主 Isolate，需要通过消息传递的方式来处理错误。
- **失败处理**：确保对每个 Isolate 的返回结果进行验证，处理可能的失败情况。

##### 7.6 生命周期管理

- **管理 Isolate 生命周期**：合理管理 Isolate 的创建、使用和销毁，避免资源浪费。
- **使用接收端口**：确保在 Isolate 完成任务后关闭接收端口，防止内存泄漏。

##### 7.7 性能监测

- **监控性能**：在使用 Isolate 时，可以考虑监控其性能指标，以便及时发现和解决性能问题。

### 8. 小结

Flutter 的多线程编程主要依赖于 Dart 的 Isolate 机制。通过合理使用 Isolate 和 `compute` 函数，可以有效提升应用的性能和用户体验。在处理耗时操作时，使用多线程是一个明智的选择，能够确保应用保持流畅与响应迅速。

