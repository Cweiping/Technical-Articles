## 深入浅出 Channel 实现机制
### 使用样例
话不多说，先上一段生产者消费者代码来认识下 channel 的使用方式:
```Go
func main() {
	taskQueue := make(chan interface{}, 5)
	// start worker
	for i := 0; i < 5; i++ {
		go worker(taskQueue)
	}
	// send task to task queue
	for i := 0; i < 100; i++ {
		taskQueue <- i
	}
}

func worker(taskQueue <-chan interface{}) {
	for {
		// Receive task
		task := <-taskQueue
		process(task)
	}
}

func process(task interface{}) {
	// do something
}
```
以上代码即实现线程安全的生产与消费，
1. `main()` 函数中 `taskQueue := make(chan interface{}, 5)` 代码行，初始化长度为 5 的工作队列;
2. `go worker(taskQueue)` 表示启动工作协程(非操作系统线程)，4~6 行初始化完成五个消费者协程;
3. `taskQueue <- i` 表示生产一个任务，8~10 行完成 100 个任务的生产工作；
4. ` worker(taskQueue <-chan interface{})` 函数实现消费者协程的工作内容，持续接收所有任务，每当任务派发到队列中，消费者协程通过 `<-taskQueue` 接收任务并通过 ` process(task interface{})`函数完成消费动作；

### 特性

通过以上代码我们可以了解到 Channel 的一些特性：
1. 它是线程安全的；
2. 它在协程之间传递消息；
3. 它是 FIFO 的数据结构；
4. 会引起协程阻塞和解除阻塞；

现在我们了解到 Channel 正确使用方式和特性，接下来让我们来撕开伪装揭露真相。

##### 1. 创建 Channel（hchan 数据结构）

首先，你需要使用 `make(chan interface{}, 5)` 内置函数来创建一个缓存容量为 5 的 Channel,当然你也可以使用 ` 或 make(chan interface{})`创建一个不带缓存的。接下来我们主要讨论带缓存的 Channel，下面是我们创建的 Channel 内存结构：![带缓存的 Channel]( https://images.contentful.com/le3mxztn6yoo/65EmHei252GgU8CoWUag0S/45204cd810750733d6dbbb9a3f60c1a6/Selection_059.png)
我们在堆内存中创建就 hchan 数据结构，并返回指针。因此我们才可以通过 `taskQueue <- i` 和 `task := <-taskQueue` 使用它。

##### 2. Channel 的发送和接收（hchan 数据结构）
`main()`自身就是一个生产者协程，通过 10~12 行代码生产任务并放入队列 `taskQueue` 中，我们通过 `go worker(taskQueue)`启动的 5
个消费消费者来完成消费任务，整个生产和消费过程都是线程安全的，不会存在数据紊乱问题。当`taskQueue`满载时，生产者协程会被挂起；
同理，消费者线程也会被挂起.这一行为是由 Go 的 运行时调度器完成的，协程也是属于用户态的实现，被 Go
的运行时调度器管理，和操作系统的线程相比它更加的轻量化，因此并发度也能进一步提高。Go
的调度器专门用于调度协程和操作系统线程的线程，比例为（1：N），即一个线程对应多个协程，调度器管理的线程数量取决于用户分配的数量，默认等于CPU核心数。下面是GMP模型的图示:![G-M-P模型](https://images.contentful.com/le3mxztn6yoo/16s9byzycQoyCoyyC6OUKy/1d67eb5b8de87a2a14eaf47219573820/Selection_061.png)
当 Channel 的数据满载时，生产者协程因为无法继续生产资源而被挂起，Channel 将会通知调度器挂起某个协程，此时调度器会将这个协程移出`runQ`队列，并改变其状态为等待中.通过这种方式调度器已经完成上下文的切换，而不需要挂起操作系统线程，成本非常低。当消费者协程完成消费后，Channel 不在数据满载，此时会通知调度器，将此前等待的生产者协程重新置为可运行状态，并加入`runQ`队列等待运行。

##### 3. 为什么要这样实现？
Go语言设计者主要考虑两点：
1. 简单性：带锁队列优先于无锁实现。
2. 性能：goroutine唤醒路径是无锁的，并且可能更少的内存副本。
在通道的实现中，在简单性和性能之间有一个精明的权衡。

### 附录
##### 参考链接
1. https://about.sourcegraph.com/go/understanding-channels-kavya-joshi/
2. https://zhuanlan.zhihu.com/p/27917262

##### 待办事项
1. 需要收集golang 官网设计文档和 Blog

