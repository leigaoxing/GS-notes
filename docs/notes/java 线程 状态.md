# Java 线程的状态

```java
public enum State {
        // new 新的线程实例，但是没有调用 start() 方法
        NEW,

    	// 可运行的线程。可运行的线程 可以被 Java虚拟机运行，
    	// 但是要等待其他的资源，比如 处理器 CPU。
        RUNNABLE,

    	// 线程阻塞等待监控锁。在此状态下，线程等待一个监控锁，
    	// 以进入 synchronized 块/方法。或者重新进入 调用
    	// {@link Object#wait() Object.wait} 之后的
    	// synchronized 块/方法。
        BLOCKED,

    	// 对于 一个等待线程的状态。一个线程处于一个等待状态
    	// 由于调用了以下方法:
    	// {@link Object#wait() Object.wait} 不带超时时间
    	// {@link #join() Thread.join}  不带超时时间
    	// {@link LockSupport#park() LockSupport.park}
    	// 线程在等待状态等待另一个线程执行特定的操作。
    	// 比如 一个线程调用了 Object.wait()等待另一个线程调用
    	// Object.notify() 或者 Object.notifyAll() 
    	// 线程 调用 Thread.join() 等待指定的线程结束。
        WAITING,
    
		// 带有指定等待时间的等待线程的状态。
    	// 线程处于有时限的等待状态，因为调用了下面指定了
    	// 正数的等待时间，
    	// {@link #sleep Thread.sleep}
    	// {@link Object#wait(long) Object.wait} 带有超时时间
    	// {@link #join(long) Thread.join} 带有超时时间
    	// {@link LockSupport#parkNanos          			         //                     LockSupport.parkNanos}
    	// {@link LockSupport#parkUntil 				             //                     LockSupport.parkUntil}
        TIMED_WAITING,

    	// 终止线程。线程已经完成执行。
        TERMINATED;
    }
```

### Java中线程的6种状态：

1. 初始 (NEW): 新创建了一个线程对象，但是没有执行 start() 方法。
2. 可运行 (RUNNABLE): Java 线程中将 就绪 (ready) 和运行中 (running) 两种状态笼统称为 “可运行”。线程对象创建后，其他线程 （比如 main 线程） 调用了该方法的 start() 方法。该状态的线程位于 可运行的线程池中，等待线程调度选中，获得 CPU 的使用权，此时处于就绪状态 （ready）。就绪状态的线程在获得 CPU 时间片后变为运行中状态 （running）。
3. 阻塞 (BLOCKED): 表示线程阻塞于锁。
4. 等待 (WAITING): 进入该状态 的线程需要等待其他线程做出一些特定动作 (通知或中断)。
5. 超时等待(TIMED_WAITING): 该状态不同于 WAITING，它可以在指定的时间后自行返回。
6. 终止 (TERMINATED): 表示线程已经执行完毕。

### 线程状态图

![](pics\thread-state.jpeg)



### 1. 初始状态

实现 Runnable 接口和继承 Thread 可以得到一个线程类，new 一个实现出来，线程就进入了初始状态。

### 2. 可运行状态

#### 2.1 就绪状态

1. 就绪状态只是说你有资格运行，调度程序没有挑选到你，你就永远是就绪状态。
2. 调用线程的 start() 方法，此线程进入就绪状态。
3. 当前线程 sleep() 方法结束，其他线程 join() 结束，当代用户输入完毕，摸个线程拿到对象锁，这些线程也将进入就绪状态。
4. 当前线程时间片用完了，调用当前线程的 yield()方法，当前线程进入就绪状态。
5. 锁池里的线程拿到对象锁后，进入就绪状态。

#### 2.2 运行中状态

线程调度程序从可运行池中选择一个线程作为当前线程时，线程所处的状态。这也是线程进入运行状态的唯一一种方式。

### 3.阻塞状态

阻塞状态是线程阻塞在进入 synchronized 关键字修饰的方法或代码块时的状态。

### 4. 等待

处于这种状态的线程不会被分配 CPU 执行时间，它们要等待被显示地唤醒，否则会处于无线等待的状态。

### 5. 超时等待

处于这种状态的线程不会被分配 CPU 时间片，不过无须无限期等待被其他线程显示地唤醒，在达到一定时间后它们会自动唤醒。

### 6. 终止状态

1. 当线程的 run() 方法完成时，或者主线程的 main() 方法完成时，我们就认为它终止了。这个线程对象也许是活的，但是，它已经不是一个单独运行的线程。线程一旦终止，就不会复生。
2. 在一个终止的线程上调用 start() 方法，会抛出 java . lang . IllegalThreadStateException 异常。

### 等待队列

调用 obj 的wait() ，notify() 方法前，必须获得 obj 锁，也就是必须写在 synchronized(obj) 代码块内。

与等待队列相关的步骤和图：

![](pics\java-thread-waiting-queue.jpg)

