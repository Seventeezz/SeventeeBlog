---
title: 'Go并发编程入门学习笔记'
description: Notes for Google Rob Pike's talks of 《Go Concurrency Patterns》 & 《Concurrency is not Parallelism》
pubDate: 'Aug 05 2024'
heroImage: '../../assets/goroutine/gopher.png'
category: '语言特性'
tags: ['Go', '并发编程']
---

> Do not communicate by sharing memory. Instead, share memory by communicating.

## 前言

学习计算机这么多年来，因为各种课程作业+课题项目，接触到形形色色的语言。

C，C++，C#，Java，JavaScript，TypeScript，Lua，Python，Go……

以下是我对这些语言**曾经的**内心写照：

- **C**不就是那种老派的、直接操控硬件和内存、写系统程序的**效率王者**嘛？
- **C++** 就在C的基础上搞了个**面向对象编程**，加了些**模板**啥的。
- **Java**不就是那个写一次到处跑，**不用操心内存管理**，企业里搞大项目的利器嘛？
- **C#** 呢，就是**微软家的Java**，搞了个.NET框架，不用手动管内存，还能写Windows应用和游戏。
- **JavaScript**就是**前端之王，动态类型**，搞网页交互特别溜，还能用Node.js搞后端。
- **TypeScript**不就是给JavaScript加了**类型检查**，更适合大型项目，让代码更靠谱嘛。
- **Lua**呢，就是**轻量级**的脚本语言，特别好嵌到C程序里，做游戏**热更新**啥的特别方便。
- **Python**不就是那个**语法简单、上手快、啥都能干**，从数据分析到Web开发，哪都能看到它的影子。
- **Go**嘛，就是Google出品的**简洁高效版C，专注并发编程**，服务器端开发超级牛。

从一种语言迁徙到另一种语言，感受无非就是xxx更方便了，yyy不太行。只不过是把编程习惯照搬过去罢了，只是表面不同的问题，有什么区别吗？

