# 探究 Repeat 中 GCD 的应用

> 这是小专栏《[彻底搞定 GCD🚦并发编程](https://xiaozhuanlan.com/complete-ios-gcd)》的一篇副产品文章

## 简介

[Repeat](https://github.com/malcommac/Repeat) 是 *Daniele* 开发的一个基于 GCD - Grand Central Dispatch 的轻量定时器，可用于替代 `NSTimer`，解决其多项[不足](#番外：NSTimer 的缺陷)。

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

## 接口的设计与使用

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

可以看到，有限次的定时器在次数达到结束后，还可以继续调用 `start()` 重新开始再次复用。

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

// TBD：10/14 - 11:52







文件目录结构：

```sh
./
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

## 番外：NSTimer 的缺陷





## 致谢与参考

* [A Background Repeating Timer in Swift](https://medium.com/over-engineering/a-background-repeating-timer-in-swift-412cecfd2ef9) by Daniel Galasko