# JUC

## 并发工具类的分类

1. 为了**并发安全**：
    
    * 互斥同步

        * 使用各种互斥同步的**锁**

            * synchronized

            * ReentrantLock
            
            * ReadWriteLock

            * ...

        * 使用互斥同步的**工具类**

            * Collections.synchronizedList(new ArrayList<>())

            * Vector 等
    
    * 非互斥同步

        * Atomic 包，原子类

            * Atomic* 基本数据类型原子类

                * AtomicInteger：整型原子类

                * AtomicLong：长整型原子类

                * AtomicBoolean：布尔类型原子类

            * Atomic*Array 数组类型原子类（ 数组中的元素同样可以保证原子性 ）

                * AtomicIntegerArray：整型数组原子类

                * AtomicLongArray：长整型数组原子类

                * AtomicReferenceArray：引用类型数组原子类

            * Atomic*Reference 引用类型原子类

                * AtomicReference：引用类型原子类

                * AtomicStampedReference：引用类型原子类的升级，附带时间戳，可以解决 ABA 问题

                * AtomicMarkableReference

            * Atomic*FieldUpdater 升级原子类

                * 用 Atomic*FieldUpdater 升级自己的变量

                * AtomicIntegerFieldUpdater：原子更新整型字段的更新器

                * AtomicLongFieldUpdater：原子更新长整型字段的更新器

            * Adder 加法器

                * LongAdder

                * DoubleAdder

            * Accumulator 累加器

                * LongAccumulator

                * DoubleAccumulator
    
    * 结合互斥和非互斥同步

        * 线程安全的并发容器

            * ConcurrentHashMap

            * CopyOnWriteArrayList

            * ConcurrentSkipListMap 和 ConcurrentSkipListSet

            * 并发队列

                * 阻塞队列

                    * ArrayBlockingQueue

                    * LinkedBlockingQueue

                    * PriorityBlockingQueue

                    * SynchronousQueue

                    * DelayedQueue

                    * TransferQueue

                    * ...

                * 非阻塞队列

                    * ConcurrentLinkedQueue

    * 无同步方案、不可变

        * 线程封闭

            * ThreadLocal

            * 栈封闭

        * final 关键字

2. **管理**线程、提高**效率**

    * 线程池相关类

        * Executor

        * Executors

        * ExecutorService

        * 常见线程池

            * FixedThreadPool

            * CachedThreadPool

            * ScheduledThreadPool

            * SingleThreadExecutor

            * ForkJoinPool

            * ...

    * 能获取子线程的运行结果

        * Callable

        * Future

        * FutureTask

        * ...

3. 线程**协作**

    * CountDownLatch

    * CyclicBarrier

    * Semaphore

    * Condition

    * Exchanger

    * Phaser

## 线程池

### 创建和停止线程池

#### 线程池构造函数的参数

1. corePoolSize

2. maxPoolSize

3. keepAliveTime

4. workQueue

5. threadFactory

6. Handler

#### 线程池应该手动创建还是自动创建

1. newFixedThreadPool

由于传进去的 LinkedBlockingQueue 是没有容量上限的，所以当请求数量越来越多时，并且无法即使处理完毕的时候，即发生了请求堆积，容易造成占用大量的内存，可能会导致 OOM 问题。

2. newSingleThreadExecutor

#### 线程池里的线程数量设定为多少比较合适？

* CPU 密集型（ 加密、计算哈希等 ）：最佳线程数为 CPU 核心数的 1~2 倍左右。

* 耗时 IO 型（ 读写数据库、文件、网络等 ）：最佳线程数一般会大于 CPU 核心数很多倍，以 JVM 线程监控繁忙情况为依据，保证线程空闲可以衔接上，参考 Brain Goetz 推荐的计算方法：

> 线程数 = CPU 核心数 *（ 1 + 平均等待时间 / 平均工作时间 ）

#### 停止线程池的正确方法

1. shutdown

2. isShutdown

3. isTerminated

4. awaitTermination

5. shutdownNow