1. 线程1获取对象A的锁，正在使用对象A。
2. 线程1调用对象A的wait()方法。
3. 线程1释放对象A的锁，并马上进入等待队列。
4. 锁池里面的对象争抢对象A的锁。
5. 线程5获得对象A的锁，进入synchronized块，使用对象A。
6. 线程5调用对象A的notifyAll()方法，唤醒所有线程，所有线程进入同步队列。若线程5调用对象A的notify()方法，则唤醒一个线程，不知道会唤醒谁，被唤醒的那个线程进入同步队列。
7. notifyAll()方法所在synchronized结束，线程5释放对象A的锁。
8. 同步队列的线程争抢对象锁，但线程1什么时候能抢到就不知道了。

### 同步队列的状态

1. 当前线程调用对象A的同步方法时，发现对象A的锁被别的线程占有，此时当前线程进入同步队列。简言之，同步队列里面放的都是想争夺对象锁的线程。
2. 当一个线程1被另外一个线程2唤醒时，1线程进入同步队列，去争夺对象锁。
3. 同步队列是在同步的环境下才有的概念，一个对象对应一个同步队列。
4. 线程等待时间到了或被 notify/notifyAll 唤醒后，会进入同步队列竞争锁，如果获得锁， 进入 RUNNABLE 状态，否则 进入 BLOCKED 状态等待获取锁。

### 几个方法的比较

1. Thread.sleep(long millis)，一定是当前线程调用此方法，当前线程进入TIMED_WAITING状态，但**不释放对象锁**，millis后线程自动苏醒进入就绪状态。**作用**：给其它线程执行机会的最佳方式。
2. Thread.yield()，一定是当前线程调用此方法，当前线程放弃获取的CPU时间片，但**不释放锁资源**，由运行状态变为就绪状态，让OS再次选择线程。**作用**：让相同优先级的线程轮流执行，但并不保证一定会轮流执行。实际中无法保证yield()达到让步目的，因为让步的线程还有可能被线程调度程序再次选中。Thread.yield()不会导致阻塞。该方法与sleep()类似，只是不能由用户指定暂停多长时间。
3. Thread.join()/thread.join(long millis)，当前线程里调用其它线程t的join方法，当前线程进入WAITING/TIMED_WAITING状态，当前线程**不会释放已经持有的对象锁**。线程t执行完毕或者millis时间到，当前线程一般情况下进入RUNNABLE状态，也有可能进入BLOCKED状态（因为join是基于wait实现的）。
4. obj.wait()，当前线程调用对象的wait()方法，当前线程释放对象锁，进入等待队列。依靠notify()/notifyAll()唤醒或者wait(long timeout) timeout时间到自动唤醒。
5. obj.notify() 唤醒在此对象监视器上等待的单个线程，选择是任意性的。obj.notifyAll() 唤醒在此对象监视器上等待的所有线程。
6. LockSupport.park()/LockSupport.parkNanos(long nanos),LockSupport.parkUntil(long deadlines), 当前线程进入WAITING/TIMED_WAITING状态。对比wait方法,不需要获得锁就可以让线程进入WAITING/TIMED_WAITING状态，需要通过LockSupport.unpark(Thread thread)唤醒。(**疑问：park是否需要获得锁**)

### 疑问

等待队列里许许多多的线程都wait()在一个对象上，此时某一线程调用了对象的notify()方法，那唤醒的到底是哪个线程？随机？队列FIFO？or sth else？Java文档就简单的写了句：选择是任意性的（The choice is arbitrary and occurs at the discretion of the implementation）。

### 实现多线程的三种方式

- 继承 Thread 类
- 实现 Runnable 接口
- 实现 Callable 接口

### 线程的描述

- 线程是比进程更轻量级的调度执行单位，CPU 调度的基本单位就是线程。
- 线程的引入，将一个进程的资源分配和执行调度分开。
- 各个线程既可以共享进程资源（内存地址、文件 I/O 等），又可独立调度。
- **Java 的 Thread 类：** 所有关键方法都是 Native 的，说明这些方法无法使用平台无关的手段实现。

### 实现线程的 3 种方式

- 使用内核线程实现
- 使用用户线程实现
- 使用用户线程加轻量级进程

### 线程的调度

- **协同式线程调度：**

   

  线程的执行时间由线程本身来控制，线程执行完自己的任务之后，主动通知系统切换到另一个线程。

  - 优点：实现简单，没有线程同步的问题。
  - 缺点：线程执行时间不可控，如果一个线程编写有问题一直无法结束，程序会一直阻塞在那里。

- **抢占式线程调度：** 每个线程由系统分配执行时间，系统决定切不切换线程。

## 确定线程池的线程数

2 * Ncpu



### 参考

[Java线程的6种状态及切换(透彻讲解)](https://blog.csdn.net/pange1991/article/details/53860651/)

[Java-Concurrency-in-Practice](https://github.com/TangBean/Java-Concurrency-in-Practice)

[[根据CPU核心数确定线程池并发线程数](https://www.cnblogs.com/dennyzhangdd/p/6909771.html)























