Java JUC锁



## 对比synchronized锁

synchronized 是 Java 语法层面的互斥锁，由 JVM 直接实现，其底层依赖于 Monitor（监视器）机制。

从 Java 5 开始，Java 引入了 java.util.concurrent（简称 JUC）包中的一套全新锁机制，被称为 JUC 锁。

与 synchronized 依赖 Monitor 不同，绝大多数 JUC 锁都是基于 AQS（AbstractQueuedSynchronizer，抽象队列同步器）实现的。AQS 是由 Java 编写的同步框架，通过维护一个表示资源状态的 status 字段和一个线程等待队列，抽象并实现了线程的阻塞、唤醒以及同步状态管理等核心机制。

AQS 的底层基于 Unsafe 提供的 CAS 操作保证状态修改的原子性，通过 volatile 变量确保可见性，并利用 LockSupport 的 park()/unpark() 实现线程的高效阻塞与唤醒，从而构建出高性能的同步框架。



ReentrantLock 是基于 AQS 实现的可重入互斥锁，它在设计上借鉴并模拟了 synchronized 的核心机制，例如支持线程的可重入性，并默认采用非公平锁策略。相较于 synchronized，ReentrantLock 更加的灵活，如支持带超时的锁获取（tryLock）、可中断的锁等待（lockInterruptibly）、以及显式地选择公平锁或非公平锁模式。

虽然 synchronized 在功能和灵活性上不及 ReentrantLock，但它作为 Java 的语法级关键字，由 JVM 直接支持，无需像 ReentrantLock 那样显式调用 unlock() 方法释放锁，会在同步代码块执行完后自动释放。自 Java 6 引入偏向锁和轻量级锁后，使得锁性能上面和ReentrantLock各有优劣。（简单场景低竞争下synchronized占优，高竞争场景下ReentrantLock占优）





## AQS的使用

AQS（AbstractQueuedSynchronizer）适用于那些**通过单一原子整型变量（state）来表示同步状态**的同步器。该状态的具体含义由 AQS 的子类自行定义。子类通过 AQS 提供的线程安全方法 `getState()`、`setState(int)` 和 `compareAndSetState(int, int)` 来读取、修改和条件更新该状态。例如：在 `ReentrantLock` 中，`state` 表示锁的重入次数；在 `Semaphore` 中，`state` 表示当前可用的许可证数量。

AQS 的子类通常被设计为**外部同步组件的内部私有类**（如 `ReentrantLock` 中的 `Sync` 类），并通过重写一系列以 `try` 开头的方法，来实现具体的获取与释放锁的逻辑。

核心可重写方法包括：

- `tryAcquire(int)`：在独占模式下尝试获取同步状态，成功返回 `true`，失败返回 `false`。
- `tryRelease(int)`：在独占模式下尝试释放同步状态，成功返回 `true`，失败返回 `false`。

在这些方法中，子类通过调用 `getState()`、`setState()` 或 `compareAndSetState()` 来检查或修改同步状态。**这些方法必须设计为简短且非阻塞的**，因为它们会被 AQS 框架在获取/释放流程中频繁调用。

> **注意**：重写这些 `try*` 方法是使用 AQS 的核心方式，子类正是通过实现这些方法来赋予 `state` 特定语义，从而实现自定义的同步语义。



AQS 内部的同步获取与释放流程如下：

**获取（AQS::acquire方法）（独占模式）**：

```java
while (!tryAcquire(arg)) {  // 尝试获取失败
    // 将当前线程加入同步队列（若尚未入队）
    // 阻塞当前线程，直到被前驱节点唤醒
}
```

**释放（AQS::release方法）（独占模式）**：

```java
if (tryRelease(arg)) {  // 尝试释放成功
    // 唤醒同步队列中的第一个等待线程（头节点的后继）
}
```

该机制通过模板方法模式将通用的线程排队、阻塞与唤醒逻辑封装在 AQS 内部，而将具体的同步策略交由子类实现，实现了高度的灵活性与复用性。





## AQS的使用示例

这两个使用示例来源于AQS类的注释。



一个非可重入的互斥锁，status为0表示解锁状态，status为1表示锁定状态。

```java
class Mutex implements Lock, java.io.Serializable {
    // 内部使用AQS实现的私有类
    private static class Sync extends AbstractQueuedSynchronizer {
        // 是否处于锁定状态
        protected boolean isHeldExclusively() {
            return getState() == 1;
        }
        // 尝试获取锁
        public boolean tryAcquire(int acquires) {
            if (compareAndSetState(0, 1)) { // 获取锁(修改状态), 通过CAS操作将0改为1
                setExclusiveOwnerThread(Thread.currentThread()); // 设置当前线程
                return true;
            }
            return false;
        }
        // 尝试释放锁
        protected boolean tryRelease(int releases) {
            if (getState() == 0) throw new IllegalMonitorStateException();
            setExclusiveOwnerThread(null); // 设置当前线程
            setState(0); // 释放锁(修改状态为0)
            return true;
        }
        // 提供条件变量
        Condition newCondition() { return new ConditionObject(); }
    }

    private final Sync sync = new Sync();
    public void lock() { sync.acquire(1); } // 调用acquire方法加锁
    public boolean tryLock() { return sync.tryAcquire(1); }
    public void unlock() { sync.release(1); } // 调用release方法解锁
    public Condition newCondition() { return sync.newCondition(); }
    public boolean isLocked() { return sync.isHeldExclusively(); }
    public boolean hasQueuedThreads() { return sync.hasQueuedThreads(); }
    public void lockInterruptibly() throws InterruptedException {
        sync.acquireInterruptibly(1);
    }
    public boolean tryLock(long timeout, TimeUnit unit) throws InterruptedException {
    	return sync.tryAcquireNanos(1, unit.toNanos(timeout));
    }
}
```



一个类似于CountDownLatch的门闩类，但只需一个信号即可触发。由于门闩是非独占的，因此它使用共享的获取和释放方法。

```java
class BooleanLatch {
    private static class Sync extends AbstractQueuedSynchronizer {
        boolean isSignalled() { return getState() != 0; }
        
        // 共享状态下的尝试获取同步状态 返回一个正数表示获取成功
        protected int tryAcquireShared(int ignore) {
            // isSignalled方法返回true表示门闩被打开
            return isSignalled() ? 1 : -1;
        }
        
        // 共享状态下的尝试释放同步状态
        protected boolean tryReleaseShared(int ignore) {
            setState(1); // 设置状态为1表示打开门闩
            return true;
        }
    }
    
    private final Sync sync = new Sync();
    public boolean isSignalled() { return sync.isSignalled(); }
    
    // 调用该方法会打开门闩
    public void signal()         { sync.releaseShared(1); }
    
    // 调用该方法会阻塞当前线程 直到门闩被打开
    // 如果门闩已经打开 则不会阻塞
    public void await() throws InterruptedException {
        sync.acquireSharedInterruptibly(1);
    }
}
```







END