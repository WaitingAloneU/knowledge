# 多线程

## 常用的几种线程池
### 单线程池
```java
// 创建方式：
Executors.newSingleThreadExecutor();
// 实现过程：固定核心线程数和最大线程数为1，
public static ExecutorService newSingleThreadExecutor() {
    return new FinalizableDelegatedExecutorService
        (new ThreadPoolExecutor(1, 1, 0L, TimeUnit.MILLISECONDS, new LinkedBlockingQueue<Runnable>()));
}
```
### 计划线程池
```java
// 创建方式：        
ScheduledExecutorService ses = Executors.newScheduledThreadPool(6);
// 5秒以后执行
ses.schedule(new BizThread(), 5000, TimeUnit.MILLISECONDS);
// 创建过程：
public static ScheduledExecutorService newScheduledThreadPool(int corePoolSize) {
    return new ScheduledThreadPoolExecutor(corePoolSize);
}
//ScheduledThreadPoolExecutor 继承 ThreadPoolExecutor
public ScheduledThreadPoolExecutor(int corePoolSize) {
    super(corePoolSize, Integer.MAX_VALUE, 0, NANOSECONDS, new DelayedWorkQueue());
}
```
### 缓存线程池
- 该线程池的队列（SynchronousQueue）中不存放任何数据，只做同步功能。所以，当有任务需要执行的时候，通过同步功能以后，直接开辟核心线程数（0）以外的线程
```java
// 创建方式：
Executors.newCachedThreadPool();
// 设置核心线程数为0，最大线程数为int最大值：2147483647
public static ExecutorService newCachedThreadPool() {
    return new ThreadPoolExecutor(0, Integer.MAX_VALUE, 60L, TimeUnit.SECONDS,
                                    new SynchronousQueue<Runnable>());
}
```
### 固定线程池
```java
// 创建方式：
Executors.newFixedThreadPool(6);
public static ExecutorService newFixedThreadPool(int nThreads) {
    return new ThreadPoolExecutor(nThreads, nThreads, 0L, TimeUnit.MILLISECONDS,
                                    new LinkedBlockingQueue<Runnable>());
}
```

- 总结： 以上代码可以看出，所有的类型的线程池都是用ThreadPoolExecuter创建，只是传参的不同。

## 线程池 ThreadPoolExecutor

### 构造函数
```java
public class ThreadPoolExecutor extends AbstractExecutorService {
    public ThreadPoolExecutor(int corePoolSize,int maximumPoolSize,long keepAliveTime,TimeUnit unit, BlockingQueue<Runnable> workQueue);
 
    public ThreadPoolExecutor(int corePoolSize,int maximumPoolSize,long keepAliveTime,TimeUnit unit, BlockingQueue<Runnable> workQueue,ThreadFactory threadFactory);
 
    public ThreadPoolExecutor(int corePoolSize,int maximumPoolSize,long keepAliveTime,TimeUnit unit, BlockingQueue<Runnable> workQueue,RejectedExecutionHandler handler);
 
    public ThreadPoolExecutor(int corePoolSize,int maximumPoolSize,long keepAliveTime,TimeUnit unit, BlockingQueue<Runnable> workQueue,ThreadFactory threadFactory,RejectedExecutionHandler handler);
}
```
- 以上四种构造器，前三种最终都是调用的第四种。 只是设置了默认的ThreadFactory和RejectedExecutionHandler
```
- corePoolSize: 核心线程数
- maximumPoolSize: 最大线程数
- keepAliveTime：线程存活时间
- unit：线程存活时间的单位
- workQueue：队列
- threadFactory：线程工厂
- handler：表示当拒绝处理任务时的策略
```
- ThreadFactory：线程工厂， 创建线程，源码中使用的默认线程工厂为：Executors.DefaultThreadFactory
```java
this(corePoolSize, maximumPoolSize, keepAliveTime, unit, workQueue, Executors.defaultThreadFactory(), defaultHandler);
```
- RejectedExecutionHandler：表示当拒绝处理任务时的策略, 有以下四种取值：
```java
ThreadPoolExecutor.AbortPolicy: 丢弃任务并抛出RejectedExecutionException异常。 (默认值)
ThreadPoolExecutor.DiscardPolicy: 也是丢弃任务，但是不抛出异常。 
ThreadPoolExecutor.DiscardOldestPolicy: 丢弃队列最前面的任务，然后重新尝试执行任务（重复此过程）
ThreadPoolExecutor.CallerRunsPolicy: 由调用线程处理该任务 
```
## 线程安全

