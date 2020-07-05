# JVM 内存结构 vs Java 内存模型 vs Java 对象模型

## 整体方向

JVM 内存结构，和 Java 虚拟机的运行时区域有关。

Java 内存模型，和 Java 的并发编程有关。

Java 对象模型，和 Java 对象在虚拟机中的表现形式有关。

## JVM 内存结构

运行时数据区（ Runtime Data Area ）：

1. 运行时数据区在所有线程间共享：堆（ Heap ）和方法区（ Method Area ）。

方法区：

* 用于存储 JVM 已经加载的类信息、static 变量以及常量。

*  还用于存放永久引用，例如 static Person p = new Person(); 的 p 应用。

* 运行时常量池

2. 运行时数据区线程私有：虚拟机栈（ VM Stack ）、本地方法栈（ Native Method Stack ）和程序计数器（ Program Counter Register ）。

## Java 对象模型（ JOM ）

* Java 对象自身的存储模型

* JVM 会给这个类创建一个 instanceKlass，并保存在方法区中，用来在 JVM 层表示这个 Java 类。

* 当我们在 Java 代码中使用 new 创建一个对象的时候，JVM 会创建一个 instanceOopDesc 对象，这个对象包含了对象头以及实例数据。

## Java 内存模型（ JMM ）

* 是规范

* 是工具类和关键字的原理

* 最重要的 3 点内容：重排序、可见性和原子性。

### 重排序的 3 种情况

#### 编译器优化

包括 JVM、JIT 编译器等。例如当前有了数据 a，那么如果把对 a 的操作放到一起效率会更高，避免了读取 b 后又返回来重新读取 a 的时间开销。在编译的过程中会进行一定程度的重排，导致生成的机器指令和之前的字节码的顺序不一致。


#### CPU 指令重排

就算编译器不发生重排序，CPU 也可能对指令进行重排序。CPU 的优化行为和编译器优化很类似，是通过乱序执行的技术来提高执行效率。所以就算编译器不发生重排，CPU 也可能对指令进行重排。

#### 内存的“重排序”

内存系统内不存在重排序，但是内存会带来看上去和重排序一样的效果，所以这里的“重排序”打了双引号。由于内存有缓存的存在，在 JMM 里表现为主存和本地内存，由于主存和本地内存的不一致，会使得程序表现出乱序的行为。在刚才的例子中，假设没发生编译器重排和指令重排，但是如果出现了内存缓存不一致的情况，也可能导致同样的情况：线程 1 修改了 a 的值，但是修改后并没有写回主存，所以线程 2 是看不到刚才线程 1 对 a 的修改，所以线程 2 看到 a 的值还是等于 0。同理，线程 2 对 b 的赋值操作也可能由于没及时写回主存，导致线程 1 看不到刚才线程 2 的修改。

### 可见性

#### 产生可见性问题的原因？

1. CPU 存在多级缓存，导致读取到的数据过期。

2. 高速缓存的容量比主内存小，但是速度仅次于寄存器，所以在 CPU 和主内存之间就多了数层的 Cache 层。

3. 线程间对于共享变量的可见性问题不是直接由多颗核心引起的，而是由多缓存引起的。每颗核心都会将自己需要的数据读取到独占缓存中，待数据修改完成后也是先写入独占缓存中，然后等待刷入到主内存中，所以会导致有些核心读取到的值是一个过期的值。

#### Happens-Before 原则

在时间上，动作 A 发生在动作 B 之前，B 保证能看见 A。
在执行顺序上，如果一个操作 Happens-Before 于另一个操作，那么我们说第一个操作对于第二个操作是可见的。

1. 单线程语义

该原则是由工作内存所决定的，在线程私有的工作内存中，先执行的语句是一定能被后执行的语句所感知的。即使发生了重排序，排在后面的语句同样也能感知到前面语句的操作。

2. 锁操作（ synchronized 和 lock ）

如果一个线程对锁解锁了，另外一个线程对锁加锁了，那么这个时候加锁的线程是一定能看到另一个线程解锁前的所有操作。

3. volatile 变量

如果 A 是对于被 volatile 修饰变量的写操作，B 是对同一个变量的读操作，那么 hb(A, B)。

4. 线程的启动

子线程所执行的所有语句都能看见在它被启动时主线程之前所有语句的执行结果。

```java
public class Main {

    private int count = 0;

    public static void main(String[] args) {
        for (int i = 0; i < 100; i++) {
            count += i;
        }

        System.out.println(count);

        Thread thread = new Thread(new ThreadRunable());
        thread.start();
    }

    static class ThreadRunable implements Runnable {
        public void run() {
            System.out.println(count);
        }
    }
}
```

5. 线程的 join 方法

6. 线程的 interrupt 方法

一个线程被其他线程 interrupt 时，那么检测中断（ isInterrupt ）或者抛出 InterruptedException 是一定能看到的。

7. 传递性

如果 hb(A, B) 而且 hb(B, C)，那么可以推出 hb(A, C)。

8. 工具类的 Happens-Before 原则

* 线程安全的容器在执行 get 操作时一定能看到在此之前的 put、remove 等一系列操作。

* CountDownLatch

* Semaphore

* CyclicBarrier

* Future

* 线程池

