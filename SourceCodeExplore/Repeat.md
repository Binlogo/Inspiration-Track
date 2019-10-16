#  探究 Repeat 中 GCD 的应用

> 这是小专栏《[彻底搞定 GCD🚦并发编程](https://xiaozhuanlan.com/complete-ios-gcd)》的一篇副产品文章

## 简介

[Repeat](https://github.com/malcommac/Repeat) 是 *Daniele* 开发的一个基于 GCD - Grand Central Dispatch 的轻量定时器，可用于替代 `NSTimer`，解决其多项[不足](#TBD)。

## 特性

Daniel 着重强调的一些特性：

* 接口简洁明了不冗余
* 避免强引用，解决潜在的内存泄露问题
* 支持注册多个观察者对象监听定时器
* 支持暂停、开始、恢复、重置等操作
* 支持设置多种重复模式
  * infinite：无限定时重复
  * finite：有限次定时重复
  * once：单次定时执行

针对以上特性，我们接下来阅读源码时可以着重看看 Daniel 是怎样实现的。

另外，这个库还扩展提供了两个有趣的特性：

* Debouncer: 防抖动，避免方法调用过于密集，总是执行每隔一定时间段内最后一个调用。
* Throttler: 节流阀，确保每隔一定时间段内仅执行一次，并忽略其他调用。

这两个特性对于用过 `RxSwift` 的开发者肯定不陌生，有了定时器，实现这两者也是顺理成章。



## 接口的设计、使用与实现

首先，从接口的设计来看看其是否「简洁明了」、「不冗余」，再逐步深入其内部实现。

### 定时器

> 注意：`Repeater` 被设计成和其他许多对象一样，需要被持有来避免被释放。

#### 创建单次定时器

* 接口设计与调用

```swift
// 创建单次执行定时器
log("当时只道是寻常")
self.timer = Repeater.once(after: .seconds(5)) { timer in
    // 5s 后运行
    log("岁月如歌,简单爱一次")
}
```

输出：

```shell
2019-10-14 15:12:04 +0000: 当时只道是寻常
2019-10-14 15:12:09 +0000: 岁月如歌,简单爱一次
```

* 单元测试设计

```swift
func test_timer_once() {
  let exp = expectation(description: "test_once")

  let timer = Repeater.once(after: .seconds(5)) { _ in
    exp.fulfill()
  }

  print("Allocated timer \(timer)")
  wait(for: [exp], timeout: 6)
}
```

#### 创建有限次重复的定时器

* 接口设计与调用

```swift
log("当时只道是寻常")
self.timer = Repeater.every(.seconds(10), count: 5) { timer in
    // 每 10s 运行 1 次，5 次后结束
    log("岁月如歌,简单爱一次")
}
```

输出：

```shell
2019-10-14 15:18:56 +0000: 当时只道是寻常
2019-10-14 15:19:06 +0000: 岁月如歌,简单爱一次
2019-10-14 15:19:16 +0000: 岁月如歌,简单爱一次
2019-10-14 15:19:26 +0000: 岁月如歌,简单爱一次
2019-10-14 15:19:36 +0000: 岁月如歌,简单爱一次
2019-10-14 15:19:46 +0000: 岁月如歌,简单爱一次
```

* 单元测试设计

```swift
func test_timer_finiteAndRestart() {
  let exp = expectation(description: "test_finiteAndRestart")

  var count: Int = 0
  var finishedFirstTime: Bool = false
  let timer = Repeater(interval: .seconds(0.5), mode: .finite(5)) { _ in
    count += 1
    print("Iteration #\(count)")
  }
  timer.onStateChanged = { (_, state) in
    print("State changed: \(state)")
    if state.isFinished {
      if finishedFirstTime == false {
        print("Now restart")
        timer.start()
        finishedFirstTime = true
      } else {
        exp.fulfill()
      }
    }
  }

  timer.start()

  wait(for: [exp], timeout: 30)
}
```

单元测试输出：

```shell
State changed: running
State changed: executing
Iteration #1
State changed: executing
Iteration #2
State changed: executing
Iteration #3
State changed: executing
Iteration #4
State changed: executing
Iteration #5
State changed: finished
Now restart
State changed: idle/paused
State changed: running
State changed: executing
Iteration #6
State changed: executing
Iteration #7
State changed: executing
Iteration #8
State changed: executing
Iteration #9
State changed: executing
Iteration #10
State changed: finished
```

可以看到，有限次的定时器在次数达到结束后，还可以继续调用 `start()` 重新开始再次复用，而不必另外创建新实例。

#### 创建无限重复定时器

* 接口设计与调用

```swift
log("当时只道是寻常")
let timer = Repeater.every(.seconds(5)) { timer in
    // 每 5s 运行一次，直到 timer 生命周期结束
    log("岁月如歌,简单爱一次")
}
```

输出：

```shell
2019-10-14 15:24:05 +0000: 当时只道是寻常
2019-10-14 15:24:10 +0000: 岁月如歌,简单爱一次
2019-10-14 15:24:15 +0000: 岁月如歌,简单爱一次
2019-10-14 15:24:20 +0000: 岁月如歌,简单爱一次
2019-10-14 15:24:25 +0000: 岁月如歌,简单爱一次
2019-10-14 15:24:30 +0000: 岁月如歌,简单爱一次
2019-10-14 15:24:35 +0000: 岁月如歌,简单爱一次
2019-10-14 15:24:40 +0000: 岁月如歌,简单爱一次
2019-10-14 15:24:45 +0000: 岁月如歌,简单爱一次
...
```

* 单元测试设计

```swift
func test_timer_infinite() {
  let exp = expectation(description: "test_once")

  var count: Int = 0
  let timer = Repeater.every(.seconds(0.5), { _ in
    count += 1
    if count == 20 {
      exp.fulfill()
    }
  })

  print("Allocated timer \(timer)")
  wait(for: [exp], timeout: 10)
}
```



#### 手动管理计时器

* 接口设计与调用

```swift
log("当时只道是寻常")
self.timer = Repeater(interval: .seconds(5), mode: .infinite) { _ in
    // 每 5s 运行一次，直到 timer 生命周期结束
    log("岁月如歌,简单爱一次")
}
timer.start()
```

输出：

```shell
2019-10-14 23:13:46 +0000: 当时只道是寻常
2019-10-14 23:13:51 +0000: 岁月如歌,简单爱一次
2019-10-14 23:13:56 +0000: 岁月如歌,简单爱一次
2019-10-14 23:14:01 +0000: 岁月如歌,简单爱一次
2019-10-14 23:14:06 +0000: 岁月如歌,简单爱一次
2019-10-14 23:14:11 +0000: 岁月如歌,简单爱一次
2019-10-14 23:14:16 +0000: 岁月如歌,简单爱一次
2019-10-14 23:14:21 +0000: 岁月如歌,简单爱一次
2019-10-14 23:14:26 +0000: 岁月如歌,简单爱一次
...
```

其他方法：

* `start()`: 开始一个已暂停或新创建的定时器
* `pause()`：暂停一个正在运行的定时器
* `reset(_ interval: Interval, restart: Bool)`：重置一个正在运行的定时器，更改时间间隔并重新开始
* `fire()`：额外手动调用一次定时器绑定事件

属性：

* `.mode`：定时器类型模式
  * `infinite`：无限定时重复
  * `finite`：有限次定时重复
  * `once`：单次定时执行
* `.remainingIterations`：针对 `.finite` 有限次重复模式结束前的剩余迭代次数



#### 添加或移除多个定时处理

一般而言，初始化定时器时会指定一个处理方法。除此之外， Repeat 的定时器还支持通过`observe()`额外添加处理方法，并且支持通过`token`再移除。

```swift
// 添加额外的处理监听
let token = timber.observe { _ in
	// 额外的新处理
  log("一个闹钟能够同时叫醒相爱的两个人。")
}
timer.start()

// 移除
timer.remove(token)
```



#### 观察定时器的状态变化

每个定时器维护着一个状态机，处在以下某个状态：

* `.paused`：空闲（未被开始过）或已暂停
* `.running`：正在计时中
* `.executing`：注册的定时处理方法正在执行
* `.finished`：计时结束

可以通过`.onStateChanged`属性添加状态变化回调监听：

```swift
timer.onStateChanged = { timer, newState in
    // 观察定时器状态变化并做相应处理
    log("你永远叫不醒一个装睡的人")
}
```



### 防抖动器

*TBD*



### 节流阀

*TBD*



## 源码探究

接下来，进一步看看以上的接口都是如何实现的吧。

### 文件目录结构

```sh
.
├── CHANGELOG.md
├── Configs
│   ├── Repeat.plist
│   └── RepeatTests.plist
├── LICENSE
├── Package.swift
├── README.md
├── Repeat.podspec
├── Repeat.xcodeproj
├── Sources
│   └── Repeat
│       ├── Debouncer.swift
│       ├── Repeater.swift
│       └── Throttler.swift
└── Tests
    ├── LinuxMain.swift
    └── RepeatTests
        └── RepeatTests.swift
```

主要文件：

* Sources/Repeat/[Repeater.swift](https://github.com/malcommac/Repeat/blob/develop/Sources/Repeat/Repeater.swift)： 核心定时器实现
* Sources/Repeat/[Debouncer.swift](https://github.com/malcommac/Repeat/blob/develop/Sources/Repeat/Debouncer.swift)： 扩展「防抖动器」实现
* Sources/Repeat/[Throttler.swift](https://github.com/malcommac/Repeat/blob/develop/Sources/Repeat/Throttler.swift)： 扩展「节流阀」实现
* Tests/RepeatTests/[RepeatTests.swift](https://github.com/malcommac/Repeat/blob/develop/Tests/RepeatTests/RepeatTests.swift)： 单元测试

### 类图与方法概览

// TBD: [配图]-UML 类图



### 工厂类方法实现以及与 `Timer` 的异同

在[接口设计与使用](#接口的设计、使用与实现)中可以看到，Repeater 提供了便捷工厂类方法，并且生成的定时器都会「自动开始」，与 `Timer` 的工厂类方法相似：

```swift
/// Alternative API for timer creation with a block.
/// - Experiment: This is a draft API currently under consideration for official import into Foundation as a suitable alternative to creation via selector
/// - Note: Since this API is under consideration it may be either removed or revised in the near future
/// - Warning: Capturing the timer or the owner of the timer inside of the block may cause retain cycles. Use with caution
open class func scheduledTimer(withTimeInterval interval: TimeInterval, 
                               repeats: Bool, 
                               block: @escaping (Timer) -> Void) -> Timer {
    let timer = Timer(fire: Date(timeIntervalSinceNow: interval), interval: interval, repeats: repeats, block: block)
    CFRunLoopAddTimer(CFRunLoopGetCurrent(), timer._timer!, kCFRunLoopDefaultMode)
    return timer
}
```

👆Tips & Declaration： [Timer.swift](https://github.com/apple/swift-corelibs-foundation/blob/master/Foundation/Timer.swift#L68)，注释中有特别提醒注意循环引用的问题。

对比一下 `Repeater` 的类工厂方法实现：

```swift
/// Create and schedule a timer that will call `handler` once after the specified time.
///
/// - Parameters:
///   - interval: interval delay for single fire
///   - queue: destination queue, if `nil` a new `DispatchQueue` is created automatically.
///   - observer: handler to call when timer fires.
/// - Returns: timer instance
@discardableResult
public class func once(after interval: Interval, 
                       tolerance: DispatchTimeInterval = .nanoseconds(0), 
                       queue: DispatchQueue? = nil, 
                       _ observer: @escaping Observer) -> Repeater {
      let timer = Repeater(interval: interval, mode: .once, tolerance: tolerance, queue: queue, observer: observer)
  timer.start()
  return timer
}

/// Create and schedule a timer that will fire every interval optionally by limiting the number of fires.
///
/// - Parameters:
///   - interval: interval of fire
///   - count: a non `nil` and > 0  value to limit the number of fire, `nil` to set it as infinite.
///   - queue: destination queue, if `nil` a new `DispatchQueue` is created automatically.
///   - handler: handler to call on fire
/// - Returns: timer
@discardableResult
public class func every(_ interval: Interval, 
                        count: Int? = nil, 
                        tolerance: DispatchTimeInterval = .nanoseconds(0), 
                        queue: DispatchQueue? = nil, 
                        _ handler: @escaping Observer) -> Repeater {
  let mode: Mode = (count != nil ? .finite(count!) : .infinite)
      let timer = Repeater(interval: interval, mode: mode, tolerance: tolerance, queue: queue, observer: handler)
  timer.start()
  return timer
}
```

两者的做法大相径庭，都是创建一个定时器实例并「自动开始」，只是开始的方式由于内部固有的实现方式有所不同：

* `Timer` 依赖 `RunLoop`，需要将创建定时器生成的 `CFRunLoopTimer` 加入当前 `Runloop`
* `Repeater` 依赖 GCD 的 `DispatchSourceTimer`，`start` 内部会调 `DispatchSourceTimer` 的 实例方法`resume`



### 初始化方法

```swift
/// Initialize a new timer.
///
/// - Parameters:
///   - interval: interval of the timer
///   - mode: mode of the timer
///   - tolerance: tolerance of the timer, 0 is default.
///   - queue: queue in which the timer should be executed; if `nil` a new queue is created automatically.
///   - observer: observer
public init(interval: Interval, mode: Mode = .infinite, tolerance: DispatchTimeInterval = .nanoseconds(0), queue: DispatchQueue? = nil, observer: @escaping Observer) {
  self.mode = mode
  self.interval = interval
  self.tolerance = tolerance
  self.remainingIterations = mode.countIterations
  self.queue = (queue ?? DispatchQueue(label: "com.repeat.queue"))
  self.timer = configureTimer()
  self.observe(observer)
}
```

初始化方法参数列表：

* `interval`：定时器时间间隔
* `mode`：定时器重复模式，默认为`.infinite`，无限重复
* `tolerance`：容许误差（这个最终是作为`DispatchSourceTimer`的`leeway`参数）
* `queue`：指定定时器运行的队列，若未指定，则自动创建默认队列
* `observer`：定时器运行的回调方法



#### 创建并配置 DispatchSourceTimer

```swift
private func configureTimer() -> DispatchSourceTimer {
  let associatedQueue = (queue ?? DispatchQueue(label: "com.repeat.\(NSUUID().uuidString)"))
  let timer = DispatchSource.makeTimerSource(queue: associatedQueue)
  let repeatInterval = interval.value
  let deadline: DispatchTime = (DispatchTime.now() + repeatInterval)
  if self.mode.isRepeating {
    timer.schedule(deadline: deadline, repeating: repeatInterval, leeway: tolerance)
  } else {
    timer.schedule(deadline: deadline, leeway: tolerance)
  }

  timer.setEventHandler { [weak self] in
    if let unwrapped = self {
      unwrapped.timeFired()
    }
  }
  return timer
}
```

* 根据初始化传入的参数，初始化并配置一个 DispatchSourceTimer
* 将 DispatchSourceTimer 的处理回调通过`timeFired`方法i处理

这一段代码是 Repeat 中 GCD 的应用关键，Repeat 的核心计时器即是 DispatchSourceTimer，进一步封装并屏蔽部分复杂逻辑，以提供简洁易用的接口。

// TBD



## 致谢与参考

文中涉及源码大部分来自开源社区，以及部分其他文献参考。

* [Repeat](https://github.com/malcommac/Repeat) by Daniele Margutti, under The MIT License (MIT)

* [swift-corelibs-foundation](https://github.com/apple/swift-corelibs-foundation) Licensed under Apache License v2.0 with Runtime Library Exception

* [A Background Repeating Timer in Swift](https://medium.com/over-engineering/a-background-repeating-timer-in-swift-412cecfd2ef9) by Daniel Galasko