### 多线程的问题
#### 原子性问题
- 原子性是指一个操作或者多个操作要么全部执行并且执行的过程不会被任何因素打断，要么就都不执行。（与事务的原子性概念相同）
#### 可见性问题
- 可见性是指当多个线程访问同一个变量时，一个线程修改了这个变量的值，其他线程能够立即看得到修改的值。
```
例：
//线程1执行的代码
int i = 0;
i = 10;
 
//线程2执行的代码
j = i;
```
假若执行线程1的是CPU1，执行线程2的是CPU2。由上面的分析可知，当线程1执行 i =10这句时，会先把i的初始值加载到CPU1的高速缓存中，然后赋值为10，那么在CPU1的高速缓存当中i的值变为10了，却没有立即写入到主存当中。此时线程2执行 j = i，它会先去主存读取i的值并加载到CPU2的缓存当中，注意此时内存当中i的值还是0，那么就会使得j的值为0，而不是10。这就是可见性问题，线程1对变量i修改了之后，线程2没有立即看到线程1修改的值。
#### 有序性问题
- 在Java中看似顺序的代码在JVM中，可能会出现编译器或者CPU对这些操作指令进行了重新排序，重排序的规则：在单线程的情况下不改变程序的运行结果。在多线程的情况下，指令重排将会给我们的程序带来不确定的结果。下面看一个例子：
```
//线程1:
context = loadContext();   //语句1
inited = true;             //语句2
 
//线程2:
while(!inited ){
  sleep()
}
doSomethingwithconfig(context);
```
上面代码中，由于语句1和语句2没有数据依赖性，因此可能会被重排序。假如发生了重排序，在线程1执行过程中先执行语句2，而此是线程2会以为初始化工作已经完成，那么就会跳出while循环，去执行doSomethingwithconfig(context)方法，而此时context并没有被初始化，就会导致程序出错。从上面可以看出，指令重排序不会影响单个线程的执行，但是会影响到线程并发执行的正确性。

**注意：要想并发程序正确地执行，必须要保证原子性、可见性以及有序性。只要有一个没有被保证，就有可能会导致程序运行不正确。**

### volatile 关键字
- 保证了不同线程对这个变量进行操作时的可见性，即一个线程修改了某个变量的值，这新值对其他线程来说是立即可见的。
- 禁止进行指令重排序。
#### volatile原理

下面这段话摘自《深入理解Java虚拟机》：

“观察加入volatile关键字和没有加入volatile关键字时所生成的汇编代码发现，加入volatile关键字时，会多出一个lock前缀指令”

lock前缀指令实际上相当于一个内存屏障（也成内存栅栏），内存屏障会提供3个功能：

- 它确保指令重排序时不会把其后面的指令排到内存屏障之前的位置，也不会把前面的指令排到内存屏障的后面；即在执行到内存屏障这句指令时，在它前面的操作已经全部完成；

- 它会强制将对缓存的修改操作立即写入主存；

- 如果是写操作，它会导致其他CPU中对应的缓存行无效。

以i = i + 1为例：下图为不加volatile关键字的执行逻辑，此逻辑在多线程环境中，会存在一定的问题。比如多个线程读取的时候都读取的是i，但是写的时候也就都写得是i+1。并没有达到预期的效果。

```mermaid
sequenceDiagram
participant a as jvm主内存
participant b as 线程工作内存
participant c as CPU高速缓存
participant d as CPU

a-->>b:从主存复制数据i到当前线程工程内存
b-->>c:从工作内存读取到cpu高速缓存
c-->>d:cpu读取高速缓存
d-->>d:cpu计算i+1
d-->>c:结果写到高速缓存
c-->>b:结果写到工作内存
b-->>a:结果写到主存
```
就算加了volatile修饰i以后，省去了CPU高速缓存（Intel的MESI协议）以及工作内存的写过程，直接进行住内存的写，也不能保证完全没有问题。原因有以下两点：
- volatile只是解决了可见性以及禁止指令重排的问题，并没有保证原子性。 因为i = i + 1, 这个过程不是原子性的。
- 保证可见性的结果是可以读到，但是下面这种情况无法避免：线程1读取i，未进行+1操作，此时线程2读取，读到的还是为修改的i，此时线程1进行+1，再写回去，接下来就是线程2也是+1，写回去。此种情况是无法避免的。

为了解决上面的这个问题可以选择i加volatile的同时，再加上i的类型改为AtomicInteger，再使用AtomicInteger的原子性的自增方法来确保线程安全。

可以参考设计模式中的[单例模式（双check模式）](https://github.com/WaitingAloneU/knowledge/blob/master/DESIGN/Design.md).

此文章需要读很多遍，反复的读，好文章记录一下：
http://www.cnblogs.com/dolphin0520/p/3920373.html


### Synchronized 关键字
- 修饰方法
- 修饰静态方法
- 修饰代码块
### 锁
#### ReentrantLock
#### ReadLock
#### WriteLock

## Callable & Future