用 submit 方法提交任务之前，对于每一个任务都可以看到在此之前的所有执行结果。

#### **核心**

线程 X 如果写入一个 volatile 变量，此时线程 Y 再读取这个变量，那么此时对于线程 X 可见的所有属性对于线程 Y 都是可见的。（ _单线程语义、volatile、传递性_ ）

```bash
假设线程 X 写入 a 和 b 两个 non-volatile 变量，然后写入 volatile C 变量；线程 Y 再去读 volatile C：
hb("线程 X 对 a,b 的写入", "线程 X 对 volatile C 写入")，这个是由单线程语义决定的；
hb("线程 X 对 volatile C 写入", "线程 Y 读取 volatile C")，这个是由 volatile 保证的；
根据这两点，可以推出：hb("线程 X 对 a,b 的写入", "线程 Y 读取 volatile C"）。
```

### volatile

#### 是什么？

* volatile 是一种**同步机制**，比起 synchronzied 或者 lock 相关的类更轻量，因为使用 volatile 并不会发生**上下文切换**这类开销很大的行为。

* 但是开销小，相应的能力也小，虽然说 volatile 是用来同步保证线程安全的，但是 volatile 做不到 synchronized 那样的原子保护，volatile 仅在很有限的场景下才能发挥作用。

#### 适用场合

* 不适用：i++

* 适用：

    1. boolean flag，如果一个共享变量自始至终只被各个线程**赋值**，而没有其他操作，那么就可以使用 volatile 来代替 synchronized 或者 Atomic 变量，因为赋值操作本身就是原子性的，而 volatile 又可以保证可见性，因此就能保证线程安全。

    2. 作为刷新之前变量的**触发器**，实现轻量级的同步（ 近朱者赤 ）。

```java
Map configOptions;
char[] configText;
volatile boolean initialized = false;

// Thread A
configOptions = new HashMap();
configText = readConfigFile(fileName);
processConfigOptions(configText, configOptions);
initialized = true;

// Thread B
while (!initialized) {
    sleep();
    // use configOptions
}
```

#### 作用

可见性：读取一个 volatile 变量前，需要先使相应的本地缓存失效，这样就必须到主内存中读取该变量的最新值；写一个 volatile 变量后会立即刷写到主内存中。

禁止重排序：用于解决单例模式的经典问题：双重检验锁。

#### 和 synchronized 的关系？

volatile 可以看作是轻量版的 synchronized。

### synchronized

synchronized 不仅防止了一个线程在操作某个共享变量时被其他线程干扰的情况，即保证了线程的安全性。同时修改完成后，可以立即被其他线程所看到，即保证了内存可见性。

### 原子性

#### 什么是原子性？

一系列的操作，要么全部执行成功，要么全部不执行，不会出现执行到一半的情况，它们是不可分割的。

i++ 操作可以分为三个步骤：load -> add -> save。

#### Java 中的原子操作有哪些？

* 除了 long 和 double 之外的基本数据类型的赋值操作。

* 所有引用类型的赋值操作，不管是 32 位还是 64 位的机器。

* java.concurrent.Atomic.* 包中所有类的原子操作。

#### long 和 double 的原子性

问题描述：对于 64 位的值的写入，是分为两个 32 位的操作执行的，可以使用 volatile 来解决（ Oracle 的神操作，强行将其变成了 JVM 的规范 ）。

结论：在 32 位的 JVM 上，long 和 double 的操作不是原子性的，但是在 64 位的 JVM 上是原子性的。

#### 原子操作 + 原子操作 ！= 原子操作

简单地把原子操作组合在一起，并不能保证整体依然具有原子性。

全同步的 HashMap 也不完全是线程安全的。

## 面试常见问题

1. JMM 应用实例，单例模式的 8 种写法、单例和并发的关系（ 极高频 ）。

    * 作用：节省内存和计算、保证结果正确（ 统计页面访问人数 ）、方便管理（ 日志工具类 ）

    * 适用场景：全局信息类、无状态的工具类

    * 写法

        * 饿汉式（ 静态常量 ）【可用】

        * 饿汉式（ 静态代码块 ）【可用】

        * 懒汉式（ 线程不安全 ）【不可用】

        * 懒汉式（ 线程安全，同步方法 ）【不推荐用】

        * 懒汉式（ 线程不安全，同步代码块 ）【不可用】

        * 懒汉式（ **双重校验** ）【推荐面试使用】

            * 为什么要使用双重校验？（ 保证线程安全 ）

            * 单次校验行不行？（ 同步代码块会导致创建出多个实例，同步方法执行效率较低 ）

            * 为什么要使用 volatile？（ 创建实例实际上包含 3 个步骤，重排序可能会带来 NPE 问题，volatile 用来禁止重排序 ）
        
        * 懒汉式（ 静态内部类 ）【推荐用】

        * 枚举【推荐实际生产使用】

            * 不仅能避免多线程的同步问题，还能实现延迟加载，而且还能防止反序列化重新创建实例。
    
    * 饿汉式的缺点：没有延迟加载

    * 懒汉式的缺点：可能无法保证线程安全性

2. 创建实例的过程是不是原子操作？（ 不是 ）

    * 创建一个空的 Person 实例；

    * 把这个实例的内存地址指向引用 p；

    * 执行 Person 类的构造方法。