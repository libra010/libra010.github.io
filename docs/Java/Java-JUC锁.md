Java JUC锁



## 对比synchronized锁

synchronized 是 Java 语法层面的互斥锁，由 JVM 直接实现，其底层依赖于 Monitor（监视器）机制。

从 Java 5 开始，Java 引入了 java.util.concurrent（简称 JUC）包中的一套全新锁机制，被称为 JUC 锁。

与 synchronized 依赖 Monitor 不同，绝大多数 JUC 锁都是基于 AQS（AbstractQueuedSynchronizer，抽象队列同步器）实现的。AQS 是由 Java 编写的同步框架，通过维护一个表示资源状态的 status 字段和一个线程等待队列，抽象并实现了线程的阻塞、唤醒以及同步状态管理等核心机制。

AQS 的底层基于 Unsafe 提供的 CAS 操作保证状态修改的原子性，通过 volatile 变量确保可见性，并利用 LockSupport 的 park()/unpark() 实现线程的高效阻塞与唤醒，从而构建出高性能的同步框架。



ReentrantLock 是基于 AQS 实现的可重入互斥锁，它在设计上借鉴并模拟了 synchronized 的核心机制，例如支持线程的可重入性，并默认采用非公平锁策略。相较于 synchronized，ReentrantLock 更加的灵活，如支持带超时的锁获取（tryLock）、可中断的锁等待（lockInterruptibly）、以及显式地选择公平锁或非公平锁模式。

虽然 synchronized 在功能和灵活性上不及 ReentrantLock，但它作为 Java 的语法级关键字，由 JVM 直接支持，无需像 ReentrantLock 那样显式调用 unlock() 方法释放锁，会在同步代码块执行完后自动释放。自 Java 6 引入偏向锁和轻量级锁后，使得锁性能上面和ReentrantLock各有优劣。（简单场景低竞争下synchronized占优，高竞争场景下ReentrantLock占优）





## AQS的使用













END