然而，在我看了Peter Norvig的 [Teach Yourself Programming in Ten Years](https://norvig.com/21-days.html) 一文后，
被啪啪打脸，自己的理解还是太浅显了。

> A language that doesn't affect the way you think about programming, is not worth knowing. ——Alan Perlis

> One possible point is that you have to learn a tiny bit of C++ (or more likely, something like JavaScript or Processing) because you need to interface with an existing tool to accomplish a specific task. But then **you're not learning how to program**; you're **learning to accomplish that task**.

没错，过往日子的多数时候，我只是在学习怎么用一门语言去完成一项特定的任务罢了，不断地repeat，并没有去汲取这门语言给我带来的编程思维上的提升。

就像玩英雄联盟一样

- 上单教会我极致细节的个人对线能力、兵线处理、中后期的边线处理
- 打野教会我发育和Gank之间的平衡，中立资源的控制，眼观六路耳听八方观察敌我全局
- 中单教会我游走的时机，与打野的联动
- AD教会我团战中的极致拉扯与自我保护
- 辅助教会我视野控制与开团保护
- 不同定位的英雄，法师教你发育后期发力，刺客教你游走与团战绕后切C，软辅教你保护与对线压力，硬辅教你抓timing开团……

当我集齐所有板块，补上短板，水平段位自然晋升飞快。

学习不同编程语言，是为了汲取其语言设计上的精华，内化为自己编程思维的内功。

> Learn at least a half dozen programming languages.Include one that **emphasizes parallelism** (like Clojure or Go).

痛定思痛，拾起Go语言，一探究竟。

吸星大法启动，并发编程的思维，是我的了！

## 并发 or 并行?

[Concurrency is not Parallelism (go.dev)](https://go.dev/talks/2012/waza.slide#1)

**并发(Concurrency)**

指在同一时间段内，多种不同类型的任务或操作可以进行**交错执行**，不要求同时执行。关注点在宏观上的任务调度，单处理器单线程也可以做到并发。
核心思想是通过将一个大任务拆分为多种**独立的**子任务，而这些子任务能不能利用多处理器去并行处理，是隐藏的细节不必关心。

操作系统中的进程调度就是并发，多个应用程序（如浏览器、音乐播放器、键盘鼠标输入）同时运行，每个应用程序被操作系统分配时间片轮流执行。即使只有一个CPU核心，但通过快速切换任务，让用户感觉多个程序在同时运行。

**并行(Parallelism)**

指在同一时刻，多个相同类型的任务同时执行，需要多核处理器。

如向量的点乘，矩阵乘法等，同一种类型的任务，利用多核去并行计算。

## 并发编程模型

### 线程（Threads）

最常见的并发模型，多个线程在同一个进程内并发执行。线程之间共享内存，但需要同步机制（如锁）来避免资源冲突。

以计数器为例，只用单线程只管往上加到n就完事了；而多线程每个线程累加1/n，等待所有线程做完再返回结果n（这个做法有点蠢但只是为了举例），为了避免数据冲突需要用互斥锁，最终开销反比单线程要大得多。

```go
func singleThreadCounter(n int) int {
	counter := 0
	for i := 0; i < n; i++ {
		counter++
	}
	return counter
}

func multiThreadCounter(n int, numGoroutines int) int {
	var counter int
	var mutex sync.Mutex
	var wg sync.WaitGroup

	incrementCounter := func() {
		defer wg.Done()
		for i := 0; i < n/numGoroutines; i++ {
			mutex.Lock()
			counter++
			mutex.Unlock()
		}
	}

	wg.Add(numGoroutines)
	for i := 0; i < numGoroutines; i++ {
		go incrementCounter()
	}
	wg.Wait()
	return counter
}
```

使用**Benchmark Testing**对两种做法进行性能分析，Profile with 'CPU Profiler'。

```go
import "testing"

const n = 1000000

func BenchmarkSingleThreadCounter(b *testing.B) {
	for i := 0; i < b.N; i++ {
		singleThreadCounter(n)
	}
}

func BenchmarkMultiThreadMutexCounter(b *testing.B) {
	const numGoroutines = 10
	for i := 0; i < b.N; i++ {
		multiThreadCounter(n, numGoroutines)
	}
}
```

结果如下

```go
goos: windows
goarch: amd64
cpu: 13th Gen Intel(R) Core(TM) i5-13400F
BenchmarkSingleThreadCounter
BenchmarkSingleThreadCounter-16    	    5262	    225,636 ns/op
BenchmarkMultiThreadCounter
BenchmarkMultiThreadCounter-16     	      22	  50,967,586 ns/op
```

![img.png](../../assets/goroutine/counter_cpu_profile.png)
单线程计数器的平均耗时为225,636纳秒（约0.225毫秒）。

多线程计数器的平均耗时为50,967,586纳秒（约50.967毫秒）。

由于多线程需要同步共享资源，因此有锁争用和线程切换的开销，导致时间开销远远高于单线程。

### C#协程（Coroutines）

更轻量级的并发模型，可以在单线程内实现并发。协程通过yield和resume的方式交替执行，避免了线程的上下文切换开销。

在C#中，协程主要用于异步编程。协程函数返回`IEnumerator`，并且可以使用`yield return`来暂停执行。每次调用`MoveNext()`，协程都会从上次暂停的地方继续执行，直到所有`yield return`语句都执行完毕。简单来说，协程让你可以在同一个线程内实现异步操作，而不用担心复杂的线程管理。

**Unity中的协程调度**

在Unity中，协程的调度由Unity引擎管理。每帧Unity都会检查是否有活跃的协程。如果某个协程达到了继续执行的条件，Unity会调用该协程的`MoveNext()`方法，从上次`yield return`的地方继续执行。这种机制允许在游戏循环中实现复杂的异步行为而不阻塞主线程。

这样，你可以让一些耗时的任务分布在多帧中进行，而不会卡住游戏的主线程。

**非并行**

需要注意的是，C#中的协程并不能实现真正的并行执行。C#运行时不会像Go运行时那样将协程分配到不同的线程上执行。因此，C#协程主要用于非阻塞异步操作，不能代替线程和任务来实现并行计算。

这意味着C#的协程本质上只是用于Unity中的非阻塞异步操作的工具。如果你需要真正的并行执行，仍然需要使用线程（Thread）和任务（Task）。

**并发性**

尽管如此，C#的协程可以实现并发调度。例如，使用`StartCoroutine()`启动多个协程，每个协程可以独立执行各自的任务，并在异步操作结束时继续执行。这种并发性使得在处理多个独立任务时非常方便，比如同时处理多个动画、多个网络请求等。

让我们来看一个实际的游戏场景，如何使用协程在FPS游戏中实现一种三连发的枪。鼠标点击一次就能发射三颗子弹，但每颗子弹之间有细小的时间间隔。假设我们已经有一个单发子弹的函数`FireSingleBullet`。

**示例代码：实现三连发枪**

```csharp
using System.Collections;
using UnityEngine;

public class ThreeRoundBurstGun : MonoBehaviour
{
    // 定义子弹之间的时间间隔（例如0.1秒）
    public float bulletInterval = 0.1f;

    // 引用单发子弹发射函数
    private void FireSingleBullet()
    {
        // 这个函数假设已经实现，会发射一颗子弹
        Debug.Log("Bullet Fired");
    }

    // 调用三连发函数
    public void FireBurst()
    {
        StartCoroutine(FireBurstCoroutine());
    }

    // 协程实现三连发
    private IEnumerator FireBurstCoroutine()
    {
        for (int i = 0; i < 3; i++)
        {
            FireSingleBullet();
            yield return new WaitForSeconds(bulletInterval);
        }
    }

    // 示例：检测鼠标点击来触发三连发
    void Update()
    {
        if (Input.GetMouseButtonDown(0))
        {
            FireBurst();
        }
    }
}
```

### JS事件驱动模型（Event-driven model）

通过事件循环处理并发任务，如JavaScript的异步编程。

JavaScript 是一种单线程语言，这意味着它同一时刻只能执行一个任务。然而，通过异步编程和事件循环机制，JavaScript 可以高效地处理多个任务。

**宏任务和微任务**

- **宏任务**：比如 `setTimeout`，在指定的时间到达后，任务会被添加到宏任务队列。
- **微任务**：比如 `Promise`，在 `then` 或 `catch` 回调完成后，任务会被添加到微任务队列。
- **宏任务队列**：存放一些较大的任务，比如 `setTimeout`，整体脚本`script`也属于宏任务。
- **微任务队列**：存放一些较小的任务，比如 `Promise` 和 `async function`。微任务在每个宏任务执行完后立即执行。

**事件循环的工作原理**

![img2.png](../../assets/goroutine/js_event_loop.png)

1. **执行宏任务**：事件循环从宏任务队列中取出第一个任务并执行。
2. **执行所有微任务**：在当前宏任务执行完毕后，事件循环会执行微任务队列中的所有任务，直到微任务队列为空。
3. **更新渲染**：如果有任何变更，浏览器会在此时更新渲染。
4. **休眠或等待新的任务**：如果宏任务队列为空，事件循环会休眠，直到有新的任务出现。
5. **重复步骤 1-4**。

**举个例子**

```javascript
console.log('script start')

setTimeout(() => {
	console.log('setTimeout')
}, 0)

Promise.resolve()
	.then(() => {
		console.log('promise1')
	})
	.then(() => {
		console.log('promise2')
	})

console.log('script end')
```

**执行流程**：

1. 执行同步代码，输出 'script start' 和 'script end'。
2. `setTimeout` 被添加到宏任务队列。
3. `Promise` 回调被添加到微任务队列。
4. 同步代码执行完毕，开始执行微任务队列，输出 'promise1' 和 'promise2'。
5. 最后，执行宏任务队列中的 `setTimeout`，输出 'setTimeout'。

通过事件循环，JavaScript 可以在单线程环境下高效地执行异步任务。宏任务和微任务队列的合理调度使得 JavaScript 能够在处理复杂任务的同时保持高响应性，确保用户体验流畅。
